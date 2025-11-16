# OWASP: Cryptography, Configuration, and Data Integrity

Cryptographic failures, security misconfiguration, vulnerable components, data integrity, and logging patterns.

## A02:2021 - Cryptographic Failures

### Description
Exposure of sensitive data due to weak or missing encryption.

### Password Hashing

```typescript
import bcrypt from 'bcrypt';

// Hash password (register)
const SALT_ROUNDS = 12;
const hashedPassword = await bcrypt.hash(plainPassword, SALT_ROUNDS);

await db.user.create({
  data: {
    email,
    passwordHash: hashedPassword,
  },
});

// Verify password (login)
const user = await db.user.findUnique({ where: { email } });
const isValid = await bcrypt.compare(plainPassword, user.passwordHash);
```

**Argon2 (More Secure Alternative)**
```typescript
import argon2 from 'argon2';

const hashedPassword = await argon2.hash(plainPassword);
const isValid = await argon2.verify(hashedPassword, plainPassword);
```

### Data Encryption at Rest

```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex'); // 32 bytes

function encrypt(text: string): { encrypted: string; iv: string; tag: string } {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  return {
    encrypted,
    iv: iv.toString('hex'),
    tag: cipher.getAuthTag().toString('hex'),
  };
}

function decrypt(encrypted: string, iv: string, tag: string): string {
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    KEY,
    Buffer.from(iv, 'hex')
  );
  decipher.setAuthTag(Buffer.from(tag, 'hex'));

  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}
```

### Sensitive Data in Logs

```typescript
// ❌ Vulnerable: Sensitive data in logs
logger.error('Payment failed', { creditCard: '4111111111111111' });

// ✓ Secure: Redact sensitive data
function redactSensitiveData(data: any): any {
  const sensitiveFields = ['password', 'token', 'creditCard', 'ssn'];

  if (typeof data !== 'object') return data;

  const redacted = { ...data };
  for (const key of Object.keys(redacted)) {
    if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
      redacted[key] = '***REDACTED***';
    }
  }
  return redacted;
}

logger.error('Payment failed', redactSensitiveData({ creditCard: '****1111' }));
```

### Prevention
- Use TLS/HTTPS for all connections
- Hash passwords with bcrypt/argon2
- Encrypt sensitive data at rest (AES-256-GCM)
- Avoid sensitive data in URLs/logs
- Implement key rotation
- Use strong random values (crypto.randomBytes)

## A04:2021 - Insecure Design

### Forgot Password Without Rate Limiting

```typescript
// ❌ Vulnerable: Email enumeration
app.post('/forgot-password', async (req, res) => {
  const { email } = req.body;
  await sendPasswordResetEmail(email);
  res.json({ success: true });
});

// ✓ Secure: Rate limiting + generic response
import rateLimit from 'express-rate-limit';

const forgotPasswordLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 3,
});

app.post('/forgot-password', forgotPasswordLimiter, async (req, res) => {
  const { email } = req.body;

  const user = await db.user.findUnique({ where: { email } });
  if (user) {
    await sendPasswordResetEmail(email);
  }

  // Always return success (don't reveal if email exists)
  res.json({ message: 'If account exists, reset email sent' });
});
```

## A05:2021 - Security Misconfiguration

### Error Handling

```typescript
// ❌ Vulnerable: Exposing stack traces
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack // Exposes internal structure!
  });
});

// ✓ Secure: Generic error in production
app.use((err, req, res, next) => {
  logger.error('Unhandled error', { error: err });

  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({ error: err.message, stack: err.stack });
  }
});
```

### Security Headers

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

### CORS Configuration

```typescript
// ❌ Vulnerable: CORS allow all
app.use(cors({ origin: '*' }));

// ✓ Secure: Specific origins
app.use(cors({
  origin: ['https://myapp.com', 'https://www.myapp.com'],
  credentials: true,
}));
```

## A06:2021 - Vulnerable and Outdated Components

### Dependency Management

```bash
# Check for vulnerabilities
npm audit

# Fix automatically (where possible)
npm audit fix

# Use automated updates (Dependabot/Renovate)
```

**GitHub Actions Security Scan**
```yaml
name: Security Scan
on: [push, pull_request]
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

### Best Practices
- Monitor security advisories
- Keep dependencies updated
- Remove unused dependencies
- Use lock files (package-lock.json)
- Automated dependency updates
- Regular security audits

## A08:2021 - Software and Data Integrity Failures

### Webhook Signature Verification

```typescript
// ❌ Vulnerable: Trust external data
app.post('/webhook', async (req, res) => {
  await processWebhook(req.body); // Trust external data!
});

// ✓ Secure: Verify signature
app.post('/webhook', async (req, res) => {
  const signature = req.headers['x-signature'];
  const payload = JSON.stringify(req.body);

  const expectedSignature = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');

  if (signature !== expectedSignature) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  await processWebhook(req.body);
  res.json({ success: true });
});
```

### Input Validation

```typescript
// ❌ Vulnerable: Deserialize untrusted data
const userData = JSON.parse(req.body.data);
eval(userData.code); // NEVER use eval!

// ✓ Secure: Validate before deserialization
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string().max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(120),
});

const userData = UserSchema.parse(req.body.data);
```

## A09:2021 - Security Logging and Monitoring Failures

### Comprehensive Logging

```typescript
// ❌ Vulnerable: No logging
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  res.json({ success: true });
});

// ✓ Secure: Log security events
app.post('/login', async (req, res) => {
  const { email } = req.body;

  logger.info('Login attempt', {
    email,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
  });

  const user = await authenticateUser(req.body);

  if (!user) {
    logger.warn('Failed login attempt', {
      email,
      ip: req.ip,
      reason: 'Invalid credentials',
    });
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  logger.info('Successful login', {
    userId: user.id,
    email,
    ip: req.ip,
  });

  res.json({ success: true });
});
```

### What to Log

**Security events:**
- Login attempts (success/failure)
- Password resets
- Account changes
- Access control failures
- Input validation failures

**DO NOT log:**
- Passwords
- Session tokens
- Credit card numbers
- API keys
- Other sensitive data

### Structured Logging

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}
```

## Testing Security

```typescript
describe('Security Tests', () => {
  test('uses HTTPS in production', () => {
    if (process.env.NODE_ENV === 'production') {
      expect(app.get('trust proxy')).toBeTruthy();
    }
  });

  test('sets security headers', async () => {
    const response = await request(app).get('/');

    expect(response.headers['x-content-type-options']).toBe('nosniff');
    expect(response.headers['x-frame-options']).toBeDefined();
    expect(response.headers['strict-transport-security']).toBeDefined();
  });

  test('does not expose stack traces in production', async () => {
    process.env.NODE_ENV = 'production';

    const response = await request(app).get('/error-route');

    expect(response.status).toBe(500);
    expect(response.body.stack).toBeUndefined();
    expect(response.body.error).toBe('Internal server error');
  });

  test('verifies webhook signatures', async () => {
    const payload = { event: 'test' };
    const invalidSignature = 'invalid';

    const response = await request(app)
      .post('/webhook')
      .set('x-signature', invalidSignature)
      .send(payload);

    expect(response.status).toBe(401);
  });
});
```

## Resources

- [OWASP Top 10 2021](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)

## Related

- [OWASP Injection](./owasp-injection.md) - Injection and SSRF prevention
- [OWASP Authentication](./owasp-auth.md) - Access control and authentication
- [Authentication Patterns](./auth-jwt.md) - JWT, OAuth, sessions
