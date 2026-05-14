# Hexagonal Architecture
* A way of structuring software so that your business logic is isolated from external concerns.
* Also called *ports and adapters*. External dependencies talk to the application through ports. Adapters connect external systems to those ports.

---

## Ports
* Ports are interfaces defining what the application needs
```typescript
// E.g. outbound port
interface FlightRepository {
    get(id: string): Promise<Flight>;
    save(flight: Flight): Promise<void>;
}
```

---

## Adapters
* Implement the ports
```typescript
class PrismaFlightRepository implements FlightRepository {
    async get(id: string): Promise<Flight> {
        const  row = await prisma.flight.findUnique(...);

        return mapToDomain(row);
    }

    async save(flight: Flight): Promise<void> {
        //
    }
}
```
* Adapters can be driving or driven:
    * Driving adapters are adapters that use the application (REST controllers, CLI commands, message consumers, etc.).
    ```typescript
    app.post("/cancel-flight", async (req, res) => {
        await cancelFlight.execute(req.body);
    });
    ```
    * Driven adapters are systems the application calls out to (database repositories, email clients, stripe, S3, kafka, etc.).

---

## Core Idea
```typescript
      [ Driving Side ]
Controllers / UI / CLI / Events
             |
         In Adapters
             |
          In Ports
             |
      -----------------
      | Application   |
      | + Domain      |
      -----------------
             |
         Out Ports
             |
         Out Adapters
             |
DB / APIs / Queues / Email
      [ Driven Side ]
```

---

## Dependency Direction
* Dependencies point inwards
    * Infrastructure --> Application --> Domain
    * Presentation --> Application --> Domain
* The domain depends on nothing.
* The application depends only on abstractions.
* Infrastructure depends on everything because it plugs into the system.

---

## Layers
### Domain
* Pure business rules
* Should not know about HTTP, databases, frameworks, and other infrastructure

### Application
* Coordinates use cases (i.e. *applies*)
* Contains ports/interfaces


### Infrastructure
* Implements ports which the application defines
* The application depends on abstractions; infrastructure depends on the application (**NOT** the other way round)

### Presentation
* Translates user actions (HTTP, CLI, events) into use-case calls and formats core outputs for the user.

---

## Example Structure
```typescript
src/
  domain/
    flight/
      Flight.ts
      FlightPolicy.ts

  application/
    cancel-flight/
      CancelFlight.ts
      FlightRepository.interface.ts
      EmailService.interface.ts

  infrastructure/
    persistence/
      PrismaFlightRepository.ts

    email/
      SESMailer.ts

  presentation/
    http/
      CancelFlightController.ts
    cli/
    gui/
```
