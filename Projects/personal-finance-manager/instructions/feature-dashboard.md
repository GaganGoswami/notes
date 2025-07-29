# Feature: Dashboard

## User Story
As a user, I want a unified view of my accounts, budgets, and goals so that I can quickly assess my financial status.

## Functional Requirements
- Display a summary of connected accounts (bank, credit card, investment).
- Show recent transactions (last 5).
- Display active budgets with progress bars.
- Show savings or debt repayment goals with progress.
- Allow navigation to detailed accounts, transactions, budgets, and goals pages.

## UI Components
- **Dashboard**: Main component with account summary, transactions, budgets, and goals.
- **AccountSummary**: List of connected accounts with balances.
- **TransactionList**: Table of recent transactions.
- **BudgetCard**: Card for each budget with progress bar.
- **GoalTracker**: Progress bar for each goal.
- **ErrorMessage**: Component for data fetch errors.

## API Endpoints
- `GET /api/accounts`: Fetch user’s connected accounts (secured with JWT, RLS).
- `GET /api/transactions?limit=5`: Fetch recent transactions (secured with JWT, RLS).
- `GET /api/budgets`: Fetch active budgets (secured with JWT, RLS).
- `GET /api/goals`: Fetch active goals (secured with JWT, RLS).

## Auth or Role-Based Behavior
- Must be authenticated; redirect unauthenticated users to `/login`.
- Data access restricted to user’s `user_id` via Supabase RLS.

## Input Validation and Edge Cases
- **Validation**: None (data fetched server-side).
- **Edge Cases**:
  - No accounts connected: Display "Connect an account" prompt.
  - No transactions: Display "No recent transactions".
  - No budgets/goals: Display "Create a budget/goal" prompt.
  - Network failure: Display "Unable to load data" error.

## Expected Output Files or Folders
- `/src/app/dashboard/page.tsx`: Dashboard page component.
- `/src/components/Dashboard.tsx`: Main dashboard component.
- `/-LazyLoad-TransactionList.tsx`: Reusable transaction list component (lazy load).
- `/src/components/BudgetCard.tsx`: Reusable budget card component.
- `/src/components/GoalTracker.tsx`: Reusable goal tracker component.
- `/src/components/ErrorMessage.tsx`: Error display component.
- `/src/lib/supabase/accounts.ts`: Supabase queries for accounts.
- `/src/lib/supabase/transactions.ts`: Supabase queries for transactions.
- `/src/lib/supabase/budgets.ts`: Supabase queries for budgets.
- `/src/lib/supabase/goals.ts`: Supabase queries for goals.
- `/src/hooks/useAccounts.ts`: Hook for fetching accounts.
- `/src/hooks/useTransactions.ts`: Hook for fetching transactions.
- `/src/hooks/useBudgets.ts`: Hook for fetching budgets.
- `/src/hooks/useGoals.ts`: Hook for fetching goals.
- `/tests/Dashboard.test.tsx`: Jest tests for dashboard.
- `/tests/accounts.test.ts`: Jest tests for account queries.
- `/tests/transactions.test.ts`: Jest tests for transaction queries.
- `/tests/budgets.test.ts`: Jest tests for budget queries.
- `/tests/goals.test.ts`: Jest tests for goal queries.

## Notes or Caveats
- Use Supabase RLS to ensure data isolation per user.
- Optimize data fetching with pagination or limits for performance.
- Deploy API routes to Vercel for serverless execution.
