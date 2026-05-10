# Architecture - Generic tips

### Naming
* Classes and objects should have noun or noun phrase name like `Customer`, `WikiPage`, `Account`, and `AddressParser`.
* Method names should have verb or verb phrase names like `postPayment`, `deletePage` or `save`.
* Pick one word for one abstract concept, e.g. `fetch`, `retrieve` and `get` are equivelent, pick one!

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

### Classes
* A class can contain multiple abstraction levels internally, but the public interface should represent a coherent level of abstraction

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


#### Resources
* Clean Code
* Code Complete
