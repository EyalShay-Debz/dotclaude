# HTTP Status Codes Reference

## Quick Reference

| Code | Name | Use Case |
|------|------|----------|
| **2xx Success** |
| 200 | OK | Successful GET, PUT, PATCH (returns data) |
| 201 | Created | Successful POST (resource created) |
| 202 | Accepted | Async operation started |
| 204 | No Content | Successful DELETE, PUT, PATCH (no data) |
| 206 | Partial Content | Range request (streaming) |
| **3xx Redirection** |
| 301 | Moved Permanently | Permanent URL change |
| 302 | Found | Temporary redirect (may change method) |
| 304 | Not Modified | Cached resource still valid |
| 307 | Temporary Redirect | Temporary redirect (preserves method) |
| **4xx Client Errors** |
| 400 | Bad Request | Invalid input/validation error |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Valid auth, insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported |
| 409 | Conflict | Resource state conflict (duplicate, version mismatch) |
| 410 | Gone | Resource permanently deleted |
| 422 | Unprocessable Entity | Semantic validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| **5xx Server Errors** |
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | Upstream service error |
| 503 | Service Unavailable | Temporary unavailability (maintenance) |
| 504 | Gateway Timeout | Upstream timeout |

## Success Codes (2xx)

### 200 OK
Successful request with response body.

**Use for:**
- GET requests returning data
- PUT/PATCH returning updated resource
- POST returning data (non-creation)

```typescript
app.get('/users/:id', async (req, res) => {
  const user = await db.user.findUnique({ where: { id: req.params.id } });
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.status(200).json(user);
});
```

### 201 Created
Resource successfully created.

**Requirements:**
- Include `Location` header
- Return created resource (recommended)

```typescript
app.post('/users', async (req, res) => {
  const user = await db.user.create({ data: req.body });
  res.status(201).location(`/users/${user.id}`).json(user);
});
```

### 202 Accepted
Request accepted for async processing.

**Use for:** Background jobs, webhooks, long-running operations

```typescript
app.post('/reports/generate', async (req, res) => {
  const jobId = await queue.enqueue('generate-report', req.body);
  res.status(202).json({
    message: 'Report generation started',
    jobId,
    statusUrl: `/jobs/${jobId}`,
  });
});
```

### 204 No Content
Successful request with no response body.

**Use for:**
- DELETE operations
- PUT/PATCH when not returning updated resource
- Actions with no data to return

```typescript
app.delete('/users/:id', async (req, res) => {
  await db.user.delete({ where: { id: req.params.id } });
  res.status(204).send();
});
```

### 206 Partial Content
Partial response to range request.

**Use for:** Video streaming, large file downloads

```typescript
app.get('/videos/:id', (req, res) => {
  const range = req.headers.range;
  // Parse range, stream partial content
  res.status(206)
    .header('Content-Range', `bytes ${start}-${end}/${total}`)
    .header('Accept-Ranges', 'bytes');
  // Stream partial content
});
```

## Redirection Codes (3xx)

### 301 Moved Permanently
Permanent URL change. Clients should update bookmarks.

```typescript
app.get('/api/v1/users', (req, res) => {
  res.redirect(301, '/api/v2/users');
});
```

### 302 Found
Temporary redirect, may change HTTP method to GET.

### 304 Not Modified
Cached resource still valid (conditional request).

```typescript
app.get('/users/:id', async (req, res) => {
  const ifNoneMatch = req.headers['if-none-match'];
  const user = await db.user.findUnique({ where: { id: req.params.id } });
  const etag = generateETag(user);

  if (ifNoneMatch === etag) {
    return res.status(304).send();
  }

  res.setHeader('ETag', etag).json(user);
});
```

### 307 Temporary Redirect
Temporary redirect, preserves HTTP method (unlike 302).

## Client Error Codes (4xx)

### 400 Bad Request
Invalid request format or validation error.

```typescript
app.post('/users', async (req, res) => {
  try {
    const validated = CreateUserSchema.parse(req.body);
    const user = await db.user.create({ data: validated });
    res.status(201).json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid request data',
          details: error.errors,
        },
      });
    }
    throw error;
  }
});
```

### 401 Unauthorized
Missing or invalid authentication credentials.

```typescript
app.use((req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token || !verifyToken(token)) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Missing or invalid authentication token',
      },
    });
  }
  next();
});
```

### 403 Forbidden
Valid authentication but insufficient permissions.

```typescript
app.delete('/users/:id', requireAuth, async (req, res) => {
  if (req.user.role !== 'admin' && req.user.id !== req.params.id) {
    return res.status(403).json({
      error: {
        code: 'FORBIDDEN',
        message: 'Insufficient permissions to delete this user',
      },
    });
  }
  await db.user.delete({ where: { id: req.params.id } });
  res.status(204).send();
});
```

### 404 Not Found
Resource doesn't exist.

```typescript
app.get('/users/:id', async (req, res) => {
  const user = await db.user.findUnique({ where: { id: req.params.id } });
  if (!user) {
    return res.status(404).json({
      error: {
        code: 'RESOURCE_NOT_FOUND',
        message: 'User not found',
      },
    });
  }
  res.json(user);
});
```

### 405 Method Not Allowed
HTTP method not supported for this endpoint.

**Include `Allow` header** listing supported methods.

```typescript
app.all('/users/:id', (req, res, next) => {
  const allowedMethods = ['GET', 'PUT', 'PATCH', 'DELETE'];
  if (!allowedMethods.includes(req.method)) {
    res.setHeader('Allow', allowedMethods.join(', '));
    return res.status(405).json({
      error: {
        code: 'METHOD_NOT_ALLOWED',
        message: `Method ${req.method} not allowed`,
      },
    });
  }
  next();
});
```

### 409 Conflict
Request conflicts with current resource state.

**Use for:**
- Duplicate resources (email already exists)
- Version conflicts (optimistic locking)
- Business rule violations

```typescript
app.post('/users', async (req, res) => {
  const existing = await db.user.findUnique({ where: { email: req.body.email } });
  if (existing) {
    return res.status(409).json({
      error: {
        code: 'RESOURCE_CONFLICT',
        message: 'User with this email already exists',
      },
    });
  }
  const user = await db.user.create({ data: req.body });
  res.status(201).json(user);
});
```

### 410 Gone
Resource permanently deleted (different from 404).

```typescript
app.get('/users/:id', async (req, res) => {
  const user = await db.user.findUnique({ where: { id: req.params.id } });
  if (user?.deletedAt) {
    return res.status(410).json({
      error: {
        code: 'RESOURCE_GONE',
        message: 'User has been permanently deleted',
      },
    });
  }
  if (!user) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'User not found' } });
  }
  res.json(user);
});
```

### 422 Unprocessable Entity
Request format valid, but semantically incorrect.

**Difference from 400:**
- 400: Syntax error (malformed JSON, missing required field)
- 422: Semantic error (valid format, violates business rules)

```typescript
app.post('/orders', async (req, res) => {
  const order = OrderSchema.parse(req.body); // Validates format

  // Semantic validation
  if (order.quantity > product.stock) {
    return res.status(422).json({
      error: {
        code: 'UNPROCESSABLE_ENTITY',
        message: 'Insufficient stock available',
        details: [
          {
            field: 'quantity',
            message: `Only ${product.stock} items available`,
          },
        ],
      },
    });
  }

  const created = await db.order.create({ data: order });
  res.status(201).json(created);
});
```

### 429 Too Many Requests
Rate limit exceeded.

**Include headers:**
- `Retry-After`: Seconds until retry allowed
- `X-RateLimit-Limit`: Total allowed requests
- `X-RateLimit-Remaining`: Requests remaining
- `X-RateLimit-Reset`: Unix timestamp when limit resets

```typescript
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  handler: (req, res) => {
    res.status(429)
      .setHeader('Retry-After', 60)
      .json({
        error: {
          code: 'RATE_LIMIT_EXCEEDED',
          message: 'Too many requests. Please try again in 60 seconds.',
          retryAfter: 60,
        },
      });
  },
}));
```

## Server Error Codes (5xx)

### 500 Internal Server Error
Unexpected server error.

**Best practice:** Log error details server-side, return generic message to client.

```typescript
app.use((err, req, res, next) => {
  logger.error('Unhandled error', { error: err, requestId: req.id });

  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      requestId: req.id,
    },
  });
});
```

### 502 Bad Gateway
Upstream service returned invalid response.

```typescript
app.get('/external-data', async (req, res) => {
  try {
    const response = await fetch('https://external-api.com/data');
    if (!response.ok) {
      return res.status(502).json({
        error: {
          code: 'BAD_GATEWAY',
          message: 'Upstream service error',
        },
      });
    }
    const data = await response.json();
    res.json(data);
  } catch (error) {
    res.status(502).json({
      error: {
        code: 'BAD_GATEWAY',
        message: 'Failed to communicate with upstream service',
      },
    });
  }
});
```

### 503 Service Unavailable
Temporary unavailability (maintenance, overload).

**Include `Retry-After` header**.

```typescript
app.use((req, res, next) => {
  if (isMaintenanceMode()) {
    return res.status(503)
      .setHeader('Retry-After', 3600) // 1 hour
      .json({
        error: {
          code: 'SERVICE_UNAVAILABLE',
          message: 'Service temporarily unavailable for maintenance',
          retryAfter: 3600,
        },
      });
  }
  next();
});
```

### 504 Gateway Timeout
Upstream service timeout.

```typescript
app.get('/slow-service', async (req, res) => {
  try {
    const data = await fetch('https://slow-api.com/data', {
      signal: AbortSignal.timeout(5000), // 5 second timeout
    });
    res.json(data);
  } catch (error) {
    if (error.name === 'AbortError') {
      return res.status(504).json({
        error: {
          code: 'GATEWAY_TIMEOUT',
          message: 'Upstream service timeout',
        },
      });
    }
    throw error;
  }
});
```

## Status Code Decision Tree

```
Request received
  ↓
Authentication valid?
  NO → 401 Unauthorized
  YES → Continue
  ↓
Permissions sufficient?
  NO → 403 Forbidden
  YES → Continue
  ↓
Resource exists?
  NO → 404 Not Found (or 410 Gone if deleted)
  YES → Continue
  ↓
Request format valid?
  NO → 400 Bad Request
  YES → Continue
  ↓
Business rules satisfied?
  NO → 422 Unprocessable Entity (or 409 Conflict)
  YES → Continue
  ↓
Operation successful?
  NO → 500 Internal Server Error (or 502/503/504 if upstream issue)
  YES → Continue
  ↓
Response type:
  - Created resource → 201 Created
  - Updated/retrieved resource → 200 OK
  - Deleted/no content → 204 No Content
  - Async operation → 202 Accepted
```

## Common Mistakes

### Using 200 for Errors
```typescript
// ❌ BAD
res.status(200).json({ success: false, error: 'User not found' });

// ✓ GOOD
res.status(404).json({ error: { code: 'NOT_FOUND', message: 'User not found' } });
```

### Using 400 for Business Logic Errors
```typescript
// ❌ BAD (use 422 for semantic errors)
res.status(400).json({ error: 'Insufficient stock' });

// ✓ GOOD
res.status(422).json({ error: { code: 'INSUFFICIENT_STOCK', message: 'Not enough items in stock' } });
```

### Returning 500 for Expected Errors
```typescript
// ❌ BAD (duplicate email is expected error)
try {
  await db.user.create({ data: { email } });
} catch (error) {
  res.status(500).json({ error: 'Error creating user' });
}

// ✓ GOOD
if (await userExists(email)) {
  return res.status(409).json({ error: { code: 'DUPLICATE_EMAIL', message: 'Email already exists' } });
}
```

### Confusing 401 and 403
```typescript
// 401: Authentication problem (who are you?)
// Missing token, expired token, invalid credentials
res.status(401).json({ error: 'Invalid credentials' });

// 403: Authorization problem (you can't do this)
// Valid user, but lacks permission
res.status(403).json({ error: 'Insufficient permissions' });
```

## Related
- [API Design](../patterns/backend/api-design.md)
- [Error Handling](../patterns/backend/error-handling.md)
- [Security Patterns](../patterns/security/authentication.md)
