# Feature: Budget Planning

## User Story
As a user, I want to create and track budgets by category so that I can control my spending.

## Functional Requirements
- Allow users to create budgets with category, amount, and time period (e.g., monthly).
- Display budget progress based on transaction data.
- Alert users when budgets are exceeded.
- Allow editing or deleting budgets.

## UI Components
- **BudgetCard**: Card displaying budget name, amount, spent, and progress bar.
- **BudgetForm**: Form to create/edit budgets (category, amount, period).
- **ErrorMessage**: Component for budget-related errors.

## API Endpoints
- `GET /api/budgets`: Fetch user’s budgets (secured with JWT, RLS).
- `POST /api/budgets`: Create a new budget.
- `PUT /api/budgets/[id]`: Update a budget.
- `DELETE /api/budgets/[id]`: Delete a budget.

## Auth or Role-Based Behavior
- Must be authenticated; redirect unauthenticated users to `/login`.
- Data access restricted to user’s `user_id` via Supabase RLS.

## Input Validation and Edge Cases
- **Validation**:
  - Category: Must be a valid category.
  - Amount: Positive number.
  - Period: Valid time period (e.g., monthly, yearly).
- **Edge Cases**:
  - Duplicate budget for category/period: Display "Budget already exists" error.
  - No transactions for category: Show 0% progress.
  - Network failure: Display "Unable to load budgets" error.

## Expected Output Files or Folders
- `/src/app/budgets/page.tsx`: Budgets page component.
- `/src/components/BudgetCard.tsx`: Reusable budget card component.
- `/src/components/BudgetForm.tsx`: Budget creation/edit form.
- `/src/lib/supabase/budgets.ts`: Supabase queries for budgets.
- `/src/hooks/useBudgets.ts`: Hook for fetching budgets.
- `/src/app/api/budgets/route.ts`: API route for budget management.
- `/src/types/budget.ts`: Type definitions for budgets.
- `/tests/BudgetCard.test.tsx`: Jest tests for budget card.
- `/tests/budgets.test.ts`: Jest tests for budget queries and API.

## Notes or Caveats
- Calculate spent amount by aggregating transactions per category.
- Use Supabase RLS for data isolation.
- Consider real-time updates with Supabase subscriptions for budget alerts.
