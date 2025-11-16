# OWASP: Access Control and Authentication

Broken access control and authentication failure prevention for TypeScript applications.

## A01:2021 - Broken Access Control

### Description
Users can act outside their intended permissions, accessing unauthorized data or functionality.

### Common Vulnerabilities
- Missing authorization checks
- Insecure Direct Object References (IDOR)
- Privilege escalation
- CORS misconfiguration

### Insecure Direct Object References (IDOR)

```typescript
// ❌ Vulnerable: No ownership check
app.delete('/api/posts/:id', authenticate, async (req, res) => {
  await db.post.delete({ where: { id: req.params.id } });
  res.json({ success: true });
});
// User can delete ANY post, not just their own

// ✓ Secure: Verify ownership
app.delete('/api/posts/:id', authenticate, async (req, res) => {
  const post = await db.post.findUnique({ where: { id: req.params.id } });

  if (!post) {
    return res.status(404).json({ error: 'Post not found' });
  }

  if (post.authorId !== req.user.id) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  await db.post.delete({ where: { id: req.params.id } });
  res.json({ success: true });
});
```

### Role-Based Access Control (RBAC)

```typescript
// Define roles and permissions
enum Role {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

interface User {
  id: string;
  role: Role;
}

// Middleware: Require specific role
function requireRole(allowedRoles: Role[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
}

// Usage
app.delete('/api/users/:id', requireRole([Role.ADMIN]), async (req, res) => {
  await db.user.delete({ where: { id: req.params.id } });
  res.json({ success: true });
});

app.post('/api/posts/:id/moderate', requireRole([Role.ADMIN, Role.MODERATOR]), async (req, res) => {
  // Moderation logic
});
```

### Resource-Level Authorization

```typescript
// Check ownership for multiple resource types
async function requireOwnership(
  resourceType: 'post' | 'comment' | 'profile',
  req: Request,
  res: Response,
  next: NextFunction
) {
  const resourceId = req.params.id;
  const userId = req.user.id;

  let resource;
  switch (resourceType) {
    case 'post':
      resource = await db.post.findUnique({ where: { id: resourceId } });
      if (resource?.authorId !== userId) {
        return res.status(403).json({ error: 'Forbidden' });
      }
      break;
    case 'comment':
      resource = await db.comment.findUnique({ where: { id: resourceId } });
      if (resource?.authorId !== userId) {
        return res.status(403).json({ error: 'Forbidden' });
      }
      break;
    case 'profile':
      if (resourceId !== userId) {
        return res.status(403).json({ error: 'Forbidden' });
      }
      break;
  }

  next();
}

// Usage
app.put('/api/posts/:id', authenticate, (req, res, next) => {
  requireOwnership('post', req, res, next);
}, async (req, res) => {
  // Update post
});
```

### Prevention
- Deny by default (require explicit permission)
- Validate ownership on every request
- Use role-based access control (RBAC)
- Test with different user roles
- Log access control failures

## A07:2021 - Identification and Authentication Failures

### Description
Broken authentication allowing attackers to compromise accounts.

### Weak Session Timeout

```typescript
// ❌ Vulnerable: Long session timeout
app.use(session({
  cookie: { maxAge: 30 * 24 * 60 * 60 * 1000 } // 30 days!
}));

// ✓ Secure: Short session timeout + refresh tokens
app.use(session({
  cookie: {
    maxAge: 15 * 60 * 1000, // 15 minutes
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
  }
}));
```

### Password Policies

```typescript
import { z } from 'zod';

const PasswordSchema = z
  .string()
  .min(12, 'Password must be at least 12 characters')
  .regex(/[a-z]/, 'Password must contain lowercase letter')
  .regex(/[A-Z]/, 'Password must contain uppercase letter')
  .regex(/[0-9]/, 'Password must contain number')
  .regex(/[^a-zA-Z0-9]/, 'Password must contain special character');

// Check against breach databases
async function isPasswordBreached(password: string): Promise<boolean> {
  const hash = crypto.createHash('sha1').update(password).digest('hex').toUpperCase();
  const prefix = hash.substring(0, 5);
  const suffix = hash.substring(5);

  const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`);
  const hashes = await response.text();

  return hashes.split('\n').some((line: string) => line.startsWith(suffix));
}

app.post('/register', async (req, res) => {
  try {
    const password = PasswordSchema.parse(req.body.password);

    if (await isPasswordBreached(password)) {
      return res.status(400).json({
        error: 'Password found in breach database. Choose a different password.',
      });
    }

    // Proceed with registration
  } catch (error) {
    return res.status(400).json({ error: 'Invalid password' });
  }
});
```

### Brute Force Protection

```typescript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true, // Don't count successful logins
});

app.post('/login', loginLimiter, async (req, res) => {
  // Login logic
});

// Account lockout after failed attempts
const MAX_FAILED_ATTEMPTS = 5;
const LOCKOUT_DURATION = 30 * 60 * 1000; // 30 minutes

app.post('/login', async (req, res) => {
  const user = await db.user.findUnique({ where: { email: req.body.email } });

  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Check if account is locked
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    return res.status(423).json({
      error: 'Account locked. Try again later.',
      lockedUntil: user.lockedUntil,
    });
  }

  const isValid = await bcrypt.compare(req.body.password, user.passwordHash);

  if (!isValid) {
    // Increment failed attempts
    const failedAttempts = user.failedLoginAttempts + 1;

    if (failedAttempts >= MAX_FAILED_ATTEMPTS) {
      await db.user.update({
        where: { id: user.id },
        data: {
          failedLoginAttempts: failedAttempts,
          lockedUntil: new Date(Date.now() + LOCKOUT_DURATION),
        },
      });
      return res.status(423).json({ error: 'Account locked due to multiple failed attempts' });
    }

    await db.user.update({
      where: { id: user.id },
      data: { failedLoginAttempts: failedAttempts },
    });

    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Successful login - reset failed attempts
  await db.user.update({
    where: { id: user.id },
    data: { failedLoginAttempts: 0, lockedUntil: null },
  });

  // Create session
  req.session.userId = user.id;
  res.json({ success: true });
});
```

### Multi-Factor Authentication

```typescript
import speakeasy from 'speakeasy';

// Enable MFA
app.post('/mfa/setup', authenticate, async (req, res) => {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${req.user.email})`,
  });

  await db.user.update({
    where: { id: req.user.id },
    data: { mfaSecret: secret.base32 },
  });

  res.json({
    secret: secret.base32,
    qrCode: secret.otpauth_url,
  });
});

// Verify MFA code
app.post('/mfa/verify', authenticate, async (req, res) => {
  const { code } = req.body;
  const user = await db.user.findUnique({ where: { id: req.user.id } });

  const verified = speakeasy.totp.verify({
    secret: user.mfaSecret,
    encoding: 'base32',
    token: code,
    window: 1,
  });

  if (!verified) {
    return res.status(401).json({ error: 'Invalid MFA code' });
  }

  await db.user.update({
    where: { id: req.user.id },
    data: { mfaEnabled: true },
  });

  res.json({ success: true });
});
```

### Prevention
- Implement multi-factor authentication
- Strong password requirements (12+ chars)
- Check against breach databases
- Rate limiting on login attempts
- Secure session management
- No default credentials
- Account lockout after failed attempts

## CORS Misconfiguration

```typescript
// ❌ Vulnerable: Allow all origins
app.use(cors({ origin: '*' }));

// ✓ Secure: Specific origins
app.use(cors({
  origin: ['https://myapp.com', 'https://www.myapp.com'],
  credentials: true,
  maxAge: 86400, // 24 hours
}));

// ✓ Secure: Dynamic origin validation
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = ['https://myapp.com', 'https://www.myapp.com'];
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
}));
```

## Testing Access Control

```typescript
describe('Access Control', () => {
  test('requires authentication', async () => {
    const response = await request(app).get('/api/profile');
    expect(response.status).toBe(401);
  });

  test('validates ownership', async () => {
    const otherUserPost = await createPost({ authorId: 'other-user' });

    const response = await request(app)
      .delete(`/api/posts/${otherUserPost.id}`)
      .set('Authorization', `Bearer ${userToken}`);

    expect(response.status).toBe(403);
  });

  test('requires admin role', async () => {
    const response = await request(app)
      .delete('/api/users/123')
      .set('Authorization', `Bearer ${regularUserToken}`);

    expect(response.status).toBe(403);
  });

  test('rate limits login attempts', async () => {
    const requests = Array.from({ length: 10 }, () =>
      request(app).post('/login').send({ email, password: 'wrong' })
    );

    const responses = await Promise.all(requests);
    const tooManyRequests = responses.filter(r => r.status === 429);

    expect(tooManyRequests.length).toBeGreaterThan(0);
  });

  test('locks account after failed attempts', async () => {
    // Make 5 failed login attempts
    for (let i = 0; i < 5; i++) {
      await request(app).post('/login').send({ email, password: 'wrong' });
    }

    const response = await request(app).post('/login').send({ email, password: 'correct' });

    expect(response.status).toBe(423);
    expect(response.body.error).toContain('locked');
  });
});
```

## Related

- [JWT Authentication](./auth-jwt.md) - JWT implementation patterns
- [OAuth Patterns](./auth-oauth.md) - OAuth 2.0 flows
- [Session Management](./auth-sessions.md) - Server-side sessions
- [OWASP Injection](./owasp-injection.md) - Injection prevention
- [OWASP Cryptography](./owasp-crypto.md) - Cryptographic failures
