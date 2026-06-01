# Architecture - Generic tips

### Naming
* Classes and objects should have noun or noun phrase name like `Customer`, `WikiPage`, `Account`, and `AddressParser`.
* Method names should have verb or verb phrase names like `postPayment`, `deletePage` or `save`.
* Pick one word for one abstract concept, e.g. `fetch`, `retrieve` and `get` are equivelent, pick one!

---

### Functions
* One level of abstraction per function - i.e. don't mix high level business intent with low level implementation detail
``` typescript
// Bad Example
function registerUser(user: User) {
    if (emailExists(user.email)) { // high level business operation
        throw new Error('Email exists');
    }
    const hashedPassword = bcrypt.encrypt(user.password); //low level implementation detail

    mail(user.email, 'Welcome', 'Thanks for registering');
}

// Good Example
function registerUser(user: User) {
    ensureEmailIsUnique(user.email);
    createUser(user);
    sendWelcomeEmail(user);
}
```
* Avoid flag arguments as they often mean a function is doing more than one thing (which it shouldn't!).
* Functions should either command (change state), or query (return state), but not both.
```typescript
const isActive = user.isActive(); // answer a question
user.deactivate(); // perform an action
```

---

### Classes
* A class can contain multiple abstraction levels internally, but the public interface should represent a coherent level of abstraction.
* Making methods private isn't the same as hiding information; e.g. having private properties with public getters/setters still exposes information and is the same as if the properties were public.

---

### Error Handling
* Distinguish expected vs exceptional erorrs. Not every failure should be an exception. Reserve exceptions for: unexpected, exceptional, infrastructure or invariant-breaking situations
```typescript
// Bad Example
function createUser(input: CreateUserInput): User {
  if (!input.email.includes("@")) { // expected/common
    throw new Error("Invalid email");
  }
  if (input.password.length < 8) { // expected/common
    throw new Error("Weak password");
  }
  return new User(input);
}

// Good Example
type ValidationResult =
  | { success: true }
  | { success: false; errors: string[] };

function validateUser(
  input: CreateUserInput
): ValidationResult {
  const errors: string[] = [];

  if (!input.email.includes("@")) {
    errors.push("Invalid email");
  }

  if (input.password.length < 8) {
    errors.push("Weak password");
  }

  if (errors.length) {
    return {
      success: false,
      errors
    };
  }

  return { success: true };
}
```
* Use boundaries. Different architectural layers should have different responsibilities for detecting, translating, propogating and finally handling failures.
```typescript
// Bad Example
// Infrastructure layer
await db.query(...); // Throws "duplicate key violates unique constraint users_email_key"

// Use Case
try {
  await repository.save(user);
} catch (err) {
  throw err;
}

// Controller
catch (err) {
  res.status(500).json({
    error: err.message
  });
}
// User sees "duplicate key violates unique constraint users_email_key"

// Good Example
// Infrastructure layer
async save(user: User): Promise<void> {
  try {
    await db.query(...);
  } catch (err) {
    if (isPgUniqueViolation(err)) {
      throw new EmailAlreadyExistsError(user.email);
    }

    throw new RepositoryUnavailableError(
      "Failed to save user",
      { cause: err }
    );
  }
}

// Use Case (no catch, or we could catch specific error)
async execute(command: RegisterUserCommand) {
  await this.userRepository.save(...);
}

// Controller - maps to transport concerns
const result = await useCase.execute(...);

if (!result.success) {
  return res.status(409).json({
    error: "Email already exists"
  });
}
```
* Reduce the number of places where exceptions must be handled
* Classes with lots of exceptions habe complex interfaces
* Generating exceptions just punts the problem to the calling code. Only throw if a higher layer has a clearer responsibility for handling it
* Exceptions are best for "the abstraction cannot fulfil its contract" e.g. a parser cannot parse, a repository cannot persist, ...
* Infrastructure should translate low-level errors into meaningul, stable, application level errors - e.g. raw axios 404 could result in "UserNotFoundError"
* Techniques for recovery - retries (e.g. for network glitches, timeouts, 5xx errors, rate limits), return partial data or last cached data, undo operations, push to sqs (worker + DLQ) for retry later, use default values, skip error and continue

---

### Modules & Sub-Systems
* Hide complexity behind simple interfaces
* Stable interfaces, flexible implementations
```typescript
storage.save(file); // uses local FS, could be changed to s3 in future without interface changing
```
* Push complexity downward. Higher level code should read like business intent. Lower level modules absorb operational/technical complexity. The top of the system should answer "What is happening?", the bottom answers "How does it work?"
```typescript
// Bad Example
async function checkout(order: Order) {
  const inventoryResponse = await fetch(
    "https://inventory/api/reserve",
    {
      method: "POST",
      headers: {
        Authorization: process.env.INVENTORY_TOKEN!
      },
      body: JSON.stringify(order.items)
    }
  );

  if (!inventoryResponse.ok) {
    if (inventoryResponse.status === 409) {
      throw new Error("Inventory conflict");
    }

    throw new Error("Inventory failed");
  }

  const paymentResponse = await stripe.paymentIntents.create({
    amount: order.total
  });

  if (paymentResponse.status !== "succeeded") {
    throw new Error("Payment failed");
  }
  //
}
// Problems: Orchestration mixed with implementation, transport concerns leaked, retry logic leaked, auth leaked, high cognitive load

// Good Example
async function checkout(order: Order) {
  await inventory.reserve(order.items);
  const payment = await paymentGateway.charge(order);
  await shipmentService.create(order, payment);
}
// Readable at business level
```
* Design for replaceability at boundaries - specifically at infrastructure boundaries, external dependencies and volatile integrations
```typescript
// Bad Example
class UserService {
  async uploadAvatar(file: Buffer) {
    return s3Client.putObject({  // infrastructure tightly coupled
      Bucket: "avatars",
      Body: file
    });
  }
}

// Good Example
interface FileStorage {
  upload(path: string, file: Buffer): Promise<void>;
}

class UploadAvatar {
  constructor(
    private readonly storage: FileStorage
  ) {}

  async execute(file: Buffer) {
    await this.storage.upload("avatar.png", file);
  }
}
// Now S3 is replacable, test doubles easy, coupling reduced
```
* Don't abstract too early. Prefer concrete implementations first, and abstract after stable patterns emerge (i.e. don't go interface mad)
* Separate orchestration from work. Orchestrators coordinate workflows, sequence operations, manage high-level flow; workers encapsulate specialised logic.
* Low in coupling and high in cohesion
* Dependency Inversion Principle (DIP) - High level modules should depend on abstractions (interfaces), not low level concrete implementations
* Avoid leaky abstractions - where abstraction fails to hide complex, underlying details, forcing users to understand the implementation.
* Code is either an internal implementation detail or part of the system's observable behaviour, but not both. For a piece of code to be part of the system's observable behaviour, it must expose an operation or state that helps the client achieve one of its goals. Exposing implementation details usually goes hand in hand with invariant violations.
```typescript
class User {
  private name: string;
  public setName(name): string { this.name = name; } // allows client to bypass the invariant
  public normaliseName(name: string) { // this is an implementation detail
    return name.trim().substring(0, 50); // invariant
  }
}

// usage
class UserService {
  public renameUser(id: number, newName: string) {
    const user = await this.repo.find(id);
    const username = user.normaliseName(newName);
    user.setName(username);
    await this.repo.save(user);
  }
}

// Better Example
class User {
  private name: string;
  public setName(name) {
    this.name = normaliseName(name);
  }
  private normaliseName(name: string) { // now private!
    return name.trim().substring(0, 50);
  }
}
```

---

### Deep vs Shallow Modules
* Information leakage occurs when the same knowledge is used in multiple places; such as two different classes that both understand the format of a particular type of file.
* Backdoor leakage - information can be leaked even if it doesn't appear in a modules interface; such as when a module claims to encapsulate complexity, but users still need insider knowledge of its implementation details to use it correctly.
```typescript
// Bad Example
const parser = new FlightPlanParser();
const flightPlan = parser.import(xmlFile);
const attribute = flightPlan.Attribute.value._; // Leaked! Has knowledge of the xml structure (this should be contained in parser)

// Good Example
interface FlightPlanParser {
  import(xml: string): Promise<FlightPlan>; // returns flight plan in own structure
}
```
* Ask yourself "How can I reorganise these classes so that this particular piece of knowledge only affects a single class?"
```typescript
/*
parser/
  index.ts              << only exposes parser and is how other parts of the system should import
  FlightPlanParser.ts   << Public. Exported in index.ts for consumption
  XmlTokeniser.ts       << Private. only used within this 'module' i.e. maybe imported in FlightPlanParser
  XmlNodeMapper.ts      << Private
*/
```
* Keep code that is related together (not necessarily the same file) - i.e. they share information, they are used together (bi-directionally), they overlap conceptually (in that a simple higher level category that includes both pieces of code), it is hard to understand one of the pieces of code without looking at the other.

---

### Abstraction
* Avoid exposing internal data structures - e.g. don't return the HTTP response from a method which is getting data from an API.
* Temporal decomposition refers to organising software around execution order and processing stages rather than arpimd stable responsibilities and encapsulated knowledge. E.g. think of a compiler where you need to call scan, parse, transform in that order in a single class. This leaks implementation details to the caller.
* Information hiding only makes sense when the information being hidden is not needed outside a module. If the information is needed, then you must not hide it.

---

### Testing
* Prefer business language over technical language - e.g. instead of `isDeliveryValid returns false with invalid date`, use `rejects deliveries scheduled in the past`
* The principle above applies to UI tests too, but describe the user-visible behaviour - avoid implementation details - e.g. instead of `confirmation popup is displayed when delete button clicked`, use `asks for confirmation before deleting an item`
* More than one line in the act section is a sign of a problem with the SUT's API.
* Use the humble object pattern to test difficult-to-test program elements - e.g. move business logic outside of UI, outside of infrastructure, etc.
* Managed dependencies are out-of-process dependencies that are only accessible through your application (e.g. DB, microservice, queues); unmanaged dependencies are dependencies that other applications have access to (Stripe, weather API's). Use real instances of managed dependencies in **integration** tests, and mocks for unmanaged dependencies.

---

#### Resources
* Clean Code
* Code Complete
* Unit Testing
