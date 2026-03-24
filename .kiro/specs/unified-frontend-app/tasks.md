# Implementation Plan: Unified Frontend Application

## Overview

This implementation plan breaks down the Next.js 14+ unified frontend application into sequential, manageable tasks. The app serves customers, sellers, and super admins through a single application with role-based routing, Zustand state management, NextAuth.js authentication, and shadcn/ui components.

The implementation follows a layered approach: project setup, authentication, shared infrastructure, customer interface, seller interface, admin interface, advanced features, and final integration. Tasks are ordered to align with the unified backend API implementation so both can be developed in parallel.

## Tasks

- [ ] 1. Project setup and infrastructure
  - Initialize Next.js 14+ project with TypeScript and App Router
  - Configure Tailwind CSS with custom design tokens
  - Install and configure shadcn/ui component library
  - Set up ESLint, Prettier, and Jest with React Testing Library
  - Configure Playwright for E2E testing
  - Install fast-check for property-based testing
  - Set up path aliases and environment variable configuration
  - Create base directory structure (app, components, lib, hooks, store, styles)
  - _Requirements: Architecture foundation_

- [ ] 2. TypeScript types and API client foundation
  - [ ] 2.1 Define core TypeScript interfaces and types
    - Create `lib/types/user.ts` with User, Address, UserRole types
    - Create `lib/types/product.ts` with Product, Category, ProductFilters types
    - Create `lib/types/order.ts` with Order, OrderItem, OrderStatus types
    - Create `lib/types/cart.ts` with CartItem, CartTotals types
    - Create `lib/types/review.ts` with Review type
    - _Requirements: All data model requirements_

  - [ ] 2.2 Implement Axios API client with interceptors
    - Create `lib/api/client.ts` with Axios instance and base URL configuration
    - Add request interceptor to attach JWT Bearer token from session
    - Add response interceptor to handle 401 redirects and error normalization
    - Create `lib/api/errorHandler.ts` with role-aware error display logic
    - _Requirements: API integration foundation_

  - [ ] 2.3 Create API endpoint modules
    - Create `lib/api/auth.ts` for authentication endpoints
    - Create `lib/api/products.ts` for product catalog endpoints
    - Create `lib/api/orders.ts` for order management endpoints
    - Create `lib/api/cart.ts` for cart endpoints
    - Create `lib/api/users.ts` for user profile endpoints
    - Create `lib/api/sellers.ts` for seller management endpoints
    - Create `lib/api/admin.ts` for admin endpoints
    - _Requirements: All API integration requirements_

- [ ] 3. Authentication with NextAuth.js
  - [ ] 3.1 Configure NextAuth.js with credentials provider
    - Create `app/api/auth/[...nextauth]/route.ts` with CredentialsProvider
    - Implement `authorize` callback calling backend `/api/auth/login`
    - Configure JWT callback to persist role, approvalStatus, accessToken, refreshToken
    - Configure session callback to expose role and accessToken to client
    - Set session strategy to JWT with 24-hour maxAge
    - _Requirements: 2.1, 2.2_

  - [ ]* 3.2 Write property test for authentication success flow
    - **Property 3: Authentication Success Flow**
    - **Validates: Requirements 2.1**

  - [ ]* 3.3 Write property test for authentication failure handling
    - **Property 4: Authentication Failure Handling**
    - **Validates: Requirements 2.2**

  - [ ] 3.4 Implement route protection middleware
    - Create `middleware.ts` with `getToken` from next-auth/jwt
    - Protect `/seller/*` routes: require seller role and approved status
    - Protect `/admin/*` routes: require super_admin role
    - Protect `/checkout`, `/orders`, `/wishlist`, `/account`: require customer role
    - Redirect unauthenticated users to `/auth/login` with callbackUrl
    - Redirect wrong-role users to home page
    - _Requirements: 19.1, 19.2_

  - [ ]* 3.5 Write property test for role-based route protection
    - **Property 25: Role-Based Route Protection**
    - **Validates: Requirements 19.1, 19.2**

- [ ] 4. Zustand state management stores
  - [ ] 4.1 Implement auth store
    - Create `store/authStore.ts` with user, isAuthenticated state
    - Add setUser and clearUser actions with persist middleware
    - _Requirements: Session management_

  - [ ] 4.2 Implement cart store with persistence
    - Create `store/cartStore.ts` with items, subtotal, tax, shipping, total state
    - Implement addItem (merge existing items), updateQuantity, removeItem, clearCart actions
    - Implement calculateTotals: subtotal = sum(price × qty), tax = 10%, free shipping over $50
    - Persist cart to localStorage via zustand/middleware persist
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ]* 4.3 Write property test for cart total calculation
    - **Property 10: Cart Total Calculation**
    - **Validates: Requirements 5.4, 20.1**

  - [ ]* 4.4 Write property test for cart addition operation
    - **Property 7: Cart Addition Operation**
    - **Validates: Requirements 5.1**

  - [ ]* 4.5 Write property test for cart quantity update
    - **Property 8: Cart Quantity Update**
    - **Validates: Requirements 5.2**

  - [ ]* 4.6 Write property test for cart item removal
    - **Property 9: Cart Item Removal**
    - **Validates: Requirements 5.3**

  - [ ]* 4.7 Write property test for guest cart persistence
    - **Property 11: Guest Cart Persistence**
    - **Validates: Requirements 5.5**

  - [ ] 4.8 Implement product and order stores
    - Create `store/productStore.ts` with products, selectedProduct, filters state
    - Create `store/orderStore.ts` with orders, selectedOrder state
    - _Requirements: Product and order state management_

- [ ] 5. Checkpoint - Foundation complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Shared UI components and layout
  - [ ] 6.1 Implement root layout and global styles
    - Create `app/layout.tsx` with SessionProvider, ErrorBoundary, and Toaster
    - Configure `styles/globals.css` with Tailwind base and custom CSS variables
    - _Requirements: Global layout_

  - [ ] 6.2 Implement shared Header component
    - Create `components/shared/Header.tsx` with role-based navigation links
    - Show cart icon with item count badge for customers
    - Show user name, role-based nav links, and logout button when authenticated
    - Show login button for guests
    - Implement mobile hamburger menu
    - _Requirements: Navigation requirements_

  - [ ] 6.3 Implement shared Footer and Sidebar components
    - Create `components/shared/Footer.tsx` with links and copyright
    - Create `components/shared/Sidebar.tsx` for seller and admin dashboards
    - _Requirements: Layout requirements_

  - [ ] 6.4 Implement skeleton loaders and error boundary
    - Create `components/ui/skeleton.tsx` with ProductCardSkeleton, TableSkeleton
    - Create `components/ErrorBoundary.tsx` as class component with fallback UI
    - _Requirements: Loading and error states_

  - [ ] 6.5 Implement toast notification utilities
    - Create `lib/utils/toast.ts` with showError and showSuccess using sonner
    - _Requirements: User feedback_

- [ ] 7. Authentication pages
  - [ ] 7.1 Implement login page
    - Create `app/auth/login/page.tsx` with React Hook Form + Zod validation
    - Fields: email, password with inline error messages
    - Call `signIn('credentials', ...)` on submit, handle errors
    - Redirect to role-appropriate dashboard on success
    - _Requirements: 2.1, 2.2_

  - [ ] 7.2 Implement registration page
    - Create `app/auth/register/page.tsx` with role selection (customer/seller)
    - Fields: name, email, password, role; seller adds businessName, businessDescription
    - Call backend `/api/auth/register` then auto-login on success
    - _Requirements: 1.1, 1.2_

  - [ ]* 7.3 Write property test for registration form validation
    - **Property 1: Registration Form Validation**
    - **Validates: Requirements 1.1**

  - [ ]* 7.4 Write property test for duplicate email prevention
    - **Property 2: Duplicate Email Prevention**
    - **Validates: Requirements 1.2**

  - [ ] 7.5 Implement password reset pages
    - Create `app/auth/reset-password/page.tsx` for forgot-password form
    - Create `app/auth/reset-password/confirm/page.tsx` for new password form with token
    - _Requirements: Password reset flow_

- [ ] 8. Customer interface - Product catalog
  - [ ] 8.1 Implement customer layout
    - Create `app/(customer)/layout.tsx` with Header and Footer
    - _Requirements: Customer layout_

  - [ ] 8.2 Implement home page
    - Create `app/(customer)/page.tsx` with featured products, categories, hero section
    - Use Next.js server component with static generation for SEO
    - _Requirements: 3.1_

  - [ ] 8.3 Implement ProductCard component
    - Create `components/customer/ProductCard.tsx` with image, name, price, stock status
    - Add "Add to Cart" button calling cartStore.addItem
    - Disable button when out of stock
    - _Requirements: 3.1_

  - [ ]* 8.4 Write unit tests for ProductCard component
    - Test product info display, add to cart action, out-of-stock disabled state
    - _Requirements: 3.1_

  - [ ]* 8.5 Write property test for product display completeness
    - **Property 5: Product Display Completeness**
    - **Validates: Requirements 3.1**

  - [ ] 8.6 Implement product listing page with filters
    - Create `app/(customer)/products/page.tsx` with server-side data fetching
    - Implement category filter, price range filter, sort options
    - Add pagination with URL search params
    - Render ProductCard grid with ProductCardSkeleton during loading
    - _Requirements: 3.1, 11.1, 12.1_

  - [ ]* 8.7 Write property test for product filter application
    - **Property 18: Product Filter Application**
    - **Validates: Requirements 11.1**

  - [ ]* 8.8 Write property test for product sort ordering
    - **Property 19: Product Sort Ordering**
    - **Validates: Requirements 12.1**

  - [ ] 8.9 Implement product search
    - Add search bar in Header calling `/api/products/search`
    - Create `app/(customer)/products/search/page.tsx` for search results
    - _Requirements: 4.1_

  - [ ]* 8.10 Write property test for search result relevance
    - **Property 6: Search Result Relevance**
    - **Validates: Requirements 4.1**

  - [ ] 8.11 Implement product detail page with SEO
    - Create `app/(customer)/products/[id]/page.tsx` with generateMetadata
    - Display images gallery, description, price, stock, seller info, reviews
    - Add "Add to Cart" with quantity selector
    - _Requirements: 3.1, 10.1_

  - [ ]* 8.12 Write property test for SEO metadata uniqueness
    - **Property 17: SEO Metadata Uniqueness**
    - **Validates: Requirements 10.1**

- [ ] 9. Customer interface - Cart and checkout
  - [ ] 9.1 Implement CartItem component
    - Create `components/customer/CartItem.tsx` with image, name, price, quantity input, remove button
    - Call cartStore.updateQuantity and cartStore.removeItem
    - _Requirements: 5.2, 5.3_

  - [ ] 9.2 Implement cart page
    - Create `app/(customer)/cart/page.tsx` rendering CartItem list and order summary
    - Show subtotal, tax, shipping, total from cartStore
    - Add "Proceed to Checkout" button (requires auth)
    - _Requirements: 5.1, 5.4_

  - [ ] 9.3 Implement checkout page
    - Create `app/(customer)/checkout/page.tsx` with address selection/entry form
    - Add payment method selection (COD / Online)
    - Validate required fields before enabling "Place Order"
    - Call POST `/api/orders` on submit, clear cart on success, redirect to order confirmation
    - _Requirements: 6.1, 6.2_

  - [ ]* 9.4 Write property test for checkout validation
    - **Property 12: Checkout Validation**
    - **Validates: Requirements 6.1**

  - [ ]* 9.5 Write property test for post-checkout cart clearing
    - **Property 13: Post-Checkout Cart Clearing**
    - **Validates: Requirements 6.2**

- [ ] 10. Customer interface - Orders, wishlist, and account
  - [ ] 10.1 Implement OrderCard component and order history page
    - Create `components/customer/OrderCard.tsx` with order number, date, status, total
    - Create `app/(customer)/orders/page.tsx` fetching GET `/api/orders` sorted newest first
    - _Requirements: 7.1_

  - [ ]* 10.2 Write property test for order history display
    - **Property 14: Order History Display**
    - **Validates: Requirements 7.1**

  - [ ] 10.3 Implement order detail page
    - Create `app/(customer)/orders/[id]/page.tsx` with items, address, payment, tracking
    - _Requirements: 7.1_

  - [ ] 10.4 Implement wishlist page
    - Create `app/(customer)/wishlist/page.tsx` fetching GET `/api/users/wishlist`
    - Add/remove wishlist items via POST/DELETE `/api/users/wishlist/:productId`
    - _Requirements: 13.1_

  - [ ]* 10.5 Write property test for wishlist state management
    - **Property 20: Wishlist State Management**
    - **Validates: Requirements 13.1**

  - [ ] 10.6 Implement customer account page
    - Create `app/(customer)/account/page.tsx` with profile edit form and address management
    - _Requirements: Profile management_

- [ ] 11. Customer interface - Reviews
  - [ ] 11.1 Implement review display on product detail page
    - Create `components/customer/ReviewList.tsx` showing approved reviews with rating stars
    - Display average rating calculated from review ratings
    - _Requirements: 15.1, 15.2_

  - [ ]* 11.2 Write property test for review display completeness
    - **Property 21: Review Display Completeness**
    - **Validates: Requirements 15.1**

  - [ ]* 11.3 Write property test for average rating calculation
    - **Property 22: Average Rating Calculation**
    - **Validates: Requirements 15.2**

  - [ ] 11.4 Implement review submission form
    - Create `components/customer/ReviewForm.tsx` with star rating and comment
    - Call POST `/api/products/:id/reviews`, show error if not verified purchaser
    - _Requirements: 15.1_

- [ ] 12. Checkpoint - Customer interface complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Seller interface - Layout and dashboard
  - [ ] 13.1 Implement seller layout with sidebar navigation
    - Create `app/seller/layout.tsx` with Sidebar showing Dashboard, Products, Orders, Inventory, Analytics, Profile links
    - Redirect unapproved sellers to `/seller/pending` page
    - _Requirements: Seller navigation_

  - [ ] 13.2 Implement seller dashboard page
    - Create `app/seller/dashboard/page.tsx` fetching GET `/api/analytics/seller/dashboard`
    - Display total sales, order count, active products, low stock alerts, pending orders
    - _Requirements: Seller dashboard_

- [ ] 14. Seller interface - Product management
  - [ ] 14.1 Implement ProductForm component
    - Create `components/seller/ProductForm.tsx` with React Hook Form + Zod
    - Fields: name, description, price, stockQuantity, categoryId with validation
    - _Requirements: 16.1_

  - [ ]* 14.2 Write property test for product form validation
    - **Property 23: Product Form Validation**
    - **Validates: Requirements 16.1**

  - [ ] 14.3 Implement seller product listing page
    - Create `app/seller/products/page.tsx` fetching seller's own products
    - Show product table with name, price, stock, status, edit/deactivate actions
    - _Requirements: 16.1_

  - [ ] 14.4 Implement product create and edit pages
    - Create `app/seller/products/new/page.tsx` calling POST `/api/products`
    - Create `app/seller/products/[id]/edit/page.tsx` calling PUT `/api/products/:id`
    - _Requirements: 16.1_

  - [ ] 14.5 Implement image upload component
    - Create `components/seller/ImageUpload.tsx` with drag-and-drop and file picker
    - Validate file type (JPEG/PNG/WebP) and size (max 5 MB) before upload
    - Call POST `/api/products/:id/images`, show preview thumbnails
    - _Requirements: 16.2_

  - [ ]* 14.6 Write property test for image upload validation
    - **Property 24: Image Upload Validation**
    - **Validates: Requirements 16.2**

- [ ] 15. Seller interface - Orders and inventory
  - [ ] 15.1 Implement OrderTable component
    - Create `components/seller/OrderTable.tsx` with order number, customer, items, status, date
    - Add status update dropdown calling PUT `/api/orders/:id/status`
    - _Requirements: Order fulfillment_

  - [ ] 15.2 Implement seller orders page
    - Create `app/seller/orders/page.tsx` with filtering by status and date range
    - _Requirements: Order management_

  - [ ] 15.3 Implement inventory management page
    - Create `app/seller/inventory/page.tsx` with stock quantity editing per product
    - Show low stock alerts (< 5 units) highlighted in red
    - Call PUT `/api/inventory/:productId` on update
    - _Requirements: Inventory management_

- [ ] 16. Seller interface - Analytics
  - [ ] 16.1 Implement AnalyticsChart component
    - Create `components/seller/AnalyticsChart.tsx` using a charting library (recharts)
    - Support line chart for sales trends and bar chart for product performance
    - _Requirements: Analytics display_

  - [ ] 16.2 Implement seller analytics page
    - Create `app/seller/analytics/page.tsx` with date range selector
    - Fetch and display sales trends, top products, revenue breakdown
    - _Requirements: Seller analytics_

- [ ] 17. Checkpoint - Seller interface complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 18. Admin interface - Layout and dashboard
  - [ ] 18.1 Implement admin layout with sidebar navigation
    - Create `app/admin/layout.tsx` with Sidebar showing Dashboard, Users, Sellers, Products, Orders, Categories, Analytics links
    - _Requirements: Admin navigation_

  - [ ] 18.2 Implement admin dashboard page
    - Create `app/admin/dashboard/page.tsx` fetching GET `/api/analytics/admin/dashboard`
    - Display total users, sellers, pending approvals, total products, revenue metrics
    - _Requirements: Admin dashboard_

- [ ] 19. Admin interface - User and seller management
  - [ ] 19.1 Implement UserTable component
    - Create `components/admin/UserTable.tsx` with email, name, role, status, block/unblock actions
    - Call POST `/api/users/:id/block` and POST `/api/users/:id/unblock`
    - _Requirements: 18.1_

  - [ ]* 19.2 Write property test for admin user blocking
    - **Property 26: Admin User Blocking**
    - **Validates: Requirements 18.1**

  - [ ] 19.3 Implement admin users page
    - Create `app/admin/users/page.tsx` with search by email/name and pagination
    - _Requirements: User management_

  - [ ] 19.4 Implement SellerApproval component
    - Create `components/admin/SellerApproval.tsx` with approve/reject buttons and reason input
    - Call POST `/api/approvals/:sellerId/approve` and POST `/api/approvals/:sellerId/reject`
    - _Requirements: Seller approval workflow_

  - [ ] 19.5 Implement admin sellers page
    - Create `app/admin/sellers/page.tsx` with pending approvals section and all sellers list
    - _Requirements: Seller management_

- [ ] 20. Admin interface - Product moderation and categories
  - [ ] 20.1 Implement PlatformMetrics component
    - Create `components/admin/PlatformMetrics.tsx` with metric cards for key platform stats
    - _Requirements: Platform analytics_

  - [ ] 20.2 Implement admin products moderation page
    - Create `app/admin/products/page.tsx` showing flagged products with approve/remove actions
    - Call POST `/api/products/:id/approve` and POST `/api/products/:id/flag`
    - _Requirements: Content moderation_

  - [ ] 20.3 Implement category management page
    - Create `app/admin/categories/page.tsx` with category tree display
    - Add create, edit, delete category forms calling respective endpoints
    - _Requirements: Category management_

  - [ ] 20.4 Implement admin analytics page
    - Create `app/admin/analytics/page.tsx` with platform revenue, user growth, seller performance charts
    - _Requirements: Admin analytics_

- [ ] 21. Responsive design and accessibility
  - [ ] 21.1 Implement responsive layouts for all pages
    - Ensure all pages render correctly from 320px to 2560px without horizontal scroll
    - Implement mobile navigation menu (hamburger) for customer and admin layouts
    - _Requirements: 8.1_

  - [ ]* 21.2 Write property test for responsive layout rendering
    - **Property 15: Responsive Layout Rendering**
    - **Validates: Requirements 8.1**

  - [ ]* 21.3 Write property test for touch target accessibility
    - **Property 16: Touch Target Accessibility**
    - **Validates: Requirements 8.2**

  - [ ] 21.4 Implement accessibility improvements
    - Add ARIA labels to all interactive elements
    - Ensure keyboard navigation works across all forms and modals
    - Add focus management for modal dialogs
    - _Requirements: Accessibility_

- [ ] 22. Checkpoint - Admin interface and responsive design complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 23. Priority 2: Advanced product features
  - [ ] 23.1 Implement advanced filter panel
    - Extend product listing with price range slider, brand filter, availability toggle
    - Maintain filter state in URL search params across pagination
    - _Requirements: Advanced filtering_

  - [ ] 23.2 Implement product recommendations section
    - Create `components/customer/RecommendationSection.tsx`
    - Fetch GET `/api/products/recommendations` and display on home and product detail pages
    - _Requirements: Product recommendations_

  - [ ] 23.3 Implement search autocomplete
    - Add debounced autocomplete to search bar calling GET `/api/products/autocomplete`
    - Display dropdown with suggestions
    - _Requirements: Advanced search_

  - [ ] 23.4 Implement coupon code input at checkout
    - Add coupon code field in checkout page calling coupon validation endpoint
    - Display discount amount when valid coupon applied
    - _Requirements: Coupon system_

- [ ] 24. Priority 2: Seller bulk operations
  - [ ] 24.1 Implement bulk product operations UI
    - Add checkbox selection to seller product table
    - Implement bulk price update, stock update, activate/deactivate actions
    - _Requirements: Bulk operations_

  - [ ] 24.2 Implement CSV import/export UI
    - Add "Import CSV" button with file picker and progress indicator
    - Add "Export CSV" button on seller products page
    - _Requirements: CSV import/export_

- [ ] 25. Priority 2: Advanced analytics UI
  - [ ] 25.1 Implement seller advanced analytics
    - Add customer demographics, seasonal trends, and product insights tabs to seller analytics page
    - _Requirements: Advanced seller analytics_

  - [ ] 25.2 Implement admin advanced analytics
    - Add user journey, retention, cohort analysis, and seller performance ranking tabs to admin analytics page
    - _Requirements: Advanced admin analytics_

- [ ] 26. Priority 3: Additional features
  - [ ] 26.1 Implement product comparison UI
    - Add "Compare" toggle to ProductCard
    - Create comparison drawer showing side-by-side specs (max 5 products)
    - _Requirements: Product comparison_

  - [ ] 26.2 Implement order dispute UI
    - Add "Raise Dispute" button on customer order detail page
    - Create dispute form and status tracking view
    - _Requirements: Dispute resolution_

  - [ ] 26.3 Implement loyalty points display
    - Show loyalty points balance and tier on customer account page
    - Display points earned on order confirmation page
    - _Requirements: Loyalty program_

  - [ ] 26.4 Implement subscription management UI
    - Create `app/(customer)/account/subscriptions/page.tsx` for managing recurring orders
    - _Requirements: Subscription management_

  - [ ] 26.5 Implement platform configuration UI for admins
    - Create `app/admin/settings/page.tsx` with feature flags and platform config editing
    - _Requirements: Platform configuration_

- [ ] 27. Checkpoint - Priority 2 and 3 features complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 28. Integration and wiring
  - [ ] 28.1 Wire all custom hooks to API modules
    - Create `hooks/useAuth.ts` wrapping useSession with login/logout and role-based redirect
    - Create `hooks/useCart.ts` wrapping cartStore with API sync (merge on login)
    - Create `hooks/useProducts.ts` with data fetching, filtering, and pagination logic
    - Create `hooks/useOrders.ts` with order fetching and status update logic
    - _Requirements: All_

  - [ ] 28.2 Implement cart merge on login
    - After successful login, call POST `/api/cart/merge` to sync guest cart with server
    - Clear local guest cart after successful merge
    - _Requirements: Cart merge_

  - [ ] 28.3 Connect all pages to real API endpoints
    - Replace any mock data with live API calls across all pages
    - Verify role-based data filtering works correctly end-to-end
    - _Requirements: All_

  - [ ]* 28.4 Write E2E test for customer purchase flow
    - Test: browse products → add to cart → checkout → order confirmation
    - _Requirements: 3.1, 5.1, 6.1, 6.2, 7.1_

  - [ ]* 28.5 Write E2E test for seller product management flow
    - Test: login as seller → create product → upload image → manage orders
    - _Requirements: 16.1, 16.2_

  - [ ]* 28.6 Write E2E test for admin management flow
    - Test: login as admin → approve seller → moderate product → view analytics
    - _Requirements: 18.1, 19.1_

- [ ] 29. Final checkpoint - All features complete
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Frontend tasks are ordered to align with backend API tasks — implement in parallel where possible
- Property tests use fast-check with minimum 100 iterations per property
- Checkpoints ensure incremental validation at major milestones
- All components use TypeScript with strict typing throughout
- shadcn/ui components are used as primitives; custom styling via Tailwind CSS
