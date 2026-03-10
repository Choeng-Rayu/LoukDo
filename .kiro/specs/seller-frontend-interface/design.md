# Design Document: Seller Frontend Interface

## Introduction

The Seller Frontend Interface is part of a unified Next.js application that provides role-based interfaces for all user types (customer, seller, super_admin). This design document outlines the seller-specific interface patterns, routing structure, and components within the unified architecture that delivers a responsive, secure, and intuitive seller experience.

## Unified Architecture Overview

## Unified Architecture Overview

### Next.js 14+ Unified Application Architecture

The seller interface is implemented within a unified Next.js 14+ application using the App Router architecture. The application dynamically renders different interfaces based on authenticated user roles through middleware and role-based routing:

```mermaid
graph TB
    subgraph "Next.js 14+ Unified Application"
        A[App Router] --> B[Middleware Layer]
        B --> C[Role Detection]
        C --> D["(customer) Routes"]
        C --> E["/seller Routes"]
        C --> F["/admin Routes"]
        
        E --> G[Seller Pages]
        E --> H[Shared Components]
        
        G --> I[Dashboard]
        G --> J[Product Management]
        G --> K[Order Management]
        G --> L[Analytics]
        
        H --> M[Authentication]
        H --> N[Navigation]
        H --> O[Forms]
        H --> P[UI Elements]
    end
    
    Q[Unified Backend API] --> A
    R[CDN/Static Assets] --> A
    S[Zustand Store] --> A
```

### Next.js App Router Structure for Sellers

The seller interface follows Next.js 14+ App Router conventions with protected seller routes:

```mermaid
graph LR
    subgraph "App Router Structure"
        A[app/] --> B["(customer)/"]
        A --> C[seller/]
        A --> D[admin/]
        
        C --> E[layout.tsx - Seller Layout]
        C --> F[page.tsx - Dashboard]
        C --> G[products/]
        C --> H[orders/]
        C --> I[analytics/]
        C --> J[profile/]
        
        G --> K[page.tsx - Product List]
        G --> L[new/page.tsx - Add Product]
        G --> M["[id]/page.tsx - Edit Product"]
        
        H --> N[page.tsx - Order List]
        H --> O["[id]/page.tsx - Order Detail"]
    end
    
    subgraph "Shared Components"
        P[components/ui/] --> Q[Button, Input, Modal]
        P --> R[components/seller/]
        P --> S[components/shared/]
        
        R --> T[ProductForm, OrderCard]
        S --> U[Header, Footer, Navigation]
    end
    
    A --> P
```

The seller interface is integrated within a single Next.js application with role-based routing and shared components:

```mermaid
graph TD
    A[Next.js App] --> B[App Router]
    B --> C[Customer Routes /]
    B --> D[Seller Routes /seller/*]
    B --> E[Admin Routes /admin/*]
    
    D --> F[Seller Dashboard]
    D --> G[Product Management]
    D --> H[Order Management]
    D --> I[Analytics]
    
    F --> J[Shared Components]
    G --> J
    H --> J
    I --> J
    
    J --> K[Unified State Store]
```

### Technology Stack

#### Core Framework & Runtime
- **Next.js 14+**: Full-stack React framework with App Router, Server Components, and Server Actions
- **React 18+**: Component-based UI with concurrent features and streaming SSR
- **TypeScript**: Type-safe development with enhanced IDE support and compile-time validation
- **Node.js**: Server-side runtime for API routes and middleware

#### State Management & Data Fetching
- **Zustand**: Lightweight global state management for user session and seller-specific state
- **TanStack Query (React Query)**: Server state management, caching, and synchronization
- **React Hook Form**: Form state management with validation and performance optimization
- **SWR**: Alternative data fetching with built-in caching (where TanStack Query isn't used)

#### Styling & UI Components
- **Tailwind CSS**: Utility-first CSS framework for rapid development and consistent design
- **shadcn/ui**: High-quality, accessible React components built on Radix UI primitives
- **Headless UI**: Unstyled, accessible UI components for custom implementations
- **Framer Motion**: Animation library for smooth interactions and page transitions
- **Lucide React**: Modern icon library with consistent design

#### Authentication & Security
- **NextAuth.js**: Complete authentication solution with JWT and session management
- **Custom JWT Handling**: Alternative JWT implementation with role-based claims
- **Next.js Middleware**: Route protection and role-based access control
- **bcrypt**: Password hashing for secure authentication

#### Charts & Analytics
- **Recharts**: Composable charting library for React
- **Chart.js**: Alternative charting solution with React wrapper
- **Date-fns**: Date manipulation and formatting utilities

## Components and Interfaces

### Next.js App Router Pages for Sellers

#### 1. Seller Route Group - `app/seller/`
```typescript
// app/seller/layout.tsx
interface SellerLayoutProps {
  children: React.ReactNode
}

export default function SellerLayout({ children }: SellerLayoutProps) {
  return (
    <div className="min-h-screen bg-gray-50">
      <SellerHeader />
      <div className="flex">
        <SellerSidebar />
        <main className="flex-1 p-6">
          {children}
        </main>
      </div>
    </div>
  )
}

// app/seller/page.tsx - Dashboard
export default function SellerDashboard() {
  return (
    <>
      <DashboardMetrics />
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mt-6">
        <RecentOrders />
        <TopProducts />
      </div>
    </>
  )
}

// app/seller/products/page.tsx - Product Management
interface ProductManagementPageProps {
  searchParams: {
    category?: string
    status?: string
    search?: string
    page?: string
  }
}

export default function ProductManagementPage({ searchParams }: ProductManagementPageProps) {
  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold">Products</h1>
        <Button asChild>
          <Link href="/seller/products/new">Add Product</Link>
        </Button>
      </div>
      <ProductFilters />
      <ProductTable searchParams={searchParams} />
    </div>
  )
}
```

#### 2. Seller Authentication with NextAuth.js
```typescript
// components/seller/auth/SellerLoginForm.tsx
interface SellerLoginFormProps {
  callbackUrl?: string
  onSuccess?: (user: Seller) => void
}

interface SellerLoginFormData {
  email: string
  password: string
  rememberMe: boolean
}

export function SellerLoginForm({ callbackUrl, onSuccess }: SellerLoginFormProps) {
  const { register, handleSubmit, formState: { errors } } = useForm<SellerLoginFormData>()
  const { data: session, status } = useSession()
  const router = useRouter()
  
  const onSubmit = async (data: SellerLoginFormData) => {
    const result = await signIn('credentials', {
      email: data.email,
      password: data.password,
      redirect: false,
    })
    
    if (result?.ok && session?.user?.role === 'seller') {
      onSuccess?.(session.user as Seller)
      router.push(callbackUrl || '/seller/dashboard')
    }
  }
  
  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      <Input
        {...register('email', { required: 'Email is required' })}
        type="email"
        placeholder="Business email address"
        error={errors.email?.message}
      />
      <Input
        {...register('password', { required: 'Password is required' })}
        type="password"
        placeholder="Password"
        error={errors.password?.message}
      />
      <Button type="submit" className="w-full">
        Sign In to Dashboard
      </Button>
    </form>
  )
}
```

#### 3. Product Management Components with Server Components
```typescript
// components/seller/products/ProductForm.tsx
interface ProductFormProps {
  product?: Product
  onSubmit: (data: ProductFormData) => Promise<void>
}

interface ProductFormData {
  name: string
  description: string
  price: number
  categoryId: string
  stockQuantity: number
  images: File[]
}

export function ProductForm({ product, onSubmit }: ProductFormProps) {
  const { register, handleSubmit, watch, formState: { errors } } = useForm<ProductFormData>({
    defaultValues: product ? {
      name: product.name,
      description: product.description,
      price: product.price,
      categoryId: product.categoryId,
      stockQuantity: product.stockQuantity,
    } : undefined
  })
  
  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-8">
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
        <div className="space-y-6">
          <Input
            {...register('name', { required: 'Product name is required' })}
            label="Product Name"
            error={errors.name?.message}
          />
          
          <Textarea
            {...register('description', { required: 'Description is required' })}
            label="Description"
            rows={4}
            error={errors.description?.message}
          />
          
          <div className="grid grid-cols-2 gap-4">
            <Input
              {...register('price', { 
                required: 'Price is required',
                min: { value: 0.01, message: 'Price must be greater than 0' }
              })}
              type="number"
              step="0.01"
              label="Price"
              error={errors.price?.message}
            />
            
            <Input
              {...register('stockQuantity', { 
                required: 'Stock quantity is required',
                min: { value: 0, message: 'Stock cannot be negative' }
              })}
              type="number"
              label="Stock Quantity"
              error={errors.stockQuantity?.message}
            />
          </div>
        </div>
        
        <div className="space-y-6">
          <CategorySelect
            {...register('categoryId', { required: 'Category is required' })}
            error={errors.categoryId?.message}
          />
          
          <ImageUpload
            {...register('images')}
            existingImages={product?.images}
            maxFiles={5}
          />
        </div>
      </div>
      
      <div className="flex justify-end space-x-4">
        <Button type="button" variant="outline" asChild>
          <Link href="/seller/products">Cancel</Link>
        </Button>
        <Button type="submit">
          {product ? 'Update Product' : 'Create Product'}
        </Button>
      </div>
    </form>
  )
}

// app/seller/products/[id]/page.tsx - Server Component
export default async function EditProductPage({ params }: { params: { id: string } }) {
  const product = await getSellerProduct(params.id) // Server-side data fetching
  
  if (!product) {
    notFound()
  }
  
  return (
    <div className="max-w-4xl mx-auto">
      <div className="mb-6">
        <h1 className="text-2xl font-bold">Edit Product</h1>
        <p className="text-gray-600">Update your product information</p>
      </div>
      <ProductForm product={product} onSubmit={updateProduct} />
    </div>
  )
}
```

#### 4. Order Management with Zustand State Management
```typescript
// store/sellerStore.ts
interface SellerState {
  orders: Order[]
  products: Product[]
  analytics: SellerAnalytics | null
  isLoading: boolean
  error: string | null
}

interface SellerActions {
  fetchOrders: () => Promise<void>
  updateOrderStatus: (orderId: string, status: OrderStatus) => Promise<void>
  fetchProducts: () => Promise<void>
  createProduct: (productData: ProductFormData) => Promise<void>
  updateProduct: (id: string, productData: ProductFormData) => Promise<void>
  deleteProduct: (id: string) => Promise<void>
  fetchAnalytics: (dateRange: DateRange) => Promise<void>
}

export const useSellerStore = create<SellerState & SellerActions>((set, get) => ({
  orders: [],
  products: [],
  analytics: null,
  isLoading: false,
  error: null,
  
  fetchOrders: async () => {
    set({ isLoading: true, error: null })
    try {
      const orders = await sellerAPI.getOrders()
      set({ orders, isLoading: false })
    } catch (error) {
      set({ error: error.message, isLoading: false })
    }
  },
  
  updateOrderStatus: async (orderId: string, status: OrderStatus) => {
    try {
      const updatedOrder = await sellerAPI.updateOrderStatus(orderId, status)
      set((state) => ({
        orders: state.orders.map(order => 
          order.id === orderId ? updatedOrder : order
        )
      }))
      toast.success('Order status updated successfully')
    } catch (error) {
      toast.error('Failed to update order status')
    }
  },
  
  createProduct: async (productData: ProductFormData) => {
    set({ isLoading: true })
    try {
      const newProduct = await sellerAPI.createProduct(productData)
      set((state) => ({
        products: [...state.products, newProduct],
        isLoading: false
      }))
      toast.success('Product created successfully')
    } catch (error) {
      set({ error: error.message, isLoading: false })
      toast.error('Failed to create product')
    }
  },
}))

// components/seller/orders/OrderCard.tsx
export function OrderCard({ order }: { order: Order }) {
  const { updateOrderStatus } = useSellerStore()
  
  return (
    <div className="bg-white rounded-lg shadow-sm border p-6">
      <div className="flex justify-between items-start mb-4">
        <div>
          <h3 className="font-semibold text-lg">Order #{order.orderNumber}</h3>
          <p className="text-gray-600">{order.customer.name}</p>
          <p className="text-sm text-gray-500">{formatDate(order.createdAt)}</p>
        </div>
        <div className="text-right">
          <p className="font-semibold text-lg">${order.total}</p>
          <OrderStatusBadge status={order.status} />
        </div>
      </div>
      
      <div className="space-y-2 mb-4">
        {order.items.map((item) => (
          <div key={item.id} className="flex justify-between text-sm">
            <span>{item.product.name} × {item.quantity}</span>
            <span>${item.totalPrice}</span>
          </div>
        ))}
      </div>
      
      <div className="flex justify-between items-center">
        <Button variant="outline" size="sm" asChild>
          <Link href={`/seller/orders/${order.id}`}>View Details</Link>
        </Button>
        
        <OrderStatusSelect
          currentStatus={order.status}
          onStatusChange={(status) => updateOrderStatus(order.id, status)}
        />
      </div>
    </div>
  )
}
```

## User Interface Design

### Design System

#### Color Palette
- Primary: #2563EB (Blue 600) - Main actions and navigation
- Secondary: #10B981 (Emerald 500) - Success states and positive metrics
- Accent: #F59E0B (Amber 500) - Warnings and pending states
- Danger: #EF4444 (Red 500) - Errors and destructive actions
- Neutral: #6B7280 (Gray 500) - Text and borders
- Background: #F9FAFB (Gray 50) - Page backgrounds

#### Typography
- Headings: Inter, system-ui, sans-serif
- Body: Inter, system-ui, sans-serif
- Code: 'Fira Code', monospace

#### Spacing Scale
- Base unit: 4px
- Scale: 4px, 8px, 12px, 16px, 24px, 32px, 48px, 64px

### Layout Structure

#### Navigation Layout
```
┌─────────────────────────────────────────────────────────────┐
│                    Top Navigation Bar                        │
│  Logo    Dashboard  Products  Orders  Reports    Profile ▼  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    Main Content Area                        │
│                                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Dashboard Layout
```
┌─────────────────────────────────────────────────────────────┐
│                    Metrics Cards Row                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────┐ │
│  │   Total     │ │   Active    │ │   Monthly   │ │  Low    │ │
│  │  Products   │ │   Orders    │ │   Sales     │ │ Stock   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────┘ │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────┐ ┌─────────────────────────┐ │
│ │      Recent Orders          │ │    Top Products         │ │
│ │                             │ │                         │ │
│ │                             │ │                         │ │
│ └─────────────────────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Technical Design

### Frontend Technology Stack

#### Core Framework
- **Next.js 14+**: React framework with App Router for role-based routing
- **TypeScript**: Type-safe JavaScript for better development experience
- **React 18+**: Component-based UI library with concurrent features

#### State Management
- **Zustand**: Lightweight state management for unified application state
- **TanStack Query (React Query)**: Server state management and caching
- **React Hook Form**: Form state management with validation

#### Styling and UI
- **Tailwind CSS**: Utility-first CSS framework for rapid UI development
- **Headless UI** or **shadcn/ui**: Unstyled, accessible UI components
- **Lucide React**: Modern icon library

#### Authentication and Routing
- **NextAuth.js** or **Custom JWT**: Authentication with role-based access control
- **Next.js App Router**: File-based routing with role-specific route groups
- **Next.js Middleware**: Route protection and role validation

### Unified Architecture Implementation

#### Next.js App Router Structure
```
src/app/
├── (customer)/              # Customer routes (public)
│   ├── page.tsx            # Home page
│   ├── products/           # Product catalog
│   └── cart/               # Shopping cart
├── seller/                 # Seller routes (protected)
│   ├── layout.tsx          # Seller-specific layout
│   ├── page.tsx            # Seller dashboard
│   ├── products/           # Product management
│   │   ├── page.tsx        # Product list
│   │   ├── new/            # Add product
│   │   └── [id]/           # Edit product
│   ├── orders/             # Order management
│   │   ├── page.tsx        # Order list
│   │   └── [id]/           # Order details
│   ├── analytics/          # Sales analytics
│   └── profile/            # Seller profile
└── admin/                  # Admin routes (protected)
    └── ...
```

#### Role-Based Layout System
```typescript
// src/app/seller/layout.tsx
import { SellerNavigation } from '@/components/seller/SellerNavigation'
import { RoleGuard } from '@/components/auth/RoleGuard'

export default function SellerLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <RoleGuard allowedRoles={['seller']}>
      <div className="min-h-screen bg-gray-50">
        <SellerNavigation />
        <main className="lg:pl-64">
          <div className="px-4 sm:px-6 lg:px-8 py-8">
            {children}
          </div>
        </main>
      </div>
    </RoleGuard>
  )
}
```

#### Unified State Management Structure
```typescript
// src/store/useAppStore.ts
interface AppState {
  // Authentication state (shared across all roles)
  auth: {
    user: User | null
    role: 'customer' | 'seller' | 'super_admin' | null
    permissions: string[]
    isAuthenticated: boolean
    isLoading: boolean
  }
  
  // Seller-specific state
  seller: {
    products: {
      items: Product[]
      isLoading: boolean
      error: string | null
    }
    orders: {
      items: Order[]
      isLoading: boolean
      error: string | null
    }
    analytics: {
      metrics: SellerMetrics
      isLoading: boolean
    }
    profile: {
      businessInfo: BusinessInfo | null
      isLoading: boolean
    }
  }
  
  // Shared state (used by multiple roles)
  shared: {
    categories: Category[]
    notifications: Notification[]
  }
  
  // Actions
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => void
  updateSellerProducts: (products: Product[]) => void
  updateSellerOrders: (orders: Order[]) => void
}

export const useAppStore = create<AppState>((set, get) => ({
  auth: {
    user: null,
    role: null,
    permissions: [],
    isAuthenticated: false,
    isLoading: false,
  },
  seller: {
    products: { items: [], isLoading: false, error: null },
    orders: { items: [], isLoading: false, error: null },
    analytics: { metrics: null, isLoading: false },
    profile: { businessInfo: null, isLoading: false },
  },
  shared: {
    categories: [],
    notifications: [],
  },
  // Actions implementation...
}))
```

### Authentication System Design

#### Next.js Middleware for Role Protection
```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { verifyJWT } from '@/lib/auth'

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value
  
  // Protect seller routes
  if (request.nextUrl.pathname.startsWith('/seller')) {
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url))
    }
    
    try {
      const payload = await verifyJWT(token)
      if (payload.role !== 'seller') {
        return NextResponse.redirect(new URL('/unauthorized', request.url))
      }
    } catch (error) {
      return NextResponse.redirect(new URL('/login', request.url))
    }
  }
  
  return NextResponse.next()
}

export const config = {
  matcher: ['/seller/:path*', '/admin/:path*']
}
```

#### Authentication Flow with NextAuth.js
```typescript
// src/lib/auth.ts
import NextAuth from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'

export const authOptions = {
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' }
      },
      async authorize(credentials) {
        const response = await fetch(`${process.env.API_URL}/auth/login`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(credentials),
        })
        
        if (response.ok) {
          const user = await response.json()
          return {
            id: user.id,
            email: user.email,
            role: user.role,
            permissions: user.permissions,
          }
        }
        return null
      }
    })
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role
        token.permissions = user.permissions
      }
      return token
    },
    async session({ session, token }) {
      session.user.role = token.role
      session.user.permissions = token.permissions
      return session
    }
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  }
}

export default NextAuth(authOptions)
```

### Data Management Design

#### API Integration Pattern
```typescript
// API service layer
class SellerAPI {
  async getProducts(sellerId: string): Promise<Product[]>
  async createProduct(product: CreateProductRequest): Promise<Product>
  async updateProduct(id: string, updates: UpdateProductRequest): Promise<Product>
  async deleteProduct(id: string): Promise<void>
  
  async getOrders(sellerId: string): Promise<Order[]>
  async updateOrderStatus(orderId: string, status: OrderStatus): Promise<Order>
  
  async getSalesReport(sellerId: string, dateRange: DateRange): Promise<SalesReport>
}
```

#### State Management Structure
```typescript
// Global state structure
interface SellerState {
  auth: {
    user: Seller | null
    isAuthenticated: boolean
    isLoading: boolean
  }
  products: {
    items: Product[]
    isLoading: boolean
    error: string | null
  }
  orders: {
    items: Order[]
    isLoading: boolean
    error: string | null
  }
  dashboard: {
    metrics: DashboardMetrics
    isLoading: boolean
  }
}
```

## Page Designs

### 1. Authentication Pages

#### Login Page
- Clean, centered form design
- Email and password fields with validation
- "Remember me" checkbox
- "Forgot password" link
- Registration link for new sellers
- Loading states and error handling

#### Registration Page
- Multi-step form for seller onboarding
- Business information collection
- Email verification flow
- Terms and conditions acceptance
- Pending approval notification

### 2. Dashboard Page

#### Overview Section
- Key performance metrics in card layout
- Visual indicators for trends (up/down arrows)
- Quick action buttons for common tasks
- Responsive grid layout for different screen sizes

#### Recent Activity
- Recent orders table with status indicators
- Low stock alerts with action buttons
- Top-selling products list
- Quick navigation to detailed views

### 3. Product Management Pages

#### Product List Page
```
┌─────────────────────────────────────────────────────────────┐
│ Products                                    [+ Add Product] │
├─────────────────────────────────────────────────────────────┤
│ Search: [_______________] Filter: [Category ▼] [Status ▼]   │
├─────────────────────────────────────────────────────────────┤
│ ┌─────┐                                                     │
│ │ IMG │ Product Name        $99.99    In Stock (25)  [Edit] │
│ └─────┘ Category: Electronics                        [Del]  │
├─────────────────────────────────────────────────────────────┤
│ ┌─────┐                                                     │
│ │ IMG │ Another Product     $49.99    Low Stock (3) [Edit]  │
│ └─────┘ Category: Clothing                           [Del]  │
└─────────────────────────────────────────────────────────────┘
```

#### Product Form Page
- Tabbed interface for different sections
- Image upload with drag-and-drop
- Rich text editor for descriptions
- Category selection dropdown
- Stock management controls
- Price validation and formatting

### 4. Order Management Pages

#### Order List Page
```
┌─────────────────────────────────────────────────────────────┐
│ Orders                                                      │
├─────────────────────────────────────────────────────────────┤
│ Filter: [Status ▼] [Date Range] Search: [_____________]     │
├─────────────────────────────────────────────────────────────┤
│ #12345  John Doe        $129.99   Processing    2024-03-10  │
│         2 items                   [Update Status ▼]        │
├─────────────────────────────────────────────────────────────┤
│ #12344  Jane Smith      $89.99    Shipped       2024-03-09  │
│         1 item                    [View Details]            │
└─────────────────────────────────────────────────────────────┘
```

#### Order Details Page
- Customer information display
- Ordered items list with images
- Shipping address
- Order timeline/status history
- Status update controls
- Print/export options

### 5. Analytics and Reporting Pages

#### Sales Dashboard
- Revenue charts (line, bar, pie)
- Time period selectors
- Key metrics comparison
- Export functionality
- Responsive chart layouts

#### Product Performance
- Product-wise sales data
- Conversion rate metrics
- Inventory turnover analysis
- Performance recommendations

## Mobile Responsive Design

### Breakpoint Strategy
- Mobile: 320px - 767px
- Tablet: 768px - 1023px  
- Desktop: 1024px+

### Mobile Adaptations
- Collapsible navigation menu
- Stacked card layouts
- Touch-friendly button sizes (44px minimum)
- Simplified data tables with horizontal scroll
- Bottom sheet modals for forms

### Navigation Pattern
```
Mobile:
┌─────────────────────┐
│ ☰ Logo        👤   │ <- Header with hamburger menu
├─────────────────────┤
│                     │
│   Main Content      │
│                     │
└─────────────────────┘

Tablet/Desktop:
┌─────────────────────────────────────┐
│ Logo  Dashboard Products Orders ... │ <- Horizontal navigation
├─────────────────────────────────────┤
│                                     │
│           Main Content              │
│                                     │
└─────────────────────────────────────┘
```

## Security Design

### Frontend Security Measures

#### Authentication Security
- JWT tokens in httpOnly cookies
- CSRF protection with double-submit cookies
- Automatic token refresh
- Secure session management

#### Data Protection
- Input validation and sanitization
- XSS prevention through proper escaping
- Content Security Policy headers
- HTTPS enforcement

#### Access Control
- Role-based route protection
- Component-level permission checks
- API request authorization headers
- Seller data isolation

### Security Headers
```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

## Performance Optimization

### Loading Performance
- Code splitting by route and feature
- Lazy loading of non-critical components
- Image optimization and lazy loading
- Bundle size optimization

### Runtime Performance
- React.memo for expensive components
- useMemo and useCallback for expensive calculations
- Virtual scrolling for large lists
- Debounced search inputs

### Caching Strategy
- API response caching with React Query
- Static asset caching
- Browser storage for user preferences
- CDN for image assets

## Integration Design

### API Integration

#### REST API Endpoints
```
Authentication:
POST /api/auth/login
POST /api/auth/register
POST /api/auth/logout
POST /api/auth/refresh

Products:
GET    /api/sellers/{sellerId}/products
POST   /api/sellers/{sellerId}/products
PUT    /api/sellers/{sellerId}/products/{productId}
DELETE /api/sellers/{sellerId}/products/{productId}

Orders:
GET /api/sellers/{sellerId}/orders
PUT /api/orders/{orderId}/status

Analytics:
GET /api/sellers/{sellerId}/analytics/sales
GET /api/sellers/{sellerId}/analytics/products
```

#### Error Handling
- Standardized error response format
- User-friendly error messages
- Retry mechanisms for failed requests
- Offline state handling

### Third-Party Integrations

#### Image Upload Service
- Direct upload to cloud storage
- Image processing and optimization
- Multiple format support
- Progress indicators

#### Analytics Service
- Event tracking for user actions
- Performance monitoring
- Error reporting
- Usage analytics

## Accessibility Design

### WCAG 2.1 AA Compliance
- Semantic HTML structure
- Proper heading hierarchy
- Alt text for images
- Keyboard navigation support
- Screen reader compatibility

### Accessibility Features
- High contrast mode support
- Focus indicators
- Skip navigation links
- ARIA labels and descriptions
- Color-blind friendly design

### Testing Strategy
- Automated accessibility testing
- Screen reader testing
- Keyboard-only navigation testing
- Color contrast validation

## Development Guidelines

### Code Organization
```
src/
├── components/          # Reusable UI components
├── pages/              # Page components
├── hooks/              # Custom React hooks
├── services/           # API services
├── stores/             # State management
├── utils/              # Utility functions
├── types/              # TypeScript type definitions
└── assets/             # Static assets
```

### Component Design Patterns
- Compound components for complex UI
- Render props for flexible composition
- Custom hooks for logic reuse
- Higher-order components for cross-cutting concerns

### Testing Strategy
- Unit tests for utility functions
- Component tests with React Testing Library
- Integration tests for user flows
- E2E tests for critical paths

This design provides a comprehensive foundation for building a modern, scalable, and user-friendly seller frontend interface that integrates seamlessly with the existing e-commerce platform while maintaining consistency with the customer frontend experience.