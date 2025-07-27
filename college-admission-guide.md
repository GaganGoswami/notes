

# Vibe Coding Prompts for College Journey Guide Application

## 📋 Project Setup & Architecture Prompts

### 1. Project Initialization
```
You are a senior full-stack developer specializing in React and Spring Boot applications. 

Create a complete project structure for "College Admission & Sponsorship Journey Guide" with:
- React frontend (Vite + TypeScript + TailwindCSS)
- Spring Boot backend (Java 21 + PostgreSQL + JPA)
- Docker configuration for development and production
- Package.json with all necessary dependencies
- Proper folder structure following best practices

Include initial configuration files, environment setup, and README with setup instructions.
```

### 2. Database Schema Design
```
Act as a database architect. Design a PostgreSQL schema for a college admission tracking application.

Create tables for:
- Users (authentication, roles)
- Student profiles (GPA, test scores, activities)
- Timeline tasks (milestones, deadlines)
- Colleges (details, requirements)
- Scholarships (criteria, deadlines)
- Applications (status tracking)

Include proper relationships, indexes, and constraints. Provide both SQL DDL and JPA entity classes.
```

## 🎨 Frontend Component Prompts

### 3. Authentication System
```
Build a complete authentication system for React with:
- Login/Register forms with validation
- JWTd routes
- Auth context provider
- Password reset functionality

Use React Hook Form for validation and Axios for API calls. Style with TailwindCSS following modern UI patterns.
```

### 4. Dashboard Component
```
Create a comprehensive dashboard component that displays:
- Upcoming timeline tasks (next 3-5)
- Application status overview (Kanban-style cards)
- Scholarship deadlines countdown
- Progress metrics and charts
- Quick action buttons

Make it responsive and include loading states. Use React Query for data fetching.
```

### 5. Timeline Tracker
```
Build an interactive timeline component showing college prep journey from 9th-12th grade:
- Horizontal timeline layout
- Color-coded milestones by category
- Task completion tracking
- Add/edit task modals
- Filter by grade/category
- Progress indicators

Include smooth animations and mobile-responsive design.
```

### 6. Profile Builder
```
Create a multi-step profile builder with:
- Academic information (GPA, test scores, courses)
- Extracurricular activities manager
- Document upload functionality
- Skills and interests tags
- AI-powered profile analysis sidebar
- Progress saving between steps

Include form validation and real-time character counts.
```

### 7. College Explorer
```
Build a college search and exploration interface with:
- Advanced filters (location, cost, acceptance rate, majors)
- Card and list view toggles
- Favorites/watchlist functionality
- College comparison tool
- Interactive maps integration
- Detail modal with comprehensive info

Include search functionality and infinite scroll pagination.
```

### 8. Scholarship Hub
```
Create a scholarship management system featuring:
- Search and filter scholarships
- Deadline tracking and alerts
- Application status tracking
- Essay helper with AI integration
- Document attachment system
- Watchlist management

Include calendar integration and notification system.
```

### 9. Application Manager
```
Build a Kanban-style application tracker with:
- Drag-and-drop columns (Idea → Draft → Submitted → Decision)
- Application cards with key details
- Status change notifications
- Document attachment
- Deadline alerts
- Progress statistics

Include responsive design and smooth animations.
```

## 🔧 Backend API Prompts

### 10. Spring Boot REST API Structure
```
Create a complete Spring Boot REST API with:
- JWT authentication and authorization
- CRUD endpoints for all entities
- Global exception handling
- Request/response validation
- API documentation with Swagger
- CORS configuration
- Security configuration

Follow REST best practices and include proper HTTP status codes.
```

### 11. User Authentication Service
```
Implement comprehensive user authentication with:
- Registration with email verification
- Login with JWT token generation
- Password reset functionality
- Role-based access control
- Account lockout after failed attempts
- Token refresh mechanism

Include BCrypt password hashing and proper security headers.
```

### 12. Profile Management API
```
Build profile management endpoints with:
- CRUD operations for student profiles
- File upload handling for documents
- Academic record management
- Activity tracking
- AI profile analysis integration
- Data validation and sanitization

Include proper error handling and data relationships.
```

### 13. Timeline Task Management
```
Create task management system with:
- CRUD operations for timeline tasks
- Bulk task creation for new users
- Task completion tracking
- Deadline notifications
- Category-based filtering
- Progress analytics

Include automated task seeding for different grade levels.
```

### 14. College Data Management
```
Implement college information system with:
- College CRUD operations
- Search and filtering capabilities
- Favorites management
- Data import/export functionality
- Integration with external APIs
- Caching for performance

Include data validation and relationship management.
```

## 🤖 AI Integration Prompts

### 15. AI Profile Analysis
```
Create an AI-powered profile analysis system that:
- Analyzes student profiles for strengths/weaknesses
- Provides personalized improvement suggestions
- Generates competitiveness scores
- Recommends colleges based on profile
- Identifies scholarship opportunities

Use OpenAI API with structured prompts and response parsing.
```

### 16. Essay Coaching Assistant
```
Build an AI essay helper that:
- Reviews scholarship and college essays
- Provides feedback on structure, content, and style
- Suggests improvements and alternatives
- Checks for grammar and clarity
- Generates essay outlines and prompts

Include confidence scoring and iterative improvement suggestions.
```

### 17. College Recommendation Engine
```
Develop an AI recommendation system that:
- Matches students with suitable colleges
- Considers academic profile, preferences, and goals
- Provides reasoning behind recommendations
- Updates recommendations based on profile changes
- Includes reach/match/safety categorization

Use machine learning approaches with fallback to rule-based logic.
```

## 📱 Advanced Feature Prompts

### 18. Real-time Notifications
```
Implement a notification system with:
- WebSocket connections for real-time updates
- Email notifications for deadlines
- Push notifications (if PWA)
- Notification preferences management
- Notification history and status tracking

Include both in-app and external notification methods.
```

### 19. Document Management System
```
Create a file management system with:
- Secure file upload/download
- Document categorization and tagging
- Version control for essays and documents
- Sharing capabilities
- Storage quota management
- File preview functionality

Include virus scanning and file type validation.
```

### 20. Calendar Integration
```
Build calendar functionality with:
- Event creation for deadlines and milestones
- Calendar view (month, week, day)
- iCal export functionality
- Google Calendar integration
- Reminder scheduling
- Event categorization and color coding

Include timezone handling and recurring events.
```

## 🎯 Testing & Deployment Prompts

### 21. Frontend Testing Suite
```
Create comprehensive React testing with:
- Unit tests for components using React Testing Library
- Integration tests for user flows
- Mock service worker for API mocking
- Accessibility testing
- Performance testing
- E2E tests with Cypress

Include test coverage reporting and CI integration.
```

### 22. Backend Testing Framework
```
Implement Spring Boot testing with:
- Unit tests for services and repositories
- Integration tests for API endpoints
- Security testing
- Database testing with TestContainers
- Performance and load testing
- Mock external service calls

Include test data fixtures and cleanup strategies.
```

### 23. Docker Deployment Configuration
```
Create production-ready Docker setup with:
- Multi-stage builds for frontend and backend
- Docker Compose for local development
- Production deployment configuration
- Environment variable management
- Health checks and monitoring
- SSL certificate handling

Include optimization for image size and security.
```

## 🔐 Security & Performance Prompts

### 24. Security Implementation
```
Implement comprehensive security measures:
- Input validation and sanitization
- SQL injection prevention
- XSS protection
- CSRF tokens
- Rate limiting
- Security headers configuration
- API key management

Include security audit checklist and penetration testing guidelines.
```

### 25. Performance Optimization
```
Optimize application performance with:
- Database query optimization and indexing
- Frontend code splitting and lazy loading
- Image optimization and CDN integration
- Caching strategies (Redis integration)
- API response compression
- Bundle size optimization

Include performance monitoring and metrics collection.
```

## 📊 Analytics & Monitoring Prompts

### 26. User Analytics System
```
Build analytics tracking with:
- User behavior tracking
- Feature usage analytics
- Performance metrics collection
- Error tracking and reporting
- Custom event tracking
- Privacy-compliant data collection

Include dashboard for analytics visualization and insights.
```

### 27. Application Monitoring
```
Implement monitoring and logging with:
- Application health checks
- Error logging and alerting
- Performance monitoring
- User session tracking
- API usage analytics
- System resource monitoring

Include integration with monitoring platforms and alerting systems.
```

## 🎨 UI/UX Enhancement Prompts

### 28. Responsive Design System
```
Create a comprehensive design system with:
- Consistent color palette and typography
- Reusable UI components library
- Responsive breakpoints and layouts
- Dark mode support
- Accessibility compliance (WCAG 2.1)
- Animation and transition guidelines

Include Storybook documentation and usage guidelines.
```

### 29. Mobile-First Experience
```
Optimize for mobile devices with:
- Touch-friendly interface elements
- Optimized navigation for small screens
- Swipe gestures and interactions
- Progressive Web App features
- Offline functionality
- Mobile-specific optimizations

Include performance optimization for mobile networks.
```

## 🔄 Advanced Workflow Prompts

### 30. Import/Export Functionality
```
Build data import/export features with:
- CSV import for bulk college/scholarship data
- Student profile export functionality
- Application data backup/restore
- Integration with external APIs
- Data validation and error handling
- Progress tracking for large imports

Include format validation and data transformation capabilities.
```

---

## 🎨 Complete CSS Design System

### **Color Tokens (Light Mode)**
```css
/* Primitive Colors */
--color-white: rgba(255, 255, 255, 1);
--color-black: rgba(0, 0, 0, 1);
--color-cream-50: rgba(252, 252, 249, 1);
--color-cream-100: rgba(255, 255, 253, 1);
--color-gray-200: rgba(245, 245, 245, 1);
--color-gray-300: rgba(167, 169, 169, 1);
--color-gray-400: rgba(119, 124, 124, 1);
--color-slate-500: rgba(98, 108, 113, 1);
--color-brown-600: rgba(94, 82, 64, 1);
--color-charcoal-700: rgba(31, 33, 33, 1);
--color-charcoal-800: rgba(38, 40, 40, 1);
--color-slate-900: rgba(19, 52, 59, 1);
--color-teal-300: rgba(50, 184, 198, 1);
--color-teal-400: rgba(45, 166, 178, 1);
--color-teal-500: rgba(33, 128, 141, 1);
--color-teal-600: rgba(29, 116, 128, 1);
--color-teal-700: rgba(26, 104, 115, 1);
--color-teal-800: rgba(41, 150, 161, 1);
--color-red-400: rgba(255, 84, 89, 1);
--color-red-500: rgba(192, 21, 47, 1);
--color-orange-400: rgba(230, 129, 97, 1);
--color-orange-500: rgba(168, 75, 47, 1);

/* Semantic Colors (Light) */
--color-background: var(--color-cream-50);
--color-surface: var(--color-cream-100);
--color-text: var(--color-slate-900);
--color-text-secondary: var(--color-slate-500);
--color-primary: var(--color-teal-500);
--color-primary-hover: var(--color-teal-600);
--color-primary-active: var(--color-teal-700);
```

### **Dark Mode Overrides**
```css
[data-color-scheme="dark"] {
  --color-background: var(--color-charcoal-700);
  --color-surface: var(--color-charcoal-800);
  --color-text: var(--color-gray-200);
  --color-text-secondary: rgba(var(--color-gray-300-rgb), 0.7);
  --color-primary: var(--color-teal-300);
  --color-primary-hover: var(--color-teal-400);
  --color-primary-active: var(--color-teal-800);
}
```

### **Typography System**
```css
--font-family-base: "FKGroteskNeue", "Geist", "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
--font-family-mono: "Berkeley Mono", ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
--font-size-xs: 11px;
--font-size-sm: 12px;
--font-size-base: 14px;
--font-size-md: 14px;
--font-size-lg: 16px;
--font-size-xl: 18px;
--font-size-2xl: 20px;
--font-size-3xl: 24px;
--font-size-4xl: 30px;
--font-weight-normal: 400;
--font-weight-medium: 500;
--font-weight-semibold: 550;
--font-weight-bold: 600;
```

### **Spacing & Layout**
```css
--space-0: 0; --space-1: 1px; --space-2: 2px; --space-4: 4px;
--space-6: 6px; --space-8: 8px; --space-10: 10px; --space-12: 12px;
--space-16: 16px; --space-20: 20px; --space-24: 24px; --space-32: 32px;
--radius-sm: 6px; --radius-base: 8px; --radius-md: 10px; --radius-lg: 12px; --radius-full: 9999px;
```
