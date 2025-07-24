# Copilot Instructions - System Design Dashboard (Vibe)

## Project Overview
You are working on "Vibe" - a System Design Dashboard that generates comprehensive system architecture designs based on user inputs. This application serves both educational and professional use cases.

## Tech Stack
- **Frontend**: React.js 18+ with TypeScript, Tailwind CSS
- **Backend**: Next.js 14+ with API routes
- **Database**: Supabase (production), SQLite (development)
- **Authentication**: Supabase Auth
- **Diagrams**: Mermaid.js for system architecture visualization
- **AI Integration**: OpenAI GPT-4 or Anthropic Claude
- **Deployment**: Vercel (frontend), Supabase (backend services)

## Code Style & Standards

### TypeScript Configuration
- Use strict TypeScript with proper typing
- Prefer interfaces over types for object shapes
- Use proper generic constraints
- Always type component props and function parameters

### React Best Practices
- Use functional components with hooks
- Implement proper error boundaries
- Use React.memo for performance optimization where needed
- Follow the composition pattern over inheritance
- Use custom hooks for reusable logic

### File Structure
```
src/
├── components/          # Reusable UI components
│   ├── ui/             # Basic UI elements (buttons, inputs, etc.)
│   ├── forms/          # Form components
│   ├── diagrams/       # Diagram-related components
│   └── layout/         # Layout components
├── pages/              # Next.js pages
├── hooks/              # Custom React hooks
├── lib/                # Utility functions and configurations
├── types/              # TypeScript type definitions
├── services/           # API services and external integrations
├── store/              # State management (Zustand)
└── styles/             # Global styles and Tailwind config
```

### Naming Conventions
- **Components**: PascalCase (e.g., `SystemDesignForm`)
- **Files**: kebab-case (e.g., `system-design-form.tsx`)
- **Functions**: camelCase (e.g., `generateSystemDesign`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_QPS_LIMIT`)
- **Types/Interfaces**: PascalCase with descriptive names

## Key Domain Concepts

### System Design Structure
Every generated system design follows this exact structure:
1. Problem Definition & Goals
2. High-Level Architecture
3. Component Design
4. Data Modeling & Storage
5. APIs & Communication
6. Scalability & Performance
7. Security
8. Observability & Monitoring
9. Testing & CI/CD
10. Infrastructure & Deployment
11. Trade-Offs & Alternatives

### Input Parameters
- **Scale Metrics**: DAU, MAU, QPS, data volume
- **Performance**: Latency requirements, availability targets
- **Geographic**: Distribution patterns, compliance needs
- **Technical**: Read/write ratios, consistency requirements

### Generation Methods
- **Rule-Based**: Predefined decision trees and templates
- **LLM-Powered**: AI-generated custom designs
- **Hybrid**: Combination of both approaches

## Component Guidelines

### Form Components
- Use controlled components with proper validation
- Implement debounced input for performance
- Show real-time validation feedback
- Support both structured and free-form inputs

### Diagram Components
- Use Mermaid.js for all architecture diagrams
- Implement lazy loading for large diagrams
- Provide zoom and pan functionality
- Support export to multiple formats (SVG, PNG, PDF)

### Data Management
- Use Zustand for client-side state management
- Implement optimistic updates for better UX
- Cache frequently accessed designs
- Support offline mode with local storage fallback

## API Design Patterns

### Endpoint Structure
```typescript
// Generation endpoints
POST /api/generate/structured    # Structured input generation
POST /api/generate/freeform     # Natural language generation
POST /api/generate/hybrid       # Combined approach

// Design management
GET    /api/designs             # List user designs
POST   /api/designs             # Save new design
PUT    /api/designs/:id         # Update existing design
DELETE /api/designs/:id         # Delete design

// Library endpoints
GET /api/library/systems        # Pre-built system designs
GET /api/library/templates      # Design templates
GET /api/library/components     # Component library
```

### Response Format
```typescript
interface APIResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  metadata?: {
    timestamp: string;
    requestId: string;
    processingTime: number;
  };
}
```

## Database Schema Patterns

### Core Entities
- `users`: User profiles and preferences
- `system_designs`: Generated and saved designs
- `design_templates`: Reusable templates
- `component_library`: Architecture components
- `generation_history`: Audit trail

### Relationships
- Users have many system designs (1:N)
- Designs can be based on templates (N:1)
- Designs use components from library (N:N)

## Error Handling

### Client-Side
- Use React Error Boundaries for component errors
- Implement toast notifications for user feedback
- Provide retry mechanisms for failed requests
- Log errors to monitoring service

### Server-Side
- Use structured error responses
- Implement proper HTTP status codes
- Log all errors with context
- Provide helpful error messages to users

## Performance Guidelines

### Frontend Optimization
- Use React.memo for expensive components
- Implement virtual scrolling for large lists
- Lazy load components and routes
- Optimize images and assets

### Backend Optimization
- Implement request caching where appropriate
- Use database indexes for common queries
- Implement rate limiting for API endpoints
- Use connection pooling for database

## Security Considerations

### Authentication
- Use Supabase Auth for user management
- Implement proper session handling
- Use JWT tokens for API authentication
- Support social login options

### Authorization
- Implement role-based access control
- Validate user permissions on all operations
- Use row-level security in Supabase
- Sanitize all user inputs

### Data Protection
- Encrypt sensitive data at rest
- Use HTTPS for all communications
- Implement CSRF protection
- Follow OWASP security guidelines

## Testing Strategy

### Unit Tests
- Test all utility functions
- Test custom hooks
- Test component logic
- Mock external dependencies

### Integration Tests
- Test API endpoints
- Test database operations
- Test authentication flows
- Test generation workflows

### E2E Tests
- Test complete user journeys
- Test cross-browser compatibility
- Test responsive design
- Test accessibility features

## Code Examples

### Component Structure
```typescript
interface SystemDesignFormProps {
  onSubmit: (data: SystemDesignInput) => void;
  loading?: boolean;
  initialData?: Partial<SystemDesignInput>;
}

export const SystemDesignForm: React.FC<SystemDesignFormProps> = ({
  onSubmit,
  loading = false,
  initialData
}) => {
  // Component implementation
};
```

### Custom Hook Pattern
```typescript
export const useSystemDesignGeneration = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const generateDesign = useCallback(async (input: SystemDesignInput) => {
    // Implementation
  }, []);
  
  return { generateDesign, loading, error };
};
```

### API Route Pattern
```typescript
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<APIResponse<SystemDesign>>
) {
  try {
    // Implementation
  } catch (error) {
    // Error handling
  }
}
```

## Documentation Standards
- Document all public APIs
- Use JSDoc for functions and components
- Maintain README files for each major feature
- Keep inline comments for complex logic only

## Git Workflow
- Use conventional commits
- Create feature branches for new development
- Require PR reviews for main branch
- Use semantic versioning for releases

Remember to always consider the user experience, performance implications, and maintainability when writing code for this project.
