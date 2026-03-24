# Implementation Plan: Seller Frontend Interface

## Overview

This implementation plan breaks down the seller-facing portion of the unified Next.js 14+ e-commerce application into discrete, actionable coding tasks. The seller interface is implemented as role-based routes under `/seller/*` within the single Next.js application using the App Router architecture, TypeScript, Tailwind CSS, Zustand for state management, and NextAuth.js for authentication.

The implementation follows an incremental approach: core infrastructure and authentication first, then product management, order management, dashboard, and finally optional Priority 2 and 3 features. Each task builds on previous work and includes property-based tests to validate correctness.

## Tasks

- [ ] 1. Set up seller route structure and shared infrastructure
  - Create `src/app/seller/` route group with App Router conventions
  - Create `src/app/seller/layout.tsx` with `RoleGuard` wrapping seller routes (allowedRoles: ['seller'])
  - Create `src/app/seller/page.tsx` as the dashboard entry point (placeholder)
  - Create `src/app/seller/products/`, `src/app/seller/orders/`, `src/app/seller/analytics/`, `src/app/seller/profile/` route directories
  - Define TypeScript interfaces: `Seller`, `Product`, `Order`, `OrderStatus`, `DashboardMetrics`, `SalesReport`, `DateRange` in `src/types/seller.ts`
  - Create `src/lib/api/seller.ts` with `SellerAPIService` class skeleton (methods: getProducts, createProduct, updateProduct, deleteProduct, getOrders, updateOrderStatus, getSalesReport)
  - _Requirements: 1.1, 2.1, 7.1, 8.1, 13.1_

- [ ] 2. Implement seller authentication flows
  - [ ] 2.1 Create seller registration form and page
    - Build `SellerRegistrationForm` component with React Hook Form and Zod validation
    - Require fields: email, password (min 8 chars), business name, contact name, phone number
    - Display inline validation errors for each field
    - Show duplicate email error from API response
    - Create `src/app/seller/auth/register/page.tsx` with form and pending approval notification on success
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.6_

  - [ ]* 2.2 Write property test for seller registration validation
    - **Property 1: Seller Registration Validation**
    - **Validates: Requirements 1.1, 1.2, 1.3, 1.4**

  - [ ] 2.3 Create seller login form and page
    - Build `SellerLoginForm` component using NextAuth `signIn('credentials', ...)`
    - Validate email and password fields with React Hook Form
    - Display pending approval message when account is not yet approved (Requirement 2.6)
    - Show authentication error messages for invalid credentials
    - Create `src/app/seller/auth/signin/page.tsx` with redirect to `/seller` on success
    - _Requirements: 2.1, 2.2, 2.6_

  - [ ]* 2.4 Write property test for seller authentication session
    - **Property 2: Seller Authentication Session Management**
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.4, 2.5**

  - [ ] 2.5 Implement seller logout
    - Add logout button to seller navigation calling NextAuth `signOut`
    - Clear seller-specific Zustand state on logout
    - Clear cached seller data from browser storage on logout
    - Redirect to `/seller/auth/signin` after logout
    - _Requirements: 3.1, 3.2, 3.3, 3.4_

  - [ ] 2.6 Implement password reset flow
    - Create `ForgotPasswordForm` component that submits email to API
    - Create `ResetPasswordForm` component with new password and confirmation fields (min 8 chars)
    - Create `src/app/seller/auth/forgot-password/page.tsx` and `src/app/seller/auth/reset-password/page.tsx`
    - Show success confirmation after password reset
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 3. Set up Zustand store for seller state
  - Create `src/store/sellerStore.ts` with `SellerState` and `SellerActions` interfaces
  - Implement state slices: `products`, `orders`, `analytics`, `profile` each with `items/data`, `isLoading`, `error`
  - Implement actions: `fetchOrders`, `updateOrderStatus`, `fetchProducts`, `createProduct`, `updateProduct`, `deleteProduct`, `fetchAnalytics`
  - Integrate with `SellerAPIService` for all async actions
  - Add toast notifications (success/error) for mutating actions
  - _Requirements: 7.1, 8.1, 10.1, 13.1, 17.1_

- [ ] 4. Build seller navigation and layout components
  - [ ] 4.1 Create `SellerNavigation` component
    - Build responsive sidebar navigation for desktop (lg:pl-64) with links: Dashboard, Products, Orders, Analytics, Profile
    - Build collapsible hamburger menu for mobile with touch-friendly buttons (min 44px)
    - Add active link highlighting based on current route
    - Include seller business name and logout button in navigation
    - _Requirements: 14.1, 14.2, 14.3_

  - [ ] 4.2 Create seller layout with role guard
    - Wire `SellerNavigation` into `src/app/seller/layout.tsx`
    - Ensure `RoleGuard` redirects unauthenticated or non-seller users to `/seller/auth/signin`
    - Apply responsive layout: sidebar on desktop, top bar on mobile
    - _Requirements: 15.6_

- [ ] 5. Implement seller profile management
  - [ ] 5.1 Create profile display and edit form
    - Create `src/app/seller/profile/page.tsx` fetching current seller profile
    - Display: business name, contact name, email, phone number
    - Build `SellerProfileForm` with React Hook Form, validating email format and phone format
    - Submit updates to API and show success/error feedback
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 5.2 Write property test for profile update validation
    - **Property 3: Seller Profile Update Validation**
    - **Validates: Requirements 4.3, 4.4**

  - [ ] 5.3 Implement email verification reminder
    - Display verification reminder banner when seller email is unverified (Requirement 5.3)
    - Handle email change flow: show verification required message after email update
    - _Requirements: 5.1, 5.3, 5.4_

- [ ] 6. Implement product management - core CRUD
  - [ ] 6.1 Create product list page
    - Create `src/app/seller/products/page.tsx` as server component fetching seller products
    - Build `ProductTable` component displaying: thumbnail, name, price, stock, status, edit/delete actions
    - Add search input (debounced 300ms), category filter dropdown, and status filter dropdown
    - Implement pagination (20 products per page)
    - Add "Add Product" button linking to `/seller/products/new`
    - Show low stock indicator (< 5 units) in stock column
    - _Requirements: 8.1, 12.3, 12.4_

  - [ ] 6.2 Create product form component
    - Build `ProductForm` component with React Hook Form and Zod validation
    - Fields: name (required), description (required), price (positive, max 2 decimal places), categoryId (required), stockQuantity (non-negative integer)
    - Add `CategorySelect` dropdown fetching available categories
    - Add `ImageUpload` component supporting JPEG/PNG/WebP, max 5 files, max 5MB each, with drag-and-drop
    - Show validation errors inline for all fields
    - _Requirements: 8.2, 8.5, 8.6, 9.1, 9.2, 9.3, 9.4_

  - [ ]* 6.3 Write property test for product form validation
    - **Property 4: Product Form Validation**
    - **Validates: Requirements 8.2, 8.5, 8.6**

  - [ ] 6.4 Create add product page
    - Create `src/app/seller/products/new/page.tsx`
    - Render `ProductForm` with empty defaults
    - On submit, call `createProduct` from seller store and redirect to product list on success
    - Set product status to active by default (Requirement 8.3)
    - _Requirements: 8.1, 8.3, 8.4_

  - [ ] 6.5 Create edit product page
    - Create `src/app/seller/products/[id]/page.tsx` as server component fetching product by ID
    - Render `ProductForm` pre-populated with existing product data
    - On submit, call `updateProduct` from seller store
    - Add product activation/deactivation toggle (hides from customer browsing, preserves data)
    - _Requirements: 10.1, 10.2, 10.3, 10.4_

  - [ ]* 6.6 Write property test for product status management
    - **Property 5: Product Status Management**
    - **Validates: Requirements 10.4**

  - [ ] 6.7 Implement product deletion with confirmation
    - Add delete button to product list and edit page
    - Show confirmation modal before deletion
    - Prevent deletion if product has existing orders (show warning message)
    - On confirmed delete, call `deleteProduct` from seller store and remove from list
    - _Requirements: 11.1, 11.2, 11.3, 11.4_

  - [ ]* 6.8 Write property test for product deletion guard
    - **Property 6: Product Deletion Guard**
    - **Validates: Requirements 11.2**

- [ ] 7. Implement image upload component
  - Build `ImageUpload` component with drag-and-drop zone and file input fallback
  - Validate file type (JPEG, PNG, WebP) and size (max 5MB) client-side before upload
  - Display upload progress indicator per file
  - Show image previews with delete button for each uploaded image
  - Enforce maximum 5 images per product
  - On image delete, call API to remove image and thumbnails
  - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

- [ ] 8. Implement inventory/stock management
  - [ ] 8.1 Add stock update controls to product form and list
    - Ensure stock quantity field in `ProductForm` enforces non-negative integer validation
    - Display out-of-stock badge when stock reaches zero
    - Display low stock warning badge when stock < 5 units
    - _Requirements: 12.1, 12.3, 12.4, 12.5_

  - [ ]* 8.2 Write property test for inventory constraints
    - **Property 7: Inventory Constraint Validation**
    - **Validates: Requirements 12.3, 12.4, 12.5**

  - [ ] 8.3 Display low stock alerts on dashboard
    - Fetch products with stock < 5 and render `LowStockAlert` list on dashboard
    - Each alert shows product name, current stock, and link to edit product
    - _Requirements: 7.3, 12.4_

- [ ] 9. Implement order management
  - [ ] 9.1 Create order list page
    - Create `src/app/seller/orders/page.tsx` fetching orders containing seller's products
    - Build `OrderTable` component displaying: order number, customer name, date, total, status
    - Sort orders by date descending (most recent first)
    - Add status filter dropdown and date range filter
    - Add search by order number
    - _Requirements: 13.1, 13.2, 13.4, 13.5_

  - [ ] 9.2 Create order detail page
    - Create `src/app/seller/orders/[id]/page.tsx` as server component
    - Display: order number, customer name, customer address, ordered items with quantities and prices
    - Show current order status with `OrderStatusBadge` component
    - _Requirements: 13.3_

  - [ ]* 9.3 Write property test for order list filtering
    - **Property 8: Order List Filtering and Sorting**
    - **Validates: Requirements 13.4, 13.5**

- [ ] 10. Implement seller dashboard
  - [ ] 10.1 Build dashboard metrics cards
    - Create `DashboardMetrics` component displaying: total products, active orders, total sales (current month)
    - Each metric card shows value, label, and trend indicator
    - Clicking a metric navigates to the corresponding detailed view
    - _Requirements: 7.1, 7.5_

  - [ ] 10.2 Build recent orders and top products widgets
    - Create `RecentOrders` component showing last 5 orders with: order number, customer name, total, status
    - Create `TopProducts` component showing top 5 selling products for current month
    - Wire both widgets into `src/app/seller/page.tsx` dashboard layout
    - _Requirements: 7.2, 7.4_

  - [ ]* 10.3 Write property test for dashboard metrics display
    - **Property 9: Dashboard Metrics Display**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4**

- [ ] 11. Checkpoint - Ensure core seller features work
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Implement security features
  - [ ] 12.1 Configure Next.js middleware for seller route protection
    - Update `middleware.ts` to verify JWT token for `/seller/*` routes
    - Redirect unauthenticated requests to `/seller/auth/signin`
    - Redirect authenticated non-seller roles to `/unauthorized`
    - _Requirements: 15.6_

  - [ ]* 12.2 Write property test for route access control
    - **Property 10: Seller Route Access Control**
    - **Validates: Requirements 15.6**

  - [ ] 12.3 Configure secure session cookies and HTTPS headers
    - Set HTTP-only and secure flags on session cookies in NextAuth config
    - Add security headers in `next.config.js`: `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`
    - Enforce HTTPS in production environment
    - _Requirements: 15.2, 15.5_

  - [ ] 12.4 Implement rate limiting display for login
    - Display rate limit error message after 5 failed login attempts within 15 minutes
    - Show lockout message (30 minutes) when rate limit is exceeded
    - _Requirements: 15.3, 15.4_

  - [ ]* 12.5 Write property test for rate limiting enforcement
    - **Property 11: Login Rate Limiting**
    - **Validates: Requirements 15.3, 15.4**

- [ ] 13. Implement mobile responsiveness
  - [ ] 13.1 Apply responsive layouts across all seller pages
    - Ensure all pages render correctly from 320px to 2560px width
    - Convert data tables to card-based layouts on mobile (< 768px)
    - Apply horizontal scroll for tables that cannot be collapsed
    - _Requirements: 14.1_

  - [ ] 13.2 Optimize forms and navigation for touch
    - Ensure all interactive elements meet 44px minimum touch target size
    - Optimize form layouts for touch input on mobile (stacked fields, larger inputs)
    - Implement bottom sheet modals for mobile form overlays
    - _Requirements: 14.3, 14.5_

  - [ ]* 13.3 Write property test for responsive layout breakpoints
    - **Property 12: Responsive Layout Breakpoints**
    - **Validates: Requirements 14.1, 14.3**

- [ ] 14. Checkpoint - Ensure security and responsiveness are complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 15. Implement Priority 2 - Order status updates
  - [ ] 15.1 Create order status update controls
    - Add `OrderStatusSelect` dropdown to `OrderCard` and order detail page
    - Enforce valid status transition sequence: order placed → processing → shipped → out for delivery → delivered
    - Disable invalid transitions in the dropdown
    - On status change, call `updateOrderStatus` from seller store
    - _Requirements: 16.1, 16.2, 16.4_

  - [ ]* 15.2 Write property test for order status transition validation
    - **Property 13: Order Status Transition Validation**
    - **Validates: Requirements 16.2, 16.4**

  - [ ] 15.3 Display status update confirmation and history
    - Show success toast after status update
    - Display order status history/timeline on order detail page with timestamps
    - _Requirements: 16.1, 16.5_

- [ ] 16. Implement Priority 2 - Sales reporting
  - [ ] 16.1 Create sales report page and date range selector
    - Create `src/app/seller/analytics/page.tsx`
    - Build `DateRangeSelector` component with presets: daily, weekly, monthly, and custom range
    - Fetch sales data from API based on selected range
    - _Requirements: 17.1, 17.2_

  - [ ] 16.2 Build sales metrics and charts
    - Display key metrics: total sales, order count, average order value
    - Implement `SalesTrendChart` using Recharts (line chart for sales over time)
    - Implement `TopProductsChart` using Recharts (bar chart for top products by revenue)
    - _Requirements: 17.1, 17.3, 17.4_

  - [ ]* 16.3 Write property test for sales report data aggregation
    - **Property 14: Sales Report Data Aggregation**
    - **Validates: Requirements 17.1, 17.2, 17.3**

  - [ ] 16.4 Implement CSV export for reports
    - Add "Export CSV" button to sales report page
    - Generate CSV with columns: date, order count, total sales, average order value
    - Trigger browser download of generated CSV file
    - _Requirements: 17.5_

- [ ] 17. Final checkpoint - Ensure all features are complete
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Priority 2 features (Requirements 16–17) are in tasks 15–16 and can be deferred
- Priority 3 features (Requirements 18–21) are not included in this plan and should be addressed in a separate iteration
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties; unit tests validate specific examples and edge cases
