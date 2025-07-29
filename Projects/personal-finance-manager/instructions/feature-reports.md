# Feature: Reports

## User Story
As a user, I want to generate reports on my spending, net worth, and investment performance so that I can make informed financial decisions.

## Functional Requirements
- Generate spending reports by category over a selected time period.
- Calculate net worth based on account balances and liabilities.
- Display investment performance for connected investment accounts.
- Allow export of reports as CSV.

## UI Components
- **ReportChart**: Chart component for visualizing spending or investment data.
- **ReportFilter**: Dropdowns for selecting report type and time period.
- **ExportButton**: Button to download report as CSV.
- **ErrorMessage**: Component for report generation errors.

## API Endpoints
- `GET /api/reports/spending`: Fetch spending data by category (secured with JWT, RLS).
- `GET /api/reports/net-worth`: Calculate net worth from accounts (secured with JWT, RLS).
- `GET /api/reports/investments`: Fetch investment performance data (secured with JWT, RLS).

## Auth or Role-Based Behavior
- Must be authenticated; redirect unauthenticated users to `/login`.
- Data access restricted to user’s `user_id` via Supabase RLS.

## Input Validation and Edge Cases
- **Validation**:
  - Report type: Valid type (spending, net-worth, investments).
  - Time period: Valid range (e.g., last 30 days, last year).
- **Edge Cases**:
  - No data for period: Display "No data available".
  - No investment accounts: Hide investment report option.
  - Network failure: Display "Unable to generate report" error.

## Expected Output Files or Folders
- `/src/app/reports/page.tsx`: Reports page component.
- `/src/components/ReportChart.tsx`: Reusable chart component.
- `/src/components/ReportFilter.tsx`: Report filter component.
- `/src/lib/supabase/reports.ts`: Supabase queries for reports.
- `/src/hooks/useReports.ts`: Hook for fetching report data.
- `/src/app/api/reports/route.ts`: API route for report generation.
- `/src/types/report.ts`: Type definitions for reports.
- `/tests/ReportChart.test.tsx`: Jest tests for report chart.
- `/tests/reports.test.ts`: Jest tests for report queries and API.

## Notes or Caveats
- Use a charting library like Chart.js for visualizations.
- Aggregate data server-side for performance.
- Ensure Supabase RLS restricts data to user’s accounts.
- Test CSV export for large datasets.
