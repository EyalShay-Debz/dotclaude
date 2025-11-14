# Code Style - Quick Reference

## Core Principles

1. **Immutability**: No data mutation
2. **Pure Functions**: Same input = same output
3. **Self-Documenting**: No comments needed
4. **Early Returns**: No deep nesting
5. **Small Functions**: Single responsibility

---

## Immutability Patterns

| Operation | ✅ DO (Immutable) | ❌ DON'T (Mutates) |
|-----------|-------------------|---------------------|
| **Arrays** |
| Add | `[...arr, item]`, `arr.concat(item)` | `arr.push(item)`, `arr.unshift(item)` |
| Remove | `arr.filter(i => i.id !== id)` | `arr.pop()`, `arr.shift()`, `arr.splice()` |
| Update | `arr.map(i => i.id === id ? {...i, x} : i)` | `arr[0] = value`, `arr.sort()`, `arr.reverse()` |
| **Objects** |
| Add/Update | `{...obj, prop: 'value'}` | `obj.prop = 'value'`, `obj['key'] = 'value'` |
| Remove | `const {remove, ...rest} = obj` | `delete obj.prop` |
| Nested | `{...obj, nested: {...obj.nested, x}}` | `obj.nested.x = 'value'` |
| Merge | `{...obj1, ...obj2}`, `Object.assign({}, a, b)` | `Object.assign(obj, updates)` |


## Functional Patterns

**Pure Functions**: Same input → same output, no side effects
✅ `calculateTotal(items)` ❌ `calculateTotal(items, externalTaxRate)`

**Side Effects**: Isolate (validate pure, then DB call) or return deferred `{data, sideEffects: {sendEmail: () => ...}}`

**Array Methods**: Use `.map()`, `.filter()`, `.reduce()`, `.find()`, `.some()`, `.every()` ❌ No `.forEach()` for transformations, no mutation inside methods


## Naming Conventions

| Type | ✅ DO | ❌ DON'T |
|------|-------|----------|
| **Functions** | Verbs: `calculateTotal`, `validateUser` | `doStuff`, `process`, `manager` |
| **Booleans** | `is/has/can/should`: `isValid`, `hasPermission` | `valid`, `permission` |
| **Event Handlers** | `handle/on`: `handleClick`, `onUserLogin` | `click`, `userLogin` |
| **Types** | PascalCase nouns: `User`, `UserProfile` | `Data`, `Info`, `Item` (too generic) |
| **Constants** | UPPER_SNAKE: `MAX_RETRY_ATTEMPTS` | `maxRetryAttempts` (use for derived) |
| **Variables** | camelCase: `userProfile`, `validationErrors` | `u`, `usr`, `cfg` (abbreviations) |
| **Files** | kebab-case: `user-service.ts`, `auth-middleware.ts` | `UserService.ts` (PascalCase) |


## Structure Preferences

### Early Returns (Guard Clauses)

**✅ DO:**
```typescript
function processUser(user: User | null): ProcessedUser {
  if (!user) {
    throw new Error('User is required');
  }

  if (!user.active) {
    throw new Error('User is not active');
  }

  if (!user.email) {
    throw new Error('User email is required');
  }

  return {
    id: user.id,
    email: user.email,
    displayName: formatDisplayName(user)
  };
}
```

**❌ DON'T:**
```typescript
function processUser(user: User | null): ProcessedUser {
  if (user) {
    if (user.active) {
      if (user.email) {
        return {
          id: user.id,
          email: user.email,
          displayName: formatDisplayName(user)
        };
      } else {
        throw new Error('User email is required');
      }
    } else {
      throw new Error('User is not active');
    }
  } else {
    throw new Error('User is required');
  }
}
```

### No Deep Nesting (Max 2 Levels)

**✅ DO:**
```typescript
function processOrders(orders: Order[]): ProcessedOrder[] {
  return orders
    .filter(isValidOrder)
    .map(calculateTotals)
    .map(applyDiscounts);
}

function isValidOrder(order: Order): boolean {
  if (!order.items.length) return false;
  if (!order.customerId) return false;
  return true;
}
```

**❌ DON'T:**
```typescript
function processOrders(orders: Order[]): ProcessedOrder[] {
  const results: ProcessedOrder[] = [];

  for (const order of orders) {
    if (order.items.length > 0) {
      if (order.customerId) {
        let total = 0;
        for (const item of order.items) {
          if (item.price > 0) {
            total += item.price;
          }
        }
        results.push({ ...order, total });
      }
    }
  }

  return results;
}
```

### Extract Helper Functions

**✅ DO:**
```typescript
function createUser(request: CreateUserRequest): User {
  validateRequest(request);
  const hashedPassword = hashPassword(request.password);
  const user = buildUserObject(request, hashedPassword);
  return user;
}

function validateRequest(request: CreateUserRequest): void {
  if (!request.email) throw new Error('Email required');
  if (!request.password) throw new Error('Password required');
}

function hashPassword(password: string): string {
  return bcrypt.hashSync(password, 10);
}

function buildUserObject(request: CreateUserRequest, hashedPassword: string): User {
  return {
    id: generateId(),
    email: request.email,
    password: hashedPassword,
    createdAt: new Date()
  };
}
```

---

## Self-Documenting Code

### Function Names

**✅ DO:**
```typescript
// Name explains purpose
function calculateTotalWithTax(items: Item[], taxRate: number): number {
  const subtotal = items.reduce((sum, item) => sum + item.price, 0);
  return subtotal * (1 + taxRate);
}

function isEligibleForDiscount(user: User): boolean {
  return user.isPremium && user.ordersCount > 10;
}
```

**❌ DON'T:**
```typescript
// Name doesn't explain purpose
function calc(items: Item[], rate: number): number { // What is rate?
  const s = items.reduce((sum, item) => sum + item.price, 0);
  return s * (1 + rate);
}

// Comment needed = bad name
function check(user: User): boolean { // Check what?
  // Check if user is eligible for discount
  return user.isPremium && user.ordersCount > 10;
}
```

### Variable Names

**✅ DO:**
```typescript
const activeUsers = users.filter(u => u.active);
const totalPrice = items.reduce((sum, item) => sum + item.price, 0);
const isAdminUser = user.role === 'admin';
```

**❌ DON'T:**
```typescript
const a = users.filter(u => u.active);  // What is 'a'?
const t = items.reduce((sum, item) => sum + item.price, 0); // What is 't'?
const f = user.role === 'admin'; // What is 'f'?
```

### Type Definitions

**✅ DO:**
```typescript
type CreateUserRequest = {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
};

type UserPermissions = {
  canRead: boolean;
  canWrite: boolean;
  canDelete: boolean;
};
```

**❌ DON'T:**
```typescript
type Request = { // Request for what?
  e: string;  // What is 'e'?
  p: string;  // What is 'p'?
  fn: string; // What is 'fn'?
  ln: string; // What is 'ln'?
};
```

---

## Function Parameters

### Options Object (>3 Parameters)

**✅ DO:**
```typescript
type CreateUserOptions = {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  role?: 'user' | 'admin';
  sendWelcomeEmail?: boolean;
};

function createUser(options: CreateUserOptions): User {
  const {
    email,
    password,
    firstName,
    lastName,
    role = 'user',
    sendWelcomeEmail = true
  } = options;

  // Implementation
}

// Usage: clear and self-documenting
createUser({
  email: 'user@example.com',
  password: 'secret',
  firstName: 'John',
  lastName: 'Doe',
  sendWelcomeEmail: false
});
```

**❌ DON'T:**
```typescript
function createUser(
  email: string,
  password: string,
  firstName: string,
  lastName: string,
  role?: string,
  sendWelcomeEmail?: boolean
): User {
  // Implementation
}

// Usage: unclear parameter order, easy to mix up
createUser('user@example.com', 'secret', 'John', 'Doe', undefined, false);
```

### Boolean Parameters

**✅ DO:**
```typescript
type FetchUsersOptions = {
  includeInactive?: boolean;
  includeDeleted?: boolean;
};

function fetchUsers(options: FetchUsersOptions = {}): User[] {
  // Implementation
}

// Usage: self-documenting
fetchUsers({ includeInactive: true });
```

**❌ DON'T:**
```typescript
function fetchUsers(includeInactive: boolean, includeDeleted: boolean): User[] {
  // Implementation
}

// Usage: unclear what true/false means
fetchUsers(true, false); // Which is which?
```

---

## TypeScript Preferences

### Use `type` Over `interface`

**✅ DO:**
```typescript
type User = {
  id: string;
  email: string;
  name: string;
};

type Admin = User & {
  permissions: string[];
};
```

**❌ DON'T (unless extending library interfaces):**
```typescript
interface User {
  id: string;
  email: string;
  name: string;
}

interface Admin extends User {
  permissions: string[];
}
```

### Avoid Type Assertions

**✅ DO:**
```typescript
// Use type guards
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value
  );
}

if (isUser(data)) {
  console.log(data.email); // TypeScript knows it's User
}
```

**❌ DON'T:**
```typescript
// Type assertion bypasses type checking
const user = data as User; // ❌ What if data isn't User?
```

### No `any` Types

**✅ DO:**
```typescript
// Use unknown for truly unknown types
function processData(data: unknown): ProcessedData {
  if (!isValidData(data)) {
    throw new Error('Invalid data');
  }
  return transformData(data);
}

// Use generics for reusable functions
function identity<T>(value: T): T {
  return value;
}
```

**❌ DON'T:**
```typescript
function processData(data: any): any { // ❌
  return data.something; // No type safety
}
```

---

## Quick Checklist

**Before committing, verify:**
- [ ] No data mutation (arrays, objects)
- [ ] Functions are pure (or side effects isolated)
- [ ] No nested conditionals (use early returns)
- [ ] No comments (code is self-documenting)
- [ ] Function/variable names describe purpose
- [ ] Functions <50 lines
- [ ] Max 2 levels of nesting
- [ ] No `any` types
- [ ] Prefer `type` over `interface`
- [ ] Options object for >3 parameters
- [ ] No boolean parameters (use options object)
