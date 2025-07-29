# System Design Dashboard - Complete Development Prompt

## 🎯 Project Overview
Build a comprehensive **System Design Dashboard** that generates professional system architecture designs based on user inputs. The application should serve both learning and professional use cases, providing detailed system designs with visual diagrams and comprehensive documentation.

## 🏗️ Core Features & Requirements

### 1. Input Methods
Create two primary input pathways:

**Free Form Input:**
- Natural language description of the system
- Single-line system goals or definitions
- Conversational interface for requirements gathering

**Structured Intake Form:**
- Daily Active Users (DAU) & Monthly Active Users (MAU)
- Queries Per Second (QPS) estimates
- Data size estimates (storage requirements)
- Traffic patterns (burst, steady, seasonal)
- Read/Write ratio percentages
- Geographic distribution (global, regional, local)
- Peak traffic multiplier factors
- Consistency requirements (eventual vs strong)
- Latency requirements (real-time, near-real-time, batch)
- Availability targets (99.9%, 99.99%, 99.999%)
- Budget constraints
- Compliance requirements (GDPR, HIPAA, etc.)
- Technology preferences/constraints

### 2. System Library
Pre-populate with popular system designs:
- **Social Media:** Facebook, Twitter, Instagram, LinkedIn, TikTok
- **Streaming:** Netflix, YouTube, Spotify, Twitch
- **E-commerce:** Amazon, eBay, Shopify
- **Transportation:** Uber, Lyft, DoorDash
- **Travel:** Airbnb, Booking.com
- **Maps & Location:** Google Maps, Waze
- **Communication:** WhatsApp, Slack, Zoom, Discord
- **Search:** Google Search, Elasticsearch
- **Cloud Storage:** Dropbox, Google Drive
- **Payment:** PayPal, Stripe, Venmo
- **Dating:** Tinder, Bumble
- **News:** Reddit, Medium, Hacker News

### 3. AI-Powered Generation Engine
Implement dual generation approaches:

**Rule-Based Engine:**
- Predefined decision trees based on scale, requirements
- Template-based architecture patterns
- Component selection based on input parameters
- Cost optimization algorithms

**LLM Integration:**
- OpenAI GPT-4 or Claude integration for creative system design
- Prompt engineering for consistent output format
- Fallback to rule engine if LLM unavailable
- Hybrid approach combining both methods

## 📋 System Design Output Format

Generate comprehensive system designs following this exact structure:

### 🔍 Problem Definition & Goals
- Clear system description and purpose
- Functional requirements (what the system must do)
- Non-functional requirements (performance, security, scalability)
- Scale assumptions with specific numbers
- Constraints, SLAs, latency & availability targets

### 🧱 High-Level Architecture
- Key components and their interactions
- Mermaid diagram representation
- Synchronous vs asynchronous flow identification
- Complete user journey: Frontend → Backend → Database flow

### 🧠 Component Design
Detailed breakdown of:
- Web/API layer architecture
- Backend services (microservice decomposition)
- Authentication & Authorization systems
- File/object storage solutions
- Business logic orchestration

### 🗃️ Data Modeling & Storage
- Data entities and relationships
- Database selection rationale (SQL/NoSQL/Time-series/Graph)
- Schema examples with sample data
- Comprehensive caching strategy

### 📡 APIs & Communication
- Internal and external API specifications
- API gateway configuration
- Communication protocols (REST/gRPC/WebSockets)
- Sample request/response formats

### 🕸️ Scalability & Performance
- Load balancing strategies
- Horizontal scaling approaches
- Database sharding techniques
- Asynchronous processing with message queues
- Rate limiting and throttling mechanisms

### 🔐 Security
- Authentication mechanisms (OAuth, JWT, SSO)
- Authorization models (RBAC/ABAC)
- Data encryption strategies
- Security best practices and OWASP compliance

### 📈 Observability & Monitoring
- Logging strategy and tools
- Metrics collection and visualization
- Alert configurations
- Dashboard designs

### 🧪 Testing & CI/CD
- Testing pyramid (unit, integration, e2e, load tests)
- CI/CD pipeline design
- Deployment strategies and rollback procedures

### ☁️ Infrastructure & Deployment
- Cloud provider selection and services
- Containerization and orchestration
- Infrastructure-as-Code implementation
- Environment management

### 🧩 Trade-Offs & Alternatives
- Decision justifications with pros/cons
- Alternative technology stack options
- Scaling considerations for different loads

## 🛠️ Technical Stack & Implementation

### Frontend (React.js + Tailwind CSS)
```typescript
// Key components to implement:
- InputForm: Structured parameter collection
- FreeFormInput: Natural language interface
- DiagramViewer: Mermaid chart rendering
- SystemLibrary: Pre-built designs browser
- UserDashboard: Personal design management
- EditableDesign: Modification interface
```

### Backend (Next.js/Node.js)
```typescript
// Core services:
- SystemDesignGenerator: Main generation logic
- RuleEngine: Parameter-based design decisions
- LLMIntegration: AI-powered generation
- MermaidGenerator: Diagram creation service
- UserManagement: Authentication & preferences
- DesignRepository: CRUD operations for designs
```

### Database Schema (Supabase)
```sql
-- Core tables:
users, system_designs, design_templates, 
user_preferences, generation_history, 
component_library, design_ratings
```

### Local Development Database
Use SQLite for development with identical schema structure.

## 🎨 User Experience Design

### Dashboard Layout
- Clean, professional interface with dark/light mode
- Sidebar navigation: New Design, Library, My Designs, Settings
- Main content area with tabbed interface
- Real-time generation progress indicators

### Design Generation Flow
1. **Input Collection:** Form or free-text with smart suggestions
2. **Processing:** Visual progress with estimation
3. **Result Display:** Tabbed output with diagram and documentation
4. **Actions:** Save, Edit, Export, Share options

### Library Experience
- Searchable grid of pre-built designs
- Filter by category, scale, technology
- Preview and detailed view options
- One-click customization starting point

## 🔧 Advanced Features

### Smart Suggestions
- Auto-complete for common requirements
- Technology recommendations based on scale
- Cost estimation integration
- Performance prediction models

### Collaboration Features
- Design sharing with view/edit permissions
- Comment system for team feedback
- Version history and comparison
- Export to various formats (PDF, PNG, JSON)

### Learning Integration
- Educational tooltips explaining design decisions
- Best practices recommendations
- Resource links for further learning
- Design pattern explanations

## 📊 Analytics & Optimization

### Usage Tracking
- Generation success rates
- Popular system types
- User journey analytics
- Performance bottleneck identification

### Quality Metrics
- Design completeness scores
- User satisfaction ratings
- Expert review integration
- Continuous improvement feedback loops

## 🚀 Development Phases

### Phase 1: Core MVP
- Basic input forms and free-text interface
- Rule-based generation for 5 popular systems
- Mermaid diagram rendering
- User authentication and basic dashboard

### Phase 2: Enhanced Features
- LLM integration for custom systems
- Complete system library (20+ designs)
- Edit and save functionality
- Export capabilities

### Phase 3: Professional Features
- Advanced collaboration tools
- Performance analytics
- Cost estimation integration
- Enterprise authentication options

## 🧪 Testing Strategy
- Unit tests for generation logic
- Integration tests for API endpoints
- E2E tests for complete user workflows
- Load testing for concurrent generations
- User acceptance testing with system design experts

## 📈 Success Metrics
- Time to generate complete system design < 2 minutes
- User satisfaction score > 4.5/5
- Design accuracy validated by senior engineers
- 90%+ uptime for generation services
- Sub-second diagram rendering

This prompt provides a comprehensive foundation for building a professional-grade system design dashboard that serves both educational and practical purposes while maintaining high standards for accuracy and usability.
