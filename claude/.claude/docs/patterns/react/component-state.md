# React Component State Patterns

State management patterns for React components using TypeScript: useState, useReducer, custom hooks, and type-safe form handling.

## Type-Safe Form Component

Complete example demonstrating composition of TypeScript patterns for form handling.

```typescript
interface FormField {
  name: string;
  label: string;
  type: 'text' | 'email' | 'number';
  required?: boolean;
}

interface FormProps<T extends Record<string, any>> {
  fields: FormField[];
  initialValues: T;
  onSubmit: (values: T) => void | Promise<void>;
  onCancel?: () => void;
  submitLabel?: string;
  children?: React.ReactNode;
}

function Form<T extends Record<string, any>>({
  fields,
  initialValues,
  onSubmit,
  onCancel,
  submitLabel = 'Submit',
  children
}: FormProps<T>) {
  const [values, setValues] = useState<T>(initialValues);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = (name: keyof T) => (value: string) => {
    setValues(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setIsSubmitting(true);
    try {
      await onSubmit(values);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {fields.map(field => (
        <div key={field.name}>
          <label>{field.label}</label>
          <input
            type={field.type}
            name={field.name}
            value={String(values[field.name] ?? '')}
            onChange={(e) => handleChange(field.name)(e.target.value)}
            required={field.required}
          />
        </div>
      ))}
      {children}
      <div className="flex gap-2">
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Submitting...' : submitLabel}
        </button>
        {onCancel && (
          <button type="button" onClick={onCancel}>
            Cancel
          </button>
        )}
      </div>
    </form>
  );
}

// Usage
interface UserFormData {
  name: string;
  email: string;
  age: number;
}

<Form<UserFormData>
  fields={[
    { name: 'name', label: 'Name', type: 'text', required: true },
    { name: 'email', label: 'Email', type: 'email', required: true },
    { name: 'age', label: 'Age', type: 'number' }
  ]}
  initialValues={{ name: '', email: '', age: 0 }}
  onSubmit={async (data) => {
    await saveUser(data);
  }}
  onCancel={() => router.back()}
  submitLabel="Create User"
>
  <p className="text-sm text-gray-500">
    All fields marked with * are required
  </p>
</Form>
```

## useState with TypeScript

### Basic Type Inference

```typescript
// Type inferred from initial value
const [count, setCount] = useState(0); // number
const [name, setName] = useState(''); // string
const [isOpen, setOpen] = useState(false); // boolean
```

### Explicit Type Annotation

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// Explicit type when initial value is null
const [user, setUser] = useState<User | null>(null);

// Array state
const [items, setItems] = useState<string[]>([]);

// Complex object state
interface FormState {
  values: Record<string, string>;
  errors: Record<string, string[]>;
  isSubmitting: boolean;
}

const [form, setForm] = useState<FormState>({
  values: {},
  errors: {},
  isSubmitting: false,
});
```

### Immutable Updates

```typescript
// ✓ Correct: Create new object
setUser(prev => ({ ...prev, name: newName }));

// ✓ Correct: Create new array
setItems(prev => [...prev, newItem]);
setItems(prev => prev.filter(item => item.id !== deleteId));

// ❌ Wrong: Mutating state
setUser(prev => {
  prev.name = newName; // Mutation!
  return prev;
});
```

## useReducer with TypeScript

### Discriminated Union Actions

```typescript
type State = {
  count: number;
  lastAction: string;
};

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'set'; payload: number }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1, lastAction: 'increment' };
    case 'decrement':
      return { count: state.count - 1, lastAction: 'decrement' };
    case 'set':
      return { count: action.payload, lastAction: 'set' };
    case 'reset':
      return { count: 0, lastAction: 'reset' };
    default:
      const _exhaustive: never = action;
      return state;
  }
}

// Usage
const [state, dispatch] = useReducer(reducer, { count: 0, lastAction: '' });

dispatch({ type: 'increment' });
dispatch({ type: 'set', payload: 10 });
```

**Benefits:**
- Exhaustive checking ensures all action types handled
- Type-safe payload for each action type
- Compile-time errors for invalid actions

### Complex State Management

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

type TodoState = {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
};

type TodoAction =
  | { type: 'add'; payload: { text: string } }
  | { type: 'toggle'; payload: { id: string } }
  | { type: 'delete'; payload: { id: string } }
  | { type: 'setFilter'; payload: { filter: TodoState['filter'] } };

function todoReducer(state: TodoState, action: TodoAction): TodoState {
  switch (action.type) {
    case 'add':
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            id: crypto.randomUUID(),
            text: action.payload.text,
            completed: false,
          },
        ],
      };
    case 'toggle':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
      };
    case 'delete':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload.id),
      };
    case 'setFilter':
      return {
        ...state,
        filter: action.payload.filter,
      };
  }
}
```

## Custom Hooks

### Type-Safe Custom Hook

```typescript
interface UseToggleReturn {
  isOn: boolean;
  toggle: () => void;
  setOn: () => void;
  setOff: () => void;
}

function useToggle(initialValue = false): UseToggleReturn {
  const [isOn, setIsOn] = useState(initialValue);

  return {
    isOn,
    toggle: () => setIsOn(prev => !prev),
    setOn: () => setIsOn(true),
    setOff: () => setIsOn(false),
  };
}

// Usage
const modal = useToggle(false);
<button onClick={modal.toggle}>Toggle Modal</button>
{modal.isOn && <Modal onClose={modal.setOff} />}
```

### Generic Custom Hook

```typescript
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((prev: T) => T)) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error saving to localStorage:`, error);
    }
  };

  return [storedValue, setValue];
}

// Usage
const [user, setUser] = useLocalStorage<User | null>('user', null);
```

## Context with TypeScript

### Type-Safe Context

```typescript
interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const user = await loginAPI(email, password);
    setUser(user);
  };

  const logout = () => {
    setUser(null);
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        login,
        logout,
        isAuthenticated: user !== null,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook for consuming context
export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Usage
function LoginButton() {
  const { login, isAuthenticated } = useAuth();
  // ...
}
```

## Summary

**Choose the right pattern:**
- **useState** - Simple state, single values, form inputs
- **useReducer** - Complex state logic, multiple sub-values, state transitions
- **Custom hooks** - Reusable stateful logic, encapsulate complexity
- **Context** - Global state, avoid prop drilling, theme/auth

**Common mistakes to avoid:**
- Mutating state directly instead of creating new objects/arrays
- Not providing explicit types when initial value is null
- Missing exhaustive checks in reducer switch statements
- Not throwing error in custom hook when context is undefined
- Over-using context (prefer composition and props)

**Remember:**
- Types document expected state shape and transitions
- Immutability is mandatory - always create new objects/arrays
- Discriminated unions provide type-safe state machines
- Custom hooks encapsulate reusable stateful behavior

## Related

- [Component Composition Patterns](./component-composition.md) - Props, generics, children
- [React Hooks Patterns](./hooks.md) - Advanced hook patterns
- [React Testing Patterns](./testing.md) - Testing stateful components
