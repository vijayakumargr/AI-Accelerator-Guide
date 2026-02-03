# TypeScript Instructions

## General Principles
- **ALWAYS enable strict mode** in tsconfig.json
- **ALWAYS use explicit types** for function parameters and returns
- **ALWAYS prefer `const`** over `let`, avoid `var`
- **ALWAYS handle Promise rejections** explicitly
- **NEVER use `any`** - use `unknown` if type is truly unknown
- **NEVER use non-null assertions (`!`)** without justification

---

# Code Style

## Naming Conventions
```typescript
// Variables and functions: camelCase
const userName = "John";
function calculateTotalPrice(items: Item[]): number {
  // ...
}

// Classes and interfaces: PascalCase
class UserAccount {
  // ...
}

interface UserData {
  id: number;
  name: string;
}

// Types: PascalCase
type UserId = string | number;

// Constants: UPPER_SNAKE_CASE or camelCase
const MAX_RETRY_COUNT = 3;
const defaultConfig = { timeout: 5000 };

// Enums: PascalCase with PascalCase members
enum UserStatus {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
  Pending = "PENDING",
}

// Private fields: # prefix (ES2022) or _ convention
class Service {
  #privateField: string;
  private _conventionalPrivate: number;
}
```

## Type Definitions
```typescript
// GOOD: Explicit, descriptive types
interface User {
  readonly id: string;
  name: string;
  email: string;
  createdAt: Date;
  metadata?: Record<string, unknown>;
}

// GOOD: Function types
type AsyncHandler<T> = (data: T) => Promise<void>;
type EventCallback = (event: Event) => void;

// GOOD: Discriminated unions for state
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

// BAD: Using any
function process(data: any): any {
  // Avoid this
}

// GOOD: Using unknown with type guards
function process(data: unknown): string {
  if (typeof data === "string") {
    return data.toUpperCase();
  }
  throw new Error("Expected string");
}
```

## Type Guards
```typescript
// Custom type guard
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    "id" in obj &&
    "name" in obj &&
    "email" in obj
  );
}

// Usage
function handleData(data: unknown) {
  if (isUser(data)) {
    // TypeScript knows data is User here
    console.log(data.name);
  }
}

// Assertion function
function assertIsUser(obj: unknown): asserts obj is User {
  if (!isUser(obj)) {
    throw new Error("Not a valid User object");
  }
}
```

---

# Error Handling

## Async/Await Patterns
```typescript
// GOOD: Proper error handling with async/await
async function fetchUser(id: string): Promise<User> {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      throw new ApiError(
        `Failed to fetch user: ${response.status}`,
        response.status
      );
    }

    return await response.json();
  } catch (error) {
    if (error instanceof ApiError) {
      throw error;
    }
    throw new ApiError("Network error", 0, { cause: error });
  }
}

// GOOD: Using Result type pattern
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safeFetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await fetchUser(id);
    return { ok: true, value: user };
  } catch (error) {
    return { ok: false, error: error as Error };
  }
}
```

## Custom Errors
```typescript
// Custom error classes
class DomainError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = this.constructor.name;
  }
}

class ValidationError extends DomainError {
  constructor(
    public readonly field: string,
    message: string
  ) {
    super(`${field}: ${message}`);
  }
}

class NotFoundError extends DomainError {
  constructor(
    public readonly resource: string,
    public readonly id: string
  ) {
    super(`${resource} not found: ${id}`);
  }
}
```

---

# Project Structure

## Recommended Layout
```
project/
├── src/
│   ├── index.ts
│   ├── types/
│   │   ├── index.ts
│   │   └── user.ts
│   ├── services/
│   │   ├── index.ts
│   │   └── user.service.ts
│   ├── utils/
│   │   ├── index.ts
│   │   └── validation.ts
│   └── errors/
│       └── index.ts
├── tests/
│   ├── setup.ts
│   └── services/
│       └── user.service.test.ts
├── tsconfig.json
├── package.json
└── README.md
```

## tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

# Testing

## Test Structure
```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { UserService } from "./user.service";
import type { UserRepository } from "./user.repository";

describe("UserService", () => {
  let userService: UserService;
  let mockRepository: UserRepository;

  beforeEach(() => {
    mockRepository = {
      findById: vi.fn(),
      findByEmail: vi.fn(),
      save: vi.fn(),
    };
    userService = new UserService(mockRepository);
  });

  describe("createUser", () => {
    it("should create user with valid data", async () => {
      // Arrange
      const userData = { name: "John", email: "john@example.com" };
      vi.mocked(mockRepository.findByEmail).mockResolvedValue(null);
      vi.mocked(mockRepository.save).mockResolvedValue({
        id: "1",
        ...userData,
        createdAt: new Date(),
      });

      // Act
      const user = await userService.createUser(userData);

      // Assert
      expect(user.id).toBeDefined();
      expect(user.name).toBe("John");
      expect(mockRepository.save).toHaveBeenCalledOnce();
    });

    it("should throw when email already exists", async () => {
      // Arrange
      const userData = { name: "John", email: "existing@example.com" };
      vi.mocked(mockRepository.findByEmail).mockResolvedValue({
        id: "existing",
        ...userData,
        createdAt: new Date(),
      });

      // Act & Assert
      await expect(userService.createUser(userData)).rejects.toThrow(
        ValidationError
      );
    });
  });
});
```

---

# React/Frontend Patterns

## Component Types
```typescript
// GOOD: Typed functional component
interface ButtonProps {
  variant?: "primary" | "secondary";
  size?: "sm" | "md" | "lg";
  disabled?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

function Button({
  variant = "primary",
  size = "md",
  disabled = false,
  onClick,
  children,
}: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} btn-${size}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}

// GOOD: Generic component
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}
```

## Hooks Patterns
```typescript
// Custom hook with proper types
function useAsync<T>(
  asyncFn: () => Promise<T>,
  deps: React.DependencyList = []
): {
  data: T | null;
  error: Error | null;
  loading: boolean;
  refetch: () => void;
} {
  const [state, setState] = useState<{
    data: T | null;
    error: Error | null;
    loading: boolean;
  }>({
    data: null,
    error: null,
    loading: true,
  });

  const execute = useCallback(async () => {
    setState((prev) => ({ ...prev, loading: true }));
    try {
      const data = await asyncFn();
      setState({ data, error: null, loading: false });
    } catch (error) {
      setState({ data: null, error: error as Error, loading: false });
    }
  }, deps);

  useEffect(() => {
    execute();
  }, [execute]);

  return { ...state, refetch: execute };
}
```

---

# Security

## Input Validation
```typescript
import { z } from "zod";

// Define schema with Zod
const UserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
});

type UserInput = z.infer<typeof UserSchema>;

// Validate input
function validateUser(input: unknown): UserInput {
  return UserSchema.parse(input);
}

// Safe parsing
function safeValidateUser(input: unknown): Result<UserInput> {
  const result = UserSchema.safeParse(input);
  if (result.success) {
    return { ok: true, value: result.data };
  }
  return { ok: false, error: new ValidationError("Invalid user data") };
}
```

## Sanitization
```typescript
// Sanitize HTML output
import DOMPurify from "dompurify";

function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty);
}

// Escape for different contexts
function escapeForAttribute(value: string): string {
  return value
    .replace(/&/g, "&amp;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#x27;");
}
```

---

# Anti-Patterns to Avoid

```typescript
// BAD: Using any
function process(data: any) {} // Avoid

// GOOD: Use unknown or specific type
function process(data: unknown) {}

// BAD: Non-null assertion without checking
const name = user!.name; // Dangerous

// GOOD: Proper null check
const name = user?.name ?? "Unknown";

// BAD: Type assertion without validation
const user = data as User; // Unsafe

// GOOD: Validate before assertion
if (isUser(data)) {
  const user = data; // Safe, type is narrowed
}

// BAD: Ignoring Promise
fetchData(); // Fire and forget

// GOOD: Handle the Promise
fetchData().catch(console.error);
// or
void fetchData(); // Explicit ignore (rare cases)

// BAD: == instead of ===
if (value == null) {} // Coercion

// GOOD: Strict equality (except for null/undefined check)
if (value === null || value === undefined) {}
// or for null/undefined specifically:
if (value == null) {} // This is actually idiomatic for null|undefined
```
