[![Nest Logo](http://kamilmysliwiec.com/public/nest-logo.png)](http://kamilmysliwiec.com/)

A lightweight **CQRS** module for [Nest](https://github.com/kamilmysliwiec/nest) framework (node.js)

  [![NPM Version][npm-image]][npm-url]
  [![NPM Downloads][downloads-image]][downloads-url]
  
## Installation

```bash
$ npm install --save @nestjs/cqrs
```
## Introduction & Usage

Why [CQRS](https://martinfowler.com/bliki/CQRS.html)? Let's have a look at the mainstream approach:

1. Controllers layer handle HTTP requests and delegate tasks to the services.
2. Services layer is the place, where the most of the business logic is being done.
3. Services uses Repositories / DAOs to change / persist entities.
4. Entities are our models - just containers for the values, with setters and getters.

Simple [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) application with layered architecture. Is it good? Yes, sure. In most cases, there is no reason to make small and medium-sized applications more complicated. So, we are done with the bigger part of logic in the services and models without any behaviour (btw. are they models still? I don't think so). When our application becomes larger it will be harder to maintain and improve it.

Let's change our way of thinking.

### Commands

To make our application easier to understand, each change has to be preceded by **Command**. If any command is dispatched - application react on it. Commands are dispatched from the services and consumed in appropriate **Command Handlers**.

**Service:**
```typescript
@Component()
export class HeroesGameService {
    constructor(private commandBus: CommandBus) {}

    async killDragon(heroId: string, killDragonDto: KillDragonDto) {
        return await this.commandBus.execute(
            new KillDragonCommand(heroId, killDragonDto.dragonId)
        );
    }
}
```

**Command:**
```typescript
export class KillDragonCommand implements ICommand {
    constructor(
        public readonly heroId: string,
        public readonly dragonId: string) {}
}
```

The **Command Bus** is a commands stream. It delegates commands to equivalent handlers.

Each Command has to have corresponding **Command Handler**:

```typescript
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
    constructor(private readonly repository: HeroRepository) {}

    execute(command: KillDragonCommand, resolve: (value?) => void) {
        const { heroId, dragonId } = command;
        const hero = this.repository.findOneById(+heroId);
        
        hero.killEnemy(dragonId);
        await this.repository.persist(hero);
        resolve();
    }
}
```

Now, every application state change is a result of **Command** occurrence. The logic is encapsulated in handlers. If we want we can simply add logging here or more - we can persist our commands in the database (e.g. for diagnostics purposes). 

Why we need `resolve()` function? Sometimes we might want to return a 'message' from handler to service. Also - we can just call this function at the beginning of the `execute()` method, so our application will firstly - turn back into the service and return response to client and then asynchronously come back here.

Our structure looks better, but it was only the **first step**. Sure, if you want, you can end up with it.

### Events

If we encapsulate commands in handlers, we prevent interaction between them - application structure is still not flexible, not **reactive**. The solution is to use **events**.

Events are asynchronous. They are dispatched by **models**. Models have to extend `AggregateRoot` class.

**Event:**
```typescript
export class HeroKilledDragonEvent implements IEvent {
    constructor(
        public readonly heroId: string,
        public readonly dragonId: string) {}
}
```

**Model:**
```typescript
export class Hero extends AggregateRoot {
    constructor(private readonly id: string) {
        super();
    }

    killEnemy(enemyId: string) {
        // logic
        this.apply(new HeroKilledDragonEvent(this.id, enemyId));
    }
}
```

The `apply()` method does not dispatch events yet, because there is no relationship between model and `EventPublisher`. So, how to tell model about the publisher? We have to use publisher `mergeObjectContext()` method inside our command handler.

```typescript
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
    constructor(
        private readonly repository: HeroRepository,
        private readonly publisher: EventPublisher) {}

    execute(command: KillDragonCommand, resolve: (value?) => void) {
        const { heroId, dragonId } = command;
        const hero = this.publisher.mergeObjectContext(
            this.repository.findOneById(+heroId)
        );
        hero.killEnemy(dragonId);
        resolve();
    }
}
```

Now, everything works as we expected. Of course, object do not have to exist already. We can easily merge type context also:

```typescript
const HeroModel = this.publisher.mergeContext(Hero);
new HeroModel('id');
```

That's it. **Model can publish events**. We have to handle them. Each event may has a lot of **Event Handlers**. They do not have to know about each other.

```typescript
@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
    constructor(private readonly repository: HeroRepository) {}

    handle(event: HeroKilledDragonEvent) {
        // logic
    }
}
```

At this time we can e.g. move our **persistance logic** into event handlers, so the command handlers will be lighter.

### Sagas

This kind of **Event-Driven Architecture** improves application **reactiveness and scalability**. Now, when we have an events, we can simply react on them. The **Sagas** - are the last building blocks from architecture point of view.

The sagas are incredibly powerful feature. Single saga is listening for 1 .. * events. It can combine, merge, filter etc. events streams. [RxJS](https://github.com/Reactive-Extensions/RxJS) library is the where magic comes from. 

Saga has to return **single command** as an Observable. This command is dispatched **asynchronously**. 

```typescript
@Component()
export class HeroesGameSagas {
    dragonKilled = (events$: EventObservable<any>): Observable<ICommand> => {
        return events$.ofType(HeroKilledDragonEvent)
            .map((event) => new DropAncientItemCommand(event.heroId, fakeItemID));
    }
}
```

We have a rule that if any hero kills dragon - the hero should obtain the ancient item. That's it. The `DropAncientItemCommand` will be dispatched and processed by appropriate handler.

### Setup

The last thing, which we have to do is to set up entire mechanism.

```typescript
export const CommandHandlers = [ KillDragonHandler, DropAncientItemHandler ];
export const EventHandlers =  [ HeroKilledDragonHandler, HeroFoundItemHandler ];

@Module({
    modules: [ CQRSModule ],
    controllers: [ HeroesGameController ],
    components: [
        HeroesGameService,
        HeroesGameSagas,
        ...CommandHandlers,
        ...EventHandlers,
        HeroRepository
    ]
})
export class HeroesGameModule implements OnModuleInit {
    constructor(
        private readonly moduleRef: ModuleRef,
        private readonly command$: CommandBus,
        private readonly event$: EventBus,
        private readonly heroesGameSagas: HeroesGameSagas) {}

    onModuleInit() {
        this.command$.setModuleRef(this.moduleRef);
        this.event$.setModuleRef(this.moduleRef);

        this.event$.register(EventHandlers);
        this.command$.register(CommandHandlers);
        this.event$.combineSagas([
            this.heroesGameSagas.dragonKilled,
        ]);
    }
}
```

### Interesting facts

- both `CommandBus` and `EventBus` are **Observables**, so it is possible to subscribe to entire streams (e.g. for logging)
- each Command and Event Handler is a Component, so it can inject another components through constructor

## Full example

- [Nest CQRS module usage example repository](https://github.com/kamilmysliwiec/nest-cqrs-example)

## People

Author - [Kamil Myśliwiec](http://kamilmysliwiec.com)

## License

[MIT](LICENSE)

[npm-image]: https://badge.fury.io/js/%40nestjs%2Fcqrs.svg
[npm-url]: https://npmjs.org/package/%40nestjs%2Fcqrs
[downloads-image]: https://img.shields.io/npm/dm/%40nestjs%2Fcqrs.svg
[downloads-url]: https://npmjs.org/package/%40nestjs%2Fcqrs
