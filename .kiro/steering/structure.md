# Project Structure

## Repository Organization

```
.
├── .kiro/
│   ├── specs/                    # Feature specifications
│   │   ├── unified-backend-api/
│   │   │   ├── requirements.md   # Unified API requirements (all user types)
│   │   │   ├── design.md         # API design with role-based access
│   │   │   └── tasks.md          # Implementation tasks
│   │   └── unified-frontend-app/
│   │       ├── requirements.md   # Frontend requirements (all interfaces)
│   │       ├── design.md         # UI design with role-based views
│   │       └── tasks.md          # Implementation tasks
│   └── steering/                 # Project guidance
│       ├── product.md            # Product overview
│       ├── tech.md               # Technology stack
│       └── structure.md          # This file
├── .vscode/
│   └── settings.json             # VSCode configuration
└── LICENSE                       # MIT License
```

## Specification Structure

The unified architecture follows this spec structure:

### Unified Backend API Requirements
- **Introduction**: Complete API overview for all user types
- **Glossary**: Shared terms and role-specific definitions
- **Requirements**: User stories for customers, sellers, and super admins
- **Role-Based Access**: Permission matrix and authorization rules
- **Priority Levels**: Core (no prefix), Priority 2 (WHERE enabled), Priority 3 (WHERE enabled)

### Unified Frontend App Requirements
- **Introduction**: Complete frontend overview for all interfaces
- **Glossary**: UI components and role-specific features
- **Requirements**: Interface requirements for each user type
- **Role-Based Routing**: Navigation and access control
- **Responsive Design**: Adaptive layouts for all user types

### Design Documents
- **Architecture**: Unified system design with role separation
- **API Specifications**: Single API with role-based endpoints
- **Database Schema**: Unified schema with user roles and permissions
- **UI/UX Design**: Role-based interface specifications

### Tasks Documents
- **Implementation Tasks**: Ordered development work for unified system
- **Role-Based Features**: User type specific functionality
- **Dependencies**: Task ordering and prerequisites

## Component Architecture

### Unified Backend API
- **Single Application**: One backend service handling all user types
- **Role-Based Access Control**: JWT tokens with role claims (customer, seller, super_admin)
- **Shared Services**: Common functionality with role-specific permissions
- **Modular Design**: Organized by feature domains with role validation

### Unified Frontend Application
- **Single Page Application**: One frontend app with role-based routing
- **Dynamic Interfaces**: UI components that adapt based on user role
- **Shared Components**: Reusable elements across all user interfaces
- **Role-Based Navigation**: Menu and access control per user type

## Development Workflow

1. **Unified Specification Review**: Review requirements for all user types in single documents
2. **Role-Based Task Planning**: Plan implementation considering all user roles
3. **Unified Development**: Build single applications with role separation
4. **Integration Testing**: Verify role-based access and functionality
5. **Property-Based Testing**: Validate correctness across all user types

## File Naming Conventions

- **Specs**: Use unified naming (e.g., `unified-backend-api`, `unified-frontend-app`)
- **Documents**: Use lowercase with extensions (e.g., `requirements.md`)
- **Components**: Follow role-based organization within unified structure
- **APIs**: Single "unified-backend-api" specification
- **Frontend**: Single "unified-frontend-app" specification

## Role-Based Dependencies

- **Authentication Flow**: Single login with role-based redirection
- **Authorization**: Middleware validates user roles for each endpoint
- **UI Rendering**: Dynamic interface based on authenticated user role
- **Data Access**: Role-based filtering and permissions in single database
- **Feature Access**: Role-specific functionality within unified codebase