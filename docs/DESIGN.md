# Design Notes

## Architecture
```
 Angular SPA (Material)  ──HTTP/JSON──>  Spring Boot API  ──JPA/JDBC──>  PostgreSQL
 (built into Spring static/,            controller → service → repository
  single deployable)                     package-by-feature: employee/, country/, insights/
```
**Decision:** serve the Angular build from Spring Boot, so there's one deploy, no CORS and one URL.
Trade-off: frontend and backend can't scale independently. That's irrelevant for 1 HR user.

## Data model
**country** (reference data, seeded via Flyway migration)
| column | type | note |
|---|---|---|
| code | CHAR(2) PK | ISO 3166, e.g. IN |
| name | VARCHAR | India |
| currency | CHAR(3) | INR |
| usd_per_unit | NUMERIC(12,6) | 0.012000, fixed rate |

**employee**
| column | type | note |
|---|---|---|
| id | BIGSERIAL PK | |
| full_name | VARCHAR(120) NOT NULL | |
| job_title | VARCHAR(80) NOT NULL | |
| department | VARCHAR(80) NOT NULL | |
| country_code | CHAR(2) FK → country | |
| annual_salary | NUMERIC(14,2) NOT NULL, > 0 | local currency |
| hire_date | DATE NOT NULL | |
| created_at, updated_at | TIMESTAMP | |

**Decisions**
- Currency is derived from the country, not stored per employee. This makes an "India employee paid in EUR" mismatch impossible.
- Money uses NUMERIC + BigDecimal, never double. 10,000 doubles summed can drift by paise; NUMERIC is exact.
- The schema is managed by Flyway migrations, not `ddl-auto`, so schema changes are versioned like code.

## Indexes
- `(country_code, job_title)` composite: serves the country filter, country+title filter, and the per-title insight GROUP BY
- `(job_title)`: title-only filter
- `LOWER(full_name)`: prefix search. Contains-search (`%priya%`) is a full scan, but on 10k rows that's ~5 ms, so it's accepted and documented.

## API
| Method | Path | Notes |
|---|---|---|
| GET | /api/employees?page=0&size=25&search=&country=&jobTitle=&department=&sort=fullName,asc | paginated |
| GET | /api/employees/{id} | 404 if missing |
| POST | /api/employees | 201 + Location header |
| PUT | /api/employees/{id} | full update |
| DELETE | /api/employees/{id} | 204 |
| GET | /api/countries | for dropdowns |
| GET | /api/insights/countries | headcount, min, max, avg per country (local currency) |
| GET | /api/insights/countries/{code}/job-titles | avg + headcount per title in that country |
| GET | /api/insights/org | cross-country comparison in USD |

Errors use RFC 7807 `ProblemDetail` (400 with field errors, 404 not found).
DTOs are kept separate from entities, so API shape changes don't leak into the schema.

## Insight queries (aggregation in SQL, not Java)
```sql
-- per country
SELECT country_code, COUNT(*), MIN(annual_salary), MAX(annual_salary), AVG(annual_salary)
FROM employee GROUP BY country_code;

-- per title within a country
SELECT job_title, COUNT(*), AVG(annual_salary)
FROM employee WHERE country_code = :code GROUP BY job_title;

-- org-wide in USD
SELECT e.country_code, AVG(e.annual_salary * c.usd_per_unit) AS avg_usd
FROM employee e JOIN country c ON c.code = e.country_code GROUP BY e.country_code;
```
**Why SQL:** loading 10k rows into Java is about 1.5 MB transferred per request. GROUP BY returns about 10 rows.

## Seed
- Deterministic: `new Random(42)`, so the same 10,000 employees are produced every run
- About 10 countries, 15 job titles, 6 departments; salary = title base × country factor × random(0.85 to 1.15)
- `JdbcTemplate.batchUpdate` in batches of 500, which is 20 round trips instead of 10,000
- Idempotent: skipped if the employee table is already populated

## Testing strategy
| Layer | Tool | Why |
|---|---|---|
| Domain and service | JUnit 5 + Mockito | fast, no Spring context, runs in ms |
| Controllers | @WebMvcTest | status codes, validation, JSON shape |
| Insight SQL | @DataJpaTest + Testcontainers Postgres | H2 behaves differently on NUMERIC and AVG; test against the real DB |
| Seed generator | unit test | same seed gives the same first N rows (determinism) |
| Angular | a few service and component tests | core flows only |

Trade-off: Testcontainers adds about 5 s of startup, so it's limited to repository tests to keep the main suite fast.

## Performance expectations (to verify and record actual numbers)
- Paginated list: about 3 KB response for 25 rows, versus about 1.5 MB for all 10k
- Insight queries: under 20 ms on 10k rows
- Seed: under 3 s for 10k rows