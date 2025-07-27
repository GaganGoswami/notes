# 🔥 Complete Project Generation Prompt - Vibe Scaffolder

## 📋 Project Overview

Generate a **complete One-Click App Scaffolding from UI + Shell Script Generator** application with the following exact specifications:

### 🎯 Core Purpose
Create a modern, visual tool for instantly scaffolding full-stack applications with customizable technology stacks and transparent shell script generation. The application should provide a step-by-step wizard interface for selecting technologies, validating compatibility, previewing project structure, and generating downloadable shell scripts with project files.

---

## 🏗️ Technical Architecture

### **Frontend Stack**
- **React 18** with hooks (useState, useEffect, useCallback, useMemo)
- **Standalone HTML version** with CDN dependencies (no build process required)
- **Vite development version** with modular components
- **Babel standalone** for JSX transformation in HTML version
- **JSZip 3.10.1** for client-side ZIP file generation
- **FileSaver.js 2.0.5** for browser-based file downloads

### **Styling System**
- **Complete CSS custom properties design system** with comprehensive theming
- **Dark/Light mode support** with automatic detection and manual toggle
- **CSS variables** for colors, typography, spacing, shadows, and animations
- **Responsive design** with mobile-first approach
- **Modern component styling** with focus states and accessibility

### **Build Tools**
- **Vite 5.4.2** as development server and build tool
- **@vitejs/plugin-react 4.3.1** for React support
- **Package.json** with development and production scripts

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

---

## 🧩 Component Architecture

### **1. Main Application Component**
```jsx
const VibeScaffolder = () => {
  const [currentStep, setCurrentStep] = useState(1);
  const [selectedTech, setSelectedTech] = useState({
    frontend: [], backend: [], database: [], tools: []
  });
  const [projectName, setProjectName] = useState('my-awesome-project');
  const [darkMode, setDarkMode] = useState(false);
  const [showTemplateModal, setShowTemplateModal] = useState(false);
  
  // Main render with step navigation
};
```

### **2. Step Components**

#### **Step 1: Project Configuration**
- Project name input with validation
- Description textarea
- Author information
- License selection (MIT, Apache, GPL, etc.)
- Project type selection (Web App, API, Desktop, Mobile)

#### **Step 2: Technology Selection**
- **Frontend Technologies**: React, Vue, Angular, Next.js, Vite
- **Backend Technologies**: Express.js, FastAPI, Spring Boot, Django, Node.js
- **Database Options**: PostgreSQL, MySQL, MongoDB, SQLite, Redis
- **Development Tools**: Docker, Git, ESLint, Prettier, Jest, TypeScript, Tailwind, Webpack

Each technology option should include:
```jsx
const TechOption = ({ tech, category, isSelected, onToggle, isIncompatible }) => (
  <div className={`tech-option ${isSelected ? 'selected' : ''} ${isIncompatible ? 'incompatible' : ''}`}
       onClick={() => !isIncompatible && onToggle(category, tech.id)}>
    <div className="checkbox">
      {isSelected && <span>✓</span>}
    </div>
    <div className="info">
      <div className="name">{tech.name}</div>
      <div className="description">{tech.description}</div>
    </div>
  </div>
);
```

#### **Step 3: Configuration Summary**
- Display selected technologies by category
- Show compatibility warnings/confirmations
- Allow final adjustments
- Preview project structure

#### **Step 4: Generation & Export**
- Live preview of generated shell script
- Interactive file tree visualization
- Download options (ZIP, Shell script, Individual files)
- Success confirmation with next steps

### **3. Advanced Components**

#### **Template Manager**
```jsx
const TemplateManager = () => {
  const [templates, setTemplates] = useState(TEMPLATES);
  const [activeCategory, setActiveCategory] = useState('fullstack');
  
  // Template categories: fullstack, frontend, backend, api, microservices
  // Save/load functionality with localStorage
  // Template sharing via JSON export/import
};
```

#### **File Tree Preview**
```jsx
const FileTreePreview = ({ structure }) => {
  // Recursive tree rendering with folders and files
  // Icons for different file types
  // Collapsible folders
  // Syntax highlighting for preview
};
```

#### **Script Generator**
```jsx
const ScriptGenerator = ({ config }) => {
  // Generate bash and PowerShell scripts
  // Include error handling and logging
  // Modular functions for each technology
  // Cross-platform compatibility
};
```

---

## 🎛️ Technology Stack Data Structure

```javascript
const TECH_STACK = {
  frontend: [
    { 
      id: 'react', 
      name: 'React', 
      description: 'Popular JavaScript library for building user interfaces',
      compatible: ['vite', 'webpack', 'typescript', 'tailwind', 'jest'],
      commands: ['npx create-react-app', 'npm install react react-dom'],
      configFiles: ['package.json', 'src/App.js', 'public/index.html']
    },
    // ... more frontend options
  ],
  backend: [
    {
      id: 'express',
      name: 'Express.js',
      description: 'Fast, unopinionated web framework for Node.js',
      compatible: ['nodejs', 'mongodb', 'postgresql', 'mysql'],
      commands: ['npm install express', 'npm install cors helmet'],
      configFiles: ['server.js', 'routes/', 'middleware/']
    },
    // ... more backend options
  ],
  database: [
    {
      id: 'postgresql',
      name: 'PostgreSQL',
      description: 'Powerful, open source object-relational database',
      compatible: ['express', 'fastapi', 'springboot', 'django'],
      commands: ['createdb myproject', 'npm install pg'],
      configFiles: ['database/schema.sql', 'config/database.js']
    },
    // ... more database options
  ],
  tools: [
    {
      id: 'docker',
      name: 'Docker',
      description: 'Containerization platform',
      compatible: ['*'],
      commands: ['docker --version', 'docker-compose --version'],
      configFiles: ['Dockerfile', 'docker-compose.yml', '.dockerignore']
    },
    // ... more tools
  ]
};
```

---

## 📋 Predefined Templates

### **Full-Stack Templates**
1. **MERN Stack**: MongoDB + Express + React + Node.js
2. **Next.js + PostgreSQL**: Modern full-stack with TypeScript
3. **Vue + FastAPI**: Vue.js frontend with Python backend
4. **Angular + Spring Boot**: Enterprise-grade application

### **Frontend Templates**
1. **React + Vite**: Fast React development setup
2. **Vue.js SPA**: Single-page application with Vue
3. **Next.js SSR**: Server-side rendered React app

### **Backend Templates**
1. **Express API**: RESTful API with Node.js
2. **FastAPI Microservice**: Python microservice architecture
3. **Spring Boot REST**: Java enterprise API

### **Specialized Templates**
1. **E-commerce Platform**: Full e-commerce with payment integration
2. **Blog/CMS**: Content management system
3. **Dashboard App**: Analytics and data visualization
4. **Real-time Chat**: WebSocket-based chat application

---

## 🔧 Shell Script Generation Engine

### **Script Structure**
```bash
#!/bin/bash
# Generated by Vibe Scaffolder
# Project: {projectName}
# Author: {author}
# Date: {currentDate}

set -e  # Exit on any error

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging functions
log_info() { echo -e "${BLUE}[INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[SUCCESS]${NC} $1"; }
log_warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# Main setup function
main() {
  log_info "Starting project setup..."
  create_project_structure
  install_dependencies
  setup_configuration
  initialize_git
  log_success "Project setup completed!"
}

# Individual setup functions for each technology
create_project_structure() { /* ... */ }
install_dependencies() { /* ... */ }
setup_configuration() { /* ... */ }
initialize_git() { /* ... */ }

# Execute main function
main "$@"
```

### **Technology-Specific Functions**
- **React Setup**: Create React app, install dependencies, configure Vite
- **Express Setup**: Initialize Node.js project, install Express, create server
- **Database Setup**: Install drivers, create connection files, run migrations
- **Docker Setup**: Generate Dockerfile and docker-compose.yml
- **Git Setup**: Initialize repository, create .gitignore, initial commit

---

## 📁 Project File Structure Generator

```javascript
const generateProjectStructure = (config) => {
  const structure = {
    [`${config.projectName}/`]: {
      'README.md': generateReadme(config),
      '.gitignore': generateGitignore(config),
      'package.json': generatePackageJson(config),
    }
  };

  // Add frontend structure
  if (config.frontend.includes('react')) {
    structure[`${config.projectName}/`]['src/'] = {
      'App.jsx': generateReactApp(),
      'main.jsx': generateReactMain(),
      'index.css': generateBasicStyles(),
      'components/': {},
      'hooks/': {},
      'utils/': {}
    };
    structure[`${config.projectName}/`]['public/'] = {
      'index.html': generateIndexHtml(),
      'vite.svg': generateViteLogo()
    };
  }

  // Add backend structure
  if (config.backend.includes('express')) {
    structure[`${config.projectName}/`]['server/'] = {
      'server.js': generateExpressServer(),
      'routes/': { 'api.js': generateApiRoutes() },
      'middleware/': { 'auth.js': generateAuthMiddleware() },
      'models/': {},
      'controllers/': {}
    };
  }

  // Add database structure
  if (config.database.length > 0) {
    structure[`${config.projectName}/`]['database/'] = {
      'schema.sql': generateDatabaseSchema(config),
      'seeds/': {},
      'migrations/': {}
    };
  }

  // Add Docker files
  if (config.tools.includes('docker')) {
    structure[`${config.projectName}/`]['Dockerfile'] = generateDockerfile(config);
    structure[`${config.projectName}/`]['docker-compose.yml'] = generateDockerCompose(config);
    structure[`${config.projectName}/`]['.dockerignore'] = generateDockerignore();
  }

  return structure;
};
```

---

## 🎨 UI/UX Specifications

### **Step Indicator Design**
```css
.step-indicator {
  width: 2.5rem; height: 2.5rem; border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  font-weight: 500; font-size: 0.875rem;
  background-color: var(--color-secondary);
  color: var(--color-text-secondary);
  transition: all var(--duration-normal) var(--ease-standard);
}

.step-indicator.active {
  background-color: var(--color-primary);
  color: var(--color-btn-primary-text);
}

.step-indicator.completed {
  background-color: var(--color-success);
  color: var(--color-btn-primary-text);
}
```

### **Technology Selection Cards**
```css
.tech-option {
  display: flex; align-items: center; padding: var(--space-16);
  border: 2px solid var(--color-border);
  border-radius: var(--radius-lg);
  background-color: var(--color-surface);
  cursor: pointer;
  transition: all var(--duration-normal) var(--ease-standard);
}

.tech-option:hover {
  border-color: var(--color-primary);
  box-shadow: var(--shadow-md);
}

.tech-option.selected {
  border-color: var(--color-primary);
  background-color: rgba(var(--color-teal-500-rgb), 0.05);
}

.tech-option.incompatible {
  opacity: 0.5; cursor: not-allowed;
  border-color: var(--color-error);
}
```

### **Interactive Elements**
- **Hover animations** with scale and shadow effects
- **Selection feedback** with color changes and checkmarks
- **Loading states** with spinners and progress indicators
- **Success/error notifications** with color-coded messages
- **Modal dialogs** for template management and confirmations

---

## 📦 File Generation and Export

### **ZIP File Creation**
```javascript
const exportProject = async (config, structure, script) => {
  const zip = new JSZip();
  
  // Add project files
  Object.entries(structure).forEach(([path, content]) => {
    if (typeof content === 'string') {
      zip.file(path, content);
    } else {
      zip.folder(path);
    }
  });
  
  // Add setup script
  zip.file('setup.sh', script.bash);
  zip.file('setup.bat', script.powershell);
  
  // Generate and download
  const blob = await zip.generateAsync({ type: 'blob' });
  saveAs(blob, `${config.projectName}-scaffold.zip`);
};
```

### **Individual File Downloads**
- **Shell script** (.sh for Unix/Linux/macOS)
- **PowerShell script** (.bat for Windows)
- **Package.json** with all dependencies
- **README.md** with setup instructions
- **Docker files** if selected

---

## 🔒 Compatibility Matrix

### **Frontend Compatibility**
- **React**: Compatible with Vite, Webpack, TypeScript, Tailwind, Jest
- **Vue.js**: Compatible with Vite, Webpack, TypeScript, Tailwind
- **Angular**: Requires TypeScript, compatible with Webpack
- **Next.js**: Requires React, compatible with TypeScript, Tailwind

### **Backend Compatibility**
- **Express.js**: Compatible with Node.js, all databases
- **FastAPI**: Requires Python, compatible with PostgreSQL, MySQL
- **Spring Boot**: Requires Java, compatible with PostgreSQL, MySQL
- **Django**: Requires Python, compatible with PostgreSQL, MySQL

### **Database Compatibility**
- **PostgreSQL**: Compatible with all backends
- **MySQL**: Compatible with all backends
- **MongoDB**: Best with Express.js and Node.js
- **SQLite**: Good for development, compatible with most backends
- **Redis**: Primarily for caching, compatible with all backends

---

## 🎯 Key Features Implementation

### **1. Real-time Compatibility Checking**
```javascript
const checkCompatibility = (selectedTech) => {
  const conflicts = [];
  const warnings = [];
  
  // Check frontend-backend compatibility
  selectedTech.frontend.forEach(frontendId => {
    selectedTech.backend.forEach(backendId => {
      const frontend = TECH_STACK.frontend.find(t => t.id === frontendId);
      const backend = TECH_STACK.backend.find(t => t.id === backendId);
      
      if (!frontend.compatible.includes(backendId) && 
          !backend.compatible.includes(frontendId)) {
        conflicts.push(`${frontend.name} may have compatibility issues with ${backend.name}`);
      }
    });
  });
  
  return { conflicts, warnings };
};
```

### **2. Live Script Preview**
```javascript
const useLiveScriptGeneration = (config) => {
  const [script, setScript] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);
  
  useEffect(() => {
    const generateScript = async () => {
      setIsGenerating(true);
      const generatedScript = await generateSetupScript(config);
      setScript(generatedScript);
      setIsGenerating(false);
    };
    
    if (Object.values(config).some(arr => arr.length > 0)) {
      generateScript();
    }
  }, [config]);
  
  return { script, isGenerating };
};
```

### **3. Template System**
```javascript
const useTemplates = () => {
  const [templates, setTemplates] = useState(() => {
    const saved = localStorage.getItem('vibe-scaffolder-templates');
    return saved ? JSON.parse(saved) : TEMPLATES;
  });
  
  const saveTemplate = (name, config) => {
    const newTemplate = {
      id: `custom-${Date.now()}`,
      name,
      description: 'Custom template',
      tags: [],
      config,
      isCustom: true
    };
    
    const updated = { ...templates, custom: [...(templates.custom || []), newTemplate] };
    setTemplates(updated);
    localStorage.setItem('vibe-scaffolder-templates', JSON.stringify(updated));
  };
  
  return { templates, saveTemplate };
};
```

---

## 📱 Responsive Design Requirements

### **Mobile (320px - 768px)**
- Single-column layout for technology selection
- Stacked step indicators
- Touch-friendly button sizes (min 44px)
- Collapsible sections for better navigation
- Simplified file tree preview

### **Tablet (768px - 1024px)**
- Two-column layout for technology categories
- Horizontal step indicators
- Medium-sized cards and buttons
- Side-by-side preview and configuration

### **Desktop (1024px+)**
- Multi-column layout for optimal space usage
- Full-featured interface with all options visible
- Large preview areas
- Advanced filtering and search capabilities

---

## 🎨 Animation and Transitions

### **Step Transitions**
```css
.wizard-step {
  display: none;
}

.wizard-step.active {
  display: block;
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}
```

### **Selection Animations**
```css
.tech-option.selected {
  animation: selectPulse 0.3s ease-out;
}

@keyframes selectPulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.02); }
  100% { transform: scale(1); }
}
```

### **Loading States**
```css
.loading::after {
  content: '';
  width: 1rem; height: 1rem;
  border: 2px solid var(--color-border);
  border-top-color: var(--color-primary);
  border-radius: 50%;
  margin-left: var(--space-8);
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

---

## 🔧 Development Setup Files

### **package.json**
```json
{
  "name": "vibe-scaffolder",
  "version": "1.0.0",
  "description": "One-Click App Scaffolding from UI + Shell Script Generator",
  "type": "module",
  "main": "index.html",
  "scripts": {
    "dev": "vite --host",
    "build": "vite build",
    "preview": "vite preview --host",
    "serve-static": "npx serve .",
    "dev-static": "npx live-server .",
    "start": "npm run serve-static",
    "clean": "rm -rf node_modules package-lock.json && npm install"
  },
  "keywords": [
    "scaffolding", "project-generator", "shell-script", "react",
    "full-stack", "developer-tools", "vite", "typescript", "tailwind"
  ],
  "author": "Vibe Scaffolder",
  "license": "MIT",
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.2"
  },
  "dependencies": {
    "file-saver": "^2.0.5",
    "jszip": "^3.10.1",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  }
}
```

### **vite.config.js**
```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
    port: 3000,
    open: true
  },
  build: {
    outDir: 'dist',
    sourcemap: true
  }
});
```

### **.gitignore**
```
node_modules/
dist/
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
*.log
coverage/
```

---

## 📖 Documentation Requirements

### **README.md Structure**
1. **Project overview** with badges and screenshots
2. **Features list** with emojis and descriptions
3. **Quick start guide** with installation steps
4. **Usage examples** with code snippets
5. **Technology stack** documentation
6. **Contributing guidelines**
7. **License information**
8. **Changelog** and roadmap

### **Inline Code Documentation**
- **JSDoc comments** for all functions and components
- **TypeScript types** for better developer experience
- **Component prop documentation**
- **Usage examples** in component files

---

## 🎯 Accessibility Requirements

### **Keyboard Navigation**
- **Tab order** for all interactive elements
- **Enter/Space** activation for custom components
- **Arrow keys** for step navigation
- **Escape** to close modals

### **Screen Reader Support**
- **ARIA labels** for all interactive elements
- **Role attributes** for custom components
- **Live regions** for dynamic content updates
- **Skip links** for main content areas

### **Visual Accessibility**
- **High contrast** color combinations
- **Focus indicators** that meet WCAG standards
- **Text alternatives** for icons and images
- **Responsive text sizing**

---

## 🎪 Demo Script

Create a `demo.sh` file that demonstrates the application:

```bash
#!/bin/bash
# Vibe Scaffolder Demo Script

echo "🔥 Welcome to Vibe Scaffolder Demo!"
echo "=================================="
echo ""

# Check if browser is available
if command -v open >/dev/null 2>&1; then
    BROWSER_CMD="open"
elif command -v xdg-open >/dev/null 2>&1; then
    BROWSER_CMD="xdg-open"
elif command -v start >/dev/null 2>&1; then
    BROWSER_CMD="start"
else
    echo "❌ No browser command found"
    exit 1
fi

echo "🚀 Starting demo..."
echo "📂 Opening index.html in your default browser..."
$BROWSER_CMD index.html

echo ""
echo "✨ Demo Features:"
echo "   • Visual technology stack selection"
echo "   • Real-time compatibility checking"  
echo "   • Interactive project structure preview"
echo "   • Shell script generation and download"
echo "   • Template system with predefined stacks"
echo "   • Dark/Light theme support"
echo ""
echo "🎯 Try creating a project with your favorite tech stack!"
echo ""
```

---

## 🔮 Future Enhancements

### **Phase 2 Features**
1. **AI-powered script optimization** using LLM integration
2. **GitHub integration** for automatic repository creation
3. **VS Code extension** for in-editor scaffolding
4. **Cloud template sharing** with community repository
5. **Team collaboration** features with shared templates

### **Phase 3 Features**
1. **Multi-language support** (Python, Java, Go, Rust)
2. **Microservices architecture** templates
3. **Cloud deployment** integration (AWS, Azure, GCP)
4. **CI/CD pipeline** generation
5. **Testing framework** integration

---

## ✅ Final Implementation Checklist

### **Core Functionality**
- [ ] Multi-step wizard with navigation
- [ ] Technology selection with compatibility checking
- [ ] Real-time script generation
- [ ] Project structure preview
- [ ] File and ZIP export functionality
- [ ] Template system with save/load
- [ ] Dark/Light theme toggle

### **UI/UX Polish**
- [ ] Responsive design for all screen sizes
- [ ] Smooth animations and transitions
- [ ] Loading states and progress indicators
- [ ] Error handling with user-friendly messages
- [ ] Accessibility compliance (WCAG 2.1)

### **Technical Requirements**
- [ ] Standalone HTML version (no build required)
- [ ] Vite development version with hot reload
- [ ] Cross-platform shell script generation
- [ ] Local storage for templates and preferences
- [ ] Clean, commented, and maintainable code

### **Documentation**
- [ ] Comprehensive README with examples
- [ ] Inline code documentation
- [ ] Setup and deployment instructions
- [ ] Contributing guidelines
- [ ] Demo script and examples

---

## 🎊 Success Criteria

The project should be considered complete when:

1. **Users can create a full project setup** in under 5 minutes
2. **Generated scripts work correctly** on Windows, macOS, and Linux
3. **All combinations of compatible technologies** work together
4. **The application is fully responsive** and accessible
5. **No build process required** for the standalone version
6. **Template system enables rapid project creation**
7. **Code is clean, well-documented, and maintainable**

---

This prompt contains everything needed to recreate the exact Vibe Scaffolder application with all its features, styling, and functionality. The implementation should follow modern React best practices, include comprehensive error handling, and provide an excellent user experience across all devices and accessibility needs.
