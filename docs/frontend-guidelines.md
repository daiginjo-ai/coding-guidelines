# Frontend Development Guidelines

This document outlines the standards and best practices for frontend development in the daiginjo-ai organization.

## Technology Stack

### Primary Technologies
- **Framework**: Next.js 14+ with TypeScript
- **State Management**: Zustand or Redux Toolkit
- **Styling**: Tailwind CSS + CSS Modules
- **Routing**: Next.js App Router
- **Build Tool**: Next.js (built-in)
- **Package Manager**: npm or yarn

### Development Tools
- **Testing**: Jest + React Testing Library
- **Linting**: ESLint + Prettier (Next.js built-in)
- **Type Checking**: TypeScript
- **Storybook**: Component documentation
- **Husky**: Git hooks for quality gates

## Project Structure

```
frontend/
├── src/
│   ├── app/                # Next.js App Router (pages and layouts)
│   │   ├── (auth)/         # Route groups
│   │   ├── api/            # API routes
│   │   ├── globals.css     # Global styles
│   │   ├── layout.tsx      # Root layout
│   │   ├── page.tsx        # Home page
│   │   └── loading.tsx     # Loading UI
│   ├── components/         # Reusable UI components
│   │   ├── ui/             # Basic UI primitives
│   │   ├── forms/          # Form-specific components
│   │   └── layout/         # Layout components
│   ├── hooks/              # Custom React hooks
│   ├── lib/                # Utility functions and configurations
│   ├── services/           # API clients and external services
│   ├── stores/             # State management
│   ├── types/              # TypeScript type definitions
│   └── constants/          # Application constants
├── public/                 # Static files (images, icons, etc.)
├── tests/                  # Test utilities and setup
├── .storybook/            # Storybook configuration
├── next.config.js         # Next.js configuration
├── tailwind.config.js     # Tailwind CSS configuration
└── tsconfig.json          # TypeScript configuration
```

## Coding Standards

### TypeScript Configuration
- Use strict mode: `"strict": true`
- Enable `exactOptionalPropertyTypes`
- Use explicit component prop types
- Prefer type imports: `import type { User } from './types'`

### Naming Conventions
- **Components**: PascalCase (`UserProfile`, `NavigationBar`)
- **Files**: Match component names (`UserProfile.tsx`, `NavigationBar.tsx`)
- **Hooks**: camelCase with 'use' prefix (`useAuth`, `useLocalStorage`)
- **Constants**: UPPER_SNAKE_CASE (`API_ENDPOINTS`, `THEME_COLORS`)
- **Variables/Functions**: camelCase (`isLoading`, `handleSubmit`)

### Component Guidelines

#### Functional Components
Always use functional components with TypeScript:

```typescript
// UserProfile.tsx
interface UserProfileProps {
  user: User;
  onEdit?: () => void;
  className?: string;
}

export const UserProfile: React.FC<UserProfileProps> = ({
  user,
  onEdit,
  className
}) => {
  return (
    <div className={cn('user-profile', className)}>
      <h2>{user.name}</h2>
      {onEdit && (
        <button onClick={onEdit}>Edit Profile</button>
      )}
    </div>
  );
};
```

#### Component Structure
1. Imports (external libraries first, then internal)
2. Type definitions
3. Component implementation
4. Default export (if applicable)

```typescript
import React, { useState, useCallback } from 'react';
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
import type { User } from '@/types/user';

interface Props {
  // prop types
}

export const ComponentName: React.FC<Props> = ({ ...props }) => {
  // hooks
  // state
  // computed values
  // event handlers
  // effects

  return (
    // JSX
  );
};
```

### State Management

#### Local State
Use `useState` for component-local state:

```typescript
const [isLoading, setIsLoading] = useState(false);
const [formData, setFormData] = useState<FormData>({
  name: '',
  email: ''
});
```

#### Global State (Zustand)
```typescript
// stores/authStore.ts
import { create } from 'zustand';

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  isAuthenticated: false,

  login: async (credentials) => {
    const user = await authService.login(credentials);
    set({ user, isAuthenticated: true });
  },

  logout: () => {
    set({ user: null, isAuthenticated: false });
  }
}));
```

### Custom Hooks

Create reusable logic with custom hooks:

```typescript
// hooks/useApi.ts
export const useApi = <T>(endpoint: string) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await apiClient.get<T>(endpoint);
      setData(response.data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  }, [endpoint]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
};
```

## Styling Guidelines

### Tailwind CSS
- Use Tailwind utility classes for styling
- Create custom components for repeated patterns
- Use CSS Modules for complex component-specific styles

```typescript
// Good: Utility classes
<button className="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded">
  Click me
</button>

// Better: Component abstraction
<Button variant="primary" size="md">
  Click me
</Button>
```

### CSS Modules
For complex component styling:

```css
/* Button.module.css */
.button {
  @apply inline-flex items-center justify-center rounded-md font-medium transition-colors;
}

.primary {
  @apply bg-blue-500 text-white hover:bg-blue-600;
}

.secondary {
  @apply bg-gray-200 text-gray-900 hover:bg-gray-300;
}
```

```typescript
// Button.tsx
import styles from './Button.module.css';

interface ButtonProps {
  variant: 'primary' | 'secondary';
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({ variant, children }) => {
  return (
    <button className={cn(styles.button, styles[variant])}>
      {children}
    </button>
  );
};
```

### Design System
- Create a consistent design system with reusable components
- Use design tokens for colors, spacing, typography
- Document components in Storybook

## API Integration

### API Client Setup
```typescript
// src/lib/apiClient.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10000,
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized access
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export { apiClient };
```

### API Service Layer
```typescript
// services/userService.ts
export const userService = {
  async getUsers(): Promise<User[]> {
    const response = await apiClient.get<ApiResponse<User[]>>('/users');
    return response.data.data;
  },

  async createUser(userData: CreateUserRequest): Promise<User> {
    const response = await apiClient.post<ApiResponse<User>>('/users', userData);
    return response.data.data;
  },

  async updateUser(id: string, userData: UpdateUserRequest): Promise<User> {
    const response = await apiClient.put<ApiResponse<User>>(`/users/${id}`, userData);
    return response.data.data;
  }
};
```

## Form Handling

### React Hook Form
Use React Hook Form for form management:

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email address'),
  age: z.number().min(18, 'Must be 18 or older')
});

type UserFormData = z.infer<typeof userSchema>;

export const UserForm: React.FC = () => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting }
  } = useForm<UserFormData>({
    resolver: zodResolver(userSchema)
  });

  const onSubmit = async (data: UserFormData) => {
    await userService.createUser(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('name')}
        placeholder="Name"
        className="border rounded px-3 py-2"
      />
      {errors.name && <span className="text-red-500">{errors.name.message}</span>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
};
```

## Testing Standards

### Component Testing
```typescript
// UserProfile.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfile } from './UserProfile';

const mockUser = {
  id: '1',
  name: 'John Doe',
  email: 'john@example.com'
};

describe('UserProfile', () => {
  it('renders user information correctly', () => {
    render(<UserProfile user={mockUser} />);

    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('calls onEdit when edit button is clicked', async () => {
    const onEdit = vi.fn();
    render(<UserProfile user={mockUser} onEdit={onEdit} />);

    await userEvent.click(screen.getByText('Edit Profile'));
    expect(onEdit).toHaveBeenCalledOnce();
  });
});
```

### Custom Hook Testing
```typescript
// useAuth.test.ts
import { renderHook, act } from '@testing-library/react';
import { useAuth } from './useAuth';

describe('useAuth', () => {
  it('should login user successfully', async () => {
    const { result } = renderHook(() => useAuth());

    await act(async () => {
      await result.current.login({ email: 'test@example.com', password: 'password' });
    });

    expect(result.current.isAuthenticated).toBe(true);
    expect(result.current.user).toBeDefined();
  });
});
```

## Performance Guidelines

### Next.js App Router Structure
```typescript
// src/app/layout.tsx - Root layout
import './globals.css';
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My App',
  description: 'Generated by create-next-app',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <header>Navigation</header>
        <main>{children}</main>
        <footer>Footer</footer>
      </body>
    </html>
  );
}

// src/app/page.tsx - Home page
export default function HomePage() {
  return (
    <div>
      <h1>Welcome to My App</h1>
    </div>
  );
}

// src/app/users/page.tsx - Users page
export default function UsersPage() {
  return (
    <div>
      <h1>Users</h1>
    </div>
  );
}

// src/app/loading.tsx - Loading UI
export default function Loading() {
  return <div>Loading...</div>;
}
```

### Route Groups and Dynamic Routes
```typescript
// src/app/(auth)/login/page.tsx - Route group for auth pages
export default function LoginPage() {
  return <div>Login Page</div>;
}

// src/app/users/[id]/page.tsx - Dynamic route
interface UserPageProps {
  params: { id: string };
}

export default function UserPage({ params }: UserPageProps) {
  return <div>User ID: {params.id}</div>;
}

// src/app/api/users/route.ts - API route
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const users = await getUsersFromDB();
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await createUser(body);
  return NextResponse.json(user, { status: 201 });
}
```

### Memoization
```typescript
// Memoize expensive calculations
const expensiveValue = useMemo(() => {
  return processLargeDataset(data);
}, [data]);

// Memoize callbacks
const handleClick = useCallback(() => {
  onItemClick(item.id);
}, [item.id, onItemClick]);

// Memoize components
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{processData(data)}</div>;
});
```

## Accessibility Guidelines

### Semantic HTML
- Use semantic HTML elements (`<main>`, `<nav>`, `<section>`)
- Provide alt text for images
- Use proper heading hierarchy (h1, h2, h3...)

### ARIA Attributes
```typescript
<button
  aria-label="Close dialog"
  aria-expanded={isOpen}
  onClick={onClose}
>
  ×
</button>

<div role="alert" aria-live="polite">
  {errorMessage}
</div>
```

### Keyboard Navigation
- Ensure all interactive elements are keyboard accessible
- Implement proper focus management
- Use `tabIndex` appropriately

## Error Handling

### Error Boundaries
```typescript
// ErrorBoundary.tsx
export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(): { hasError: boolean } {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('Error caught by boundary:', error, errorInfo);
  }

  render(): React.ReactNode {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong.</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Environment Configuration

### Environment Variables
```bash
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
NEXT_PUBLIC_ENVIRONMENT=development
NEXT_PUBLIC_SENTRY_DSN=your-sentry-dsn
DATABASE_URL=your-database-url
JWT_SECRET=your-jwt-secret
```

### Configuration Management
```typescript
// src/lib/config.ts
export const config = {
  // Client-side environment variables (NEXT_PUBLIC_ prefix)
  apiUrl: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001/api/v1',
  environment: process.env.NEXT_PUBLIC_ENVIRONMENT || 'development',

  // Server-side environment variables (no prefix needed)
  databaseUrl: process.env.DATABASE_URL,
  jwtSecret: process.env.JWT_SECRET,

  // Common
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production'
};

// Validate required environment variables
if (!config.databaseUrl && typeof window === 'undefined') {
  throw new Error('DATABASE_URL is required');
}
```

### Next.js Configuration
```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
  images: {
    domains: ['example.com'],
  },
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
};

module.exports = nextConfig;
```

## Next.js Specific Features

### Server Components vs Client Components
```typescript
// Server Component (default in app directory)
// src/app/users/page.tsx
import { getUsers } from '@/lib/api';

export default async function UsersPage() {
  const users = await getUsers(); // This runs on the server

  return (
    <div>
      <h1>Users</h1>
      {users.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
}

// Client Component (add 'use client' directive)
// src/components/Counter.tsx
'use client';

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

### Data Fetching Patterns
```typescript
// Server-side data fetching
async function getUser(id: string) {
  const res = await fetch(`${process.env.API_URL}/users/${id}`, {
    next: { revalidate: 3600 } // Revalidate every hour
  });

  if (!res.ok) {
    throw new Error('Failed to fetch user');
  }

  return res.json();
}

// Static generation with params
// src/app/users/[id]/page.tsx
export async function generateStaticParams() {
  const users = await getUsers();

  return users.map((user) => ({
    id: user.id,
  }));
}

export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await getUser(params.id);

  return <div>User: {user.name}</div>;
}
```

### Middleware
```typescript
// middleware.ts (root level)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Authentication check
  const token = request.cookies.get('token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*']
};
```

### Image Optimization
```typescript
// Use Next.js Image component for optimized images
import Image from 'next/image';

export default function ProfileImage({ user }: { user: User }) {
  return (
    <Image
      src={user.avatarUrl}
      alt={`${user.name}'s avatar`}
      width={100}
      height={100}
      className="rounded-full"
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,..."
    />
  );
}
```

### Internationalization (i18n)
```typescript
// next.config.js
const nextConfig = {
  i18n: {
    locales: ['en', 'fr', 'es'],
    defaultLocale: 'en',
  },
};

// Usage in components
import { useRouter } from 'next/router';

export default function LocaleSwitcher() {
  const router = useRouter();

  const changeLocale = (locale: string) => {
    router.push(router.asPath, router.asPath, { locale });
  };

  return (
    <select onChange={(e) => changeLocale(e.target.value)}>
      <option value="en">English</option>
      <option value="fr">Français</option>
      <option value="es">Español</option>
    </select>
  );
}
```

### Performance Optimization
```typescript
// Dynamic imports for code splitting
import dynamic from 'next/dynamic';

const DynamicComponent = dynamic(() => import('../components/HeavyComponent'), {
  loading: () => <p>Loading...</p>,
  ssr: false, // Disable server-side rendering for this component
});

// Bundle analyzer
const nextConfig = {
  experimental: {
    bundleAnalyzer: {
      enabled: process.env.ANALYZE === 'true',
    },
  },
};
```

## Documentation

### Component Documentation
- Document all public props and their types
- Include usage examples
- Document any complex behavior or edge cases
- Use Storybook for interactive component documentation

### README Requirements
- Setup instructions
- Available scripts
- Environment variables
- Deployment instructions
- Contributing guidelines