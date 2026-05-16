# Domain Driven Design

## Ubiquitous Language
*The code should use the same terminology as the business/domain experts.

---

## Layered Architecture
#### UI / Presentation layer
* Responsible for showing information to the user and interpreting the user's commands.

#### Application Layer
* Defines the jobs software is supposed to do & directs the expressive domain objects to work out problems.
* This layer is kept thin. It does not contain business rules, but only coordinates tasks and deletegates work to collaborations of domain objects in the next layer down.

#### Domain Layer
* Responsible for representing concepts of the business, info about the business situation, and business rules.
* The technical details of storing it are delegated to the infrastructure.

#### Infrastructure Layer
* Provides generic technical capabilities that support higher layers: message sending, persistance for the domain, drawing widgets for the UI, etc.

---

## Entity
* An object defined primarily by its identity
* The ID of an entity could be anything but it has to be unique to the system (do 2 similar objects - like a person - need to be distinguishable? Yes? Then they are entities)
* Entities can be mutable

---

## Value Objects
* Many objects have no conceptual identity. These objects describe some characteristics of a thing.
* Value objects can reference entities - e.g. a flight could go from EGLL to EGCC - the route could be a value object even though the airports would be entities.
* In order for an object to be shared safely, it must be immutable.

---

## Services
* A service tends to be named for an activity, rather than an entity - a verb rather than a noun.
* A good service has 3 characteristics:
    1. The operation relates to a domain concept that is not a natural part of an entity or value object
    2. The interface is defined in terms of other elements of the domain model
    3. The operation is stateless. I.e. the client can use any instance of a particular service without regard to the instances individual history.
    ```typescript
    // stateful
    class CartService {
        private cart: CartEntity; // internal state

        public createCart();
        public addArticleToCart(articleId);
        public removeArticleFromCart(articleId);
    }

    // stateless
    class CartService {
        public createCart(): CartId;
        public addArticleToCart(cartId, articleId);
        public removeArticleFromCart(cartId, articleId);
    }
    ```
* Services don't just belong in the domain layer:
```typescript
/*
E.g. Funds transfer operations for a bank

Application Layer (answers "what steps happen?"):
    - Digest input (such as XML)
    - Sends message to domain service for fulfillment
    - Listens for confirmation
    - Decodes tp semd notification using infrastructure service

Domain Layer (answers "what business rules/calculations/policy applies?"):
    - Interacts with necessary account and ledger objects, making appropriate debits and credits
    - Supplies confirmation of result (transferred allowed or not, etc)

Infrastructure Layer (answers "How do we technically interact with external systems?"):
    - Sends email, letters and other comms as directed by the application.
*/
```

---

## Aggregates
* A cluster of associated objects that we treat as a unit for the purpose of data changes.
* Each aggregate has a root and a boundary:
    * A root is a single, specific entity contained in the aggregate. The root is the only member of the aggregate that outside objects are allowed to hold references to.
    * A boundary defines what is inside the aggregate.
* An aggregate is an abstrasct concept, it is not the class. The aggregate is the rule.
* An aggregate is a consistency boundary, implemented by a root enttiy that controls all changes inside that boundary.
* You can only change some group of data through the aggregate root.
* A solo entity by itself is classed as an aggregate.

---

## Domain Events
* Represent meaningful things that happened. E.g. "FlightCancelled", "UserRegistered".
* Immutable facts.
* Past tense (something happened)
```typescript
// Bad
await emailService.sendCancellationEmail();
await analytics.track();
await refundService.process();
await loyalty.removePoints();

// Better
class FlightCancelled { // Domain Event
  constructor(
    readonly flightId: string
  ) {}
}

class Flight { // Domain model
    public domainEvents = [];

    cancel(reason: string) {
        // do something then...
        this.domainEvents.push(new FlightCancelled(...))
    }
}

class CancelFlightUseCase { // Application
    async execute() {
        const flight = await this.repo.getById(123);
        flight.cancel('some reason');
        await this.repo.save(flight);
        for (const event of flight.domainEvents) {
            this.eventBus.publish(event);
        }
    }
}

class RefundOnFlightCancelled {} // Handler responding to FlightCancelled event
class NotifyCustomerOnFlightCancelled {} // Handler responding to FlightCancelled event
```

---

## Shared Kernel
* A special type of bounded context that contains code and data shared across multiple bounded contexts within the same domain.
```typescript
/*
src
  moduleA/
  moduleB/
  shared/
    kernel/
      BaseEntity.ts
      DomainEvent.ts
*/
```