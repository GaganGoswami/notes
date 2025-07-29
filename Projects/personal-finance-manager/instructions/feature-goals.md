# Feature: Goal Setting

## User Story
As a user, I want to set and track savings or debt repayment goals so that I can achieve my financial objectives.

## Functional Requirements
- Allow users to create goals with target amount, deadline, and type (savings or debt).
- Display goal progress based on linked account balances or manual updates.
- Allow editing or deleting goals.
- Notify users of progress milestones (e.g., 50% complete).

## UI Components
- **GoalTracker**: Component displaying goal name, target, progress, and deadline.
- **GoalForm**: Form to create/edit goals (name, amount, deadline, type).
- **ErrorMessage**: Component for goal-related errors.

## API Endpoints
- `GET /api/goals`: Fetch user’s goals (secured with JWT, RLS).
- `POST /api/goals`: Create a new goal.
- `PUT /api/goals/[id]`: Update a goal.
- `DELETE /api/goals/[id]`: Delete a goal.

## Auth or Role-Based Behavior
- Must be authenticated; redirect unauthenticated users to `/login`.
- Data access restricted to user’s `user_id` via Supabase RLS.

## Input Validation and Edge Cases
- **Validation**:
  - Name: Non-empty string.
  - Amount: Positive number.
  - Deadline: Valid future date.
  - Type: Either "savings" or "debt".
- **Edge Cases**:
  - No linked account: Prompt user to connect an account or allow manual updates.
  - Past deadline: Mark goal as expired.
  - Network failure: Display "Unable to load goals" error.

## Expected Output Files or Folders
- `/src/app/goals/page.tsx`: Goals page component.
- `/src/components/GoalTracker.tsx`: Reusable goal tracker component.
- `/src/components/GoalForm.tsx`: Goal creation/edit form.
- `/src/lib/supabase/goals.ts`: Supabase queries for goals.
- `/src/hooks/useGoals.ts`: Hook for fetching goals.
- `/src/app/api/goals/route.ts`: API route for goal management.
- `/src/types/goal.ts`: Type definitions for goals.
- `/tests/GoalTracker.test.tsx`: Jest tests for goal tracker.
- `/tests/goals.test.ts`: Jest tests for goal queries and API.

## Notes or Caveats
- Link goals to accounts for automatic progress tracking via Plaid.
- Use Supabase RLS for data isolation.
- Consider Supabase subscriptions for real-time milestone notifications.
