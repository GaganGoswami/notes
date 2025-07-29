# Feature: Login

## User Story
As a user, I want to securely log in with my email/password or Google account so that I can access my financial data.

## Functional Requirements
- Support login via email/password using Supabase Auth.
- Support Google OAuth login via Supabase Auth.
- Redirect to `/dashboard` upon successful login.
- Display error messages for invalid credentials or network issues.

## UI Components
- **LoginForm**: Form with email and password fields, submit button.
- **GoogleButton**: Button for Google OAuth login.
- **ErrorMessage**: Component to display auth errors.

## API Endpoints
- None (handled client-side with Supabase Auth).

## Auth or Role-Based Behavior
- No role-based access; all users can attempt login.
- Redirect authenticated users to `/dashboard`.

## Input Validation and Edge Cases
- **Validation**:
  - Email: Valid format (e.g., `user@domain.com`).
  - Password: Minimum 8 characters.
- **Edge Cases**:
  - Invalid email/password: Display "Invalid credentials" error.
  - Network failure: Display "Unable to connect" error.
  - Google OAuth failure: Fallback to email login with error message.

## Expected Output Files or Folders
- `/src/app/login/page.tsx`: Login page component.
- `/src/components/LoginForm.tsx`: Reusable login form component.
- `/src/components/ErrorMessage.tsx`: Error display component.
- `/src/lib/supabase/auth.ts`: Supabase Auth utilities.
- `/src/hooks/useAuth.ts`: Custom hook for auth state.
- `/tests/LoginForm.test.tsx`: Jest tests for login form.
- `/tests/auth.test.ts`: Jest tests for auth utilities.

## Notes or Caveats
- Configure Google OAuth in Supabase dashboard.
- Store JWT in HTTP-only cookies for security.
- Use Supabase client for auth operations.
