I want to build a modern presentation-style web app using **Next.js**, with the following features:

## Core Features

- Render a **full-page scrollable presentation** where each section behaves like a **slide**.
- Presentation **content is sourced from a Markdown file** (e.g., `content.md` or MDX).
- Each slide should support Markdown formatting: headings, lists, images, code blocks, etc.
- Slides must support **scroll snapping** with smooth transitions (vertical).
- A **footer section must be visible on every slide**, similar to the design at https://www.ryder-and-davis.com.
- Implement a **responsive, minimalist layout** that works well across devices.
- Provide **keyboard navigation (←/→ or ↑/↓)** and **mobile gesture support**.

## Technical Requirements

- Use **Next.js (latest version)** with **Static Site Generation (SSG)** and optional **Server-Side Rendering (SSR)** for dynamic content.
- Use **Tailwind CSS** for styling.
- Parse Markdown using a library like `remark`, `gray-matter`, or `next-mdx-remote`.
- Optional: allow **frontmatter metadata per slide** (e.g., title, bg color, animations).
- Light/Dark mode toggle with persisted preference (localStorage or cookie).

## PDF Export

- Add ability to **export the full presentation to a styled PDF**.
- Use a tool like `react-to-print`, `html2pdf`, or `puppeteer` to generate a printable version.
- Ensure **slide boundaries, layout, and footers are preserved** in the PDF.
- Add a “Download as PDF” button.

## Bonus Features (Optional)

- Support **custom Markdown extensions** for video embedding, callouts, etc.
- Add **animated transitions** between slides.
- Slide progress indicator or mini nav menu.
- Deployable via Vercel or static export with `next export`.

---

Generate:
1. Initial project scaffolding (Next.js + Tailwind)
2. Markdown loader and slide splitter
3. Layout for scroll-snapped slides
4. Reusable footer component
5. PDF export feature
6. Sample `content.md` file

Ensure code is clean, modular, and extendable.
