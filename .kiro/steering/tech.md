# Technology Stack

## Architecture

### Unified API Design
- Single RESTful API with role-based access control
- JWT tokens with user role and permissions
- Shared services with role-specific endpoints
- Event-driven architecture for real-time updates
- Role-based middleware for authorization

### Backend Technologies
- **Authentication**: JWT tokens with role-based claims (customer, seller, super_admin)
- **Authorization**: Middleware-based role checking and permission validation
- **Database**: Single database with user roles and permissions tables
- **Security**: HTTPS/TLS 1.3, CORS policies, rate limiting per role, input validation
- **Performance**: Connection pooling, read replicas, caching strategies
- **Architecture**: Modular services within single application

### Frontend Technologies
- **Single Page Application**: Role-based routing and interface switching
- **Responsive Design**: Adaptive UI for desktop and mobile across all user types
- **Role-Based UI**: Dynamic interface rendering based on user permissions
- **SEO Optimization**: Search engine friendly for customer-facing pages
- **Security**: Secure session management and CSRF protection

### External Integrations
- **Payment Gateways**: Stripe, PayPal for online payments
- **Email Services**: SMTP integration for notifications
- **Image Storage**: Secure image upload and management
- **Search**: Full-text search with relevance scoring

## Development Standards

### API Design
- OpenAPI/Swagger documentation for all endpoints
- Consistent error response formats
- API versioning with backward compatibility
- Health check endpoints for monitoring

### Security Requirements
- Multi-factor authentication for admin accounts
- Comprehensive audit logging
- Data encryption at rest and in transit
- GDPR compliance and data privacy controls
- Rate limiting and DDoS protection

### Performance Standards
- 95% of requests respond within 200-300ms
- Horizontal scaling support
- Database query optimization
- Graceful degradation under load

## Common Commands

Since this is a specification-driven project with unified architecture:

```bash
# View project specifications
ls .kiro/specs/

# Check backend API requirements (unified)
cat .kiro/specs/unified-backend-api/requirements.md

# Check frontend requirements (unified with role-based interfaces)
cat .kiro/specs/unified-frontend-app/requirements.md

# Review design documents
cat .kiro/specs/*/design.md

# Check implementation tasks
cat .kiro/specs/*/tasks.md
```

## Testing Strategy

- Property-based testing for correctness validation
- API endpoint testing with comprehensive coverage
- Integration testing across system boundaries
- Security testing for authentication and authorization
- Performance testing for scalability requirements