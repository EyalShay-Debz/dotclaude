# React Testing: Common Patterns

Common testing scenarios and patterns for React components.

## Form Submission

```typescript
describe('ContactForm', () => {
  it('should submit form with valid data', async () => {
    const onSubmit = jest.fn();
    render(<ContactForm onSubmit={onSubmit} />);

    // Fill out form
    await userEvent.type(screen.getByLabelText('Name'), 'John Doe');
    await userEvent.type(screen.getByLabelText('Email'), 'john@example.com');
    await userEvent.type(screen.getByLabelText('Message'), 'Hello world');

    // Submit
    await userEvent.click(screen.getByRole('button', { name: 'Send' }));

    // Assert submission
    expect(onSubmit).toHaveBeenCalledTimes(1);
    expect(onSubmit).toHaveBeenCalledWith({
      name: 'John Doe',
      email: 'john@example.com',
      message: 'Hello world'
    });
  });

  it('should show validation errors for invalid input', async () => {
    render(<ContactForm onSubmit={jest.fn()} />);

    // Submit empty form
    await userEvent.click(screen.getByRole('button', { name: 'Send' }));

    // Assert validation errors visible
    expect(screen.getByText('Name is required')).toBeInTheDocument();
    expect(screen.getByText('Email is required')).toBeInTheDocument();
  });
});
```

---

## Conditional Rendering

```typescript
describe('UserProfile', () => {
  it('should show loading state initially', () => {
    render(<UserProfile userId="123" />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();
    expect(screen.queryByText('Profile')).not.toBeInTheDocument();
  });

  it('should show user data after loading', async () => {
    render(<UserProfile userId="123" />);

    // Wait for data to load
    const heading = await screen.findByRole('heading', { name: 'John Doe' });
    expect(heading).toBeInTheDocument();

    // Loading spinner should be gone
    expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
  });

  it('should show error state on failure', async () => {
    // Mock API to fail
    jest.spyOn(api, 'fetchUser').mockRejectedValue(new Error('API error'));

    render(<UserProfile userId="123" />);

    // Wait for error message
    const error = await screen.findByText('Failed to load user');
    expect(error).toBeInTheDocument();
  });
});
```

---

## Toggle/Show/Hide

```typescript
describe('Dropdown', () => {
  it('should show menu when clicked', async () => {
    render(<Dropdown />);

    // Menu initially hidden
    expect(screen.queryByRole('menu')).not.toBeInTheDocument();

    // Click trigger
    await userEvent.click(screen.getByRole('button', { name: 'Options' }));

    // Menu now visible
    expect(screen.getByRole('menu')).toBeInTheDocument();
    expect(screen.getByRole('menuitem', { name: 'Edit' })).toBeInTheDocument();
  });

  it('should hide menu when clicking outside', async () => {
    render(<Dropdown />);

    // Open menu
    await userEvent.click(screen.getByRole('button', { name: 'Options' }));
    expect(screen.getByRole('menu')).toBeInTheDocument();

    // Click outside
    await userEvent.click(document.body);

    // Menu closed
    expect(screen.queryByRole('menu')).not.toBeInTheDocument();
  });
});
```

---

## List Rendering

```typescript
describe('TodoList', () => {
  it('should render all todo items', () => {
    const todos = [
      { id: '1', text: 'Buy milk', completed: false },
      { id: '2', text: 'Walk dog', completed: true },
      { id: '3', text: 'Write code', completed: false }
    ];

    render(<TodoList todos={todos} />);

    const items = screen.getAllByRole('listitem');
    expect(items).toHaveLength(3);

    expect(screen.getByText('Buy milk')).toBeInTheDocument();
    expect(screen.getByText('Walk dog')).toBeInTheDocument();
    expect(screen.getByText('Write code')).toBeInTheDocument();
  });

  it('should toggle todo completion on click', async () => {
    const onToggle = jest.fn();
    const todos = [{ id: '1', text: 'Buy milk', completed: false }];

    render(<TodoList todos={todos} onToggle={onToggle} />);

    // Click checkbox
    const checkbox = screen.getByRole('checkbox', { name: 'Buy milk' });
    await userEvent.click(checkbox);

    expect(onToggle).toHaveBeenCalledWith('1');
  });
});
```

---

## Accessibility Testing

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('LoginForm', () => {
  it('should have no accessibility violations', async () => {
    const { container } = render(<LoginForm />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should have proper ARIA labels', () => {
    render(<LoginForm />);

    expect(screen.getByLabelText('Email')).toHaveAttribute('type', 'email');
    expect(screen.getByLabelText('Password')).toHaveAttribute('type', 'password');
    expect(screen.getByRole('button', { name: 'Sign In' })).toBeEnabled();
  });
});
```

---

## Disabled State Testing

```typescript
describe('PaymentForm', () => {
  it('should not submit while processing', async () => {
    const onSubmit = jest.fn(() => new Promise(() => {})); // Never resolves
    render(<PaymentForm onSubmit={onSubmit} />);

    const button = screen.getByRole('button', { name: 'Pay' });

    await userEvent.click(button);

    // Button disabled during processing
    expect(button).toBeDisabled();

    // Second click doesn't call onSubmit again
    await userEvent.click(button);
    expect(onSubmit).toHaveBeenCalledTimes(1);
  });
});
```

---

## Multi-Step Forms

```typescript
describe('RegistrationWizard', () => {
  it('should navigate through steps', async () => {
    render(<RegistrationWizard />);

    // Step 1: Personal Info
    expect(screen.getByRole('heading', { name: 'Personal Information' })).toBeInTheDocument();

    await userEvent.type(screen.getByLabelText('First Name'), 'John');
    await userEvent.type(screen.getByLabelText('Last Name'), 'Doe');
    await userEvent.click(screen.getByRole('button', { name: 'Next' }));

    // Step 2: Account Info
    expect(screen.getByRole('heading', { name: 'Account Information' })).toBeInTheDocument();

    await userEvent.type(screen.getByLabelText('Email'), 'john@example.com');
    await userEvent.type(screen.getByLabelText('Password'), 'password123');
    await userEvent.click(screen.getByRole('button', { name: 'Next' }));

    // Step 3: Confirmation
    expect(screen.getByRole('heading', { name: 'Confirm Registration' })).toBeInTheDocument();
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('should allow going back to previous step', async () => {
    render(<RegistrationWizard />);

    // Go to step 2
    await userEvent.click(screen.getByRole('button', { name: 'Next' }));

    // Go back to step 1
    await userEvent.click(screen.getByRole('button', { name: 'Back' }));

    expect(screen.getByRole('heading', { name: 'Personal Information' })).toBeInTheDocument();
  });
});
```

---

## Search/Filter Patterns

```typescript
describe('ProductList', () => {
  it('should filter products by search query', async () => {
    const products = [
      { id: '1', name: 'Widget', category: 'tools' },
      { id: '2', name: 'Gadget', category: 'electronics' },
      { id: '3', name: 'Gizmo', category: 'tools' }
    ];

    render(<ProductList products={products} />);

    // All products visible initially
    expect(screen.getAllByRole('listitem')).toHaveLength(3);

    // Filter by search
    await userEvent.type(screen.getByLabelText('Search'), 'gad');

    // Only matching product visible
    expect(screen.getByText('Gadget')).toBeInTheDocument();
    expect(screen.queryByText('Widget')).not.toBeInTheDocument();
    expect(screen.queryByText('Gizmo')).not.toBeInTheDocument();
  });

  it('should filter by category', async () => {
    const products = [
      { id: '1', name: 'Widget', category: 'tools' },
      { id: '2', name: 'Gadget', category: 'electronics' }
    ];

    render(<ProductList products={products} />);

    await userEvent.selectOptions(screen.getByLabelText('Category'), 'tools');

    expect(screen.getByText('Widget')).toBeInTheDocument();
    expect(screen.queryByText('Gadget')).not.toBeInTheDocument();
  });
});
```

---

## Pagination

```typescript
describe('PaginatedList', () => {
  it('should paginate through items', async () => {
    const items = Array.from({ length: 50 }, (_, i) => ({
      id: String(i),
      name: `Item ${i}`
    }));

    render(<PaginatedList items={items} pageSize={10} />);

    // First page visible
    expect(screen.getByText('Item 0')).toBeInTheDocument();
    expect(screen.getByText('Item 9')).toBeInTheDocument();
    expect(screen.queryByText('Item 10')).not.toBeInTheDocument();

    // Go to next page
    await userEvent.click(screen.getByRole('button', { name: 'Next' }));

    // Second page visible
    expect(screen.queryByText('Item 0')).not.toBeInTheDocument();
    expect(screen.getByText('Item 10')).toBeInTheDocument();
    expect(screen.getByText('Item 19')).toBeInTheDocument();
  });
});
```

---

## Modal/Dialog Testing

```typescript
describe('ConfirmationModal', () => {
  it('should confirm action', async () => {
    const onConfirm = jest.fn();
    render(<ConfirmationModal onConfirm={onConfirm} onCancel={jest.fn()} />);

    expect(screen.getByRole('dialog')).toBeInTheDocument();
    expect(screen.getByText('Are you sure?')).toBeInTheDocument();

    await userEvent.click(screen.getByRole('button', { name: 'Confirm' }));

    expect(onConfirm).toHaveBeenCalledTimes(1);
  });

  it('should close on cancel', async () => {
    const onCancel = jest.fn();
    render(<ConfirmationModal onConfirm={jest.fn()} onCancel={onCancel} />);

    await userEvent.click(screen.getByRole('button', { name: 'Cancel' }));

    expect(onCancel).toHaveBeenCalledTimes(1);
  });

  it('should close on escape key', async () => {
    const onCancel = jest.fn();
    render(<ConfirmationModal onConfirm={jest.fn()} onCancel={onCancel} />);

    await userEvent.keyboard('{Escape}');

    expect(onCancel).toHaveBeenCalledTimes(1);
  });
});
```

---

## Error Boundaries

```typescript
describe('ErrorBoundary', () => {
  it('should catch errors and display fallback', () => {
    const ThrowError = () => {
      throw new Error('Test error');
    };

    // Suppress console.error for this test
    const spy = jest.spyOn(console, 'error').mockImplementation(() => {});

    render(
      <ErrorBoundary fallback={<div>Something went wrong</div>}>
        <ThrowError />
      </ErrorBoundary>
    );

    expect(screen.getByText('Something went wrong')).toBeInTheDocument();

    spy.mockRestore();
  });

  it('should render children when no error', () => {
    render(
      <ErrorBoundary fallback={<div>Error</div>}>
        <div>Normal content</div>
      </ErrorBoundary>
    );

    expect(screen.getByText('Normal content')).toBeInTheDocument();
    expect(screen.queryByText('Error')).not.toBeInTheDocument();
  });
});
```

---

## Summary

**Common patterns covered:**
- Form submission and validation
- Conditional rendering (loading, error, success states)
- Toggle/show/hide interactions
- List rendering and iteration
- Accessibility testing with jest-axe
- Disabled state and button prevention
- Multi-step forms and wizards
- Search and filter functionality
- Pagination
- Modal/dialog interactions
- Error boundaries

**Key principles:**
- Always test through user interactions
- Assert on visible outcomes
- Use accessible queries (roles, labels)
- Handle async operations with findBy* or waitFor
- Test edge cases (empty states, errors, disabled states)

See also:
- @~/.claude/docs/patterns/react/testing-queries.md for query fundamentals
- @~/.claude/docs/patterns/react/testing-mocks.md for mocking patterns
