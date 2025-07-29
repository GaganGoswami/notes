# Feature: Signup

## User Story
As a new user, I want to create an account with my email/password or Google account so that I can start managing my finances.

## Functional Requirements
- Support signup via email/password using Supabase Auth.
- Support Google OAuth signup via Supabase Auth.
- Validate email uniqueness and password strength.
- Redirect to `/dashboard` or `/accounts` setup upon successful signup.
- Send email verification if required by Supabase.

## UI Components
- **SignupForm**: Form with email, password, and confirm password fields, submit button.
- **GoogleButton**: Button for Google OAuth signup.
- **ErrorMessage**: Component to display signup errors.

## API Endpoints
- None (handled client-side with Supabase Auth).

## Auth or Role-Based Behavior
- No role-based access; all users can attempt signup.
- Redirect authenticated users to `/dashboard` or `/accounts`.

## Input Validation and Edge Cases
- **Validation**:
  - Email: Valid format, unique in Supabase `auth.users`.
  - Password: Minimum 8 characters, at least one number and one special character.
  - Confirm Password: Must match password.
- **Edge Cases**:
  - Duplicate email: Display "Email already in use" error.
  - Weak password: Display "Password too weak" error.
  - Network failure: Display "Unable to connect" error.
  - Google OAuth failure: Fallback to email signup with error message.

## Expected Output Files or Folders
- `/src/app/signup/page.tsx`: Signup page component.
- `/src/components/SignupForm.tsx`: Reusable signup form component.
- `/src/components/ErrorMessage.tsx`: Error display component.
- `/src/lib/supabase/auth.ts`: Supabase Auth utilities (shared with login).
- `/src/hooks/useAuth.ts`: Custom hook for auth state (shared with login).
- `/tests/SignupForm.test.tsx`: Jest tests for signup form.
- `/tests/auth.test.ts`: Jest tests for auth utilities (shared with login).

## Notes or Caveats
- Enable email verification in Supabase if required.
- Ensure Supabase Auth is configured for Google OAuth.
- Store JWT in HTTP-only cookies for security.
