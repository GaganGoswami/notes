# Journey2College : College Admission & Sponsorship Journey Guide

## Project Overview
Create a comprehensive full-stack College Admission & Sponsorship Journey Guide application with React + TypeScript frontend and Spring Boot backend. The application helps students navigate the college admission process and find scholarship opportunities.

## Architecture Requirements

### Frontend Stack
- **Framework**: React 18 with TypeScript
- **Build Tool**: Vite
- **Styling**: Custom CSS with CSS variables (NOT TailwindCSS due to PostCSS conflicts)
- **State Management**: React Query + Context API
- **Routing**: React Router v6
- **UI Components**: Custom components with inline styles
- **Icons**: Emoji-based icons for consistency
- **Animations**: CSS transitions and hover effects

### Backend Stack (Ready for Integration)
- **Framework**: Spring Boot 3.2
- **Language**: Java 21
- **Database**: PostgreSQL 15
- **ORM**: Spring Data JPA
- **Security**: Spring Security with JWT

## Design System Specifications

### Color Palette
```css
:root {
  /* Primary Colors */
  --color-teal-500: rgba(33, 128, 141, 1);
  --color-teal-600: rgba(29, 116, 128, 1);
  --color-teal-700: rgba(26, 104, 115, 1);
  
  /* Background Colors */
  --color-cream-50: rgba(252, 252, 249, 1);
  --color-cream-100: rgba(255, 255, 253, 1);
  
  /* Text Colors */
  --color-slate-900: rgba(19, 52, 59, 1);
  --color-slate-600: rgba(71, 85, 105, 1);
  --color-slate-700: rgba(51, 65, 85, 1);
  --color-slate-500: rgba(98, 108, 113, 1);
  
  /* Utility Colors */
  --color-gray-200: rgba(245, 245, 245, 1);
  --color-gray-300: rgba(167, 169, 169, 1);
  --color-gray-400: rgba(119, 124, 124, 1);
  --color-red-500: rgba(192, 21, 47, 1);
  --color-orange-500: rgba(168, 75, 47, 1);
  --color-green-600: rgba(22, 163, 74, 1);
  --color-blue-600: rgba(37, 99, 235, 1);
  
  /* Semantic Colors */
  --color-background: var(--color-cream-50);
  --color-surface: var(--color-cream-100);
  --color-text: var(--color-slate-900);
  --color-primary: var(--color-teal-500);
}
```

### Typography & Layout
```css
:root {
  --font-family-base: "FKGroteskNeue", "Geist", "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  --radius-base: 8px;
  --radius-lg: 12px;
  --radius-large: 16px;
}
```

### Button System
```css
.btn-primary {
  background: var(--color-teal-500);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: var(--radius-base);
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-secondary {
  background: var(--color-gray-200);
  color: var(--color-slate-900);
  border: 1px solid var(--color-gray-300);
  padding: 8px 16px;
  border-radius: var(--radius-base);
  cursor: pointer;
  transition: all 0.2s ease;
}
```

## Project Structure

```
src/
├── App-simple.tsx                 # Main application router
├── main.tsx                       # Application entry point
├── index.css                      # Complete CSS system
├── components/
│   ├── ui/
│   │   ├── SimpleNavbar.tsx       # Sidebar navigation
│   │   ├── Card.tsx               # Card components
│   │   ├── Button.tsx             # Button components
│   │   └── Input.tsx              # Input components
│   └── dashboard/
│       └── SimpleDashboard.tsx    # Dashboard content
├── pages/
│   ├── Dashboard.tsx              # Overview page
│   ├── Timeline.tsx               # Grade-based task management
│   ├── Profile.tsx                # Student profile management
│   ├── Colleges.tsx               # College explorer
│   ├── Scholarships.tsx           # Scholarship hub
│   └── Applications.tsx           # Application manager
├── contexts/
│   ├── AuthContext.tsx            # Authentication context
│   ├── ThemeContext.tsx           # Theme management
│   └── index.ts                   # Context exports
├── types/
│   └── index.ts                   # TypeScript definitions
└── lib/
    └── utils.ts                   # Utility functions
```

## Feature Implementation Requirements

### 1. Dashboard (/dashboard)
**Purpose**: Overview statistics and progress tracking
**Key Features**:
- Statistics cards (5 applications in progress, 3 scholarships applied, 12 tasks remaining)
- Timeline progress bar showing 65% completion
- Recent activity feed with timestamped updates
- Quick action buttons for navigation
- Responsive grid layout with proper styling

**Implementation**:
```tsx
// Statistics with large numbers and descriptive labels
// Progress bar with teal color and percentage indicators
// Activity feed with left border accent and clean typography
// Action buttons using .btn-primary and .btn-secondary classes
```

### 2. Timeline (/timeline)
**Purpose**: Grade-based task management system
**Key Features**:
- Grade sections (9th, 10th, 11th, 12th)
- Interactive task completion toggles
- Priority-based color coding (high/medium/low)
- Category filtering with emoji icons (📋 application, 📝 test, ✍️ essay, 💰 scholarship)
- Progress tracking with completion percentages
- Deadline management with urgency indicators

**Task Structure**:
```tsx
interface TimelineTask {
  id: string;
  title: string;
  description: string;
  deadline: string;
  category: 'application' | 'test' | 'essay' | 'scholarship';
  priority: 'high' | 'medium' | 'low';
  completed: boolean;
  grade: 9 | 10 | 11 | 12;
}
```

### 3. Profile (/profile)
**Purpose**: Comprehensive student information management
**Key Features**:
- Multi-tab interface: Personal Info, Academic, Test Scores, Activities, Achievements
- Editable forms with save/edit toggle functionality
- Personal information (name, email, phone, address)
- Academic records (GPA, class rank, coursework)
- Test scores (SAT/ACT with dates and sections)
- Extracurricular activities with leadership indicators
- Awards and achievements with categorization
- Badge system for activities and achievements

**Profile Structure**:
```tsx
interface UserProfile {
  personalInfo: PersonalInfo;
  academicInfo: AcademicInfo;
  testScores: TestScores;
  activities: Activity[];
  achievements: Achievement[];
}
```

### 4. Colleges (/colleges)
**Purpose**: College exploration and comparison tool
**Key Features**:
- Advanced filtering system (type, region, size, tuition, SAT scores)
- Dual view modes (grid cards and detailed list)
- College comparison with side-by-side metrics
- Favorites system with heart-based bookmarking
- Application status tracking integration
- Comprehensive college data (rankings, admission rates, majors)
- Search functionality with real-time filtering

**College Data Structure**:
```tsx
interface College {
  id: string;
  name: string;
  location: string;
  type: 'public' | 'private' | 'community';
  tuition: number;
  enrollment: number;
  acceptanceRate: number;
  satRange: { min: number; max: number };
  ranking: number;
  majors: string[];
  applicationStatus?: string;
  isFavorite: boolean;
}
```

### 5. Scholarships (/scholarships)
**Purpose**: Scholarship discovery and application tracking
**Key Features**:
- Smart matching system with percentage-based compatibility
- Category organization (merit, need-based, athletic, creative, minority, career-specific, community)
- Advanced filtering (difficulty, deadlines, amounts, match scores)
- Application status management with progress tracking
- Deadline urgency indicators with color coding
- Scholarship statistics dashboard
- Financial tracking with total value calculations

**Scholarship Structure**:
```tsx
interface Scholarship {
  id: string;
  title: string;
  provider: string;
  amount: number;
  deadline: string;
  category: string;
  difficulty: 'low' | 'medium' | 'high';
  matchScore: number;
  description: string;
  requirements: string[];
  applicationStatus: 'not-applied' | 'in-progress' | 'submitted' | 'awarded' | 'rejected';
}
```

### 6. Applications (/applications)
**Purpose**: Comprehensive application management system
**Key Features**:
- Kanban board interface with drag-and-drop-style columns
- Alternative list view for detailed spreadsheet-like management
- Application status workflow (not-started → in-progress → submitted → accepted/rejected/waitlisted)
- Document completion progress tracking
- Essay management with word counts and status
- Priority classification (safety/target/reach schools)
- Financial tracking with application fee totals
- Deadline proximity warnings with color coding
- Scholarship opportunity integration

**Application Structure**:
```tsx
interface Application {
  id: string;
  collegeName: string;
  program: string;
  deadline: string;
  applicationFee: number;
  status: 'not-started' | 'in-progress' | 'submitted' | 'accepted' | 'rejected' | 'waitlisted';
  priority: 'safety' | 'target' | 'reach';
  documents: DocumentProgress;
  essays: Essay[];
  notes: string;
  scholarshipOpportunities: number;
}
```

## Navigation System

### SimpleNavbar Component
**Requirements**:
- Fixed sidebar with cream background (#FCFCF9)
- React Router Link integration with active state indicators
- Navigation items: Dashboard, Timeline, Profile, Colleges, Scholarships, Applications
- Theme toggle functionality (light/dark mode ready)
- User profile display section
- Sign out functionality
- Hover effects and smooth transitions

## Styling Guidelines

### Core Principles
1. **No TailwindCSS**: Use custom CSS with CSS variables only
2. **Inline Styles**: Primary styling method for components
3. **CSS Variables**: Consistent color and spacing system
4. **Responsive Design**: Mobile-first approach with proper breakpoints
5. **Hover Effects**: Subtle animations for interactive elements
6. **Color Coding**: Meaningful color usage for status and priority

### Layout Patterns
```tsx
// Container styling pattern
const containerStyle = {
  padding: '24px',
  maxWidth: '1200px',
  margin: '0 auto',
};

// Card styling pattern
const cardStyle = {
  background: 'var(--color-cream-100)',
  border: '1px solid var(--color-gray-200)',
  borderRadius: 'var(--radius-lg)',
  padding: '24px',
  marginBottom: '24px',
};

// Button hover pattern
onMouseEnter={(e) => {
  const target = e.target as HTMLButtonElement;
  target.style.background = 'var(--color-teal-600)';
}}
```

## Data Management

### Mock Data Requirements
Create comprehensive mock data for all features:
- Student profiles with realistic academic information
- Diverse college database with accurate statistics
- Varied scholarship opportunities across categories
- Complex application workflows with multiple documents
- Timeline tasks organized by grade and priority

### State Management Patterns
- React hooks for local component state
- Context providers for global state (auth, theme)
- React Query integration for future API connections
- Consistent data structures across all features

## Technical Implementation Details

### Package.json Dependencies
```json
{
  "dependencies": {
    "@tanstack/react-query": "^5.83.0",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "react-router-dom": "^7.7.1",
    "react-hook-form": "^7.61.1",
    "react-hot-toast": "^2.5.2",
    "axios": "^1.11.0"
  }
}
```

### App-simple.tsx Structure
```tsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'react-hot-toast';

// All page imports
import { Dashboard } from './pages/Dashboard';
import { Timeline } from './pages/Timeline';
import { Profile } from './pages/Profile';
import { Colleges } from './pages/Colleges';
import { Scholarships } from './pages/Scholarships';
import { Applications } from './pages/Applications';
import { SimpleNavbar } from './components/ui/SimpleNavbar';

// Context providers
import { AuthProvider, ThemeProvider } from './contexts';

// Main layout with sidebar + content area
```

### Context Setup
```tsx
// AuthContext: Mock user authentication
// ThemeContext: Light/dark mode management
// Both with TypeScript interfaces and proper error handling
```

## Quality Standards

### User Experience
- Professional, clean design with consistent spacing
- Intuitive navigation with clear visual hierarchy
- Responsive layout that works on all screen sizes
- Loading states and error handling
- Accessibility considerations with proper contrast

### Code Quality
- TypeScript for all components and data structures
- Consistent naming conventions and file organization
- Proper error boundaries and validation
- Performance optimization with React best practices
- Clean, maintainable code with proper comments

### Visual Design
- Consistent color usage with semantic meaning
- Professional typography with proper font weights
- Subtle animations and hover effects
- Clean card-based layouts with proper spacing
- Status indicators with meaningful color coding

## Final Implementation Checklist

✅ **Project Setup**
- [x] Vite + React + TypeScript configuration
- [x] Custom CSS system with variables
- [x] React Router setup
- [x] Context providers (Auth, Theme)

✅ **Core Components**
- [x] SimpleNavbar with routing
- [x] Button system (primary/secondary)
- [x] Card components
- [x] Layout patterns

✅ **Feature Pages**
- [x] Dashboard with statistics and progress
- [x] Timeline with grade-based task management
- [x] Profile with multi-tab interface
- [x] Colleges with filtering and comparison
- [x] Scholarships with matching and tracking
- [x] Applications with Kanban and list views

✅ **Data & State**
- [x] Comprehensive TypeScript interfaces
- [x] Mock data for all features
- [x] State management patterns
- [x] React Query integration ready

✅ **Styling & UX**
- [x] Complete CSS variable system
- [x] Responsive design patterns
- [x] Hover effects and transitions
- [x] Professional visual hierarchy

## Current Status
The application is **100% complete** with all 6 major features implemented and fully functional. Navigation works perfectly, all pages are interactive, and the design system is consistent throughout. The app is ready for backend integration and advanced features.

**Live Development Server**: `npm run dev` (runs on http://localhost:5173)
**Main Route**: `/dashboard` (default landing page)

This prompt will recreate the exact College Admission & Sponsorship Journey Guide application with all features, styling, and functionality as implemented.
