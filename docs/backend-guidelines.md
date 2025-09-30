# Backend Development Guidelines

This document outlines the standards and best practices for backend development in the daiginjo-ai organization.

## Technology Stack

### Primary Technologies
- **Runtime**: Node.js (LTS version)
- **Framework**: Apollo GraphQL with Express.js
- **Language**: TypeScript (required)
- **Database**: MongoDB (primary), Redis (caching)
- **ODM**: Mongoose
- **GraphQL**: Apollo Server

### Development Tools
- **Package Manager**: npm or yarn
- **Testing**: Jest + Supertest + Apollo Server Testing
- **Linting**: ESLint + Prettier
- **Type Checking**: TypeScript
- **Documentation**: GraphQL introspection + Apollo Studio
- **Code Generation**: GraphQL Code Generator (optional)

## Project Structure

```
backend/
├── src/
│   ├── graphql/         # GraphQL schema and resolvers
│   │   ├── resolvers/   # GraphQL resolvers
│   │   ├── schema/      # GraphQL schema definitions
│   │   └── types/       # GraphQL type definitions
│   ├── services/        # Business logic layer
│   ├── models/          # Mongoose models and schemas
│   ├── repositories/    # Data access layer
│   ├── middleware/      # Express and Apollo middleware
│   ├── routes/          # Express route definitions (health, etc.)
│   ├── utils/           # Utility functions
│   ├── types/           # TypeScript type definitions
│   ├── config/          # Configuration management
│   ├── database/        # MongoDB connection and setup
│   ├── server.ts        # GraphQL server setup
│   └── app.ts           # Express application setup
├── tests/               # Test files
├── docs/                # API documentation
└── scripts/             # Build and deployment scripts
```

## Coding Standards

### TypeScript Configuration
- Use strict mode: `"strict": true`
- Enable all recommended rules
- Use explicit return types for functions
- Prefer interfaces over types for object shapes

### Naming Conventions
- **Files**: camelCase (`userController.ts`, `emailService.ts`)
- **Classes**: PascalCase (`UserService`, `DatabaseConnection`)
- **Functions/Variables**: camelCase (`getUserById`, `isAuthenticated`)
- **Constants**: UPPER_SNAKE_CASE (`API_BASE_URL`, `MAX_RETRY_ATTEMPTS`)
- **Interfaces**: PascalCase with 'I' prefix (`IUser`, `IApiResponse`)

### Code Organization

#### GraphQL Resolvers
- Handle GraphQL queries and mutations
- Validate input parameters
- Delegate business logic to services
- Return consistent data structures

```typescript
// userResolver.ts
export const userResolvers = {
  Query: {
    user: async (_: any, { id }: { id: string }, context: Context): Promise<User> => {
      return await context.services.userService.getUserById(id);
    },
    users: async (_: any, args: any, context: Context): Promise<User[]> => {
      return await context.services.userService.getAllUsers();
    }
  },
  Mutation: {
    createUser: async (_: any, { input }: { input: CreateUserInput }, context: Context): Promise<User> => {
      return await context.services.userService.createUser(input);
    }
  }
};
```

#### Express Controllers (Health & Utility)
- Handle non-GraphQL endpoints
- Health checks and system status
- File uploads and downloads

```typescript
// healthController.ts
export const healthCheck = (req: Request, res: Response): void => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version || '1.0.0'
  });
};
```

#### Services
- Contain business logic
- Be framework-agnostic
- Handle data validation
- Manage transactions

```typescript
// userService.ts
export class UserService {
  async getUserById(id: string): Promise<IUser> {
    if (!id) {
      throw new Error('User ID is required');
    }
    return await this.userRepository.findById(id);
  }
}
```

#### Repositories
- Handle data access only
- Abstract database operations
- Implement consistent error handling

```typescript
// userRepository.ts
import { User, IUser } from '../models/User';

export class UserRepository {
  async findById(id: string): Promise<IUser | null> {
    return await User.findById(id).exec();
  }

  async findByEmail(email: string): Promise<IUser | null> {
    return await User.findOne({ email }).exec();
  }

  async create(userData: Partial<IUser>): Promise<IUser> {
    const user = new User(userData);
    return await user.save();
  }

  async updateById(id: string, updates: Partial<IUser>): Promise<IUser | null> {
    return await User.findByIdAndUpdate(id, updates, { new: true }).exec();
  }

  async deleteById(id: string): Promise<boolean> {
    const result = await User.findByIdAndDelete(id).exec();
    return !!result;
  }
}
```

## API Design

### GraphQL Schema Design
- Use Apollo GraphQL as the primary API layer
- Define clear type definitions and resolvers
- Implement proper error handling in resolvers
- Use DataLoader for efficient data fetching

### Express Routes
- Use Express for health checks and non-GraphQL endpoints
- Health endpoint must be available at `/health`
- Authentication endpoints can use Express routes
- File upload endpoints should use Express

### GraphQL Schema Example
```typescript
// schema/user.ts
import { gql } from 'apollo-server-express';

export const userTypeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
    updatedAt: DateTime!
  }

  input CreateUserInput {
    name: String!
    email: String!
  }

  input UpdateUserInput {
    name: String
    email: String
  }

  extend type Query {
    user(id: ID!): User
    users(limit: Int, offset: Int): [User!]!
  }

  extend type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
  }
`;
```

### Express Health Endpoint
```typescript
// Required health endpoint response format
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "uptime": 3600,
  "version": "1.0.0"
}
```

### Application Setup

#### Express App Configuration
```typescript
// app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { healthCheck } from './routes/health';

const app = express();

// Security middleware
app.use(helmet({
  contentSecurityPolicy: false, // Allow GraphQL Playground in development
  crossOriginEmbedderPolicy: false
}));

// CORS configuration
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true
}));

// Health check endpoint (required)
app.get('/health', healthCheck);

export { app };
```

#### Apollo Server Setup
```typescript
// server.ts
import { ApolloServer } from 'apollo-server-express';
import { buildSchema } from 'type-graphql';
import { app } from './app';
import { Context } from './types/context';
import { userResolvers } from './graphql/resolvers';
import { typeDefs } from './graphql/schema';
import { DatabaseConnection } from './database/connection';

interface ServerContext {
  req: Request;
  res: Response;
}

const createApolloServer = async (): Promise<ApolloServer> => {
  const server = new ApolloServer({
    typeDefs,
    resolvers: userResolvers,
    context: ({ req, res }: ServerContext): Context => ({
      req,
      res,
      services: {
        userService: new UserService(),
        // Add other services
      },
      user: req.user, // From authentication middleware
    }),
    introspection: process.env.NODE_ENV !== 'production',
    playground: process.env.NODE_ENV !== 'production',
    formatError: (error) => {
      console.error('GraphQL Error:', error);
      return {
        message: error.message,
        code: error.extensions?.code,
        path: error.path,
      };
    },
  });

  return server;
};

const startServer = async (): Promise<void> => {
  try {
    // Initialize database connection
    const db = DatabaseConnection.getInstance();
    await db.connect({
      uri: process.env.MONGODB_URI || 'mongodb://localhost:27017/your-app',
    });

    const server = await createApolloServer();
    await server.start();

    server.applyMiddleware({
      app,
      path: '/graphql',
      cors: false // Use Express CORS instead
    });

    const PORT = process.env.PORT || 4000;
    app.listen(PORT, () => {
      console.log(`🚀 Server ready at http://localhost:${PORT}${server.graphqlPath}`);
      console.log(`💚 Health check available at http://localhost:${PORT}/health`);
      console.log(`📊 MongoDB connected and ready`);
    });

    // Graceful shutdown
    process.on('SIGTERM', async () => {
      console.log('SIGTERM received, shutting down gracefully');
      await server.stop();
      await db.disconnect();
      process.exit(0);
    });

  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
};

startServer().catch(error => {
  console.error('Failed to start server:', error);
  process.exit(1);
});
```

## Database Guidelines

### MongoDB Schema Design with Mongoose
- Use MongoDB ObjectIds for primary keys
- Include `createdAt` and `updatedAt` timestamps (use Mongoose timestamps)
- Use descriptive field names following camelCase convention
- Implement proper schema validation
- Define indexes for frequently queried fields

```typescript
// models/User.ts
import mongoose, { Schema, Document } from 'mongoose';

export interface IUser extends Document {
  _id: mongoose.Types.ObjectId;
  name: string;
  email: string;
  passwordHash: string;
  role: 'admin' | 'user';
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const UserSchema = new Schema<IUser>({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    minlength: [2, 'Name must be at least 2 characters'],
    maxlength: [100, 'Name cannot exceed 100 characters']
  },
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    lowercase: true,
    trim: true,
    validate: {
      validator: (email: string) => {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
      },
      message: 'Please provide a valid email address'
    }
  },
  passwordHash: {
    type: String,
    required: [true, 'Password hash is required'],
    select: false // Don't include in queries by default
  },
  role: {
    type: String,
    enum: ['admin', 'user'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, {
  timestamps: true, // Automatically adds createdAt and updatedAt
  toJSON: {
    transform: (doc, ret) => {
      ret.id = ret._id;
      delete ret._id;
      delete ret.__v;
      delete ret.passwordHash;
      return ret;
    }
  }
});

// Indexes
UserSchema.index({ email: 1 }, { unique: true });
UserSchema.index({ createdAt: 1 });
UserSchema.index({ role: 1, isActive: 1 });

// Pre-save middleware
UserSchema.pre('save', function(next) {
  if (this.isModified('email')) {
    this.email = this.email.toLowerCase();
  }
  next();
});

export const User = mongoose.model<IUser>('User', UserSchema);
```

### Database Connection Setup
```typescript
// database/connection.ts
import mongoose from 'mongoose';

interface ConnectionOptions {
  uri: string;
  options?: mongoose.ConnectOptions;
}

export class DatabaseConnection {
  private static instance: DatabaseConnection;
  private isConnected = false;

  public static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  public async connect({ uri, options = {} }: ConnectionOptions): Promise<void> {
    if (this.isConnected) {
      console.log('Already connected to MongoDB');
      return;
    }

    const defaultOptions: mongoose.ConnectOptions = {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
      bufferCommands: false,
      bufferMaxEntries: 0,
      ...options
    };

    try {
      await mongoose.connect(uri, defaultOptions);
      this.isConnected = true;
      console.log('Connected to MongoDB successfully');

      // Handle connection events
      mongoose.connection.on('disconnected', () => {
        console.log('MongoDB disconnected');
        this.isConnected = false;
      });

      mongoose.connection.on('error', (error) => {
        console.error('MongoDB connection error:', error);
      });

    } catch (error) {
      console.error('Failed to connect to MongoDB:', error);
      throw error;
    }
  }

  public async disconnect(): Promise<void> {
    if (!this.isConnected) {
      return;
    }

    await mongoose.disconnect();
    this.isConnected = false;
    console.log('Disconnected from MongoDB');
  }

  public getConnectionStatus(): boolean {
    return this.isConnected && mongoose.connection.readyState === 1;
  }
}
```

### Query Optimization
- Use proper indexes for frequently queried fields
- Implement pagination using `skip()` and `limit()`
- Use `lean()` for read-only operations to improve performance
- Use aggregation pipelines for complex queries
- Monitor query performance with MongoDB profiler

```typescript
// Example of optimized queries
export class UserRepository {
  // Paginated query with lean for better performance
  async findWithPagination(page: number, limit: number) {
    const skip = (page - 1) * limit;
    return await User.find({ isActive: true })
      .select('name email role createdAt')
      .sort({ createdAt: -1 })
      .skip(skip)
      .limit(limit)
      .lean()
      .exec();
  }

  // Aggregation example
  async getUserStats() {
    return await User.aggregate([
      { $match: { isActive: true } },
      {
        $group: {
          _id: '$role',
          count: { $sum: 1 },
          lastCreated: { $max: '$createdAt' }
        }
      }
    ]);
  }
}
```

## Security Best Practices

### Authentication & Authorization
- Use JWT tokens for stateless authentication
- Implement role-based access control (RBAC)
- Validate all input parameters
- Use HTTPS in production

### Data Protection
- Hash passwords using bcrypt
- Sanitize user inputs
- Implement rate limiting
- Use environment variables for secrets

### CORS Configuration
```typescript
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true
}));
```

## Testing Standards

### Test Structure
- Unit tests for services and utilities
- Integration tests for GraphQL resolvers
- End-to-end tests using Apollo Server Testing
- Use test databases
- Achieve >80% code coverage

### GraphQL Resolver Testing
```typescript
// userResolver.test.ts
import { createTestClient } from 'apollo-server-testing';
import { ApolloServer } from 'apollo-server-express';
import { typeDefs } from '../graphql/schema';
import { userResolvers } from '../graphql/resolvers';

const server = new ApolloServer({
  typeDefs,
  resolvers: userResolvers,
  context: () => ({
    services: {
      userService: mockUserService,
    },
  }),
});

const { query, mutate } = createTestClient(server);

describe('User Resolvers', () => {
  describe('Query.user', () => {
    it('should return user when valid ID is provided', async () => {
      const GET_USER = gql`
        query GetUser($id: ID!) {
          user(id: $id) {
            id
            name
            email
          }
        }
      `;

      const response = await query({
        query: GET_USER,
        variables: { id: '1' },
      });

      expect(response.errors).toBeUndefined();
      expect(response.data?.user).toEqual({
        id: '1',
        name: 'John Doe',
        email: 'john@example.com',
      });
    });
  });
});
```

### Service Testing
```typescript
describe('UserService', () => {
  describe('getUserById', () => {
    it('should return user when valid ID is provided', async () => {
      // Test implementation
    });

    it('should throw error when user not found', async () => {
      // Test implementation
    });
  });
});
```

## Error Handling

### Custom Error Classes
```typescript
export class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
    this.name = 'ValidationError';
  }
}
```

### GraphQL Error Handling
```typescript
// Custom error classes for GraphQL
export class UserInputError extends Error {
  constructor(message: string, public code: string = 'USER_INPUT_ERROR') {
    super(message);
    this.name = 'UserInputError';
  }
}

export class AuthenticationError extends Error {
  constructor(message: string = 'Authentication required') {
    super(message);
    this.name = 'AuthenticationError';
  }
}

// Error formatter in Apollo Server
formatError: (error) => {
  // Log the original error
  console.error('GraphQL Error:', error);

  // Return formatted error to client
  if (error.originalError instanceof UserInputError) {
    return {
      message: error.message,
      code: 'USER_INPUT_ERROR',
      path: error.path,
    };
  }

  if (error.originalError instanceof AuthenticationError) {
    return {
      message: error.message,
      code: 'UNAUTHENTICATED',
      path: error.path,
    };
  }

  // Hide internal errors in production
  if (process.env.NODE_ENV === 'production') {
    return {
      message: 'Internal server error',
      code: 'INTERNAL_ERROR',
    };
  }

  return {
    message: error.message,
    code: error.extensions?.code || 'INTERNAL_ERROR',
    path: error.path,
  };
}
```

### Express Error Handler (for health and utility routes)
```typescript
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error('Express Error:', err);

  res.status(500).json({
    error: 'Internal server error',
    timestamp: new Date().toISOString(),
  });
});
```

## Environment Configuration

### Required Environment Variables
```bash
NODE_ENV=development|production|test
PORT=4000
MONGODB_URI=mongodb://username:password@localhost:27017/your-database-name
JWT_SECRET=your-secret-key
REDIS_URL=redis://localhost:6379
ALLOWED_ORIGINS=http://localhost:3000,https://app.example.com
GRAPHQL_INTROSPECTION=true|false
GRAPHQL_PLAYGROUND=true|false
```

### Configuration Management

#### Development Environment (.env file)
- Use a `.env` file for local development environment variables
- Load environment variables using `dotenv` package
- **NEVER commit `.env` files to version control** - add to `.gitignore`
- Provide a `.env.example` file with placeholder values

```bash
# .env (local development only - DO NOT COMMIT)
NODE_ENV=development
PORT=4000
MONGODB_URI=mongodb://username:password@localhost:27017/your-app-dev
JWT_SECRET=your-local-jwt-secret-min-32-chars
REDIS_URL=redis://localhost:6379
ALLOWED_ORIGINS=http://localhost:3000
GRAPHQL_INTROSPECTION=true
GRAPHQL_PLAYGROUND=true
```

```bash
# .env.example (committed to version control)
NODE_ENV=development
PORT=4000
MONGODB_URI=mongodb://username:password@localhost:27017/your-app-dev
JWT_SECRET=your-jwt-secret-here
REDIS_URL=redis://localhost:6379
ALLOWED_ORIGINS=http://localhost:3000
GRAPHQL_INTROSPECTION=true
GRAPHQL_PLAYGROUND=true
```

#### Environment Variable Loading
```typescript
// src/config/environment.ts
import dotenv from 'dotenv';

// Load environment variables from .env file in development
if (process.env.NODE_ENV !== 'production') {
  dotenv.config();
}

interface Config {
  nodeEnv: string;
  port: number;
  mongodbUri: string;
  jwtSecret: string;
  redisUrl: string;
  allowedOrigins: string[];
  graphql: {
    introspection: boolean;
    playground: boolean;
  };
}

export const config: Config = {
  nodeEnv: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '4000', 10),
  mongodbUri: process.env.MONGODB_URI || '',
  jwtSecret: process.env.JWT_SECRET || '',
  redisUrl: process.env.REDIS_URL || 'redis://localhost:6379',
  allowedOrigins: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  graphql: {
    introspection: process.env.GRAPHQL_INTROSPECTION === 'true',
    playground: process.env.GRAPHQL_PLAYGROUND === 'true',
  },
};

// Validate required environment variables
const requiredEnvVars = ['MONGODB_URI', 'JWT_SECRET'];
const missingEnvVars = requiredEnvVars.filter(envVar => !process.env[envVar]);

if (missingEnvVars.length > 0) {
  throw new Error(`Missing required environment variables: ${missingEnvVars.join(', ')}`);
}
```

#### .gitignore Requirements
```bash
# .gitignore
# Environment variables
.env
.env.local
.env.development
.env.test
.env.production

# Keep example file
!.env.example
```

#### Best Practices
- Use environment-specific config files for different deployment environments
- Validate required environment variables on startup
- Use type-safe configuration objects
- Provide sensible defaults for optional variables
- Document all environment variables in `.env.example`

## Documentation

### API Documentation
- GraphQL schema serves as self-documenting API
- Use GraphQL introspection for schema exploration
- Apollo Studio for production monitoring and analytics
- Document complex business logic in resolver functions
- Maintain schema descriptions and field documentation

```typescript
// Example of documented GraphQL schema
export const userTypeDefs = gql`
  """
  Represents a user in the system
  """
  type User {
    "Unique identifier for the user"
    id: ID!

    "Full name of the user"
    name: String!

    "Email address (must be unique)"
    email: String!

    "Account creation timestamp"
    createdAt: DateTime!

    "Last update timestamp"
    updatedAt: DateTime!
  }

  """
  Input for creating a new user
  """
  input CreateUserInput {
    "Full name (2-100 characters)"
    name: String!

    "Valid email address"
    email: String!
  }
`;
```

### Code Documentation
- Use JSDoc comments for public APIs
- Document complex business logic
- Include usage examples in README files