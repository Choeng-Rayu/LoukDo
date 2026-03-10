# Design Document: Customer Backend API (Unified Architecture)

## Overview

The Customer Backend API represents the customer-facing portion of a unified NestJS backend application that serves all user types (customers, sellers, super_admin) in the multi-vendor e-commerce platform. This design implements role-based access control within a single application architecture, providing secure, scalable, and performant backend services for customer operations including product browsing, cart management, order placement, and purchase tracking.

The unified architecture eliminates the complexity of separate microservices while maintaining clear separation of concerns through NestJS modules, role-based guards, and service-layer filtering. This approach ensures data consistency, simplifies deployment, and provides seamless integration across all user types within the platform.

### Key Design Principles

- **Unified Architecture**: Single NestJS application serving all user types with role-based access control
- **Role-Based Security**: JWT tokens with role claims and NestJS guards for authorization
- **Database Consistency**: Single MySQL database with TypeORM entities and role-based filtering
- **Performance Optimized**: Sub-200ms response times through caching and query optimization
- **Scalable Design**: Horizontal scaling support through stateless NestJS architecture
- **Developer Experience**: TypeScript throughout with comprehensive OpenAPI documentation
- **Container-Ready**: Docker development environment with MySQL containerization

## Architecture

### Unified NestJS Application Architecture

The Customer Backend API is implemented as part of a unified NestJS application that serves all user types through role-based access control:

```mermaid
graph TB
    subgraph "Client Layer"
        CF[Customer Frontend]
        SF[Seller Frontend]
        AF[Admin Frontend]
        MA[Mobile Apps]
    end
    
    subgraph "Unified NestJS Application"
        AG[API Gateway/Guards]
        
        subgraph "Authentication Module"
            AS[Auth Service]
            JG[JWT Guard]
            RG[Roles Guard]
        end
        
        subgraph "Customer Module"
            PC[Products Controller]
            CC[Cart Controller]
            OC[Orders Controller]
            UC[Users Controller]
        end
        
        subgraph "Shared Services"
            PS[Product Service]
            CS[Cart Service]
            OS[Order Service]
            US[User Service]
            NS[Notification Service]
        end
        
        subgraph "Data Access Layer"
            PR[Product Repository]
            CR[Cart Repository]
            OR[Order Repository]
            UR[User Repository]
        end
    end
    
    subgraph "Infrastructure"
        subgraph "Database"
            MY[(MySQL 8.0)]
            DC[Docker Container]
        end
        
        subgraph "Caching"
            RD[(Redis Cache)]
        end
        
        subgraph "External Services"
            ES[Email Service]
            PS[Payment Service]
        end
    end
    
    CF --> AG
    SF --> AG
    AF --> AG
    MA --> AG
    
    AG --> JG
    JG --> RG
    RG --> PC
    RG --> CC
    RG --> OC
    RG --> UC
    
    PC --> PS
    CC --> CS
    OC --> OS
    UC --> US
    
    PS --> PR
    CS --> CR
    OS --> OR
    US --> UR
    
    PR --> MY
    CR --> MY
    OR --> MY
    UR --> MY
    
    MY --> DC
    
    PS --> RD
    CS --> RD
    
    NS --> ES
    OS --> PS
```

### NestJS Module Structure

The unified application is organized into focused NestJS modules with clear separation of concerns:

```typescript
// Application module structure
@Module({
  imports: [
    // Core modules
    ConfigModule.forRoot(),
    TypeOrmModule.forRoot(databaseConfig),
    
    // Feature modules
    AuthModule,
    UsersModule,
    ProductsModule,
    CartModule,
    OrdersModule,
    NotificationModule,
    
    // Infrastructure modules
    CacheModule,
    LoggerModule
  ]
})
export class AppModule {}
```

### Role-Based Access Control Architecture

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Guards
    participant S as Service
    participant R as Repository
    participant DB as MySQL
    
    C->>G: Request with JWT
    G->>G: Validate Token
    G->>G: Check Role Permissions
    G->>S: Authorized Request
    S->>S: Apply Role-Based Filtering
    S->>R: Query with Role Context
    R->>DB: Execute Filtered Query
    DB-->>R: Results
    R-->>S: Filtered Data
    S-->>G: Response
    G-->>C: Final Response
```

### Technology Stack

#### Backend Framework
- **NestJS**: TypeScript-first framework with decorators and dependency injection
- **TypeScript**: Full type safety throughout the application
- **Passport.js**: Authentication middleware with JWT strategy
- **Class Validator**: Decorator-based validation for DTOs
- **Class Transformer**: Object transformation and serialization

#### Database Technologies
- **MySQL 8.0+**: Primary relational database with ACID compliance
- **TypeORM**: Object-relational mapping with decorators and migrations
- **Docker**: Containerized MySQL for development environment
- **Redis 7+**: In-memory caching and session storage

#### Authentication & Security
- **JWT (JSON Web Tokens)**: Stateless authentication with role-based claims
- **bcrypt**: Password hashing with configurable cost factor
- **Helmet**: Security middleware for HTTP headers
- **CORS**: Cross-origin resource sharing configuration
- **Rate Limiting**: Built-in NestJS throttling with Redis backend

#### Development & Deployment
- **Docker Compose**: Development environment orchestration
- **Swagger/OpenAPI**: Automatic API documentation generation
- **Winston**: Structured logging with multiple transports
- **Jest**: Testing framework with TypeScript support

## Components and Interfaces

### NestJS Controller Design

#### Role-Based Controller Architecture
```mermaid
sequenceDiagram
    participant C as Client
    participant PC as Products Controller
    participant RG as Roles Guard
    participant PS as Product Service
    participant PR as Product Repository
    participant DB as MySQL
    
    C->>PC: GET /api/products
    PC->>RG: Check Role Permissions
    RG-->>PC: Role: customer
    PC->>PS: getProducts(filters, role)
    PS->>PS: Apply Customer Filtering
    PS->>PR: findWithFilters(customerFilters)
    PR->>DB: SELECT with role constraints
    DB-->>PR: Filtered Results
    PR-->>PS: Products
    PS-->>PC: Customer-Visible Products
    PC-->>C: HTTP Response
```

#### Authentication Module Interface
```typescript
@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '24h' }
    }),
    PassportModule
  ],
  providers: [AuthService, JwtStrategy, LocalStrategy],
  controllers: [AuthController],
  exports: [AuthService]
})
export class AuthModule {}

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(email: string, password: string): Promise<any> {
    const user = await this.usersService.findByEmail(email)
    if (user && await bcrypt.compare(password, user.password)) {
      const { password, ...result } = user
      return result
    }
    return null
  }

  async login(user: any) {
    const payload = { 
      email: user.email, 
      sub: user.id, 
      role: user.role,
      permissions: user.permissions 
    }
    return {
      access_token: this.jwtService.sign(payload),
      user: user
    }
  }

  async register(createUserDto: CreateUserDto): Promise<User> {
    const hashedPassword = await bcrypt.hash(createUserDto.password, 12)
    return this.usersService.create({
      ...createUserDto,
      password: hashedPassword,
      role: UserRole.CUSTOMER
    })
  }
}
```

#### Products Controller with Role-Based Access
```typescript
@Controller('api/products')
@UseGuards(JwtAuthGuard, RolesGuard)
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  @Roles(UserRole.CUSTOMER, UserRole.SELLER, UserRole.SUPER_ADMIN)
  async getProducts(
    @Query() filters: ProductFiltersDto,
    @Query() pagination: PaginationDto,
    @Request() req
  ): Promise<ProductPageDto> {
    return this.productsService.findByRole(
      filters, 
      pagination, 
      req.user.role
    )
  }

  @Get(':id')
  @Roles(UserRole.CUSTOMER, UserRole.SELLER, UserRole.SUPER_ADMIN)
  async getProduct(
    @Param('id') id: string,
    @Request() req
  ): Promise<ProductDetailDto> {
    return this.productsService.findOneByRole(id, req.user.role)
  }

  @Get('search')
  @Roles(UserRole.CUSTOMER)
  async searchProducts(
    @Query() searchQuery: SearchQueryDto
  ): Promise<ProductPageDto> {
    return this.productsService.search(searchQuery)
  }
}
```

#### Cart Controller with Customer-Specific Logic
```typescript
@Controller('api/cart')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.CUSTOMER)
export class CartController {
  constructor(private readonly cartService: CartService) {}

  @Get()
  async getCart(@Request() req): Promise<CartDto> {
    return this.cartService.getCartByUserId(req.user.id)
  }

  @Post('items')
  async addToCart(
    @Body() addItemDto: AddCartItemDto,
    @Request() req
  ): Promise<CartDto> {
    return this.cartService.addItem(req.user.id, addItemDto)
  }

  @Put('items/:itemId')
  async updateCartItem(
    @Param('itemId') itemId: string,
    @Body() updateDto: UpdateCartItemDto,
    @Request() req
  ): Promise<CartDto> {
    return this.cartService.updateItem(req.user.id, itemId, updateDto)
  }

  @Delete('items/:itemId')
  async removeFromCart(
    @Param('itemId') itemId: string,
    @Request() req
  ): Promise<CartDto> {
    return this.cartService.removeItem(req.user.id, itemId)
  }
}
```

#### Orders Controller with Multi-Seller Support
```typescript
@Controller('api/orders')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.CUSTOMER)
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Post()
  async createOrder(
    @Body() createOrderDto: CreateOrderDto,
    @Request() req
  ): Promise<OrderDto> {
    return this.ordersService.createOrder(req.user.id, createOrderDto)
  }

  @Get()
  async getOrderHistory(
    @Query() pagination: PaginationDto,
    @Request() req
  ): Promise<OrderPageDto> {
    return this.ordersService.getOrderHistory(req.user.id, pagination)
  }

  @Get(':id')
  async getOrder(
    @Param('id') orderId: string,
    @Request() req
  ): Promise<OrderDetailDto> {
    return this.ordersService.getOrderById(orderId, req.user.id)
  }

  @Put(':id/cancel')
  async cancelOrder(
    @Param('id') orderId: string,
    @Request() req
  ): Promise<OrderDto> {
    return this.ordersService.cancelOrder(orderId, req.user.id)
  }
}
```

### Service Layer with Role-Based Filtering

#### Product Service Implementation
```typescript
@Injectable()
export class ProductsService {
  constructor(
    @InjectRepository(Product)
    private productRepository: Repository<Product>,
    private cacheManager: CacheManager
  ) {}

  async findByRole(
    filters: ProductFiltersDto,
    pagination: PaginationDto,
    userRole: UserRole
  ): Promise<ProductPageDto> {
    const cacheKey = `products:${userRole}:${JSON.stringify(filters)}:${pagination.page}`
    
    let products = await this.cacheManager.get(cacheKey)
    if (!products) {
      const queryBuilder = this.productRepository
        .createQueryBuilder('product')
        .leftJoinAndSelect('product.category', 'category')
        .leftJoinAndSelect('product.seller', 'seller')

      // Apply role-based filtering
      this.applyRoleFilters(queryBuilder, userRole)
      this.applyUserFilters(queryBuilder, filters)
      
      const [items, total] = await queryBuilder
        .skip((pagination.page - 1) * pagination.limit)
        .take(pagination.limit)
        .getManyAndCount()

      products = {
        products: items.map(product => this.transformForRole(product, userRole)),
        pagination: {
          page: pagination.page,
          limit: pagination.limit,
          total,
          hasNextPage: total > pagination.page * pagination.limit
        }
      }

      await this.cacheManager.set(cacheKey, products, 300) // 5 minutes
    }

    return products
  }

  private applyRoleFilters(
    queryBuilder: SelectQueryBuilder<Product>,
    userRole: UserRole
  ): void {
    switch (userRole) {
      case UserRole.CUSTOMER:
        queryBuilder
          .andWhere('product.isActive = :isActive', { isActive: true })
          .andWhere('product.stockQuantity > 0')
          .andWhere('seller.isApproved = :isApproved', { isApproved: true })
        break
      case UserRole.SELLER:
        // Sellers see their own products regardless of status
        queryBuilder.andWhere('product.sellerId = :sellerId', { 
          sellerId: this.getCurrentUserId() 
        })
        break
      case UserRole.SUPER_ADMIN:
        // Admin sees all products
        break
    }
  }

  private transformForRole(product: Product, userRole: UserRole): any {
    const base = {
      id: product.id,
      name: product.name,
      price: product.price,
      images: product.images,
      category: product.category
    }

    switch (userRole) {
      case UserRole.CUSTOMER:
        return {
          ...base,
          availability: product.stockQuantity > 0,
          averageRating: product.averageRating,
          reviewCount: product.reviewCount
        }
      case UserRole.SELLER:
        return {
          ...base,
          stockQuantity: product.stockQuantity,
          isActive: product.isActive,
          createdAt: product.createdAt
        }
      case UserRole.SUPER_ADMIN:
        return {
          ...base,
          stockQuantity: product.stockQuantity,
          isActive: product.isActive,
          seller: product.seller,
          createdAt: product.createdAt,
          updatedAt: product.updatedAt
        }
    }
  }
}
```

### Guard Implementation

#### JWT Authentication Guard
```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context)
  }

  handleRequest(err, user, info) {
    if (err || !user) {
      throw err || new UnauthorizedException()
    }
    return user
  }
}
```

#### Role-Based Authorization Guard
```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()]
    )

    if (!requiredRoles) {
      return true
    }

    const { user } = context.switchToHttp().getRequest()
    return requiredRoles.some((role) => user.role === role)
  }
}

export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles)
export const ROLES_KEY = 'roles'
```

## Data Models

### TypeORM Entity Relationships

The unified architecture uses a single MySQL database with TypeORM entities that support role-based access:

```mermaid
erDiagram
    USERS ||--o{ ORDERS : places
    USERS ||--o{ CART_ITEMS : has
    USERS ||--o{ ADDRESSES : owns
    USERS ||--o{ REVIEWS : writes
    USERS ||--o{ WISHLISTS : maintains
    USERS ||--o{ PRODUCTS : sells
    
    ORDERS ||--o{ ORDER_ITEMS : contains
    ORDERS }o--|| ADDRESSES : ships_to
    
    PRODUCTS ||--o{ ORDER_ITEMS : ordered
    PRODUCTS ||--o{ CART_ITEMS : added_to_cart
    PRODUCTS ||--o{ REVIEWS : reviewed
    PRODUCTS ||--o{ WISHLIST_ITEMS : saved_to
    PRODUCTS }o--|| CATEGORIES : belongs_to
    
    WISHLISTS ||--o{ WISHLIST_ITEMS : contains
    
    USERS {
        uuid id PK
        string email UK
        string password
        string name
        string phone
        enum role
        boolean email_verified
        boolean is_approved
        timestamp created_at
        timestamp updated_at
        timestamp last_login
    }
    
    ADDRESSES {
        uuid id PK
        uuid user_id FK
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
        uuid user_id FK
        string session_token
        uuid product_id FK
        integer quantity
        timestamp created_at
        timestamp updated_at
    }
    
    REVIEWS {
        uuid id PK
        uuid user_id FK
        uuid product_id FK
        uuid order_id FK
        integer rating
        text comment
        boolean is_verified_purchase
        timestamp created_at
    }
    
    WISHLISTS {
        uuid id PK
        uuid user_id FK
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

### TypeORM Entity Definitions

#### User Entity (Unified for All Roles)
```typescript
export enum UserRole {
  CUSTOMER = 'customer',
  SELLER = 'seller',
  SUPER_ADMIN = 'super_admin'
}

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column({ unique: true })
  email: string

  @Column()
  password: string

  @Column()
  name: string

  @Column({ nullable: true })
  phone: string

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.CUSTOMER
  })
  role: UserRole

  @Column({ default: false })
  emailVerified: boolean

  @Column({ default: true })
  isApproved: boolean // For seller approval workflow

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date

  @Column({ nullable: true })
  lastLogin: Date

  // Relations
  @OneToMany(() => Address, address => address.user)
  addresses: Address[]

  @OneToMany(() => Order, order => order.customer)
  orders: Order[]

  @OneToMany(() => CartItem, cartItem => cartItem.user)
  cartItems: CartItem[]

  @OneToMany(() => Review, review => review.user)
  reviews: Review[]

  @OneToMany(() => Wishlist, wishlist => wishlist.user)
  wishlists: Wishlist[]

  @OneToMany(() => Product, product => product.seller)
  products: Product[] // For sellers
}
```

#### Product Entity with Role-Based Access
```typescript
@Entity('products')
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  name: string

  @Column('text')
  description: string

  @Column('decimal', { precision: 10, scale: 2 })
  price: number

  @Column('int')
  stockQuantity: number

  @Column({ default: true })
  isActive: boolean

  @Column('json', { default: '[]' })
  images: string[]

  @Column('decimal', { precision: 3, scale: 2, default: 0 })
  averageRating: number

  @Column('int', { default: 0 })
  reviewCount: number

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date

  // Relations
  @ManyToOne(() => User, user => user.products)
  @JoinColumn({ name: 'seller_id' })
  seller: User

  @Column('uuid')
  sellerId: string

  @ManyToOne(() => Category, category => category.products)
  @JoinColumn({ name: 'category_id' })
  category: Category

  @Column('uuid')
  categoryId: string

  @OneToMany(() => OrderItem, orderItem => orderItem.product)
  orderItems: OrderItem[]

  @OneToMany(() => CartItem, cartItem => cartItem.product)
  cartItems: CartItem[]

  @OneToMany(() => Review, review => review.product)
  reviews: Review[]

  @OneToMany(() => WishlistItem, wishlistItem => wishlistItem.product)
  wishlistItems: WishlistItem[]
}
```

#### Order Entity with Multi-Seller Support
```typescript
export enum OrderStatus {
  PLACED = 'placed',
  CONFIRMED = 'confirmed',
  PROCESSING = 'processing',
  SHIPPED = 'shipped',
  OUT_FOR_DELIVERY = 'out_for_delivery',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled'
}

export enum PaymentMethod {
  COD = 'cod',
  ONLINE = 'online',
  WALLET = 'wallet'
}

export enum PaymentStatus {
  PENDING = 'pending',
  COMPLETED = 'completed',
  FAILED = 'failed',
  REFUNDED = 'refunded'
}

@Entity('orders')
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column({ unique: true })
  orderNumber: string

  @Column({
    type: 'enum',
    enum: OrderStatus,
    default: OrderStatus.PLACED
  })
  status: OrderStatus

  @Column('decimal', { precision: 10, scale: 2 })
  subtotal: number

  @Column('decimal', { precision: 10, scale: 2, default: 0 })
  taxAmount: number

  @Column('decimal', { precision: 10, scale: 2, default: 0 })
  shippingCost: number

  @Column('decimal', { precision: 10, scale: 2 })
  totalAmount: number

  @Column({
    type: 'enum',
    enum: PaymentMethod
  })
  paymentMethod: PaymentMethod

  @Column({
    type: 'enum',
    enum: PaymentStatus,
    default: PaymentStatus.PENDING
  })
  paymentStatus: PaymentStatus

  @Column('text', { nullable: true })
  specialInstructions: string

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date

  @Column({ nullable: true })
  estimatedDelivery: Date

  // Relations
  @ManyToOne(() => User, user => user.orders)
  @JoinColumn({ name: 'customer_id' })
  customer: User

  @Column('uuid')
  customerId: string

  @ManyToOne(() => Address, address => address.orders)
  @JoinColumn({ name: 'shipping_address_id' })
  shippingAddress: Address

  @Column('uuid')
  shippingAddressId: string

  @OneToMany(() => OrderItem, orderItem => orderItem.order)
  items: OrderItem[]
}
```

#### Cart Entity for Customer Sessions
```typescript
@Entity('cart_items')
export class CartItem {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column('int')
  quantity: number

  @Column({ nullable: true })
  sessionToken: string // For guest carts

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date

  // Relations
  @ManyToOne(() => User, user => user.cartItems, { nullable: true })
  @JoinColumn({ name: 'user_id' })
  user: User

  @Column('uuid', { nullable: true })
  userId: string

  @ManyToOne(() => Product, product => product.cartItems)
  @JoinColumn({ name: 'product_id' })
  product: Product

  @Column('uuid')
  productId: string
}
```

### Docker Database Configuration

#### Docker Compose Setup
```yaml
# docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: ecommerce_mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ecommerce
      MYSQL_USER: app_user
      MYSQL_PASSWORD: app_password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init:/docker-entrypoint-initdb.d
    command: --default-authentication-plugin=mysql_native_password
    
  redis:
    image: redis:7-alpine
    container_name: ecommerce_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  mysql_data:
  redis_data:
```

#### TypeORM Configuration
```typescript
// src/config/database.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm'
import { ConfigService } from '@nestjs/config'

export const getDatabaseConfig = (configService: ConfigService): TypeOrmModuleOptions => ({
  type: 'mysql',
  host: configService.get('DB_HOST', 'localhost'),
  port: configService.get('DB_PORT', 3306),
  username: configService.get('DB_USERNAME', 'root'),
  password: configService.get('DB_PASSWORD', 'root'),
  database: configService.get('DB_NAME', 'ecommerce'),
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: configService.get('NODE_ENV') === 'development',
  migrations: [__dirname + '/../database/migrations/*{.ts,.js}'],
  migrationsRun: true,
  logging: configService.get('NODE_ENV') === 'development',
  extra: {
    connectionLimit: 10,
    acquireTimeout: 60000,
    timeout: 60000
  }
})
```

### Repository Pattern with Role-Based Filtering

#### Base Repository with Role Support
```typescript
export abstract class BaseRepository<T> {
  constructor(
    protected repository: Repository<T>,
    protected entityName: string
  ) {}

  async findByRole(
    filters: any,
    userRole: UserRole,
    userId?: string
  ): Promise<T[]> {
    const queryBuilder = this.repository.createQueryBuilder(this.entityName)
    
    this.applyRoleFilters(queryBuilder, userRole, userId)
    this.applyUserFilters(queryBuilder, filters)
    
    return queryBuilder.getMany()
  }

  protected abstract applyRoleFilters(
    queryBuilder: SelectQueryBuilder<T>,
    userRole: UserRole,
    userId?: string
  ): void

  protected abstract applyUserFilters(
    queryBuilder: SelectQueryBuilder<T>,
    filters: any
  ): void
}
```

#### Product Repository Implementation
```typescript
@Injectable()
export class ProductRepository extends BaseRepository<Product> {
  constructor(
    @InjectRepository(Product)
    productRepository: Repository<Product>
  ) {
    super(productRepository, 'product')
  }

  protected applyRoleFilters(
    queryBuilder: SelectQueryBuilder<Product>,
    userRole: UserRole,
    userId?: string
  ): void {
    switch (userRole) {
      case UserRole.CUSTOMER:
        queryBuilder
          .andWhere('product.isActive = :isActive', { isActive: true })
          .andWhere('product.stockQuantity > 0')
          .leftJoin('product.seller', 'seller')
          .andWhere('seller.isApproved = :isApproved', { isApproved: true })
        break
        
      case UserRole.SELLER:
        if (userId) {
          queryBuilder.andWhere('product.sellerId = :sellerId', { sellerId: userId })
        }
        break
        
      case UserRole.SUPER_ADMIN:
        // Admin sees all products
        break
    }
  }

  protected applyUserFilters(
    queryBuilder: SelectQueryBuilder<Product>,
    filters: any
  ): void {
    if (filters.categoryId) {
      queryBuilder.andWhere('product.categoryId = :categoryId', { 
        categoryId: filters.categoryId 
      })
    }
    
    if (filters.priceRange) {
      queryBuilder.andWhere('product.price BETWEEN :minPrice AND :maxPrice', {
        minPrice: filters.priceRange.min,
        maxPrice: filters.priceRange.max
      })
    }
    
    if (filters.searchTerm) {
      queryBuilder.andWhere(
        '(product.name LIKE :searchTerm OR product.description LIKE :searchTerm)',
        { searchTerm: `%${filters.searchTerm}%` }
      )
    }
  }
}
```

### Caching Strategy with Redis

#### NestJS Cache Configuration
```typescript
// src/config/cache.config.ts
import { CacheModule } from '@nestjs/cache-manager'
import { redisStore } from 'cache-manager-redis-store'

export const getCacheConfig = () => CacheModule.register({
  store: redisStore,
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
  ttl: 300, // 5 minutes default
  max: 1000 // Maximum number of items in cache
})
```

#### Cache Service Implementation
```typescript
@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  // Product caching
  async cacheProduct(productId: string, product: Product, ttl = 300): Promise<void> {
    const key = CacheKeys.PRODUCT(productId)
    await this.cacheManager.set(key, product, ttl)
  }

  async getCachedProduct(productId: string): Promise<Product | null> {
    const key = CacheKeys.PRODUCT(productId)
    return await this.cacheManager.get(key)
  }

  async invalidateProductCache(productId: string): Promise<void> {
    const key = CacheKeys.PRODUCT(productId)
    await this.cacheManager.del(key)
  }

  // Cart caching
  async cacheCart(userId: string, cart: any, ttl = 600): Promise<void> {
    const key = CacheKeys.CART(userId)
    await this.cacheManager.set(key, cart, ttl)
  }

  async getCachedCart(userId: string): Promise<any | null> {
    const key = CacheKeys.CART(userId)
    return await this.cacheManager.get(key)
  }

  async invalidateCartCache(userId: string): Promise<void> {
    const key = CacheKeys.CART(userId)
    await this.cacheManager.del(key)
  }
}

const CacheKeys = {
  PRODUCT: (id: string) => `product:${id}`,
  PRODUCT_LIST: (filters: string) => `products:${filters}`,
  SEARCH_RESULTS: (query: string) => `search:${Buffer.from(query).toString('base64')}`,
  CART: (userId: string) => `cart:${userId}`,
  USER_SESSION: (sessionId: string) => `session:${sessionId}`,
  CATEGORY_TREE: () => 'categories:tree'
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

### NestJS Exception Handling Strategy

The unified NestJS application implements comprehensive error handling through built-in exception filters and custom error responses that maintain consistency across all user roles.

#### Global Exception Filter
```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name)

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse<Response>()
    const request = ctx.getRequest<Request>()

    let status: number
    let message: string
    let code: string

    if (exception instanceof HttpException) {
      status = exception.getStatus()
      const exceptionResponse = exception.getResponse()
      
      if (typeof exceptionResponse === 'object' && exceptionResponse !== null) {
        message = (exceptionResponse as any).message || exception.message
        code = (exceptionResponse as any).code || this.getErrorCode(status)
      } else {
        message = exceptionResponse as string
        code = this.getErrorCode(status)
      }
    } else if (exception instanceof QueryFailedError) {
      status = 400
      message = 'Database operation failed'
      code = 'DATABASE_ERROR'
      this.logger.error('Database error:', exception)
    } else {
      status = 500
      message = 'Internal server error'
      code = 'INTERNAL_ERROR'
      this.logger.error('Unexpected error:', exception)
    }

    const errorResponse = {
      error: {
        code,
        message,
        timestamp: new Date().toISOString(),
        requestId: request.headers['x-request-id'] || 'unknown',
        path: request.url
      }
    }

    response.status(status).json(errorResponse)
  }

  private getErrorCode(status: number): string {
    const codes = {
      400: 'BAD_REQUEST',
      401: 'UNAUTHORIZED',
      403: 'FORBIDDEN',
      404: 'NOT_FOUND',
      409: 'CONFLICT',
      422: 'VALIDATION_ERROR',
      429: 'RATE_LIMIT_EXCEEDED',
      500: 'INTERNAL_ERROR'
    }
    return codes[status] || 'UNKNOWN_ERROR'
  }
}
```

#### Custom Business Exception Classes
```typescript
export class BusinessException extends HttpException {
  constructor(
    message: string,
    code: string,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
    details?: any
  ) {
    super(
      {
        message,
        code,
        details
      },
      status
    )
  }
}

export class ValidationException extends BusinessException {
  constructor(message: string, details?: any) {
    super(message, 'VALIDATION_ERROR', HttpStatus.UNPROCESSABLE_ENTITY, details)
  }
}

export class ResourceNotFoundException extends BusinessException {
  constructor(resource: string, id: string) {
    super(
      `${resource} with id ${id} not found`,
      'RESOURCE_NOT_FOUND',
      HttpStatus.NOT_FOUND
    )
  }
}

export class InsufficientStockException extends BusinessException {
  constructor(productId: string, requested: number, available: number) {
    super(
      'Insufficient stock available',
      'INSUFFICIENT_STOCK',
      HttpStatus.CONFLICT,
      { productId, requested, available }
    )
  }
}
```

#### Validation Pipe Configuration
```typescript
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    exceptionFactory: (errors: ValidationError[]) => {
      const details = errors.reduce((acc, error) => {
        acc[error.property] = Object.values(error.constraints || {}).join(', ')
        return acc
      }, {})
      
      return new ValidationException('Request validation failed', details)
    }
  })
)
```

### Error Response Format

#### Standardized Error Structure
```typescript
interface APIErrorResponse {
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
    "message": "Request validation failed",
    "details": {
      "email": "must be a valid email",
      "password": "must be at least 8 characters"
    },
    "timestamp": "2024-03-10T10:30:00Z",
    "requestId": "req_123456789",
    "path": "/api/auth/register"
  }
}
```

### Role-Based Error Handling

#### Service-Level Error Handling
```typescript
@Injectable()
export class ProductsService {
  async findOneByRole(id: string, userRole: UserRole, userId?: string): Promise<Product> {
    const product = await this.productRepository.findOne({
      where: { id },
      relations: ['seller', 'category']
    })

    if (!product) {
      throw new ResourceNotFoundException('Product', id)
    }

    // Role-based access validation
    switch (userRole) {
      case UserRole.CUSTOMER:
        if (!product.isActive || product.stockQuantity <= 0) {
          throw new ResourceNotFoundException('Product', id)
        }
        if (!product.seller.isApproved) {
          throw new BusinessException(
            'Product not available',
            'PRODUCT_UNAVAILABLE',
            HttpStatus.NOT_FOUND
          )
        }
        break

      case UserRole.SELLER:
        if (product.sellerId !== userId) {
          throw new BusinessException(
            'Access denied to this product',
            'ACCESS_DENIED',
            HttpStatus.FORBIDDEN
          )
        }
        break

      case UserRole.SUPER_ADMIN:
        // Admin can access any product
        break
    }

    return product
  }
}
```

### Rate Limiting and Throttling

#### NestJS Throttler Configuration
```typescript
// app.module.ts
@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60, // 1 minute
      limit: 100, // 100 requests per minute
      storage: new ThrottlerStorageRedisService(redisClient)
    })
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard
    }
  ]
})
export class AppModule {}

// Custom rate limiting for auth endpoints
@Controller('api/auth')
@UseGuards(ThrottlerGuard)
export class AuthController {
  @Post('login')
  @Throttle(5, 60) // 5 attempts per minute
  async login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto)
  }

  @Post('register')
  @Throttle(3, 300) // 3 registrations per 5 minutes
  async register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto)
  }
}
```

## Testing Strategy

### NestJS Testing Framework

The unified NestJS application employs a comprehensive testing strategy that combines traditional testing methodologies with property-based testing to ensure both functional correctness and mathematical guarantees of system behavior across all user roles.

#### Testing Pyramid Structure

**Unit Tests (Foundation - 70% of tests)**
- **Scope**: Individual services, controllers, and isolated components
- **Framework**: Jest with NestJS testing utilities and TypeScript support
- **Coverage Target**: 95%+ code coverage for business logic
- **Focus Areas**:
  - Role-based business logic validation
  - Service method functionality and error handling
  - Guard and pipe behavior verification
  - Repository method testing with mocked TypeORM

**Integration Tests (Middle Layer - 20% of tests)**
- **Scope**: Module interactions, database operations, and API endpoints
- **Framework**: Jest with NestJS Test Module and Supertest for HTTP testing
- **Environment**: In-memory SQLite or Docker MySQL for isolated testing
- **Focus Areas**:
  - Controller endpoint functionality across all roles
  - Database transaction integrity and TypeORM operations
  - Authentication and authorization workflows
  - Role-based access control validation

**Property-Based Tests (Correctness Layer - 8% of tests)**
- **Framework**: fast-check for JavaScript/TypeScript property testing
- **Configuration**: Minimum 100 iterations per property with configurable seed values
- **Focus Areas**:
  - Universal properties that must hold for all valid inputs
  - Role-based filtering correctness across user types
  - Mathematical correctness of calculations and transformations
  - System invariants and consistency guarantees

**End-to-End Tests (Integration Layer - 2% of tests)**
- **Scope**: Complete user workflows across the entire unified application
- **Framework**: Jest with full application bootstrap
- **Environment**: Docker containers with MySQL and Redis
- **Focus Areas**:
  - Critical customer, seller, and admin journey validation
  - Cross-role integration verification
  - Performance under realistic load conditions
  - Security vulnerability and penetration testing

#### NestJS Unit Testing Implementation

**Service Testing with Dependency Injection**
```typescript
describe('ProductsService', () => {
  let service: ProductsService
  let repository: Repository<Product>
  let cacheService: CacheService

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ProductsService,
        {
          provide: getRepositoryToken(Product),
          useClass: Repository
        },
        {
          provide: CacheService,
          useValue: {
            getCachedProduct: jest.fn(),
            cacheProduct: jest.fn(),
            invalidateProductCache: jest.fn()
          }
        }
      ]
    }).compile()

    service = module.get<ProductsService>(ProductsService)
    repository = module.get<Repository<Product>>(getRepositoryToken(Product))
    cacheService = module.get<CacheService>(CacheService)
  })

  describe('findByRole', () => {
    it('should filter products for customer role', async () => {
      const mockProducts = [
        { id: '1', name: 'Product 1', isActive: true, stockQuantity: 10 },
        { id: '2', name: 'Product 2', isActive: false, stockQuantity: 5 }
      ]

      jest.spyOn(repository, 'createQueryBuilder').mockReturnValue({
        leftJoinAndSelect: jest.fn().mockReturnThis(),
        andWhere: jest.fn().mockReturnThis(),
        skip: jest.fn().mockReturnThis(),
        take: jest.fn().mockReturnThis(),
        getManyAndCount: jest.fn().mockResolvedValue([mockProducts, 2])
      } as any)

      const result = await service.findByRole(
        {},
        { page: 1, limit: 10 },
        UserRole.CUSTOMER
      )

      expect(result.products).toHaveLength(2)
      expect(repository.createQueryBuilder).toHaveBeenCalled()
    })

    it('should show seller-specific products for seller role', async () => {
      const sellerId = 'seller-123'
      const mockProducts = [
        { id: '1', name: 'Seller Product', sellerId, isActive: false }
      ]

      jest.spyOn(repository, 'createQueryBuilder').mockReturnValue({
        leftJoinAndSelect: jest.fn().mockReturnThis(),
        andWhere: jest.fn().mockReturnThis(),
        skip: jest.fn().mockReturnThis(),
        take: jest.fn().mockReturnThis(),
        getManyAndCount: jest.fn().mockResolvedValue([mockProducts, 1])
      } as any)

      const result = await service.findByRole(
        {},
        { page: 1, limit: 10 },
        UserRole.SELLER,
        sellerId
      )

      expect(result.products).toHaveLength(1)
      expect(result.products[0].sellerId).toBe(sellerId)
    })
  })
})
```

**Controller Testing with Guards**
```typescript
describe('ProductsController', () => {
  let controller: ProductsController
  let service: ProductsService

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [ProductsController],
      providers: [
        {
          provide: ProductsService,
          useValue: {
            findByRole: jest.fn(),
            findOneByRole: jest.fn()
          }
        }
      ]
    })
    .overrideGuard(JwtAuthGuard)
    .useValue({ canActivate: () => true })
    .overrideGuard(RolesGuard)
    .useValue({ canActivate: () => true })
    .compile()

    controller = module.get<ProductsController>(ProductsController)
    service = module.get<ProductsService>(ProductsService)
  })

  it('should get products with role-based filtering', async () => {
    const mockRequest = {
      user: { id: 'user-123', role: UserRole.CUSTOMER }
    }
    const mockProducts = {
      products: [{ id: '1', name: 'Product 1' }],
      pagination: { page: 1, limit: 10, total: 1, hasNextPage: false }
    }

    jest.spyOn(service, 'findByRole').mockResolvedValue(mockProducts)

    const result = await controller.getProducts(
      {},
      { page: 1, limit: 10 },
      mockRequest as any
    )

    expect(result).toEqual(mockProducts)
    expect(service.findByRole).toHaveBeenCalledWith(
      {},
      { page: 1, limit: 10 },
      UserRole.CUSTOMER
    )
  })
})
```

#### Integration Testing with Test Database

**Database Integration Testing**
```typescript
describe('ProductsController (Integration)', () => {
  let app: INestApplication
  let productRepository: Repository<Product>
  let userRepository: Repository<User>

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'sqlite',
          database: ':memory:',
          entities: [User, Product, Category],
          synchronize: true
        }),
        TypeOrmModule.forFeature([User, Product, Category]),
        ProductsModule,
        AuthModule
      ]
    }).compile()

    app = moduleFixture.createNestApplication()
    await app.init()

    productRepository = moduleFixture.get<Repository<Product>>(
      getRepositoryToken(Product)
    )
    userRepository = moduleFixture.get<Repository<User>>(
      getRepositoryToken(User)
    )
  })

  beforeEach(async () => {
    await productRepository.clear()
    await userRepository.clear()
    
    // Seed test data
    await this.seedTestData()
  })

  it('should return customer-visible products only', async () => {
    const customerToken = await this.getCustomerToken()

    const response = await request(app.getHttpServer())
      .get('/api/products')
      .set('Authorization', `Bearer ${customerToken}`)
      .expect(200)

    expect(response.body.products).toHaveLength(2) // Only active products
    response.body.products.forEach(product => {
      expect(product.isActive).toBe(true)
      expect(product.stockQuantity).toBeGreaterThan(0)
    })
  })

  it('should return seller-specific products for seller', async () => {
    const sellerToken = await this.getSellerToken()

    const response = await request(app.getHttpServer())
      .get('/api/products')
      .set('Authorization', `Bearer ${sellerToken}`)
      .expect(200)

    expect(response.body.products).toHaveLength(1) // Only seller's products
    expect(response.body.products[0].sellerId).toBe(this.testSellerId)
  })

  private async seedTestData() {
    // Create test users and products
    const customer = await userRepository.save({
      email: 'customer@test.com',
      password: 'hashedpassword',
      name: 'Test Customer',
      role: UserRole.CUSTOMER
    })

    const seller = await userRepository.save({
      email: 'seller@test.com',
      password: 'hashedpassword',
      name: 'Test Seller',
      role: UserRole.SELLER,
      isApproved: true
    })

    await productRepository.save([
      {
        name: 'Active Product 1',
        price: 100,
        stockQuantity: 10,
        isActive: true,
        sellerId: seller.id
      },
      {
        name: 'Active Product 2',
        price: 200,
        stockQuantity: 5,
        isActive: true,
        sellerId: seller.id
      },
      {
        name: 'Inactive Product',
        price: 150,
        stockQuantity: 0,
        isActive: false,
        sellerId: seller.id
      }
    ])

    this.testSellerId = seller.id
  }
})
```

#### Property-Based Testing Implementation

**Property Test Examples with Required Tags**
```typescript
describe('Customer Backend API Property Tests', () => {
  
  it('Cart total calculation consistency across roles', () => {
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
  
  it('Role-based product filtering consistency', () => {
    fc.assert(fc.property(
      fc.array(productGenerator, { minLength: 5, maxLength: 50 }),
      fc.constantFrom(UserRole.CUSTOMER, UserRole.SELLER, UserRole.SUPER_ADMIN),
      (products, userRole) => {
        // Feature: customer-backend-api, Property 15: Product listing response format
        const filteredProducts = applyRoleBasedFiltering(products, userRole)
        
        switch (userRole) {
          case UserRole.CUSTOMER:
            filteredProducts.forEach(product => {
              expect(product.isActive).toBe(true)
              expect(product.stockQuantity).toBeGreaterThan(0)
            })
            break
          case UserRole.SELLER:
            // Seller sees their own products regardless of status
            expect(filteredProducts.length).toBeLessThanOrEqual(products.length)
            break
          case UserRole.SUPER_ADMIN:
            // Admin sees all products
            expect(filteredProducts.length).toBe(products.length)
            break
        }
      }
    ), { numRuns: 100 })
  })
  
  it('Order splitting preserves item integrity', () => {
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

  it('JWT token role claims consistency', () => {
    fc.assert(fc.property(
      fc.record({
        id: fc.uuid(),
        email: fc.emailAddress(),
        role: fc.constantFrom(UserRole.CUSTOMER, UserRole.SELLER, UserRole.SUPER_ADMIN),
        permissions: fc.array(fc.string(), { minLength: 0, maxLength: 10 })
      }),
      (userData) => {
        // Feature: customer-backend-api, Property 12: JWT token structure and expiration
        const token = generateJWTToken(userData)
        const decoded = verifyJWTToken(token)
        
        expect(decoded.sub).toBe(userData.id)
        expect(decoded.role).toBe(userData.role)
        expect(decoded.permissions).toEqual(userData.permissions)
        expect(decoded.exp).toBeGreaterThan(Date.now() / 1000)
      }
    ), { numRuns: 100 })
  })
})
```

#### Performance and Load Testing

**NestJS Performance Testing**
```typescript
describe('Performance Tests', () => {
  let app: INestApplication

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule]
    }).compile()

    app = moduleFixture.createNestApplication()
    await app.init()
  })

  it('should handle concurrent requests efficiently', async () => {
    const concurrentRequests = 100
    const startTime = Date.now()

    const requests = Array(concurrentRequests).fill(null).map(() =>
      request(app.getHttpServer())
        .get('/api/products')
        .set('Authorization', `Bearer ${validToken}`)
    )

    const responses = await Promise.all(requests)
    const endTime = Date.now()

    // All requests should succeed
    responses.forEach(response => {
      expect(response.status).toBe(200)
    })

    // Average response time should be under 200ms
    const averageResponseTime = (endTime - startTime) / concurrentRequests
    expect(averageResponseTime).toBeLessThan(200)
  })

  it('should maintain database connection pool under load', async () => {
    const heavyRequests = 50

    const requests = Array(heavyRequests).fill(null).map(async (_, index) => {
      return request(app.getHttpServer())
        .post('/api/cart/items')
        .set('Authorization', `Bearer ${validToken}`)
        .send({
          productId: `product-${index}`,
          quantity: Math.floor(Math.random() * 5) + 1
        })
    })

    const responses = await Promise.all(requests)

    // All database operations should succeed
    responses.forEach(response => {
      expect([200, 201]).toContain(response.status)
    })
  })
})
```