# Test Data Factory Patterns: Advanced

Advanced factory patterns for complex object construction, builder patterns, and sophisticated composition strategies.

## Builder Pattern for Complex Objects

For objects with many optional fields or complex setup requirements.

```typescript
// order.ts
import { z } from 'zod';

const OrderItemSchema = z.object({
  productId: z.string(),
  productName: z.string(),
  quantity: z.number().int().positive(),
  unitPrice: z.number().positive(),
  subtotal: z.number().nonnegative()
});

const OrderSchema = z.object({
  id: z.string().uuid(),
  customerId: z.string(),
  items: z.array(OrderItemSchema).min(1),
  subtotal: z.number().nonnegative(),
  tax: z.number().nonnegative(),
  shipping: z.number().nonnegative(),
  total: z.number().nonnegative(),
  status: z.enum(['pending', 'paid', 'shipped', 'delivered', 'cancelled']),
  createdAt: z.date()
});

type OrderItem = z.infer<typeof OrderItemSchema>;
type Order = z.infer<typeof OrderSchema>;

export { OrderSchema, OrderItemSchema };
export type { Order, OrderItem };
```

```typescript
// order.test-factory.ts
import { type Order, type OrderItem, OrderSchema, OrderItemSchema } from './order';

class TestOrderBuilder {
  private order: Partial<Order> = {
    id: crypto.randomUUID(),
    customerId: 'cust-default',
    items: [],
    subtotal: 0,
    tax: 0,
    shipping: 0,
    total: 0,
    status: 'pending',
    createdAt: new Date()
  };

  withId(id: string): this {
    this.order.id = id;
    return this;
  }

  withCustomerId(customerId: string): this {
    this.order.customerId = customerId;
    return this;
  }

  withItem(productId: string, productName: string, quantity: number, unitPrice: number): this {
    const subtotal = quantity * unitPrice;
    const item: OrderItem = {
      productId,
      productName,
      quantity,
      unitPrice,
      subtotal
    };

    const validated = OrderItemSchema.parse(item);
    this.order.items = [...(this.order.items || []), validated];

    return this;
  }

  withStatus(status: Order['status']): this {
    this.order.status = status;
    return this;
  }

  withShipping(shipping: number): this {
    this.order.shipping = shipping;
    return this;
  }

  withTaxRate(taxRate: number): this {
    const subtotal = this.calculateSubtotal();
    this.order.subtotal = subtotal;
    this.order.tax = subtotal * taxRate;
    return this;
  }

  private calculateSubtotal(): number {
    return (this.order.items || []).reduce((sum, item) => sum + item.subtotal, 0);
  }

  private calculateTotal(): number {
    return (this.order.subtotal || 0) + (this.order.tax || 0) + (this.order.shipping || 0);
  }

  build(): Order {
    // Calculate totals before building
    this.order.subtotal = this.calculateSubtotal();
    this.order.total = this.calculateTotal();

    return OrderSchema.parse(this.order as Order);
  }
}

const createTestOrderBuilder = (): TestOrderBuilder => new TestOrderBuilder();

const createTestOrder = (builderFn?: (builder: TestOrderBuilder) => TestOrderBuilder): Order => {
  const builder = createTestOrderBuilder()
    .withItem('prod-1', 'Default Product', 1, 10.00);

  const configured = builderFn ? builderFn(builder) : builder;
  return configured.build();
};

export { createTestOrderBuilder, createTestOrder };
```

```typescript
// order.test.ts
import { describe, it, expect } from 'vitest';
import { createTestOrder, createTestOrderBuilder } from './order.test-factory';

describe('Order Factory', () => {
  it('should create order with single default item', () => {
    const order = createTestOrder();

    expect(order.items).toHaveLength(1);
    expect(order.subtotal).toBe(10.00);
  });

  it('should create order with multiple items', () => {
    const order = createTestOrder(builder =>
      builder
        .withItem('prod-1', 'Product 1', 2, 10.00)
        .withItem('prod-2', 'Product 2', 1, 25.00)
    );

    expect(order.items).toHaveLength(2);
    expect(order.subtotal).toBe(45.00);
  });

  it('should calculate tax and shipping', () => {
    const order = createTestOrder(builder =>
      builder
        .withItem('prod-1', 'Product', 2, 10.00)
        .withTaxRate(0.08)
        .withShipping(5.00)
    );

    expect(order.subtotal).toBe(20.00);
    expect(order.tax).toBe(1.60);
    expect(order.shipping).toBe(5.00);
    expect(order.total).toBe(26.60);
  });

  it('should use builder directly for complex setup', () => {
    const order = createTestOrderBuilder()
      .withCustomerId('cust-premium')
      .withItem('prod-1', 'Widget', 3, 15.00)
      .withItem('prod-2', 'Gadget', 1, 50.00)
      .withTaxRate(0.10)
      .withShipping(10.00)
      .withStatus('paid')
      .build();

    expect(order.customerId).toBe('cust-premium');
    expect(order.items).toHaveLength(2);
    expect(order.subtotal).toBe(95.00);
    expect(order.tax).toBe(9.50);
    expect(order.total).toBe(114.50);
  });
});
```

**Key Benefits:**
- Fluent API for complex object construction
- Automatic calculation of derived fields
- Type-safe chaining
- Readable test setup

**When to Use Builder Pattern:**
- Objects with many optional fields (5+)
- Complex validation dependencies between fields
- Derived/calculated fields based on other fields
- Need for readable, self-documenting test setup

---

## Advanced Composition: Multi-Level Nesting

Creating factories for deeply nested structures with multiple levels of composition.

```typescript
// organization.test-factory.ts
import { type Organization, type Team, type User } from './types';
import { createTestUser } from './user.test-factory';

type TeamOverrides = Partial<Omit<Team, 'members'> & { members?: User[] }>;
type OrgOverrides = Partial<Omit<Organization, 'teams'> & { teams?: Team[] }>;

const createTestTeam = (overrides?: TeamOverrides): Team => {
  const defaultMembers = [
    createTestUser({ role: 'admin' }),
    createTestUser({ role: 'user' }),
    createTestUser({ role: 'user' })
  ];

  return {
    id: crypto.randomUUID(),
    name: 'Engineering',
    members: defaultMembers,
    createdAt: new Date(),
    ...overrides
  };
};

const createTestOrganization = (overrides?: OrgOverrides): Organization => {
  const defaultTeams = [
    createTestTeam({ name: 'Engineering' }),
    createTestTeam({ name: 'Product' })
  ];

  return {
    id: crypto.randomUUID(),
    name: 'Acme Corp',
    teams: defaultTeams,
    createdAt: new Date(),
    ...overrides
  };
};

export { createTestTeam, createTestOrganization };
```

```typescript
// organization.test.ts
describe('Organization Factory', () => {
  it('should create organization with custom teams', () => {
    const admin = createTestUser({ role: 'admin', username: 'cto' });
    const engineers = [
      createTestUser({ username: 'dev1' }),
      createTestUser({ username: 'dev2' })
    ];

    const org = createTestOrganization({
      name: 'Tech Startup',
      teams: [
        createTestTeam({
          name: 'Engineering',
          members: [admin, ...engineers]
        })
      ]
    });

    expect(org.name).toBe('Tech Startup');
    expect(org.teams[0].members[0].username).toBe('cto');
    expect(org.teams[0].members).toHaveLength(3);
  });
});
```

---

## Factory Traits: Reusable State Variations

Creating reusable state variations that can be composed.

```typescript
// user.test-factory.ts
import { type User } from './user';
import { createTestUser } from './user.test-factory';

// Trait functions - reusable state variations
const asAdmin = (user: User): User => ({
  ...user,
  role: 'admin'
});

const asInactive = (user: User): User => ({
  ...user,
  isActive: false
});

const withVerifiedEmail = (user: User): User => ({
  ...user,
  emailVerified: true,
  emailVerifiedAt: new Date()
});

// Compose traits
const createAdminUser = (overrides?: Partial<User>): User =>
  asAdmin(createTestUser(overrides));

const createInactiveUser = (overrides?: Partial<User>): User =>
  asInactive(createTestUser(overrides));

const createVerifiedAdminUser = (overrides?: Partial<User>): User =>
  withVerifiedEmail(asAdmin(createTestUser(overrides)));

export { asAdmin, asInactive, withVerifiedEmail, createAdminUser, createInactiveUser };
```

```typescript
// user.test.ts
describe('User Traits', () => {
  it('should create verified admin user', () => {
    const user = createVerifiedAdminUser({ username: 'superadmin' });

    expect(user.role).toBe('admin');
    expect(user.emailVerified).toBe(true);
    expect(user.username).toBe('superadmin');
  });

  it('should compose traits manually', () => {
    const user = withVerifiedEmail(
      asAdmin(
        createTestUser({ username: 'custom' })
      )
    );

    expect(user.role).toBe('admin');
    expect(user.emailVerified).toBe(true);
  });
});
```

**Key Benefits:**
- Reusable state transformations
- Compose multiple traits
- Clear, expressive test setup
- Avoid duplication across tests

---

## Factory Registry Pattern

Centralized factory management for large test suites.

```typescript
// factory-registry.ts
type FactoryFn<T> = (overrides?: Partial<T>) => T;
type FactoryMap = Map<string, FactoryFn<any>>;

class FactoryRegistry {
  private factories: FactoryMap = new Map();

  register<T>(name: string, factory: FactoryFn<T>): void {
    this.factories.set(name, factory);
  }

  create<T>(name: string, overrides?: Partial<T>): T {
    const factory = this.factories.get(name);
    if (!factory) {
      throw new Error(`Factory "${name}" not registered`);
    }
    return factory(overrides);
  }

  has(name: string): boolean {
    return this.factories.has(name);
  }
}

const registry = new FactoryRegistry();

export { registry };
```

```typescript
// factories/index.ts
import { registry } from './factory-registry';
import { createTestUser } from './user.test-factory';
import { createTestPost } from './post.test-factory';
import { createTestOrder } from './order.test-factory';

// Register all factories
registry.register('user', createTestUser);
registry.register('post', createTestPost);
registry.register('order', createTestOrder);

export { registry as factories };
```

```typescript
// Usage in tests
import { factories } from './factories';

describe('Factory Registry', () => {
  it('should create entities via registry', () => {
    const user = factories.create('user', { username: 'testuser' });
    const post = factories.create('post', { title: 'Test Post' });

    expect(user.username).toBe('testuser');
    expect(post.title).toBe('Test Post');
  });
});
```

**When to Use Registry:**
- Large test suites with many factories
- Dynamic factory selection
- Centralized factory configuration
- Plugin-based testing systems

---

## Advanced Sequences: Context-Aware Generation

Sequences that generate contextual, realistic data.

```typescript
// advanced-sequence.test-factory.ts

const emailDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'company.com'];
const firstNames = ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve'];
const lastNames = ['Smith', 'Johnson', 'Williams', 'Brown', 'Jones'];

let userCounter = 0;

const generateRealisticEmail = (): string => {
  const firstName = firstNames[userCounter % firstNames.length];
  const lastName = lastNames[userCounter % lastNames.length];
  const domain = emailDomains[userCounter % emailDomains.length];
  const number = Math.floor(userCounter / firstNames.length) + 1;

  userCounter++;

  return `${firstName.toLowerCase()}.${lastName.toLowerCase()}${number > 1 ? number : ''}@${domain}`;
};

const generateUsername = (): string => {
  const firstName = firstNames[userCounter % firstNames.length];
  const number = Math.floor(userCounter / firstNames.length) + 1;

  return `${firstName.toLowerCase()}${number > 1 ? number : ''}`;
};

const resetRealisticGenerators = (): void => {
  userCounter = 0;
};

export { generateRealisticEmail, generateUsername, resetRealisticGenerators };
```

**Use cases:**
- More realistic test data for demos
- Testing uniqueness constraints with realistic patterns
- Debugging with identifiable test data

---

## Summary: Advanced Factory Patterns

**Builder Pattern:**
- Use for objects with many optional fields or computed values
- Provides fluent, readable API
- Handles complex validation and dependencies

**Multi-Level Composition:**
- Compose factories for deeply nested structures
- Override at any level
- Maintain type safety throughout

**Factory Traits:**
- Reusable state transformations
- Compose multiple traits for complex states
- Clear, expressive test setup

**Factory Registry:**
- Centralized factory management
- Dynamic factory selection
- Useful for large test suites

**Advanced Sequences:**
- Context-aware, realistic data generation
- Better for demos and debugging
- Maintain uniqueness with realistic patterns

**Decision Guide:**
- Simple objects → Basic factory with partial overrides
- Nested objects → Composed factories
- Complex objects with many fields → Builder pattern
- Reusable state variations → Traits
- Large test suites → Registry pattern
- Realistic test data → Advanced sequences

See also: @~/.claude/docs/examples/factory-basics.md for fundamental patterns
