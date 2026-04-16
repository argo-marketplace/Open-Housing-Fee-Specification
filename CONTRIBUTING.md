
# Contributing to OHFS

This repository documents how to contribute data to the **Open Housing Fee Specification** (OHFS), a machine-readable format for specifying and sharing housing permit fee information.

OHFS is designed so that anyone can contribute standardized housing fee data for California cities and counties.

### Getting Started

1. Read the [OHFS README](README.md) for background on the format and examples.
2. Pick a California city whose fee schedule has not yet been added. Check the `full_fee_schedules/California/` directory for what already exists.
3. Find the city's current fee schedule. Most cities publish fee schedules on their Building and Safety or Community Development department websites, typically as a PDF resolution or fee schedule document.

### Writing an OHFS File

1. Fork this repository and clone it locally.
2. Navigate to `full_fee_schedules/California/` and create a new folder for the city:
   ```
   mkdir "full_fee_schedules/California/[City Name]"
   ```
3. Create an OHFS file in that folder using the naming convention `[ACRONYM]-YYYY-MM-DD.ohfs`, where the date is the fee schedule's effective date:
   ```
   touch "full_fee_schedules/California/[City Name]/[ACRONYM]-YYYY-MM-DD.ohfs"
   ```
4. Start from one of the templates in the `templates/` directory:
   - `template.ohfs` — base template for any fee structure
   - `tiered_template.ohfs` — for fees tiered by project valuation or square footage
   - `depends_on_template.ohfs` — for fees that vary by a single variable (e.g., bedroom count)
   - `depends_on_multiple_template.ohfs` — for fees that vary by multiple variables

5. Fill in the fee structure based on the city's published fee schedule. Key things to capture:
   - **Building permit fee** — usually tiered by project valuation. Note whether tiers are per-dollar or per-$1,000 of valuation.
   - **Plan check fee** — usually a percentage of the building permit fee (commonly 65–80%).
   - **Impact fees** — transportation, park, school, fire. These are often flat per-unit amounts that vary by bedroom count or project type.
   - **Connection fees** — water and sewer connection or capacity fees.
   - **State fees** — strong motion (SMIP) fee (typically 0.01% of valuation) and school fees (Education Code Level 1 fee).
   - **Other surcharges** — technology fees, general plan fees, green building fees.

6. Use the standard fee component names and input variable names from the [README reference tables](README.md#fee-components) wherever possible to ensure consistency across files.

7. Add a `notes` field in `metadata` to record any caveats, exceptions, or links to the source document.

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Folder | Full city name | `Long Beach` |
| File | `[ACRONYM]-YYYY-MM-DD.ohfs` | `LB-2024-07-01.ohfs` |
| Effective date | ISO 8601 (`YYYY-MM-DD`) | `2024-07-01` |
| Permit classes | `UPPER_SNAKE_CASE` | `SINGLE_FAMILY`, `ADU`, `MULTI_FAMILY` |
| Fee components | `lower_snake_case` | `building_permit_fee`, `plan_check_fee` |

### Permit Classes

Use these standard permit class names:

| Class | Description |
|-------|-------------|
| `SINGLE_FAMILY` | Single-family detached residential |
| `ADU` | Accessory Dwelling Unit |
| `JADU` | Junior Accessory Dwelling Unit |
| `MULTI_FAMILY` | Multi-family residential (any size) |
| `COMMERCIAL` | Commercial construction |
| `MIXED_USE` | Mixed-use development |
| `OTHER` | Other project types not covered above |

### Submitting Your Contribution

1. Push your changes to your forked repository.
2. Open a pull request against the `master` branch of this repository.
3. In the PR description, link to the source fee schedule document and note the effective date.
4. A reviewer will check the file for completeness and accuracy before merging.

### Questions

Open an [issue](https://github.com/argo-marketplace/open-housing-fee-specification/issues) if you have questions about encoding a particular fee structure.
