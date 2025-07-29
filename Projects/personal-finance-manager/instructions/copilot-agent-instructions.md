# Copilot Agent Instructions

## Purpose
This file provides guidelines for GitHub Copilot Agent in VSCode Agent Mode to ensure consistent, secure, and high-quality code for the Personal Finance Manager application, a Mint-like platform for managing personal finances.

## Coding Guidelines
- **Language**: TypeScript for type safety.
- **Framework**: Next.js 14 (App Router) for frontend and backend.
- **Conventions**:
  - Follow Airbnb TypeScript style guide.
  - Use PascalCase for React components, camelCase for functions/variables.
  - File naming: `kebab-case` (e.g., `login-page.tsx`, `auth-service.ts`).
- **Comments**: Add JSDoc for all functions and inline comments for complex logic.
- **Error Handling**: Include try-catch for API calls, Supabase queries, and user input validation.
- **Security**:
  - Sanitize inputs to prevent injection attacks.
  - Use environment variables for Supabase and Plaid API keys.
  - Store JWT in HTTP-only cookies.
  - Enable Supabase Row-Level Security (RLS) for all database tables.

## Folder and File Structure
Follow the structure in `/instructions/recommended-folder-structure.md`. Key expectations:
- React components in `/src/components`.
- Next.js pages in `/src/app`.
- API routes in `/src/app/api`.
- Supabase utilities in `/src/lib/supabase`.
- Custom hooks in `/src/hooks`.
- Type definitions in `/src/types`.
- Plaid integration utilities in `/src/utils/plaid.ts`.

## Communication Style
- **Clarity**: If requirements are unclear, ask for clarification with a proposed solution.
- **Minimal Boilerplate**: Generate functional code without unnecessary abstractions.
- **Documentation**: Include a README section for each feature explaining its purpose and usage.
- **Supabase**: Use Supabase client for auth and database; follow official Supabase TypeScript docs.
- **Vercel**: Ensure code is compatible with Vercel’s serverless environment.

## Behavior Expectations
- **No Hallucination**: Generate code strictly based on `spec.md` and `feature-<name>.md` files.
- **Security First**: Prioritize secure coding practices (e.g., parameterized Supabase queries, RLS).
- **Testable Code**: Write modular, testable code with clear inputs/outputs.
- **Feature Isolation**: Implement one feature at a time, following the order in `/instructions`.
- **Error Handling**: Return appropriate HTTP status codes (e.g., 401 for auth errors, 500 for server errors).
- **Testing**: Generate Jest tests for each component and API route after implementation.

## Example Prompts and Outputs
### Prompt
"Generate a login page component with Supabase Auth integration, supporting email/password and Google OAuth."

### Expected Output
- File: `/src/app/login/page.tsx`
- Content: A React component using Supabase Auth client, with form validation, error handling, and Google OAuth.
- Additional Files: `/src/lib/supabase/auth.ts` for auth utilities.
- Comments: JSDoc for functions, inline comments for OAuth flow.

### Prompt
"Create an API route to fetch user transactions from Supabase, secured with JWT and RLS."

### Expected Output
- File: `/src/app/api/transactions/route.ts`
- Content: A Next.js API route that verifies JWT via Supabase Auth, queries the `transactions` table with RLS, and returns paginated results.
- Error Handling: Return 401 for invalid tokens, 500 for server errors.
- Tests: `/tests/transactions.test.ts` with Jest tests for success and error cases.

### Prompt
"Generate a budget card component to display budget details and progress."

### Expected Output
- File: `/src/components/BudgetCard.tsx`
- Content: A React component displaying budget name, amount, spent, and progress bar.
- Additional Files: `/src/types/budget.ts` for type definitions.
- Tests: `/tests/BudgetCard.test.tsx` with Jest tests for rendering and props.
