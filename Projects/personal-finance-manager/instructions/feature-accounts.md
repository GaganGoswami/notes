# Feature: Account Integration

## User Story
As a user, I want to connect my bank, credit card, and investment accounts so that I can track their balances and transactions.

## Functional Requirements
- Integrate with Plaid API to connect financial accounts.
- Display a list of connected accounts with balances.
- Allow users to add or remove accounts.
- Store account metadata in Supabase (e.g., account ID, institution name).

## UI Components
- **AccountList**: List of connected accounts with balances and institution names.
- **AddAccountButton**: Button to initiate Plaid Link flow.
- **RemoveAccountButton**: Button to remove an account.
- **ErrorMessage**: Component for Plaid or Supabase errors.

## API Endpoints
- `POST /api/accounts`: Exchange Plaid public token for access token, store account in Supabase.
- `DELETE /api/accounts/[id]`: Remove an account from Supabase.
- `GET /api/accounts`: Fetch user’s connected accounts (secured with JWT, RLS).

## Auth or Role-Based Behavior
- Must be authenticated; redirect unauthenticated users to `/login`.
- Data access restricted to user’s `user_id` via Supabase RLS.

## Input Validation and Edge Cases
- **Validation**:
  - Plaid public token: Valid token from Plaid Link.
  - Account ID: Valid UUID for deletion.
- **Edge Cases**:
  - Plaid Link failure: Display "Unable to connect account" error.
  - Invalid access token: Log error, prompt user to reconnect.
  - Network failure: Display "Unable to load accounts" error.

## Expected Output Files or Folders
- `/src/app/accounts/page.tsx`: Accounts page component.
- `/src/components/AccountList.tsx`: Reusable account list component.
- `/src/utils/plaid.ts`: Plaid API utilities.
- `/src/lib/supabase/accounts.ts`: Supabase queries for accounts.
- `/src/hooks/useAccounts.ts`: Hook for fetching accounts.
- `/src/app/api/accounts/route.ts`: API route for account management.
- `/src/types/account.ts`: Type definitions for accounts.
- `/tests/AccountList.test.tsx`: Jest tests for account list.
- `/tests/accounts.test.ts`: Jest tests for account queries and API.

## Notes or Caveats
- Configure Plaid API keys in `.env` and Vercel.
- Store Plaid access tokens securely in Supabase with RLS.
- Use Plaid Link SDK for client-side account connection.
