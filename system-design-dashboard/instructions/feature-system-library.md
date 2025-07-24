# Feature: System Library

## Overview
The System Library provides a curated collection of pre-built system designs for popular applications and services. Users can browse, preview, and use these designs as starting points for their own projects, with the ability to customize parameters and generate variations.

## User Stories

### As a learning user
- I want to explore system designs of popular apps so that I can understand how they work
- I want to see different architectural approaches so that I can learn best practices
- I want to compare similar systems so that I can understand trade-offs

### As a professional user
- I want quick access to proven architectures so that I can accelerate my design process
- I want to use existing designs as templates so that I don't start from scratch
- I want to customize library designs so that they fit my specific requirements

### As a system design student
- I want to study real-world examples so that I can prepare for interviews
- I want to see complete documentation so that I understand all aspects of the design
- I want interactive exploration so that I can dive deep into specific components

## System Categories & Examples

### Social Media Platforms
```typescript
const socialMediaSystems = [
  {
    id: 'facebook',
    name: 'Facebook',
    description: 'Social networking platform with news feed, messaging, and content sharing',
    category: 'social-media',
    scale: {
      dau: 2_900_000_000,
      mau: 3_000_000_000,
      qps: 63_000,
      dataVolume: '100+ PB'
    },
    keyFeatures: ['News Feed', 'Real-time Chat', 'Photo/Video Sharing', 'Friend Graph'],
    architecturalHighlights: ['Graph Database', 'CDN for Media', 'Real-time Messaging', 'ML Recommendations']
  },
  {
    id: 'twitter',
    name: 'Twitter',
    description: 'Microblogging platform with real-time updates and social following',
    category: 'social-media',
    scale: { dau: 450_000_000, mau: 450_000_000, qps: 18_000, dataVolume: '40+ TB/day' },
    keyFeatures: ['Tweet Timeline', 'Real-time Updates', 'Hashtag System', 'Trending Topics'],
    architecturalHighlights: ['Fan-out Service', 'Real-time Processing', 'Distributed Cache', 'Search Engine']
  },
  // ... more social media systems
];
```

### Streaming & Media
```typescript
const streamingSystems = [
  {
    id: 'netflix',
    name: 'Netflix',
    description: 'Video streaming platform with personalized recommendations',
    category: 'streaming',
    scale: { dau: 230_000_000, mau: 260_000_000, qps: 1_000_000, dataVolume: '15+ PB' },
    keyFeatures: ['Video Streaming', 'Recommendation Engine', 'Content Catalog', 'User Profiles'],
    architecturalHighlights: ['CDN Architecture', 'Microservices', 'A/B Testing Platform', 'ML Pipeline']
  },
  {
    id: 'youtube',
    name: 'YouTube',
    description: 'Video sharing platform with user-generated content',
    category: 'streaming',
    scale: { dau: 2_000_000_000, mau: 2_700_000_000, qps: 500_000, dataVolume: '720,000 hours/hour' },
    keyFeatures: ['Video Upload', 'Video Streaming', 'Comments System', 'Channel Subscriptions'],
    architecturalHighlights: ['Video Processing Pipeline', 'Global CDN', 'Real-time Analytics', 'Content Moderation']
  }
];
```

### E-commerce & Marketplace
```typescript
const ecommerceSystems = [
  {
    id: 'amazon',
    name: 'Amazon',
    description: 'E-commerce marketplace with logistics and cloud services',
    category: 'ecommerce',
    scale: { dau: 200_000_000, mau: 300_000_000, qps: 80_000, dataVolume: '50+ PB' },
    keyFeatures: ['Product Catalog', 'Order Management', 'Payment Processing', 'Inventory Management'],
    architecturalHighlights: ['Service-Oriented Architecture', 'Event-Driven Architecture', 'Distributed Database', 'ML Recommendations']
  }
];
```

### Transportation & Mobility
```typescript
const transportationSystems = [
  {
    id: 'uber',
    name: 'Uber',
    description: 'Ride-sharing platform with real-time matching and GPS tracking',
    category: 'transportation',
    scale: { dau: 25_000_000, mau: 130_000_000, qps: 12_000, dataVolume: '100+ TB' },
    keyFeatures: ['Real-time Matching', 'GPS Tracking', 'Dynamic Pricing', 'Payment Processing'],
    architecturalHighlights: ['Geospatial Database', 'Real-time Processing', 'Microservices', 'Event Streaming']
  }
];
```

## Data Models

### Library System Schema
```typescript
interface LibrarySystem {
  id: string;
  name: string;
  description: string;
  category: SystemCategory;
  thumbnail: string;
  tags: string[];
  
  // Scale Information
  scale: {
    dailyActiveUsers: number;
    monthlyActiveUsers: number;
    queriesPerSecond: number;
    dataVolume: string;
    geographicReach: 'global' | 'regional' | 'local';
  };
  
  // System Design Details
  systemDesign: SystemDesign;
  
  // Metadata
  createdAt: Date;
  updatedAt: Date;
  version: string;
  difficulty: 'beginner' | 'intermediate' | 'advanced';
  estimatedReadTime: number; // minutes
  
  // Analytics
  views: number;
  likes: number;
  bookmarks: number;
  difficulty_rating: number; // 1-5 stars
}

type SystemCategory = 
  | 'social-media'
  | 'streaming'
  | 'ecommerce'
  | 'transportation'
  | 'communication'
  | 'maps-location'
  | 'search'
  | 'cloud-storage'
  | 'payment'
  | 'dating'
  | 'news-content';
```

### User Interaction Data
```typescript
interface UserLibraryInteraction {
  userId: string;
  systemId: string;
  action: 'view' | 'like' | 'bookmark' | 'customize' | 'export';
  timestamp: Date;
  sessionId: string;
  metadata?: Record<string, any>;
}

interface UserPreferences {
  userId: string;
  favoriteCategories: SystemCategory[];
  bookmarkedSystems: string[];
  difficultyPreference: 'beginner' | 'intermediate' | 'advanced';
  viewHistory: {
    systemId: string;
    viewedAt: Date;
    timeSpent: number; // seconds
  }[];
}
```

## Component Architecture

### LibraryBrowser Component
```typescript
interface LibraryBrowserProps {
  initialCategory?: SystemCategory;
  searchQuery?: string;
  onSystemSelect: (system: LibrarySystem) => void;
}

const LibraryBrowser: React.FC<LibraryBrowserProps> = ({
  initialCategory,
  searchQuery,
  onSystemSelect
}) => {
  // Features:
  // - Grid/list view toggle
  // - Category filtering
  // - Search functionality
  // - Sort options (popularity, difficulty, name)
  // - Infinite scroll/pagination
  // - Quick preview on hover
};
```

### SystemCard Component
```typescript
interface SystemCardProps {
  system: LibrarySystem;
  viewMode: 'grid' | 'list';
  onSelect: (system: LibrarySystem) => void;
  onBookmark: (systemId: string) => void;
  onLike: (systemId: string) => void;
}

const SystemCard: React.FC<SystemCardProps> = ({
  system,
  viewMode,
  onSelect,
  onBookmark,
  onLike
}) => {
  // Features:
  // - Thumbnail with category badge
  // - Scale metrics preview
  // - Difficulty indicator
  // - Quick action buttons
  // - Progress indicator for viewed systems
};
```

### SystemDetail Component
```typescript
interface SystemDetailProps {
  system: LibrarySystem;
  onCustomize: (system: LibrarySystem) => void;
  onExport: (format: 'pdf' | 'png' | 'json') => void;
  onClose: () => void;
}

const SystemDetail: React.FC<SystemDetailProps> = ({
  system,
  onCustomize,
  onExport,
  onClose
}) => {
  // Features:
  // - Full system design display
  // - Interactive Mermaid diagrams
  // - Tabbed content sections
  // - Related systems suggestions
  // - Learning resources links
};
```

## Search & Filtering

### Search Implementation
```typescript
interface SearchFilters {
  category?: SystemCategory[];
  difficulty?: ('beginner' | 'intermediate' | 'advanced')[];
  scale?: {
    minDAU?: number;
    maxDAU?: number;
    minQPS?: number;
    maxQPS?: number;
  };
  technologies?: string[];
  features?: string[];
}

const useLibrarySearch = () => {
  const [query, setQuery] = useState('');
  const [filters, setFilters] = useState<SearchFilters>({});
  const [sortBy, setSortBy] = useState<'popularity' | 'name' | 'difficulty' | 'date'>('popularity');
  
  const searchSystems = useMemo(() => {
    return debounce(async (searchQuery: string, searchFilters: SearchFilters) => {
      // Implementation with Fuse.js or similar
      const fuse = new Fuse(systems, {
        keys: ['name', 'description', 'keyFeatures', 'architecturalHighlights'],
        threshold: 0.3
      });
      
      let results = searchQuery ? fuse.search(searchQuery).map(r => r.item) : systems;
      
      // Apply filters
      results = applyFilters(results, searchFilters);
      
      // Apply sorting
      results = applySorting(results, sortBy);
      
      return results;
    }, 300);
  }, [sortBy]);
  
  return { query, setQuery, filters, setFilters, searchSystems, sortBy, setSortBy };
};
```

### Advanced Filtering UI
```typescript
const FilterPanel: React.FC<{
  filters: SearchFilters;
  onFiltersChange: (filters: SearchFilters) => void;
}> = ({ filters, onFiltersChange }) => {
  return (
    <div className="filter-panel">
      {/* Category Filter */}
      <FilterSection title="Category">
        <CategoryCheckboxes 
          selected={filters.category} 
          onChange={(categories) => onFiltersChange({ ...filters, category: categories })}
        />
      </FilterSection>
      
      {/* Scale Filter */}
      <FilterSection title="Scale">
        <RangeSlider
          label="Daily Active Users"
          min={1000}
          max={3_000_000_000}
          value={[filters.scale?.minDAU || 1000, filters.scale?.maxDAU || 3_000_000_000]}
          onChange={(range) => onFiltersChange({
            ...filters,
            scale: { ...filters.scale, minDAU: range[0], maxDAU: range[1] }
          })}
        />
      </FilterSection>
      
      {/* Technology Filter */}
      <FilterSection title="Technologies">
        <TechnologyTags
          available={getAllTechnologies()}
          selected={filters.technologies}
          onChange={(techs) => onFiltersChange({ ...filters, technologies: techs })}
        />
      </FilterSection>
    </div>
  );
};
```

## Customization Features

### Parameter Customization
```typescript
interface CustomizationState {
  baseSystem: LibrarySystem;
  modifications: Partial<SystemDesignInput>;
  previewMode: boolean;
}

const useSystemCustomization = (baseSystem: LibrarySystem) => {
  const [state, setState] = useState<CustomizationState>({
    baseSystem,
    modifications: {},
    previewMode: false
  });
  
  const updateParameter = (key: keyof SystemDesignInput, value: any) => {
    setState(prev => ({
      ...prev,
      modifications: { ...prev.modifications, [key]: value }
    }));
  };
  
  const generateCustomized = async () => {
    const customInput = { ...extractInputFromSystem(baseSystem), ...state.modifications };
    return await generateSystemDesign(customInput);
  };
  
  return { state, updateParameter, generateCustomized };
};
```

### Customization UI
```typescript
const SystemCustomizer: React.FC<{
  system: LibrarySystem;
  onGenerate: (customDesign: SystemDesign) => void;
  onCancel: () => void;
}> = ({ system, onGenerate, onCancel }) => {
  const { state, updateParameter, generateCustomized } = useSystemCustomization(system);
  
  return (
    <div className="customizer-modal">
      <div className="customizer-header">
        <h2>Customize {system.name}</h2>
        <p>Adjust parameters to fit your specific requirements</p>
      </div>
      
      <div className="customizer-body">
        <div className="parameters-panel">
          <ParameterGroup title="Scale">
            <NumberInput
              label="Daily Active Users"
              value={state.modifications.dailyActiveUsers || system.scale.dailyActiveUsers}
              onChange={(value) => updateParameter('dailyActiveUsers', value)}
              min={1000}
              max={10_000_000_000}
            />
            <NumberInput
              label="Queries Per Second"
              value={state.modifications.queriesPerSecond || deriveQPS(system)}
              onChange={(value) => updateParameter('queriesPerSecond', value)}
            />
          </ParameterGroup>
          
          <ParameterGroup title="Performance">
            <SelectInput
              label="Latency Requirement"
              value={state.modifications.latencyRequirement || deriveLatency(system)}
              options={['real-time', 'near-real-time', 'batch']}
              onChange={(value) => updateParameter('latencyRequirement', value)}
            />
            <SelectInput
              label="Availability Target"
              value={state.modifications.availabilityTarget || deriveAvailability(system)}
              options={['99.9', '99.99', '99.999']}
              onChange={(value) => updateParameter('availabilityTarget', value)}
            />
          </ParameterGroup>
        </div>
        
        <div className="preview-panel">
          {state.previewMode && (
            <SystemPreview 
              baseSystem={system}
              modifications={state.modifications}
            />
          )}
        </div>
      </div>
      
      <div className="customizer-footer">
        <Button variant="secondary" onClick={onCancel}>Cancel</Button>
        <Button variant="primary" onClick={generateCustomized}>Generate Custom Design</Button>
      </div>
    </div>
  );
};
```

## Export & Sharing

### Export Functionality
```typescript
interface ExportOptions {
  format: 'pdf' | 'png' | 'svg' | 'json' | 'markdown';
  sections: SystemDesignSection[];
  includeDiagrams: boolean;
  includeMetadata: boolean;
}

const useSystemExport = () => {
  const exportSystem = async (system: LibrarySystem, options: ExportOptions) => {
    switch (options.format) {
      case 'pdf':
        return await exportToPDF(system, options);
      case 'png':
        return await exportToPNG(system, options);
      case 'json':
        return await exportToJSON(system, options);
      case 'markdown':
        return await exportToMarkdown(system, options);
      default:
        throw new Error(`Unsupported export format: ${options.format}`);
    }
  };
  
  return { exportSystem };
};

// Export implementations
const exportToPDF = async (system: LibrarySystem, options: ExportOptions) => {
  const doc = new jsPDF();
  
  // Add title page
  doc.setFontSize(24);
  doc.text(system.name, 20, 30);
  doc.setFontSize(12);
  doc.text(system.description, 20, 50);
  
  // Add system design sections
  let yPosition = 70;
  for (const section of options.sections) {
    yPosition = addSectionToPDF(doc, section, yPosition);
  }
  
  // Add diagrams if requested
  if (options.includeDiagrams) {
    await addDiagramsToPDF(doc, system);
  }
  
  return doc.output('blob');
};

const exportToMarkdown = async (system: LibrarySystem, options: ExportOptions) => {
  let markdown = `# ${system.name}\n\n`;
  markdown += `${system.description}\n\n`;
  
  // Add metadata if requested
  if (options.includeMetadata) {
    markdown += `## System Metadata\n\n`;
    markdown += `- **Category**: ${system.category}\n`;
    markdown += `- **Scale**: ${system.scale.dailyActiveUsers.toLocaleString()} DAU\n`;
    markdown += `- **Difficulty**: ${system.difficulty}\n\n`;
  }
  
  // Add each section
  for (const section of options.sections) {
    markdown += formatSectionAsMarkdown(section);
  }
  
  return new Blob([markdown], { type: 'text/markdown' });
};
```

### Sharing Features
```typescript
interface ShareOptions {
  includeCustomizations: boolean;
  allowEditing: boolean;
  expirationDate?: Date;
  accessLevel: 'public' | 'team' | 'private';
}

const useSystemSharing = () => {
  const shareSystem = async (system: LibrarySystem, options: ShareOptions) => {
    const shareData = {
      systemId: system.id,
      customizations: options.includeCustomizations ? system.customizations : null,
      permissions: {
        allowEditing: options.allowEditing,
        accessLevel: options.accessLevel
      },
      expiresAt: options.expirationDate,
      createdBy: getCurrentUser().id
    };
    
    const response = await fetch('/api/systems/share', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(shareData)
    });
    
    const { shareId, shareUrl } = await response.json();
    return { shareId, shareUrl };
  };
  
  return { shareSystem };
};
```

## Analytics & Insights

### Usage Analytics
```typescript
interface LibraryAnalytics {
  popularSystems: {
    systemId: string;
    name: string;
    views: number;
    likes: number;
    customizations: number;
  }[];
  categoryTrends: {
    category: SystemCategory;
    viewGrowth: number;
    popularityRank: number;
  }[];
  userEngagement: {
    averageSessionTime: number;
    systemsPerSession: number;
    customizationRate: number;
  };
}

const useLibraryAnalytics = () => {
  const [analytics, setAnalytics] = useState<LibraryAnalytics | null>(null);
  
  useEffect(() => {
    const fetchAnalytics = async () => {
      const response = await fetch('/api/library/analytics');
      const data = await response.json();
      setAnalytics(data);
    };
    
    fetchAnalytics();
  }, []);
  
  return analytics;
};
```

### Recommendation Engine
```typescript
interface RecommendationEngine {
  getSimilarSystems(systemId: string): Promise<LibrarySystem[]>;
  getPersonalizedRecommendations(userId: string): Promise<LibrarySystem[]>;
  getTrendingSystems(): Promise<LibrarySystem[]>;
}

const useSystemRecommendations = (currentSystem?: LibrarySystem) => {
  const [recommendations, setRecommendations] = useState<{
    similar: LibrarySystem[];
    personalized: LibrarySystem[];
    trending: LibrarySystem[];
  }>({ similar: [], personalized: [], trending: [] });
  
  useEffect(() => {
    const loadRecommendations = async () => {
      const [similar, personalized, trending] = await Promise.all([
        currentSystem ? getSimilarSystems(currentSystem.id) : Promise.resolve([]),
        getPersonalizedRecommendations(getCurrentUser().id),
        getTrendingSystems()
      ]);
      
      setRecommendations({ similar, personalized, trending });
    };
    
    loadRecommendations();
  }, [currentSystem]);
  
  return recommendations;
};
```

## API Endpoints

### Library Management API
```typescript
// GET /api/library/systems
interface GetSystemsRequest {
  category?: SystemCategory;
  difficulty?: string;
  search?: string;
  limit?: number;
  offset?: number;
  sortBy?: 'popularity' | 'name' | 'difficulty' | 'date';
}

// GET /api/library/systems/:id
interface GetSystemRequest {
  includeAnalytics?: boolean;
  includeRecommendations?: boolean;
}

// POST /api/library/systems/:id/interact
interface SystemInteractionRequest {
  action: 'view' | 'like' | 'bookmark' | 'customize';
  metadata?: Record<string, any>;
}

// POST /api/library/systems/:id/customize
interface CustomizeSystemRequest {
  modifications: Partial<SystemDesignInput>;
  saveAsNew?: boolean;
}
```

### Search API
```typescript
// POST /api/library/search
interface SearchRequest {
  query: string;
  filters: SearchFilters;
  sortBy: string;
  limit: number;
  offset: number;
}

interface SearchResponse {
  systems: LibrarySystem[];
  totalCount: number;
  facets: {
    categories: { name: string; count: number }[];
    difficulties: { name: string; count: number }[];
    technologies: { name: string; count: number }[];
  };
  suggestions: string[];
}
```

## Performance Optimization

### Caching Strategy
```typescript
// Client-side caching
const systemCache = new Map<string, LibrarySystem>();
const searchCache = new Map<string, SearchResponse>();

const useCachedSystems = () => {
  const getCachedSystem = (id: string): LibrarySystem | null => {
    return systemCache.get(id) || null;
  };
  
  const setCachedSystem = (system: LibrarySystem) => {
    systemCache.set(system.id, system);
    // Expire after 1 hour
    setTimeout(() => systemCache.delete(system.id), 60 * 60 * 1000);
  };
  
  return { getCachedSystem, setCachedSystem };
};

// Server-side caching with Redis
const cacheSystem = async (system: LibrarySystem) => {
  await redis.setex(`system:${system.id}`, 3600, JSON.stringify(system));
};

const getCachedSystem = async (id: string): Promise<LibrarySystem | null> => {
  const cached = await redis.get(`system:${id}`);
  return cached ? JSON.parse(cached) : null;
};
```

### Lazy Loading & Virtualization
```typescript
const VirtualizedSystemGrid: React.FC<{
  systems: LibrarySystem[];
  onSystemSelect: (system: LibrarySystem) => void;
}> = ({ systems, onSystemSelect }) => {
  const { height, width } = useWindowSize();
  
  const itemRenderer = ({ index, style }: any) => (
    <div style={style}>
      <SystemCard
        system={systems[index]}
        viewMode="grid"
        onSelect={onSystemSelect}
        onBookmark={() => {}}
        onLike={() => {}}
      />
    </div>
  );
  
  return (
    <FixedSizeGrid
      height={height - 200}
      width={width}
      columnCount={Math.floor(width / 300)}
      columnWidth={300}
      rowCount={Math.ceil(systems.length / Math.floor(width / 300))}
      rowHeight={400}
      itemData={systems}
    >
      {itemRenderer}
    </FixedSizeGrid>
  );
};
```

## Testing Strategy

### Unit Tests
```typescript
describe('LibraryBrowser', () => {
  test('filters systems by category', () => {
    const systems = [
      createMockSystem({ category: 'social-media' }),
      createMockSystem({ category: 'streaming' })
    ];
    
    const filtered = filterSystemsByCategory(systems, ['social-media']);
    expect(filtered).toHaveLength(1);
    expect(filtered[0].category).toBe('social-media');
  });
  
  test('searches systems by name and description', () => {
    const systems = [
      createMockSystem({ name: 'Facebook', description: 'Social network' }),
      createMockSystem({ name: 'Netflix', description: 'Video streaming' })
    ];
    
    const results = searchSystems(systems, 'social');
    expect(results).toHaveLength(1);
    expect(results[0].name).toBe('Facebook');
  });
});
```

### Integration Tests
```typescript
describe('System Library API', () => {
  test('GET /api/library/systems returns paginated results', async () => {
    const response = await request(app)
      .get('/api/library/systems?limit=10&offset=0')
      .expect(200);
    
    expect(response.body.systems).toHaveLength(10);
    expect(response.body.totalCount).toBeGreaterThan(10);
  });
  
  test('POST /api/library/search filters and sorts correctly', async () => {
    const searchRequest = {
      query: 'social',
      filters: { category: ['social-media'] },
      sortBy: 'popularity',
      limit: 5,
      offset: 0
    };
    
    const response = await request(app)
      .post('/api/library/search')
      .send(searchRequest)
      .expect(200);
    
    expect(response.body.systems.every(s => s.category === 'social-media')).toBe(true);
  });
});
```

## Success Metrics

### User Engagement
- Library page views per session > 3
- System detail page time spent > 2 minutes
- Customization rate > 15% of views
- Bookmark rate > 8% of views

### Content Performance
- Search result click-through rate > 25%
- Category distribution balance (no single category > 40%)
- User satisfaction with search results > 4.0/5
- Export completion rate > 80%

### Technical Performance
- Search response time < 200ms
- System detail load time < 1 second
- Mobile performance score > 85
- Cache hit rate > 70%
