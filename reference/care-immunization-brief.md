# Project brief — `care_immunization`

This brief turns the immunization-programme idea from
[issue #18](https://github.com/sabinss4/care_scaffold/issues/18) into an implementation-ready
scope for a future `care_immunization` / `care_immunization_fe` plugin pair. It is a design
artifact, not application code.

## Outcome

Provide staff with a reliable immunization programme workflow: define vaccine schedules, identify
people who are due or overdue, record administered doses against vaccine lots, capture adverse
events following immunization (AEFI), and report coverage. The plugin owns this domain and
references CARE records; neither CARE core nor `care_fe` should contain immunization-specific
models or logic.

## Scope

### In scope

- Vaccine products and programme-specific schedules, including dose order and eligibility rules.
- Due-list generation for a facility, programme, date range, and patient cohort.
- Administration records linked to a patient, dose, vaccinator, facility, date, and vaccine lot.
- Lot inventory metadata: lot number, product, expiry, quantities received/used/discarded, and
  optional cold-chain notes.
- AEFI reports with symptoms, onset, severity, outcome, reporter, and follow-up status.
- Coverage summaries by programme, dose, facility, cohort, and reporting period.
- Staff-facing pages and dashboard components exposed through the plugin manifest.

### Out of scope for the first release

- National registry or health-information-exchange submission.
- Automated clinical recommendations beyond configured schedule rules.
- Cold-chain sensor integrations.
- Patient self-registration or OTP administration workflows.
- Stock purchasing, invoices, and supply-chain procurement.

## Architecture decisions

| Requirement | Plugin design | Core changes |
| --- | --- | --- |
| Vaccine, schedule, dose, lot, administration, AEFI, and coverage concepts | New plugin models inheriting CARE base models | 0 |
| Link an administration or AEFI to a person | Foreign keys into the existing patient/person model | 0 |
| Identify a patient’s programme enrolment or cohort | Plugin-owned relationship and eligibility snapshot | 0 |
| Due-list calculation | Plugin service/query and an idempotent scheduled task | 0 |
| Staff pages and dashboard | `manifest.routes` and existing generic extension points | 0 unless an attachment point is missing |
| Public configuration | Plugin `PluginSettings` and `config/` endpoint | 0 |
| Notifications or reminders | Optional integration guarded by installed-app checks | 0 |

Do not add columns to CARE models. If a small annotation on an existing core record is eventually
needed, namespace it under `immunization` in that record’s `meta` JSON field.

## Proposed domain model

All records are soft-delete aware and expose CARE external IDs through serializers.

- `ImmunizationProgramme`: name, code, description, active status, and owning facility scope.
- `VaccineProduct`: antigen/product name, manufacturer, route, dose volume, and active status.
- `ProgrammeDose`: programme, dose order, vaccine product, minimum/target age, interval, and
  eligibility constraints.
- `PatientProgramme`: patient, programme, enrolment date, status, and eligibility snapshot.
- `VaccineLot`: product, lot number, expiry, facility, received quantity, and status.
- `ImmunizationAdministration`: patient programme, dose, administered date, facility, vaccinator,
  lot, route/site, and administration status.
- `AEFIReport`: administration (when known), patient, report details, severity, outcome, and
  follow-up status.
- `CoverageSnapshot`: programme, facility, reporting period, cohort, numerator, denominator, and
  calculation metadata.

The lot’s available quantity must be derived from immutable transaction records if inventory
adjustment/audit requirements are confirmed; otherwise the first implementation may use explicit
receipt/use/discard totals with validation that quantities cannot become negative.

## API surface

The backend should mount under `/api/care_immunization/`.

| Method | Path | Purpose |
| --- | --- | --- |
| GET/POST | `/programmes/` | List and create programmes |
| GET/PATCH | `/programmes/{id}/` | Read or update a programme |
| GET/POST | `/programmes/{id}/doses/` | Manage ordered programme doses |
| GET/POST | `/products/` | Manage vaccine products |
| GET/POST | `/lots/` | List and register vaccine lots |
| GET/PATCH | `/lots/{id}/` | Review or update lot status and quantities |
| GET | `/due-list/` | Facility- and date-scoped due/overdue patients |
| GET/POST | `/administrations/` | List and record administered doses |
| GET/PATCH | `/administrations/{id}/` | Correct or review an administration |
| GET/POST | `/aefi-reports/` | Report and list AEFI cases |
| GET/PATCH | `/aefi-reports/{id}/` | Review and update AEFI follow-up |
| GET | `/coverage/` | Return coverage aggregates for a reporting period |
| GET | `/config/` | Return client-safe plugin configuration |

Every list endpoint must enforce facility and role permissions server-side. Do not trust patient,
facility, or programme IDs supplied by the client to establish access.

## Frontend surfaces

The frontend package should expose:

- A route for programme and schedule administration.
- A due-list route with programme, facility, date, and due-status filters.
- A patient immunization timeline showing completed and upcoming doses.
- An administration form with lot selection and remaining-quantity feedback.
- An AEFI queue and detail workflow.
- A coverage dashboard with period and facility filters.

Use the existing extension-point catalog before proposing a core change. All user-facing strings
belong in `care_immunization_fe/public/locale/en.json` under the `immunization__` prefix, and
all roots must use the plugin Tailwind scope.

## Roles and lifecycle

These are proposed defaults and require product confirmation before implementation:

| Record | Proposed states | Proposed transitions |
| --- | --- | --- |
| Programme | `draft`, `active`, `archived` | programme manager activates/archives |
| Administration | `recorded`, `voided` | vaccinator records; authorised supervisor voids |
| AEFI report | `reported`, `under_review`, `closed` | reporter submits; supervisor reviews/closes |
| Vaccine lot | `available`, `exhausted`, `expired`, `quarantined` | inventory staff update |

Voiding an administration must preserve the audit trail and restore lot availability only through
an explicit, audited adjustment. Coverage must exclude voided administrations.

## Open product decisions

Before scaffolding the plugin, confirm:

1. Which patient/person model and facility relationship are authoritative in the target CARE
   version?
2. Are programmes configured per facility, globally, or both?
3. Which age, pregnancy, risk, and catch-up rules are required for the first release?
4. Must lot inventory be transaction-based from day one?
5. Which roles may create lots, administer doses, void records, and close AEFI reports?
6. Is patient-portal read access required after the first release?
7. Which coverage denominator source is authoritative?
8. Is an external reporting format or integration required?
9. Which optional notifications plugin, if any, should receive due-list reminders?
10. Which frontend preview port and workspace root should be used?

## Verification gates

The implementation should not proceed past each gate until it passes:

1. The model and permission decisions above are confirmed and recorded in the generated
   `.agent/plugin-brief.md`.
2. Backend migrations, serializer validation, facility scoping, lot quantity invariants, and
   administration/AEFI lifecycle tests pass.
3. Due-list and coverage calculations have deterministic fixture tests for on-time, overdue,
   skipped, voided, and catch-up cases.
4. The API is exercised before frontend work; then the frontend type-check and production build
   pass.
5. A cold-start verification confirms the plugin registers without adding immunization-specific
   code to CARE or `care_fe`.

