# Design Document

## Overview

The Customer Backend API is a comprehensive RESTful API system that serves as the backbone for all customer-facing operations in the multi-vendor e-commerce platform. This system provides secure, scalable, and performant backend services that enable customers to browse products, manage shopping carts, place orders, and track purchases across multiple sellers.

The API follows modern microservices architecture principles while maintaining data consistency and providing seamless integration with seller backend systems and admin oversight tools. The design emphasizes security, performance, and maintainability to support a growing marketplace with thousands of customers and sellers.

### Key Design Principles

- **Security First**: All endpoints implement comprehensive authentication, authorization, and data protection
- **Performance Optimized**: Sub-200ms response times for 95% of requests through caching and optimization
- **Scalable Architecture**: Horizontal scaling support through stateless design and load balancing
- **Data Integrity**: ACID transactions and foreign key constraints ensure consistent data state
- **Integration Ready**: Event-driven communication with seller systems and admin tools
- **Developer Friendly**: Comprehensive OpenAPI documentation and consistent error handling

## Architecture

### System Architecture Overview

The Customer Backend API follows a layered microservices architecture with clear separation of concerns:

```mermaid
graph TB
    subgraph "Client Layer"
        CF[Customer Frontend]
        MA[Mobile App]
        TP[Third Party]
    end
    
    subgraph "API Gateway Layer"
        AG[API Gateway]
        LB[Load Balancer]
        RL[Rate Limiter]
    end
    
    subgraph "Application Layer"
        AS[Authentication Service]
        PS[Product Service]
        CS[Cart Service]
        OS[Order Service]
        US[User Service]
        RS[Review Service]
        WS[Wishlist Service]
        NS[Notification Service]
        SS[Search Service]
        PYS[Payment Service]
    end
    
    subgraph "Integration Layer"
        IL[Integration Layer]
        EQ[Event Queue]
        SB[Seller Backend]
        AB[Admin Backend]
    end
    
    subgraph "Data Layer"
        PG[(PostgreSQL)]
        RD[(Redis Cache)]
        ES[(Elasticsearch)]
        FS[File Storage]
    end
    
    CF --> AG
    MA --> AG
    TP --> AG
    
    AG --> LB
    LB --> RL
    RL --> AS
    RL --> PS
    RL --> CS
    RL --> OS
    RL --> US
    RL --> RS
    RL --> WS
    RL --> NS
    RL --> SS
    RL --> PYS
    
    AS --> PG
    PS --> PG
    PS --> ES
    CS --> PG
    CS --> RD
    OS --> PG
    US --> PG
    RS --> PG
    WS --> PG
    SS --> ES
    PYS --> PG
    
    PS --> IL
    OS --> IL
    IL --> EQ
    EQ --> SB
    EQ --> AB
    
    PS --> FS
```

### Microservices Architecture

#### Service Decomposition Strategy

The system is decomposed into focused microservices based on business capabilities:

**Authentication Service**
- Handles user registration, login, and session management
- JWT token generation and validation
- Password reset and email verification
- Rate limiting and security monitoring

**Product Service**
- Product catalog management and retrieval
- Category-based filtering and search
- Product availability and pricing
- Integration with seller product updates

**Cart Service**
- Shopping cart persistence and operations
- Guest cart management with session tokens
- Cart merging for user authentication
- Real-time price and availability validation

**Order Service**
- Order placement and processing
- Multi-seller order coordination
- Order status tracking and history
- Payment integration and confirmation

**User Service**
- Customer profile management
- Address management and validation
- User preferences and settings
- Account security and privacy controls

**Search Service**
- Full-text product search with relevance scoring
- Advanced filtering and sorting capabilities
- Search analytics and optimization
- Autocomplete and suggestion features

**Review Service**
- Product review and rating management
- Purchase verification for review eligibility
- Review moderation and content filtering
- Aggregate rating calculations

**Wishlist Service**
- Product wishlist management
- Wishlist sharing and collaboration
- Availability notifications
- Cart integration for saved items

**Notification Service**
- Email notification delivery
- Template management and personalization
- Delivery tracking and retry logic
- Preference management and unsubscribe handling

**Payment Service**
- Cash on delivery processing
- Online payment gateway integration
- Transaction recording and reconciliation
- Refund and chargeback handling

### Technology Stack

#### Backend Framework
- **Node.js with Express.js**: High-performance JavaScript runtime with mature ecosystem
- **TypeScript**: Type safety and enhanced developer experience
- **Helmet.js**: Security middleware for HTTP headers
- **Compression**: Response compression for improved performance

#### Database Technologies
- **PostgreSQL 15+**: Primary relational database with ACID compliance
- **Redis 7+**: In-memory caching and session storage
- **Elasticsearch 8+**: Full-text search and analytics engine

#### Authentication & Security
- **JWT (JSON Web Tokens)**: Stateless authentication with RS256 signing
- **bcrypt**: Password hashing with configurable cost factor
- **express-rate-limit**: API rate limiting and DDoS protection
- **CORS**: Cross-origin resource sharing configuration

#### Integration & Communication
- **Bull Queue**: Redis-based job queue for background processing
- **Socket.io**: Real-time communication for notifications
- **Axios**: HTTP client for external API integration
- **Joi**: Request validation and schema enforcement

#### Monitoring & Logging
- **Winston**: Structured logging with multiple transports
- **Prometheus**: Metrics collection and monitoring
- **Jaeger**: Distributed tracing for performance analysis
- **New Relic**: Application performance monitoring

## Components and Interfaces

### API Gateway Design

#### Request Flow Architecture
```mermaid
sequenceDiagram
    participant C as Client
    participant AG as API Gateway
    participant AS as Auth Service
    participant MS as Microservice
    participant DB as Database
    
    C->>AG: HTTP Request
    AG->>AG: Rate Limiting Check
    AG->>AS: Token Validation
    AS-->>AG: User Context
    AG->>MS: Forwarded Request
    MS->>DB: Data Operation
    DB-->>MS: Result
    MS-->>AG: Response
    AG-->>C: HTTP Response
```

#### Gateway Responsibilities
- **Request Routing**: Intelligent routing based on URL patterns and service health
- **Authentication**: JWT token validation and user context injection
- **Rate Limiting**: Per-user and per-IP rate limiting with Redis backend
- **Request/Response Transformation**: Header injection and response formatting
- **Circuit Breaking**: Automatic failover and service health monitoring
- **Logging**: Comprehensive request/response logging for audit and debugging

### Service Interface Definitions

#### Authentication Service Interface
```typescript
interface AuthenticationService {
  // Registration and login
  register(userData: RegisterRequest): Promise<AuthResponse>
  login(credentials: LoginRequest): Promise<AuthResponse>
  logout(token: string): Promise<void>
  refreshToken(refreshToken: string): Promise<AuthResponse>
  
  // Password management
  requestPasswordReset(email: string): Promise<void>
  resetPassword(token: string, newPassword: string): Promise<void>
  changePassword(userId: string, oldPassword: string, newPassword: string): Promise<void>
  
  // Token validation
  validateToken(token: string): Promise<UserContext>
  revokeToken(token: string): Promise<void>
}

interface RegisterRequest {
  name: string
  email: string
  password: string
}

interface LoginRequest {
  email: string
  password: string
}

interface AuthResponse {
  token: string
  refreshToken: string
  user: UserProfile
  expiresIn: number
}
```

#### Product Service Interface
```typescript
interface ProductService {
  // Product retrieval
  getProducts(filters: ProductFilters, pagination: Pagination): Promise<ProductPage>
  getProduct(productId: string): Promise<ProductDetail>
  searchProducts(query: SearchQuery): Promise<ProductPage>
  
  // Category management
  getCategories(): Promise<Category[]>
  getProductsByCategory(categoryId: string, pagination: Pagination): Promise<ProductPage>
  
  // Availability and pricing
  checkAvailability(productIds: string[]): Promise<AvailabilityMap>
  getCurrentPrices(productIds: string[]): Promise<PriceMap>
}

interface ProductFilters {
  categoryId?: string
  sellerId?: string
  priceRange?: { min: number; max: number }
  availability?: boolean
  brand?: string
  rating?: number
}

interface ProductPage {
  products: Product[]
  totalCount: number
  hasNextPage: boolean
  filters: FilterOptions
}
```

#### Cart Service Interface
```typescript
interface CartService {
  // Cart operations
  getCart(userId: string): Promise<Cart>
  addToCart(userId: string, item: CartItem): Promise<Cart>
  updateCartItem(userId: string, itemId: string, quantity: number): Promise<Cart>
  removeFromCart(userId: string, itemId: string): Promise<Cart>
  clearCart(userId: string): Promise<void>
  
  // Guest cart operations
  createGuestCart(): Promise<GuestCartToken>
  getGuestCart(token: string): Promise<Cart>
  mergeGuestCart(userId: string, guestToken: string): Promise<Cart>
  
  // Cart validation
  validateCart(cart: Cart): Promise<CartValidation>
  calculateTotals(cart: Cart): Promise<CartTotals>
}

interface Cart {
  id: string
  userId?: string
  items: CartItem[]
  totals: CartTotals
  lastUpdated: Date
}

interface CartItem {
  productId: string
  sellerId: string
  quantity: number
  price: number
  availability: boolean
}
```

#### Order Service Interface
```typescript
interface OrderService {
  // Order placement
  createOrder(userId: string, orderData: CreateOrderRequest): Promise<Order>
  validateOrderData(orderData: CreateOrderRequest): Promise<OrderValidation>
  
  // Order management
  getOrder(orderId: string, userId: string): Promise<Order>
  getOrderHistory(userId: string, pagination: Pagination): Promise<OrderPage>
  cancelOrder(orderId: string, userId: string): Promise<Order>
  
  // Order tracking
  getOrderStatus(orderId: string): Promise<OrderStatus>
  getTrackingInfo(orderId: string): Promise<TrackingInfo>
}

interface CreateOrderRequest {
  cartId: string
  shippingAddress: Address
  paymentMethod: PaymentMethod
  specialInstructions?: string
}

interface Order {
  id: string
  orderNumber: string
  userId: string
  items: OrderItem[]
  shippingAddress: Address
  paymentMethod: PaymentMethod
  status: OrderStatus
  totals: OrderTotals
  createdAt: Date
  estimatedDelivery?: Date
}
```

### Database Integration Layer

#### Connection Management
```typescript
interface DatabaseManager {
  // Connection pooling
  getConnection(): Promise<DatabaseConnection>
  executeTransaction<T>(operation: (conn: DatabaseConnection) => Promise<T>): Promise<T>
  
  // Health monitoring
  checkHealth(): Promise<HealthStatus>
  getConnectionStats(): ConnectionStats
}

interface DatabaseConnection {
  query<T>(sql: string, params?: any[]): Promise<T[]>
  execute(sql: string, params?: any[]): Promise<ExecutionResult>
  beginTransaction(): Promise<void>
  commit(): Promise<void>
  rollback(): Promise<void>
}
```

#### Repository Pattern Implementation
```typescript
abstract class BaseRepository<T> {
  protected tableName: string
  protected db: DatabaseManager
  
  async findById(id: string): Promise<T | null>
  async findMany(filters: FilterCriteria): Promise<T[]>
  async create(entity: Partial<T>): Promise<T>
  async update(id: string, updates: Partial<T>): Promise<T>
  async delete(id: string): Promise<void>
  
  protected abstract mapToEntity(row: any): T
  protected abstract mapToRow(entity: Partial<T>): any
}
```

## Data Models

### Core Entity Relationships

```mermaid
erDiagram
    CUSTOMERS ||--o{ ORDERS : places
    CUSTOMERS ||--o{ CART_ITEMS : has
    CUSTOMERS ||--o{ ADDRESSES : owns
    CUSTOMERS ||--o{ REVIEWS : writes
    CUSTOMERS ||--o{ WISHLISTS : maintains
    
    ORDERS ||--o{ ORDER_ITEMS : contains
    ORDERS }o--|| ADDRESSES : ships_to
    ORDERS }o--|| PAYMENT_METHODS : paid_with
    
    PRODUCTS ||--o{ ORDER_ITEMS : ordered
    PRODUCTS ||--o{ CART_ITEMS : added_to_cart
    PRODUCTS ||--o{ REVIEWS : reviewed
    PRODUCTS ||--o{ WISHLIST_ITEMS : saved_to
    PRODUCTS }o--|| CATEGORIES : belongs_to
    PRODUCTS }o--|| SELLERS : sold_by
    
    WISHLISTS ||--o{ WISHLIST_ITEMS : contains
    
    CUSTOMERS {
        uuid id PK
        string email UK
        string password_hash
        string name
        string phone
        boolean email_verified
        timestamp created_at
        timestamp updated_at
        timestamp last_login
    }
    
    ADDRESSES {
        uuid id PK
        uuid customer_id FK
        string street_address
        string city
        string state
        string postal_code
        string country
        boolean is_default
        timestamp created_at
    }
    
    PRODUCTS {
        uuid id PK
        uuid seller_id FK
        uuid category_id FK
        string name
        text description
        decimal price
        integer stock_quantity
        boolean is_active
        json images
        decimal average_rating
        integer review_count
        timestamp created_at
        timestamp updated_at
    }
    
    CATEGORIES {
        uuid id PK
        string name UK
        string slug UK
        text description
        uuid parent_id FK
        integer sort_order
        boolean is_active
    }
    
    ORDERS {
        uuid id PK
        string order_number UK
        uuid customer_id FK
        uuid shipping_address_id FK
        enum status
        decimal subtotal
        decimal tax_amount
        decimal shipping_cost
        decimal total_amount
        enum payment_method
        enum payment_status
        text special_instructions
        timestamp created_at
        timestamp updated_at
        timestamp estimated_delivery
    }
    
    ORDER_ITEMS {
        uuid id PK
        uuid order_id FK
        uuid product_id FK
        uuid seller_id FK
        integer quantity
        decimal unit_price
        decimal total_price
        json product_snapshot
    }
    
    CART_ITEMS {
        uuid id PK
        uuid customer_id FK
        string session_token
        uuid product_id FK
        integer quantity
        timestamp created_at
        timestamp updated_at
    }
    
    REVIEWS {
        uuid id PK
        uuid customer_id FK
        uuid product_id FK
        uuid order_id FK
        integer rating
        text comment
        boolean is_verified_purchase
        timestamp created_at
    }
    
    WISHLISTS {
        uuid id PK
        uuid customer_id FK
        string name
        boolean is_public
        string share_token
        timestamp created_at
    }
    
    WISHLIST_ITEMS {
        uuid id PK
        uuid wishlist_id FK
        uuid product_id FK
        timestamp added_at
    }
```

### Database Schema Design

#### Customers Table
```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT valid_phone CHECK (phone IS NULL OR phone ~* '^\+?[1-9]\d{1,14}$')
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_created_at ON customers(created_at);
CREATE INDEX idx_customers_last_login ON customers(last_login);
```

#### Products Table
```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id UUID NOT NULL,
    category_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL CHECK (stock_quantity >= 0),
    is_active BOOLEAN DEFAULT TRUE,
    images JSONB DEFAULT '[]',
    average_rating DECIMAL(3,2) DEFAULT 0 CHECK (average_rating >= 0 AND average_rating <= 5),
    review_count INTEGER DEFAULT 0 CHECK (review_count >= 0),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    FOREIGN KEY (category_id) REFERENCES categories(id),
    FOREIGN KEY (seller_id) REFERENCES sellers(id)
);

CREATE INDEX idx_products_seller_id ON products(seller_id);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_is_active ON products(is_active);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_average_rating ON products(average_rating);
CREATE INDEX idx_products_created_at ON products(created_at);
CREATE INDEX idx_products_name_gin ON products USING gin(to_tsvector('english', name));
CREATE INDEX idx_products_description_gin ON products USING gin(to_tsvector('english', description));
```

#### Orders Table
```sql
CREATE TYPE order_status AS ENUM ('placed', 'confirmed', 'processing', 'shipped', 'out_for_delivery', 'delivered', 'cancelled');
CREATE TYPE payment_method AS ENUM ('cod', 'online', 'wallet');
CREATE TYPE payment_status AS ENUM ('pending', 'completed', 'failed', 'refunded');

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    shipping_address_id UUID NOT NULL,
    status order_status DEFAULT 'placed',
    subtotal DECIMAL(10,2) NOT NULL CHECK (subtotal >= 0),
    tax_amount DECIMAL(10,2) DEFAULT 0 CHECK (tax_amount >= 0),
    shipping_cost DECIMAL(10,2) DEFAULT 0 CHECK (shipping_cost >= 0),
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
    payment_method payment_method NOT NULL,
    payment_status payment_status DEFAULT 'pending',
    special_instructions TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    estimated_delivery TIMESTAMP WITH TIME ZONE,
    
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(id)
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_orders_order_number ON orders(order_number);
```

### Data Validation and Constraints

#### Business Rule Constraints
```sql
-- Ensure cart items reference valid products
ALTER TABLE cart_items 
ADD CONSTRAINT fk_cart_items_product 
FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE;

-- Ensure reviews are from verified purchases
ALTER TABLE reviews 
ADD CONSTRAINT fk_reviews_order 
FOREIGN KEY (order_id) REFERENCES orders(id);

-- Ensure order items maintain product snapshot integrity
ALTER TABLE order_items 
ADD CONSTRAINT valid_order_item_price 
CHECK (unit_price > 0 AND total_price = unit_price * quantity);

-- Ensure addresses belong to customers
ALTER TABLE addresses 
ADD CONSTRAINT fk_addresses_customer 
FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE;
```

#### Data Integrity Triggers
```sql
-- Update product average rating when reviews change
CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products 
    SET average_rating = (
        SELECT COALESCE(AVG(rating), 0) 
        FROM reviews 
        WHERE product_id = COALESCE(NEW.product_id, OLD.product_id)
    ),
    review_count = (
        SELECT COUNT(*) 
        FROM reviews 
        WHERE product_id = COALESCE(NEW.product_id, OLD.product_id)
    )
    WHERE id = COALESCE(NEW.product_id, OLD.product_id);
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_product_rating
    AFTER INSERT OR UPDATE OR DELETE ON reviews
    FOR EACH ROW EXECUTE FUNCTION update_product_rating();
```

### Caching Strategy

#### Redis Cache Design
```typescript
interface CacheManager {
  // Product caching
  cacheProduct(productId: string, product: Product, ttl?: number): Promise<void>
  getCachedProduct(productId: string): Promise<Product | null>
  invalidateProductCache(productId: string): Promise<void>
  
  // Search result caching
  cacheSearchResults(query: string, results: ProductPage, ttl?: number): Promise<void>
  getCachedSearchResults(query: string): Promise<ProductPage | null>
  
  // Cart caching
  cacheCart(userId: string, cart: Cart, ttl?: number): Promise<void>
  getCachedCart(userId: string): Promise<Cart | null>
  invalidateCartCache(userId: string): Promise<void>
  
  // Session management
  storeSession(sessionId: string, sessionData: SessionData, ttl: number): Promise<void>
  getSession(sessionId: string): Promise<SessionData | null>
  invalidateSession(sessionId: string): Promise<void>
}
```

#### Cache Key Patterns
```typescript
const CacheKeys = {
  PRODUCT: (id: string) => `product:${id}`,
  PRODUCT_LIST: (filters: string) => `products:${filters}`,
  SEARCH_RESULTS: (query: string) => `search:${Buffer.from(query).toString('base64')}`,
  CART: (userId: string) => `cart:${userId}`,
  USER_SESSION: (sessionId: string) => `session:${sessionId}`,
  CATEGORY_TREE: () => 'categories:tree',
  SELLER_PRODUCTS: (sellerId: string) => `seller:${sellerId}:products`
}
```

## Error Handling

### Error Response Format

#### Standardized Error Structure
```typescript
interface APIError {
  error: {
    code: string
    message: string
    details?: any
    timestamp: string
    requestId: string
    path: string
  }
}

// Example error responses
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data provided",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    },
    "timestamp": "2024-03-10T10:30:00Z",
    "requestId": "req_123456789",
    "path": "/api/auth/register"
  }
}
```

### Error Categories and HTTP Status Codes

#### Authentication Errors (4xx)
- **401 Unauthorized**: Invalid or expired JWT token
- **403 Forbidden**: Insufficient permissions for resource access
- **429 Too Many Requests**: Rate limit exceeded

#### Validation Errors (4xx)
- **400 Bad Request**: Invalid request format or missing required fields
- **422 Unprocessable Entity**: Valid format but business rule violations

#### Resource Errors (4xx)
- **404 Not Found**: Requested resource does not exist
- **409 Conflict**: Resource conflict (e.g., duplicate email registration)
- **410 Gone**: Resource permanently deleted

#### Server Errors (5xx)
- **500 Internal Server Error**: Unexpected server error
- **502 Bad Gateway**: External service unavailable
- **503 Service Unavailable**: Temporary service overload
- **504 Gateway Timeout**: External service timeout

### Error Handling Middleware

```typescript
interface ErrorHandler {
  handleValidationError(error: ValidationError): APIError
  handleDatabaseError(error: DatabaseError): APIError
  handleAuthenticationError(error: AuthError): APIError
  handleExternalServiceError(error: ExternalError): APIError
  handleUnknownError(error: Error): APIError
}

class GlobalErrorHandler implements ErrorHandler {
  handleValidationError(error: ValidationError): APIError {
    return {
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid input data provided',
        details: error.details,
        timestamp: new Date().toISOString(),
        requestId: error.requestId,
        path: error.path
      }
    }
  }
  
  handleDatabaseError(error: DatabaseError): APIError {
    // Log sensitive database details internally
    logger.error('Database error:', error)
    
    return {
      error: {
        code: 'DATABASE_ERROR',
        message: 'A database error occurred',
        timestamp: new Date().toISOString(),
        requestId: error.requestId,
        path: error.path
      }
    }
  }
}
```

### Circuit Breaker Pattern

```typescript
interface CircuitBreaker {
  execute<T>(operation: () => Promise<T>): Promise<T>
  getState(): CircuitState
  reset(): void
}

enum CircuitState {
  CLOSED = 'closed',
  OPEN = 'open',
  HALF_OPEN = 'half_open'
}

class ServiceCircuitBreaker implements CircuitBreaker {
  private failureCount = 0
  private lastFailureTime?: Date
  private state = CircuitState.CLOSED
  
  constructor(
    private failureThreshold: number = 5,
    private recoveryTimeout: number = 60000
  ) {}
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (this.shouldAttemptReset()) {
        this.state = CircuitState.HALF_OPEN
      } else {
        throw new Error('Circuit breaker is OPEN')
      }
    }
    
    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }
}
```

### Retry Logic and Backoff

```typescript
interface RetryConfig {
  maxAttempts: number
  baseDelay: number
  maxDelay: number
  backoffMultiplier: number
  retryableErrors: string[]
}

class RetryHandler {
  constructor(private config: RetryConfig) {}
  
  async executeWithRetry<T>(
    operation: () => Promise<T>,
    context: string
  ): Promise<T> {
    let lastError: Error
    
    for (let attempt = 1; attempt <= this.config.maxAttempts; attempt++) {
      try {
        return await operation()
      } catch (error) {
        lastError = error
        
        if (!this.isRetryableError(error) || attempt === this.config.maxAttempts) {
          throw error
        }
        
        const delay = this.calculateDelay(attempt)
        logger.warn(`Attempt ${attempt} failed for ${context}, retrying in ${delay}ms`, error)
        await this.sleep(delay)
      }
    }
    
    throw lastError!
  }
  
  private calculateDelay(attempt: number): number {
    const delay = this.config.baseDelay * Math.pow(this.config.backoffMultiplier, attempt - 1)
    return Math.min(delay, this.config.maxDelay)
  }
}
```

## Testing Strategy

### Testing Pyramid Approach

The testing strategy follows a comprehensive pyramid approach with emphasis on both unit testing and property-based testing to ensure correctness and reliability.

#### Unit Testing (Foundation Layer)
- **Scope**: Individual functions, methods, and components
- **Framework**: Jest with TypeScript support
- **Coverage Target**: 90%+ code coverage
- **Focus Areas**:
  - Business logic validation
  - Error handling scenarios
  - Edge cases and boundary conditions
  - Integration points between services

#### Integration Testing (Middle Layer)
- **Scope**: Service-to-service communication and database interactions
- **Framework**: Jest with Supertest for API testing
- **Database**: Test database with Docker containers
- **Focus Areas**:
  - API endpoint functionality
  - Database transaction integrity
  - External service integration
  - Authentication and authorization flows

#### Property-Based Testing (Correctness Layer)
- **Framework**: fast-check for JavaScript/TypeScript
- **Configuration**: Minimum 100 iterations per property test
- **Focus Areas**:
  - Universal properties that must hold for all inputs
  - Data transformation correctness
  - Business rule enforcement
  - System invariants and consistency

#### End-to-End Testing (Top Layer)
- **Scope**: Complete user workflows across the entire system
- **Framework**: Playwright for browser automation
- **Environment**: Staging environment with production-like data
- **Focus Areas**:
  - Critical user journeys
  - Cross-browser compatibility
  - Performance under load
  - Security vulnerability testing

### Property-Based Testing Configuration

#### Test Framework Setup
```typescript
import fc from 'fast-check'

// Custom generators for domain objects
const customerGenerator = fc.record({
  name: fc.string({ minLength: 1, maxLength: 100 }),
  email: fc.emailAddress(),
  password: fc.string({ minLength: 8, maxLength: 50 })
})

const productGenerator = fc.record({
  name: fc.string({ minLength: 1, maxLength: 200 }),
  price: fc.float({ min: 0.01, max: 10000, noNaN: true }),
  stockQuantity: fc.integer({ min: 0, max: 1000 }),
  description: fc.string({ maxLength: 2000 })
})

const cartItemGenerator = fc.record({
  productId: fc.uuid(),
  quantity: fc.integer({ min: 1, max: 10 }),
  price: fc.float({ min: 0.01, max: 1000, noNaN: true })
})
```

#### Property Test Structure
```typescript
describe('Property-Based Tests', () => {
  it('should maintain cart total consistency', () => {
    fc.assert(fc.property(
      fc.array(cartItemGenerator, { minLength: 1, maxLength: 20 }),
      (cartItems) => {
        // Feature: customer-backend-api, Property 1: Cart total calculation consistency
        const calculatedTotal = calculateCartTotal(cartItems)
        const manualTotal = cartItems.reduce((sum, item) => sum + (item.price * item.quantity), 0)
        
        expect(Math.abs(calculatedTotal - manualTotal)).toBeLessThan(0.01)
      }
    ), { numRuns: 100 })
  })
})
```

### Test Data Management

#### Test Database Strategy
```typescript
interface TestDatabaseManager {
  setupTestDatabase(): Promise<void>
  seedTestData(): Promise<void>
  cleanupTestData(): Promise<void>
  createTestTransaction(): Promise<DatabaseTransaction>
}

class PostgreSQLTestManager implements TestDatabaseManager {
  async setupTestDatabase(): Promise<void> {
    // Create isolated test database
    await this.createDatabase(`test_${Date.now()}`)
    await this.runMigrations()
  }
  
  async seedTestData(): Promise<void> {
    // Insert minimal required data for tests
    await this.insertTestCategories()
    await this.insertTestSellers()
    await this.insertTestProducts()
  }
  
  async cleanupTestData(): Promise<void> {
    // Clean up test data after each test
    await this.truncateAllTables()
  }
}
```

#### Mock Service Configuration
```typescript
interface MockServiceConfig {
  sellerBackendService: MockSellerService
  paymentGateway: MockPaymentGateway
  emailService: MockEmailService
  searchEngine: MockSearchEngine
}

class MockSellerService {
  async notifyNewOrder(order: Order): Promise<void> {
    // Mock implementation for testing
    this.recordedNotifications.push(order)
  }
  
  async updateProductAvailability(productId: string, quantity: number): Promise<void> {
    this.productUpdates.set(productId, quantity)
  }
}
```

### Performance Testing

#### Load Testing Strategy
- **Tool**: Artillery.js for API load testing
- **Scenarios**: Realistic user behavior patterns
- **Metrics**: Response time, throughput, error rate
- **Targets**: 
  - 95% of requests under 200ms
  - Support 1000 concurrent users
  - 99.9% uptime under normal load

#### Performance Test Configuration
```yaml
# artillery-config.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
    - duration: 120
      arrivalRate: 50
    - duration: 60
      arrivalRate: 100
  defaults:
    headers:
      Content-Type: 'application/json'

scenarios:
  - name: 'Customer Journey'
    weight: 70
    flow:
      - post:
          url: '/api/auth/login'
          json:
            email: 'test@example.com'
            password: 'password123'
      - get:
          url: '/api/products'
      - post:
          url: '/api/cart/items'
          json:
            productId: '{{ $randomUUID }}'
            quantity: 2
      - get:
          url: '/api/cart'
```

### Security Testing

#### Security Test Categories
- **Authentication Testing**: Token validation, session management
- **Authorization Testing**: Role-based access control
- **Input Validation Testing**: SQL injection, XSS prevention
- **Rate Limiting Testing**: DDoS protection verification
- **Data Protection Testing**: Encryption and data masking

#### Automated Security Scanning
```typescript
describe('Security Tests', () => {
  it('should prevent SQL injection attacks', async () => {
    const maliciousInputs = [
      "'; DROP TABLE customers; --",
      "1' OR '1'='1",
      "admin'/*",
      "1; DELETE FROM orders WHERE 1=1; --"
    ]
    
    for (const input of maliciousInputs) {
      const response = await request(app)
        .get(`/api/products/search?q=${encodeURIComponent(input)}`)
        .expect(400)
      
      expect(response.body.error.code).toBe('VALIDATION_ERROR')
    }
  })
  
  it('should enforce rate limiting', async () => {
    const requests = Array(10).fill(null).map(() => 
      request(app).post('/api/auth/login').send({
        email: 'test@example.com',
        password: 'wrongpassword'
      })
    )
    
    const responses = await Promise.all(requests)
    const rateLimitedResponses = responses.filter(r => r.status === 429)
    
    expect(rateLimitedResponses.length).toBeGreaterThan(0)
  })
})
```

### Continuous Integration Testing

#### CI/CD Pipeline Configuration
```yaml
# .github/workflows/test.yml
name: Test Suite
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
      
      - name: Run property-based tests
        run: npm run test:property
      
      - name: Generate coverage report
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
```

This comprehensive testing strategy ensures that the Customer Backend API maintains high quality, reliability, and security standards while supporting rapid development and deployment cycles. The combination of traditional testing approaches with property-based testing provides both concrete validation and mathematical correctness guarantees.
## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Valid Registration Creates Account

*For any* valid registration data (proper email format, password ≥8 characters, required fields present), submitting it to the registration endpoint should result in a new customer account being created in the database with matching information.

**Validates: Requirements 1.1**

### Property 2: Duplicate Email Registration Rejection

*For any* email address that already exists in the customer database, attempting to register with that email should return a 409 Conflict error regardless of other registration data.

**Validates: Requirements 1.2**

### Property 3: Registration Input Validation

*For any* registration request with invalid data (malformed email, password <8 characters, or missing required fields), the Authentication_Service should reject the request with appropriate validation errors.

**Validates: Requirements 1.3**

### Property 4: Successful Registration Response Format

*For any* valid registration request, successful processing should return a 201 Created status code along with a verification token.

**Validates: Requirements 1.4**

### Property 5: Registration Email Notification

*For any* successful customer registration, the Notification_Service should send a verification email to the registered email address.

**Validates: Requirements 1.5**

### Property 6: Password Security Storage

*For any* customer password stored in the database, it should be a bcrypt hash with cost factor 12, never stored as plaintext.

**Validates: Requirements 1.6**

### Property 7: Registration Audit Logging

*For any* registration attempt (successful or failed), the Audit_System should create a log entry containing timestamp, IP address, and outcome.

**Validates: Requirements 1.7**

### Property 8: Valid Authentication Token Generation

*For any* valid customer credentials (existing email and correct password), authentication should generate and return a valid JWT token.

**Validates: Requirements 2.1**

### Property 9: Invalid Authentication Rejection

*For any* invalid credentials (non-existent email or incorrect password), authentication should return a 401 Unauthorized error.

**Validates: Requirements 2.2**

### Property 10: Authentication Rate Limiting

*For any* email address, after 5 failed authentication attempts within 15 minutes, subsequent attempts should be rate limited.

**Validates: Requirements 2.3**

### Property 11: Rate Limit Enforcement Response

*For any* email address that has exceeded the rate limit, authentication attempts should return 429 Too Many Requests and remain blocked for 30 minutes.

**Validates: Requirements 2.4**

### Property 12: JWT Token Structure and Expiration

*For any* generated JWT token, it should have a 24-hour expiration time and contain the customer ID and role in its payload.

**Validates: Requirements 2.5**

### Property 13: Token Refresh Functionality

*For any* valid refresh token, the token refresh endpoint should generate and return a new valid access token without requiring re-authentication.

**Validates: Requirements 2.6**

### Property 14: Protected Endpoint Token Validation

*For any* protected API endpoint, requests with invalid or missing JWT tokens should be rejected with a 401 Unauthorized response.

**Validates: Requirements 2.7**

### Property 15: Product Listing Response Format

*For any* product listing request, the response should contain paginated products with name, price, images, and availability information in the specified format.

**Validates: Requirements 5.1**

### Property 16: Pagination Constraints

*For any* product listing request, the pagination should enforce a default page size of 20 products and a maximum page size of 100 products per page.

**Validates: Requirements 5.2**

### Property 17: Individual Product Retrieval

*For any* valid product ID, requesting that specific product should return detailed product information including seller details.

**Validates: Requirements 5.3**

### Property 18: Cart Item Addition

*For any* authenticated customer and valid product, adding the product to cart should store the item with correct quantity and seller information in the customer's cart.

**Validates: Requirements 7.1**

### Property 19: Cart Contents Accuracy

*For any* customer's cart, retrieving cart contents should return all stored items with current prices and availability status.

**Validates: Requirements 7.2**

### Property 20: Cart Quantity Updates with Validation

*For any* cart item quantity update, the system should validate against stock availability and only update the quantity if sufficient stock exists.

**Validates: Requirements 7.3**

### Property 21: Cart Item Removal

*For any* cart item that exists in a customer's cart, removing that item should result in it no longer being present in the cart.

**Validates: Requirements 7.4**

### Property 22: Cart Total Calculation Accuracy

*For any* cart with items, the calculated totals (subtotal, taxes, shipping estimates) should be mathematically correct based on item prices, quantities, and applicable rates.

**Validates: Requirements 7.5**

### Property 23: Valid Order Creation

*For any* valid order request (valid cart, shipping address, and payment method), the Order_Service should successfully create order records and initiate payment processing.

**Validates: Requirements 9.1**

### Property 24: Order Data Validation

*For any* order request with invalid data (empty cart, invalid address, or invalid payment information), the Order_Service should reject the request before creating any order records.

**Validates: Requirements 9.2**

### Property 25: Order Confirmation Generation

*For any* successfully placed order, the system should generate a unique order number and return complete order confirmation data.

**Validates: Requirements 9.3**

### Property 26: Inventory Integration Consistency

*For any* successfully placed order, the Inventory_System should reserve the ordered products and update stock levels to reflect the purchase.

**Validates: Requirements 9.4**

### Property 27: Multi-Seller Order Splitting

*For any* order containing products from multiple sellers, the Order_Service should create separate order records for each seller while maintaining order relationship integrity.

**Validates: Requirements 9.5**

### Property 28: JSON Request Parsing

*For any* valid JSON request body that conforms to defined schemas, the Customer_API should successfully parse the request according to the schema specifications.

**Validates: Requirements 28.1**

### Property 29: Invalid JSON Error Handling

*For any* malformed or invalid JSON request, the Customer_API should return a 400 Bad Request response with descriptive error messages.

**Validates: Requirements 28.2**

### Property 30: Response JSON Serialization

*For any* API response, the data should be serialized to valid JSON format with consistent structure across all endpoints.

**Validates: Requirements 28.3**

### Property 31: Serialization Round-Trip Integrity

*For any* valid API request object, the sequence of parsing then serializing then parsing should produce an equivalent object, ensuring data integrity through the serialization cycle.

**Validates: Requirements 28.4**

### Property 32: Input Schema Validation

*For any* API request, input data should be validated against defined schemas before any processing occurs, rejecting invalid data early in the request lifecycle.

**Validates: Requirements 28.5**

### Property 33: Error Response Format Consistency

*For any* error condition across all API endpoints, the error response should follow the standardized error format structure with consistent fields and formatting.

**Validates: Requirements 28.6**

### Property 34: International Character Encoding

*For any* customer data containing international characters (Unicode), the system should handle character encoding properly throughout storage, retrieval, and processing operations.

**Validates: Requirements 28.7**
## Error Handling

### Comprehensive Error Management Strategy

The Customer Backend API implements a multi-layered error handling approach that ensures consistent, informative, and secure error responses across all endpoints while maintaining system stability and user experience.

#### Error Classification and Response Codes

**Client Errors (4xx Series)**
- **400 Bad Request**: Malformed requests, invalid JSON, or missing required parameters
- **401 Unauthorized**: Missing, invalid, or expired authentication tokens
- **403 Forbidden**: Valid authentication but insufficient permissions for the requested resource
- **404 Not Found**: Requested resource does not exist or has been deleted
- **409 Conflict**: Resource conflicts such as duplicate email registration or concurrent modifications
- **422 Unprocessable Entity**: Valid request format but business rule violations
- **429 Too Many Requests**: Rate limiting triggered due to excessive requests

**Server Errors (5xx Series)**
- **500 Internal Server Error**: Unexpected application errors with sensitive details hidden
- **502 Bad Gateway**: External service dependencies unavailable or returning errors
- **503 Service Unavailable**: Temporary service overload or maintenance mode
- **504 Gateway Timeout**: External service timeouts or database connection issues

#### Error Response Structure

All error responses follow a standardized format for consistency:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error description",
    "details": {
      "field": "specific_field",
      "reason": "detailed_explanation"
    },
    "timestamp": "2024-03-10T10:30:00Z",
    "requestId": "req_abc123def456",
    "path": "/api/endpoint/path"
  }
}
```

#### Error Handling Middleware Pipeline

The error handling system processes errors through a structured middleware pipeline:

1. **Error Detection**: Catch all errors from controllers, services, and database operations
2. **Error Classification**: Determine error type and appropriate HTTP status code
3. **Security Filtering**: Remove sensitive information from error messages
4. **Response Formatting**: Apply standardized error response structure
5. **Logging**: Record error details for monitoring and debugging
6. **Client Response**: Send formatted error response to client

#### Validation Error Handling

Input validation errors provide detailed feedback while maintaining security:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": {
      "email": "Invalid email format",
      "password": "Password must be at least 8 characters",
      "name": "Name is required"
    },
    "timestamp": "2024-03-10T10:30:00Z",
    "requestId": "req_validation_123",
    "path": "/api/auth/register"
  }
}
```

#### Database Error Handling

Database errors are handled with careful attention to security and information disclosure:

- **Connection Errors**: Graceful degradation with retry mechanisms
- **Constraint Violations**: Meaningful business-level error messages
- **Transaction Failures**: Automatic rollback with consistent error responses
- **Timeout Errors**: Clear timeout messages with suggested retry strategies

#### External Service Error Handling

Integration with external services includes comprehensive error handling:

- **Circuit Breaker Pattern**: Prevent cascading failures from external service outages
- **Retry Logic**: Exponential backoff for transient failures
- **Fallback Mechanisms**: Alternative responses when external services are unavailable
- **Timeout Management**: Configurable timeouts with graceful error responses

#### Rate Limiting Error Responses

Rate limiting errors provide clear information about limits and retry timing:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "details": {
      "limit": 100,
      "window": "1 hour",
      "retryAfter": 3600
    },
    "timestamp": "2024-03-10T10:30:00Z",
    "requestId": "req_ratelimit_456",
    "path": "/api/auth/login"
  }
}
```

## Testing Strategy

### Comprehensive Testing Approach

The Customer Backend API employs a multi-tiered testing strategy that combines traditional testing methodologies with property-based testing to ensure both functional correctness and mathematical guarantees of system behavior.

#### Testing Pyramid Structure

**Unit Tests (Foundation - 70% of tests)**
- **Scope**: Individual functions, methods, and isolated components
- **Framework**: Jest with TypeScript support and comprehensive mocking
- **Coverage Target**: 95%+ code coverage for business logic
- **Focus Areas**:
  - Business rule validation and enforcement
  - Data transformation and calculation accuracy
  - Error handling and edge case scenarios
  - Security validation and input sanitization

**Integration Tests (Middle Layer - 20% of tests)**
- **Scope**: Service interactions, database operations, and API endpoints
- **Framework**: Jest with Supertest for HTTP testing and Docker for database isolation
- **Environment**: Containerized test environment with PostgreSQL and Redis
- **Focus Areas**:
  - API endpoint functionality and response formats
  - Database transaction integrity and rollback scenarios
  - Service-to-service communication patterns
  - Authentication and authorization workflows

**Property-Based Tests (Correctness Layer - 8% of tests)**
- **Framework**: fast-check for JavaScript/TypeScript property testing
- **Configuration**: Minimum 100 iterations per property with configurable seed values
- **Focus Areas**:
  - Universal properties that must hold for all valid inputs
  - Mathematical correctness of calculations and transformations
  - System invariants and consistency guarantees
  - Round-trip properties for serialization and data integrity

**End-to-End Tests (Integration Layer - 2% of tests)**
- **Scope**: Complete user workflows across the entire system
- **Framework**: Playwright for API workflow testing
- **Environment**: Staging environment with production-like data and load
- **Focus Areas**:
  - Critical customer journey validation
  - Cross-service integration verification
  - Performance under realistic load conditions
  - Security vulnerability and penetration testing

#### Property-Based Testing Implementation

**Test Configuration and Generators**
```typescript
import fc from 'fast-check'

// Domain-specific generators for comprehensive property testing
const validEmailGenerator = fc.emailAddress()
const strongPasswordGenerator = fc.string({ minLength: 8, maxLength: 50 })
  .filter(pwd => /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(pwd))

const customerDataGenerator = fc.record({
  name: fc.string({ minLength: 1, maxLength: 100 }),
  email: validEmailGenerator,
  password: strongPasswordGenerator,
  phone: fc.option(fc.string({ minLength: 10, maxLength: 15 }))
})

const productGenerator = fc.record({
  id: fc.uuid(),
  name: fc.string({ minLength: 1, maxLength: 200 }),
  price: fc.float({ min: 0.01, max: 10000, noNaN: true }),
  stockQuantity: fc.integer({ min: 0, max: 1000 }),
  sellerId: fc.uuid(),
  categoryId: fc.uuid()
})

const cartItemGenerator = fc.record({
  productId: fc.uuid(),
  quantity: fc.integer({ min: 1, max: 10 }),
  price: fc.float({ min: 0.01, max: 1000, noNaN: true }),
  sellerId: fc.uuid()
})
```

**Property Test Examples with Required Tags**
```typescript
describe('Customer Backend API Property Tests', () => {
  
  it('Cart total calculation consistency', () => {
    fc.assert(fc.property(
      fc.array(cartItemGenerator, { minLength: 1, maxLength: 20 }),
      (cartItems) => {
        // Feature: customer-backend-api, Property 22: Cart total calculation accuracy
        const calculatedTotal = calculateCartTotal(cartItems)
        const expectedTotal = cartItems.reduce((sum, item) => 
          sum + (item.price * item.quantity), 0)
        
        expect(Math.abs(calculatedTotal - expectedTotal)).toBeLessThan(0.01)
      }
    ), { numRuns: 100 })
  })
  
  it('Registration input validation consistency', () => {
    fc.assert(fc.property(
      customerDataGenerator,
      (customerData) => {
        // Feature: customer-backend-api, Property 3: Registration input validation
        const validationResult = validateRegistrationData(customerData)
        
        // Valid data should always pass validation
        expect(validationResult.isValid).toBe(true)
        expect(validationResult.errors).toHaveLength(0)
      }
    ), { numRuns: 100 })
  })
  
  it('JSON serialization round-trip integrity', () => {
    fc.assert(fc.property(
      fc.record({
        products: fc.array(productGenerator),
        pagination: fc.record({
          page: fc.integer({ min: 1, max: 100 }),
          limit: fc.integer({ min: 1, max: 100 }),
          total: fc.integer({ min: 0, max: 10000 })
        })
      }),
      (apiResponse) => {
        // Feature: customer-backend-api, Property 31: Serialization round-trip integrity
        const serialized = JSON.stringify(apiResponse)
        const parsed = JSON.parse(serialized)
        
        expect(parsed).toEqual(apiResponse)
      }
    ), { numRuns: 100 })
  })
  
  it('Order splitting for multi-seller carts', () => {
    fc.assert(fc.property(
      fc.array(cartItemGenerator, { minLength: 2, maxLength: 10 }),
      (cartItems) => {
        // Feature: customer-backend-api, Property 27: Multi-seller order splitting
        const sellerIds = [...new Set(cartItems.map(item => item.sellerId))]
        const splitOrders = splitOrderBySeller(cartItems)
        
        // Should have one order per unique seller
        expect(splitOrders).toHaveLength(sellerIds.length)
        
        // Each order should contain only items from one seller
        splitOrders.forEach(order => {
          const orderSellerIds = [...new Set(order.items.map(item => item.sellerId))]
          expect(orderSellerIds).toHaveLength(1)
        })
        
        // Total items should be preserved
        const totalOriginalItems = cartItems.length
        const totalSplitItems = splitOrders.reduce((sum, order) => sum + order.items.length, 0)
        expect(totalSplitItems).toBe(totalOriginalItems)
      }
    ), { numRuns: 100 })
  })
})
```

#### Unit Testing Strategy

**Service Layer Testing**
```typescript
describe('AuthenticationService', () => {
  let authService: AuthenticationService
  let mockUserRepository: jest.Mocked<UserRepository>
  let mockHashService: jest.Mocked<HashService>
  
  beforeEach(() => {
    mockUserRepository = createMockUserRepository()
    mockHashService = createMockHashService()
    authService = new AuthenticationService(mockUserRepository, mockHashService)
  })
  
  it('should create user account for valid registration data', async () => {
    const registrationData = {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'SecurePass123'
    }
    
    mockUserRepository.findByEmail.mockResolvedValue(null)
    mockHashService.hash.mockResolvedValue('hashedPassword')
    mockUserRepository.create.mockResolvedValue({ id: 'user123', ...registrationData })
    
    const result = await authService.register(registrationData)
    
    expect(result.success).toBe(true)
    expect(mockHashService.hash).toHaveBeenCalledWith('SecurePass123', 12)
    expect(mockUserRepository.create).toHaveBeenCalledWith({
      ...registrationData,
      password: 'hashedPassword'
    })
  })
  
  it('should reject registration for existing email', async () => {
    const registrationData = {
      name: 'Jane Doe',
      email: 'existing@example.com',
      password: 'SecurePass123'
    }
    
    mockUserRepository.findByEmail.mockResolvedValue({ id: 'existing', email: 'existing@example.com' })
    
    await expect(authService.register(registrationData))
      .rejects.toThrow('Email already registered')
    
    expect(mockUserRepository.create).not.toHaveBeenCalled()
  })
})
```

#### Integration Testing Framework

**API Endpoint Testing**
```typescript
describe('Product API Integration Tests', () => {
  let app: Express
  let testDb: TestDatabase
  
  beforeAll(async () => {
    testDb = await setupTestDatabase()
    app = createTestApp(testDb)
  })
  
  afterAll(async () => {
    await testDb.cleanup()
  })
  
  beforeEach(async () => {
    await testDb.seed()
  })
  
  afterEach(async () => {
    await testDb.reset()
  })
  
  it('should return paginated products with correct format', async () => {
    const response = await request(app)
      .get('/api/products')
      .query({ page: 1, limit: 10 })
      .expect(200)
    
    expect(response.body).toMatchObject({
      products: expect.arrayContaining([
        expect.objectContaining({
          id: expect.any(String),
          name: expect.any(String),
          price: expect.any(Number),
          images: expect.any(Array),
          availability: expect.any(Boolean)
        })
      ]),
      pagination: expect.objectContaining({
        page: 1,
        limit: 10,
        total: expect.any(Number),
        hasNextPage: expect.any(Boolean)
      })
    })
  })
  
  it('should enforce pagination limits', async () => {
    const response = await request(app)
      .get('/api/products')
      .query({ page: 1, limit: 150 }) // Exceeds maximum of 100
      .expect(200)
    
    expect(response.body.products.length).toBeLessThanOrEqual(100)
    expect(response.body.pagination.limit).toBe(100)
  })
})
```

#### Performance and Load Testing

**Load Testing Configuration**
```typescript
describe('Performance Tests', () => {
  it('should handle concurrent cart operations', async () => {
    const userId = 'test-user-123'
    const concurrentOperations = 50
    
    const operations = Array(concurrentOperations).fill(null).map(async (_, index) => {
      return request(app)
        .post('/api/cart/items')
        .set('Authorization', `Bearer ${validToken}`)
        .send({
          productId: `product-${index}`,
          quantity: Math.floor(Math.random() * 5) + 1
        })
    })
    
    const results = await Promise.all(operations)
    
    // All operations should succeed
    results.forEach(result => {
      expect(result.status).toBe(200)
    })
    
    // Final cart should contain all items
    const finalCart = await request(app)
      .get('/api/cart')
      .set('Authorization', `Bearer ${validToken}`)
      .expect(200)
    
    expect(finalCart.body.items).toHaveLength(concurrentOperations)
  })
  
  it('should maintain response time under load', async () => {
    const startTime = Date.now()
    const requests = 100
    
    const operations = Array(requests).fill(null).map(() => 
      request(app)
        .get('/api/products')
        .expect(200)
    )
    
    await Promise.all(operations)
    
    const totalTime = Date.now() - startTime
    const averageResponseTime = totalTime / requests
    
    // 95% of requests should be under 200ms
    expect(averageResponseTime).toBeLessThan(200)
  })
})
```

#### Security Testing Integration

**Security Vulnerability Testing**
```typescript
describe('Security Tests', () => {
  it('should prevent SQL injection in search queries', async () => {
    const maliciousQueries = [
      "'; DROP TABLE customers; --",
      "1' OR '1'='1",
      "admin'/*",
      "1; DELETE FROM orders; --"
    ]
    
    for (const query of maliciousQueries) {
      const response = await request(app)
        .get('/api/products/search')
        .query({ q: query })
        .expect(400)
      
      expect(response.body.error.code).toBe('VALIDATION_ERROR')
    }
    
    // Verify database integrity after malicious attempts
    const productCount = await testDb.query('SELECT COUNT(*) FROM products')
    expect(productCount[0].count).toBeGreaterThan(0)
  })
  
  it('should enforce authentication on protected endpoints', async () => {
    const protectedEndpoints = [
      { method: 'get', path: '/api/cart' },
      { method: 'post', path: '/api/cart/items' },
      { method: 'post', path: '/api/orders' },
      { method: 'get', path: '/api/orders/history' }
    ]
    
    for (const endpoint of protectedEndpoints) {
      await request(app)
        [endpoint.method](endpoint.path)
        .expect(401)
    }
  })
  
  it('should implement proper rate limiting', async () => {
    const email = 'test@example.com'
    const invalidPassword = 'wrongpassword'
    
    // Make 5 failed login attempts
    for (let i = 0; i < 5; i++) {
      await request(app)
        .post('/api/auth/login')
        .send({ email, password: invalidPassword })
        .expect(401)
    }
    
    // 6th attempt should be rate limited
    await request(app)
      .post('/api/auth/login')
      .send({ email, password: invalidPassword })
      .expect(429)
  })
})
```

This comprehensive testing strategy ensures that the Customer Backend API maintains high quality, security, and performance standards while providing mathematical guarantees of correctness through property-based testing. The combination of unit tests for detailed validation, integration tests for system behavior, property tests for universal correctness, and security tests for vulnerability prevention creates a robust quality assurance framework that supports confident deployment and maintenance of the system.