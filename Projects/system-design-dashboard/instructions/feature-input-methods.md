# Feature: Input Methods

## Overview
The Input Methods feature provides two distinct pathways for users to specify their system requirements: structured forms and natural language processing. This dual approach caters to both technical users who prefer precise parameter specification and non-technical users who want to describe their needs conversationally.

## User Stories

### As a technical user
- I want to specify exact system parameters (DAU, QPS, latency) so that I get precise system designs
- I want to use dropdown menus and sliders so that I can quickly input common values
- I want to see real-time validation so that I know my inputs are valid

### As a non-technical user
- I want to describe my system in plain English so that I don't need to understand technical metrics
- I want the system to ask clarifying questions so that I provide complete requirements
- I want suggestions for common system types so that I can get started quickly

## Technical Requirements

### Structured Input Form
```typescript
interface SystemDesignInput {
  // Scale Parameters
  dailyActiveUsers: number;
  monthlyActiveUsers: number;
  queriesPerSecond: number;
  dataVolumeGB: number;
  
  // Performance Requirements
  readWriteRatio: number; // 0-100 (percentage reads)
  latencyRequirement: 'real-time' | 'near-real-time' | 'batch';
  availabilityTarget: '99.9' | '99.99' | '99.999';
  
  // Geographic & Traffic
  geographicDistribution: 'global' | 'regional' | 'local';
  trafficPattern: 'steady' | 'burst' | 'seasonal';
  peakMultiplier: number; // 1-100x
  
  // Technical Constraints  
  consistencyRequirement: 'strong' | 'eventual';
  budgetConstraint: 'low' | 'medium' | 'high' | 'unlimited';
  complianceRequirements: string[];
  technologyPreferences: string[];
}
```

### Free-Form Input Processing
```typescript
interface FreeFormInput {
  description: string;
  systemType?: string;
  clarifyingQuestions?: string[];
  extractedParameters?: Partial<SystemDesignInput>;
}

interface NLPProcessor {
  extractParameters(input: string): Promise<Partial<SystemDesignInput>>;
  generateClarifyingQuestions(input: string): Promise<string[]>;
  suggestSystemTypes(input: string): Promise<string[]>;
}
```

## Component Architecture

### StructuredInputForm Component
```typescript
interface StructuredInputFormProps {
  onSubmit: (data: SystemDesignInput) => void;
  initialData?: Partial<SystemDesignInput>;
  loading?: boolean;
}

// Features:
// - Multi-step wizard interface
// - Real-time validation
// - Smart defaults based on system type
// - Parameter interdependency validation
// - Progress saving
```

### FreeFormInputForm Component
```typescript
interface FreeFormInputFormProps {
  onSubmit: (data: FreeFormInput) => void;
  onParametersExtracted: (params: Partial<SystemDesignInput>) => void;
  loading?: boolean;
}

// Features:
// - Rich text editor with syntax highlighting
// - Auto-suggestions for common terms
// - Real-time parameter extraction preview
// - Clarifying question interface
// - System type suggestions
```

### InputMethodSelector Component
```typescript
interface InputMethodSelectorProps {
  selectedMethod: 'structured' | 'freeform';
  onMethodChange: (method: 'structured' | 'freeform') => void;
}

// Features:
// - Toggle between input methods
// - Preview of each method
// - Recommendation based on user profile
```

## Form Validation Rules

### Structured Input Validation
```typescript
const validationRules = {
  dailyActiveUsers: {
    min: 1,
    max: 10_000_000_000,
    required: true
  },
  queriesPerSecond: {
    min: 1,
    max: 1_000_000,
    required: true,
    dependsOn: ['dailyActiveUsers'] // QPS should be reasonable for DAU
  },
  readWriteRatio: {
    min: 0,
    max: 100,
    required: true
  },
  // ... additional rules
};
```

### Smart Defaults
```typescript
const getSmartDefaults = (systemType: string): Partial<SystemDesignInput> => {
  const defaults = {
    'social-media': {
      readWriteRatio: 80, // Heavy read
      latencyRequirement: 'near-real-time',
      trafficPattern: 'burst'
    },
    'streaming': {
      readWriteRatio: 95, // Very heavy read
      latencyRequirement: 'real-time',
      trafficPattern: 'steady'
    },
    // ... other system types
  };
  return defaults[systemType] || {};
};
```

## Natural Language Processing

### Parameter Extraction Logic
```typescript
const extractParametersFromText = async (text: string): Promise<Partial<SystemDesignInput>> => {
  // Use LLM or rule-based extraction
  const prompt = `
    Extract system design parameters from this description:
    "${text}"
    
    Return JSON with estimated values for:
    - dailyActiveUsers
    - queriesPerSecond  
    - readWriteRatio
    - latencyRequirement
    - geographicDistribution
    - trafficPattern
  `;
  
  // Process with LLM or regex patterns
};
```

### Clarifying Questions Generation
```typescript
const generateQuestions = (input: string): string[] => {
  const questions = [];
  
  if (!containsScaleInfo(input)) {
    questions.push("How many users do you expect to have daily?");
  }
  
  if (!containsPerformanceInfo(input)) {
    questions.push("What are your latency requirements (real-time, near-real-time, or batch)?");
  }
  
  if (!containsGeographicInfo(input)) {
    questions.push("Will your users be global, regional, or local?");
  }
  
  return questions;
};
```

## UI/UX Design

### Structured Form Layout
- **Step 1**: System Overview (type, basic scale)
- **Step 2**: Performance Requirements (latency, availability)
- **Step 3**: Scale & Traffic (DAU, QPS, patterns)
- **Step 4**: Technical Constraints (budget, compliance, tech stack)
- **Step 5**: Review & Generate

### Free-Form Interface
- Large text area with placeholder examples
- Side panel showing extracted parameters in real-time
- Suggestion chips for common system types
- Interactive Q&A section for clarifications

### Responsive Design
```css
/* Mobile-first approach */
.input-form {
  @apply w-full max-w-2xl mx-auto p-4;
}

.form-section {
  @apply mb-8 p-6 bg-white rounded-lg shadow-sm;
}

.parameter-input {
  @apply w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500;
}

/* Tablet and desktop */
@media (min-width: 768px) {
  .input-form {
    @apply max-w-4xl grid grid-cols-2 gap-6;
  }
}
```

## State Management

### Form State
```typescript
interface InputFormState {
  method: 'structured' | 'freeform';
  structuredData: Partial<SystemDesignInput>;
  freeformData: FreeFormInput;
  currentStep: number;
  validationErrors: Record<string, string>;
  isSubmitting: boolean;
}

const useInputForm = () => {
  const [state, setState] = useState<InputFormState>({
    method: 'structured',
    structuredData: {},
    freeformData: { description: '' },
    currentStep: 1,
    validationErrors: {},
    isSubmitting: false
  });
  
  // Form management methods
};
```

## API Integration

### Structured Input Endpoint
```typescript
// POST /api/generate/structured
interface StructuredGenerationRequest {
  input: SystemDesignInput;
  generationMethod: 'rule-based' | 'llm' | 'hybrid';
}

interface StructuredGenerationResponse {
  design: SystemDesign;
  processingTime: number;
  confidence: number;
}
```

### Free-Form Processing Endpoint
```typescript
// POST /api/process/freeform
interface FreeFormProcessingRequest {
  description: string;
  context?: string;
}

interface FreeFormProcessingResponse {
  extractedParameters: Partial<SystemDesignInput>;
  clarifyingQuestions: string[];
  suggestedSystemTypes: string[];
  confidence: number;
}
```

## Testing Strategy

### Unit Tests
- Form validation logic
- Parameter extraction algorithms
- Default value calculations
- Input sanitization

### Integration Tests
- End-to-end form submission
- API integration for both methods
- State persistence across sessions
- Cross-browser form behavior

### User Testing
- A/B test structured vs free-form preference
- Usability testing for form flow
- Accessibility testing for screen readers
- Mobile experience optimization

## Performance Considerations

### Form Optimization
- Debounce input validation (300ms)
- Lazy load form steps
- Cache user preferences
- Optimize bundle size with code splitting

### NLP Processing
- Cache common parameter extractions
- Implement request debouncing
- Use streaming responses for real-time feedback
- Fallback to rule-based extraction if LLM fails

## Accessibility

### WCAG Compliance
- Proper form labels and ARIA attributes
- Keyboard navigation support
- Screen reader compatibility
- High contrast mode support
- Focus management across form steps

### Progressive Enhancement
- JavaScript-free form submission
- Graceful degradation for older browsers
- Offline form saving capability
- Mobile-optimized input types

## Error Handling

### Validation Errors
```typescript
interface ValidationError {
  field: string;
  message: string;
  type: 'required' | 'format' | 'range' | 'dependency';
  suggestedFix?: string;
}
```

### User-Friendly Messages
- "Please enter a number between 1 and 1,000,000 for queries per second"
- "Your QPS seems high for the number of daily users. Consider adjusting one of these values."
- "We couldn't extract some parameters from your description. Please answer these questions to proceed."

## Success Metrics

### User Experience
- Form completion rate > 85%
- Average time to submit < 3 minutes
- User satisfaction score > 4.2/5
- Error resolution rate > 90%

### Technical Performance
- Form validation response < 100ms
- Parameter extraction accuracy > 85%
- API response time < 2 seconds
- Mobile form performance score > 90
