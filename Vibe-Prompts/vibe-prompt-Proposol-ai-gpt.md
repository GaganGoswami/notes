You are an expert full-stack engineer specializing in AI-native productivity platforms and beautiful UI/UX ("vibe coding").  
Create a **complete, client-only TypeScript React application** (Vite or Next.js) called **"ProposalAI"** for pre-sales and proposal management workflows.

---

## 🔧 Project Scope
Build a modular, scalable, and fully functional **frontend-only** application with:
- **React + TypeScript**
- **Tailwind CSS** for styling
- **Mock/local JSON and interfaces** for all data (no backend)
- **Clean folder architecture**, **reusable components**, **animated transitions**, and **accessible UI**

---

## ✨ Features & Requirements

### 📊 1. Dashboard
- Stats cards (e.g., total proposals, win rate)
- Monthly activity chart (e.g., bar or line)
- Recent Proposals Table with columns: Title, Client, Date, Status

### 📁 2. PDF Upload + Parsing Simulator
- Drag-and-drop file upload (PDF/DOCX)
- Simulated parsing with progress bar
- Fake extracted sections: Introduction, Scope, Deliverables, etc.
- Use realistic sample RFP content in TypeScript files

### 🧠 3. Semantic Generation
- Prompt Config Form (tone, target persona, win themes)
- Display side-by-side layout:
  - Left: Parsed RFP sections
  - Right: Simulated SOW + Executive Summary generated from sample prompts

### 📚 4. Proposal Library
- Grid/List view with mock proposals
- Filters (client, status, tags)
- Simulated similarity score bar/visual
- Semantic search (simulated with fuzzy text match)

### ✍️ 5. Rich Editor
- Rich text editor (TipTap or Quill)
- Formatting toolbar (bold, italic, lists, headings)
- Version history sidebar (mock timeline)
- Export to PDF/DOCX (use print-to-pdf browser trick or JS library)

### ⚙️ 6. Settings Page
- Profile settings: name, role, email
- Prompt templates manager (CRUD in local state)
- Theme toggle (dark/light using Tailwind and context)
- Team roles simulator: Admin/Manager/User → affects visible UI options only

### 🌐 7. Global UI
- Responsive layout
- NavBar with logo, nav links, user avatar
- Sidebar navigation for desktop
- Modal dialogs (upload, confirm actions)
- Toast notifications for success/errors
- Smooth page transitions (Framer Motion or Tailwind animations)
- Keyboard nav and ARIA compliance

---

## 🗂️ Folder Structure

/src
/assets
/components
/data
/hooks
/pages
/types
/utils
/contexts
/layouts
/constants
App.tsx
main.tsx

---

## 🧪 Data Requirements

Use **local TS mock data** and define **clean interfaces**:

- `Proposal`: id, title, client, sections, createdAt, status
- `User`: id, name, email, role
- `Template`: id, name, description, defaultPrompt
- `ParsedRFPSection`: id, name, content

---

## 📄 README Instructions

Include `README.md` with:
- App overview
- Installation: `npm install && npm run dev`
- How to switch mock data or simulate new proposals
- Technologies used
- App screenshot(s)

---

## 🧠 UX/Design Guidelines

- Modern, clean aesthetic: Tailwind gray/blue/green palette
- Use readable sans-serif fonts (e.g., Inter, DM Sans)
- Intuitive flows: Upload → Review → Generate → Edit → Export
- CTA buttons should be consistent and bold
- No lorem ipsum — use domain-accurate mock data

---

## 🧼 Code Quality

- Strong typing with `interface` and `type`
- Clear component naming and separation of concerns
- Comments for logic-heavy areas
- Reusable layout and UI primitives
- Accessibility features: ARIA, tabIndex, keyboard-friendly

---

## ✅ Output Expectation

> Generate full working code for all pages/components, including:
- Dashboard
- File Upload
- Parsing simulator
- AI Generation view
- Proposal Library
- Rich Text Editor
- Settings
- Nav/Sidebar/Toasts/Modals
- TypeScript types and mock data

> Final output must be:
- **Client-only**
- **Fully functional in-browser**
- **Zero server code**
- **Easy to extend for real backend integration later**

---
