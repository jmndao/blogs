# Introducing AuthFlow: Universal Authentication Made Simple

Following the successful release of our [mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) plugin for intelligent MongoDB document processing, we're excited to introduce another developer productivity tool that solves a fundamental challenge in modern web development.

Authentication is one of those features every developer implements, yet it remains surprisingly complex to get right across different environments and frameworks. While building applications that needed to work seamlessly across browser, server-side rendering, and API contexts, we encountered the same authentication challenges repeatedly: token management, automatic refresh, request queuing, and environment-specific storage handling.

That's why we built [AuthFlow](https://github.com/jmndao/auth-flow)—a universal authentication client that handles these complexities automatically while providing a clean, consistent API across all JavaScript environments.

## The Authentication Challenge

Modern applications often need authentication that works across multiple contexts:

- **Client-side React apps** that need localStorage token management
- **Next.js applications** requiring both client and server-side authentication
- **Node.js APIs** that validate tokens from various sources
- **Universal apps** that render on both server and client

Each context has different requirements for token storage, extraction, and validation. Developers typically end up writing custom authentication logic for each environment, leading to inconsistent implementations and maintenance overhead.

## The AuthFlow Solution

[AuthFlow](https://github.com/jmndao/auth-flow) eliminates this complexity with intelligent defaults and automatic environment detection:

**Universal Compatibility**: Works seamlessly in browser, Node.js, and server-side rendering environments without configuration changes.

**Automatic Token Management**: Handles token refresh, storage, and validation automatically with configurable retry logic.

**Smart Environment Detection**: Automatically selects the optimal storage strategy (localStorage, cookies, or memory) based on the runtime environment.

**Framework Agnostic**: Clean integration with React, Next.js, Vue.js, Express, and vanilla JavaScript applications.

## Quick Start

Getting started with AuthFlow requires minimal configuration:

### Installation

```bash
npm install @jmndao/auth-flow
```

### Basic Setup

```typescript
import { createAuthFlow } from "@jmndao/auth-flow";

// Minimal setup - just pass your API URL
const auth = createAuthFlow("https://api.example.com");

// Login
const user = await auth.login({
  username: "user@example.com",
  password: "password",
});

// Make authenticated requests - tokens handled automatically
const profile = await auth.get("/user/profile");
const posts = await auth.get("/user/posts");

// Logout
await auth.logout();
```

### Advanced Configuration

For more control, AuthFlow provides comprehensive configuration options:

```typescript
const auth = createAuthFlow({
  baseURL: "https://api.example.com",
  endpoints: {
    login: "/auth/signin",
    refresh: "/auth/refresh-token",
    logout: "/auth/signout",
  },
  tokens: {
    access: "access_token",
    refresh: "refresh_token",
  },
  storage: "localStorage",
  timeout: 15000,
  retry: { attempts: 5, delay: 2000 },
  onTokenRefresh: (tokens) => {
    console.log("Tokens refreshed:", tokens);
  },
  onAuthError: (error) => {
    if (error.status === 401) {
      window.location.href = "/login";
    }
  },
});
```

## Framework Integration

AuthFlow's universal design enables consistent authentication across different frameworks:

### React Integration

```typescript
// Custom hook for React applications
export function useAuth() {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  const login = async (credentials) => {
    setIsLoading(true);
    try {
      const userData = await auth.login(credentials);
      setUser(userData);
      return true;
    } catch (error) {
      console.error("Login failed:", error);
      return false;
    } finally {
      setIsLoading(false);
    }
  };

  return { user, login, logout: auth.logout, isLoading };
}
```

### Next.js Server-Side

```typescript
// Server-side authentication in Next.js API routes
export default async function handler(req, res) {
  const auth = createAuthFlow("https://api.example.com", { req, res });

  try {
    if (await auth.hasValidTokens()) {
      const data = await auth.get("/protected-data");
      res.json(data);
    } else {
      res.status(401).json({ error: "Unauthorized" });
    }
  } catch (error) {
    res.status(500).json({ error: "Authentication failed" });
  }
}
```

### Express.js Middleware

```typescript
// Authentication middleware for Express
export const authMiddleware = async (req, res, next) => {
  const auth = createAuthFlow("https://api.example.com", { req, res });

  try {
    if (await auth.hasValidTokens()) {
      req.auth = auth;
      next();
    } else {
      res.status(401).json({ error: "Unauthorized" });
    }
  } catch (error) {
    res.status(401).json({ error: "Authentication failed" });
  }
};
```

## Key Features

### Intelligent Token Management

AuthFlow automatically handles token refresh, storage, and validation:

- **Automatic Refresh**: Tokens are refreshed automatically before expiration
- **Request Queuing**: Multiple requests during token refresh are queued and executed after refresh completes
- **Storage Optimization**: Automatically selects the best storage method for each environment
- **Error Recovery**: Comprehensive error handling with configurable retry logic

### Universal Storage Strategy

The library automatically adapts to different environments:

- **Browser**: Uses localStorage with cookie fallback
- **Server**: Uses cookies for persistent authentication
- **Universal**: Detects environment and chooses optimal storage
- **Custom**: Supports custom storage adapters for specialized requirements

### TypeScript Support

Full TypeScript definitions provide excellent developer experience:

```typescript
interface User {
  id: string;
  email: string;
  name: string;
}

// Type-safe authentication
const user = await auth.login<User>({ email, password });
const profile = await auth.get<User>("/profile");
```

## Production Considerations

### Performance

AuthFlow is designed for production use with efficient token management:

- **Lazy Loading**: Authentication client is only created when needed
- **Memory Efficient**: Minimal memory footprint with automatic cleanup
- **Request Optimization**: Intelligent request batching during token refresh

### Security

Built-in security features protect against common vulnerabilities:

- **Secure Token Storage**: Tokens are stored securely based on environment capabilities
- **Automatic Cleanup**: Tokens are cleared on logout and authentication errors
- **CSRF Protection**: Cookie-based storage includes proper security attributes

### Scalability

The library scales from simple prototypes to enterprise applications:

- **Environment Agnostic**: Same code works across development and production
- **Framework Flexible**: Easy migration between frameworks without authentication rewrites
- **Extensible**: Custom storage adapters and error handlers for specialized needs

## Real-World Applications

AuthFlow excels in diverse authentication scenarios:

- **Single Page Applications**: Seamless token management for React, Vue, and Angular apps
- **Server-Side Rendered Apps**: Consistent authentication across client and server contexts
- **API Services**: Robust authentication for Node.js microservices and APIs
- **Mobile Web Apps**: Optimized storage strategies for mobile browsers

## Getting Started

AuthFlow is available through standard JavaScript package managers:

- **GitHub Repository**: https://github.com/jmndao/auth-flow
- **Documentation Site**: https://auth-flow-virid.vercel.app/
- **npm Package**: `@jmndao/auth-flow`

The documentation site provides comprehensive guides, API reference, and interactive examples. The repository includes practical examples for all major frameworks and deployment scenarios.

## Contributing and Community

AuthFlow is an open-source project that welcomes community contributions. The modular architecture makes it easy to add new features and storage adapters. Current development priorities include:

- Enhanced cookie handling for complex server configurations
- Built-in session management for backends without refresh tokens
- Additional framework integrations and examples
- Performance optimizations for high-traffic applications

## Conclusion

AuthFlow solves the universal authentication challenge by providing a single, consistent API that works across all JavaScript environments. By handling token management, storage optimization, and error recovery automatically, developers can focus on building features instead of wrestling with authentication complexity.

The library's intelligent defaults and automatic environment detection make it perfect for rapid prototyping, while its comprehensive configuration options and TypeScript support make it suitable for enterprise applications. Whether you're building a simple React app or a complex universal application, AuthFlow provides the authentication foundation you need.

Experience simplified authentication management—try [AuthFlow](https://github.com/jmndao/auth-flow) today and see how universal authentication should work.

**Get started**: https://github.com/jmndao/auth-flow  
**Documentation**: https://auth-flow-virid.vercel.app/
