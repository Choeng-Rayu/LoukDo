---
inclusion: always
---

# Unified Architecture Guidelines

## MANDATORY Architecture Principles

**CRITICAL**: This project uses a unified architecture approach. When working with specifications or code:

- **Single Backend Application**: One NestJS app serving all user types (customer, seller, super_admin)
- **Single Frontend Application**: One Next.js app with role-based interfaces
- **Role-Based Access Control**: JWT tokens with role claims, not separate authentication systems
- **Shared Database**: Single MySQL schema with role-based permissions, not separate databases

## Technology Stack

### Backend: NestJS
- **Framework**: NestJS with TypeScript
- **Database**: MySQL with TypeORM
- **Authentication**: JWT tokens with Passport.js
- **Development**: Docker container for MySQL database
- **API Documentation**: Swagger/OpenAPI

### Frontend: Next.js
- **Framework**: Next.js 14+ with TypeScript
- **Styling**: Tailwind CSS or styled-components
- **State Management**: Zustand or Redux Toolkit
- **Authentication**: NextAuth.js or custom JWT handling
- **UI Components**: Headless UI or shadcn/ui

### Database: MySQL with Docker
- **Database**: MySQL 8.0+
- **ORM**: TypeORM with decorators
- **Development**: Docker Compose for local development
- **Migrations**: TypeORM migrations
- **Connection**: Connection pooling with retry logic

## Code Organization Standards

### Backend Structure (NestJS)
```
src/
├── auth/           # JWT authentication & Passport strategies
├── users/          # User management with role enum
├── products/       # Product domain with role-based access
├── orders/         # Order domain with role-based access
├── common/         # Shared DTOs, decorators, guards
├── guards/         # Role-based authorization guards
├── database/       # TypeORM entities and migrations
└── config/         # Configuration modules
```

### Frontend Structure (Next.js)
```
src/
├── app/            # Next.js 14 app router
│   ├── (customer)/ # Customer routes (public)
│   ├── seller/     # Seller dashboard routes
│   └── admin/      # Admin panel routes
├── components/
│   ├── ui/         # Reusable UI components
│   ├── customer/   # Customer-specific components
│   ├── seller/     # Seller-specific components
│   └── admin/      # Admin-specific components
├── lib/            # Utilities and configurations
├── hooks/          # Custom React hooks
└── store/          # State management
```

### Docker Development Setup
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ecommerce
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
```

## API Design Patterns

### NestJS Endpoint Structure
- Use domain-based controllers: `ProductsController`, `OrdersController`, `UsersController`
- Apply role-based filtering in services, not separate endpoints
- Use NestJS guards for role validation

### Authentication Flow (NestJS + JWT)
```typescript
// JWT payload structure
interface JWTPayload {
  sub: string;        // user ID
  role: 'customer' | 'seller' | 'super_admin';
  permissions: string[];
  iat: number;
  exp: number;
}

// NestJS Guard Implementation
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    // Role validation logic
  }
}
```

### Role-Based Controllers (NestJS)
```typescript
@Controller('api/products')
@UseGuards(JwtAuthGuard, RolesGuard)
export class ProductsController {
  @Get()
  @Roles('customer', 'seller', 'super_admin')
  async getProducts(@Request() req) {
    // Role-based filtering in service layer
    return this.productsService.findByRole(req.user.role);
  }
}
```

## Database Schema Rules (MySQL + TypeORM)

### User Management Entity
```typescript
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column({
    type: 'enum',
    enum: ['customer', 'seller', 'super_admin'],
    default: 'customer'
  })
  role: UserRole;

  @OneToMany(() => UserPermission, permission => permission.user)
  permissions: UserPermission[];
}
```

### Domain Tables
- Products, orders, reviews use TypeORM entities
- Foreign keys reference `users.id` with proper relations
- Role-based filtering in repository methods, not separate tables
- Use MySQL-specific features like JSON columns where appropriate

### Docker Database Configuration
```typescript
// TypeORM configuration for Docker MySQL
export const databaseConfig: TypeOrmModuleOptions = {
  type: 'mysql',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT) || 3306,
  username: process.env.DB_USERNAME || 'root',
  password: process.env.DB_PASSWORD || 'root',
  database: process.env.DB_NAME || 'ecommerce',
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: process.env.NODE_ENV === 'development',
  migrations: [__dirname + '/migrations/*{.ts,.js}'],
};
```

## Frontend Patterns (Next.js)

### Role-Based Routing (App Router)
```typescript
// Next.js 14 App Router structure
src/app/
├── (customer)/          # Customer routes (public)
│   ├── page.tsx         # Home page
│   ├── products/        # Product catalog
│   └── cart/            # Shopping cart
├── seller/              # Seller dashboard (protected)
│   ├── layout.tsx       # Seller layout with auth
│   ├── dashboard/       # Seller dashboard
│   └── products/        # Product management
└── admin/               # Admin panel (protected)
    ├── layout.tsx       # Admin layout with auth
    ├── dashboard/       # Admin dashboard
    └── users/           # User management
```

### Next.js Component Design
```typescript
// Role-based component with Next.js
interface RoleBasedComponentProps {
  userRole: 'customer' | 'seller' | 'super_admin';
  children: React.ReactNode;
}

export function RoleBasedComponent({ userRole, children }: RoleBasedComponentProps) {
  return (
    <div className={`interface-${userRole}`}>
      {children}
    </div>
  );
}
```

### State Management (Zustand)
```typescript
interface AppState {
  auth: {
    user: User | null;
    role: UserRole | null;
    permissions: string[];
    isAuthenticated: boolean;
  };
  // Domain state shared across roles
  products: ProductState;
  orders: OrderState;
  // Actions
  login: (user: User) => void;
  logout: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  auth: {
    user: null,
    role: null,
    permissions: [],
    isAuthenticated: false,
  },
  // ... rest of state
}));
```

### Next.js Authentication Middleware
```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token');
  
  if (request.nextUrl.pathname.startsWith('/seller')) {
    // Verify seller role
    if (!token || !hasRole(token, 'seller')) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }
  
  if (request.nextUrl.pathname.startsWith('/admin')) {
    // Verify admin role
    if (!token || !hasRole(token, 'super_admin')) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }
}
```

## Specification Writing Rules

### When Creating New Specs
1. **Always specify unified implementation**: Never create separate backend/frontend specs for different roles
2. **Define role-based access patterns**: Specify what each role can access for every feature
3. **Use permission matrices**: Document role capabilities in tabular format
4. **Plan shared components**: Identify reusable elements across user interfaces

### When Updating Existing Specs
1. **Consolidate duplicate requirements**: Merge similar features across role-specific specs
2. **Maintain role distinctions**: Preserve role-specific behaviors within unified features
3. **Update task dependencies**: Ensure implementation tasks reflect unified architecture

## Implementation Guidelines

### Security Patterns
- Implement role validation at the controller level
- Use decorators for role-based access control
- Apply principle of least privilege for each role

### Performance Considerations
- Use role-based data filtering in database queries
- Implement caching strategies that respect role permissions
- Design APIs to minimize role-specific data exposure

### Testing Strategy
- Write tests that cover all role scenarios for each feature
- Use property-based testing to validate role-based access controls
- Test role transitions and permission inheritance

## Common Anti-Patterns to Avoid

### Backend Anti-Patterns
- ❌ Separate controllers for each role (`CustomerController`, `SellerController`)
- ❌ Role-specific endpoints (`/customer-api/*`, `/seller-api/*`)
- ❌ Duplicate business logic across role-specific services

### Frontend Anti-Patterns
- ❌ Separate applications for each role
- ❌ Duplicate components with minor role differences
- ❌ Role-specific state management stores

### Database Anti-Patterns
- ❌ Separate tables for role-specific data (`customer_orders`, `seller_orders`)
- ❌ Role-specific user tables (`customers`, `sellers`, `admins`)
- ❌ Duplicate schemas across role boundaries