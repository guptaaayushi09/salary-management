# Salary Management: Requirements

## Goal
Replace ACME's Excel-based salary tracking with a web app that lets an HR Manager
manage salary data for ~10,000 employees across multiple countries and quickly
answer "how does the org pay people?"

## User
HR Manager (single trusted user). Comfortable with Excel, not technical.
Needs speed: find an employee in seconds, answer pay questions without pivot tables.

## Assumptions
- Salary = one annual gross amount per employee, stored in the employee's local currency
- Cross-country comparison uses a fixed exchange-rate table (USD base), not live rates
- Single HR user, no login
- Frontend: modern Angular (v17+)

## In scope
1. Employee management
   - Create, view, update, delete employees
   - Fields: full name, job title, department, country, currency, annual salary, hire date
   - Validation: salary > 0, required fields, valid currency code per country
2. Find employees fast
   - Server-side pagination (default 25/page)
   - Search by name; filter by country, job title, department
3. Salary insights (the core HR questions)
   - Per country: headcount, min, max, average salary (local currency)
   - Per job title within a country: average salary and headcount
   - Org-wide comparison across countries in USD using the fixed rate table
4. Data
   - Deterministic seed script generating 10,000 employees
5. Delivery
   - Deployed app, README, video demo, AI usage log

## Deliberately out of scope, and why
| Left out | Reason |
|---|---|
| Authentication and roles | Single trusted user in the brief; adds effort without testing core judgment. Would add before real production use. |
| Payroll processing and tax/net salary | Country-specific tax rules are a product in themselves; the brief asks about pay insights, not payroll. |
| Salary history and audit trail | Valuable, but doubles data model complexity. First candidate for v2. |
| Live currency conversion | External dependency and non-deterministic tests; fixed rates are enough for comparison. |
| Excel/CSV import | Strong fit for HR's current workflow; planned as a stretch goal if time permits. |

## Success criteria
- HR can find any employee in under 3 clicks
- Insight queries return in under 200 ms on 10,000 rows
- Core domain and service logic covered by fast, deterministic unit tests