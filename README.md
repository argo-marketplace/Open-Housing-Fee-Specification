
# Open Housing Fee Specification

This repository documents the **Open Housing Fee Specification** (OHFS), a machine-readable format for specifying and sharing housing permit fee information.

OHFS is designed for planners, housing advocates, developers, and researchers interested in analyzing housing permit fees across California jurisdictions. OHFS attempts to fully encode a jurisdiction's permit fee structure in a form that is easy to store, share, modify, and apply programmatically.

### Table of Contents

* [Why a Standard for Sharing Housing Fees?](#why)
   - [Benefits of OHFS](#benefits)
* [Getting Started](#getting-started)
   - [Example 1 - Simple Flat Fee](#example1)
   - [Example 2 - Per-Square-Foot Fee](#example2)
   - [Example 3 - Tiered Building Permit Fee](#example3)
   - [Example 4 - Formula-Based Plan Check Fee](#example4)
   - [Example 5 - Impact Fee Depending on Bedroom Count](#example5)
   - [Example 6 - Multi-Variable Dependencies](#example6)
* [Full City Examples](#city-examples)
* [Fee Components Reference](#fee-components)
* [Input Variables Reference](#input-variables)

### Contribute

Please see the [CONTRIBUTING](CONTRIBUTING.md) file for details.

### Contact

Please reach out with comments, ideas, or suggestions using the [issues page](https://github.com/argo-marketplace/open-housing-fee-specification/issues).

---

## <a name="why"></a>Why a Standard for Sharing Housing Fees?

Housing permit fees in California are published in a patchwork of PDF documents, HTML tables, and council resolution attachments that are difficult to compare, analyze, or integrate into permitting tools. These formats serve human readers but create significant barriers for systematic analysis:

1. Fee schedules are published in inconsistent formats across cities, making cross-jurisdictional comparison labor-intensive.
2. The formulas used to calculate total permit costs—especially interdependent fees like plan check fees—are often not explicitly stated.
3. PDF and HTML formats are difficult to store, transmit, and parse programmatically.

California's housing crisis has placed renewed scrutiny on permit fees as a component of housing production costs. Understanding how fees vary across jurisdictions—and how they affect affordability—requires a machine-readable standard.

OHFS overcomes these barriers by specifying a plain-text format that fully encodes a jurisdiction's permit fee structure. Specifically:

1. OHFS is based on [YAML](http://yaml.org/), designed to be easy to store, transmit, and parse in any programming language while remaining human-readable.
2. All fee components—including formulas, tiered structures, and conditional charges—are captured in a single flat file.
3. The format supports the full range of California fee structures: flat fees, valuation-based tiers, per-unit impact fees, and formula-derived fees.

### <a name="benefits"></a>Benefits of OHFS

* Cross-jurisdictional analysis of permit fee levels and structures
* Quantifying the impact of fees on housing production costs and affordability
* Identifying fee structures that may disproportionately burden affordable or smaller-scale projects
* Integration with permitting estimation tools and development pro forma models
* Tracking fee changes over time as cities update their fee schedules

---

## <a name="getting-started"></a>Getting Started

The best way to understand OHFS is through examples. Files use the `.ohfs` extension.

#### <a name="example1"></a>Example 1 - Simple Flat Fee

The simplest OHFS file represents a jurisdiction that charges a single fixed fee for ADU applications.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  ADU:
    planning_application_fee: 850
    total_fee: planning_application_fee
```

`metadata` captures descriptive information about the fee schedule. `fee_structure` contains the fee calculation data, organized by permit class (e.g., `ADU`, `SINGLE_FAMILY`, `MULTI_FAMILY`).

In Example 1, the only fee is a flat `planning_application_fee` of $850, and `total_fee` is equal to it.

#### <a name="example2"></a>Example 2 - Per-Square-Foot Fee

Many cities charge building permit fees on a per-square-foot basis, sometimes combined with a base fee.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  SINGLE_FAMILY:
    base_permit_fee: 200
    per_sqft_rate: 0.38
    building_permit_fee: "base_permit_fee+per_sqft_rate*gross_sq_ft"
    total_fee: building_permit_fee
```

Here `gross_sq_ft` is an input variable representing the gross floor area of the project. `building_permit_fee` is a formula combining a fixed base fee with a per-square-foot charge. When OHFS parsers evaluate this file, they substitute the actual `gross_sq_ft` value from the project data.

#### <a name="example3"></a>Example 3 - Tiered Building Permit Fee

Most cities use a tiered fee structure based on project valuation, where the per-dollar rate decreases as valuation increases.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  SINGLE_FAMILY:
    tier_starts:
      - 0
      - 25000
      - 50000
      - 100000
      - 500000
      - 1000000
    tier_prices:
      - 0.0200
      - 0.0175
      - 0.0150
      - 0.0120
      - 0.0090
      - 0.0060
    building_permit_fee: Tiered
    total_fee: building_permit_fee
```

When `building_permit_fee` is set to `Tiered`, OHFS parsers calculate the fee by multiplying the portion of `project_valuation` that falls within each tier by the corresponding `tier_price`. The `tier_starts` list defines the lower bound of each valuation tier in dollars. `tier_prices` are in dollars per dollar of project valuation.

For example, on a $75,000 project: the first $25,000 is billed at $0.0200/dollar ($500), the next $25,000 at $0.0175/dollar ($437.50), and the remaining $25,000 at $0.0150/dollar ($375), for a total of $1,312.50.

#### <a name="example4"></a>Example 4 - Formula-Based Plan Check Fee

Plan check (plan review) fees are typically calculated as a percentage of the building permit fee—this interdependency is expressed with a formula.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  SINGLE_FAMILY:
    tier_starts:
      - 0
      - 25000
      - 50000
      - 100000
      - 500000
    tier_prices:
      - 0.0200
      - 0.0175
      - 0.0150
      - 0.0120
      - 0.0090
    building_permit_fee: Tiered
    plan_check_rate: 0.65
    plan_check_fee: "plan_check_rate*building_permit_fee"
    strong_motion_fee: "0.0001*project_valuation"
    total_fee: "building_permit_fee+plan_check_fee+strong_motion_fee"
```

Here `plan_check_fee` is defined as 65% of the `building_permit_fee`, a common structure in California cities. `strong_motion_fee` is the state-mandated SMIP fee expressed as a fraction of project valuation. `total_fee` sums all components.

#### <a name="example5"></a>Example 5 - Impact Fee Depending on Bedroom Count

Impact fees often vary by unit type or bedroom count. The `depends_on` field captures this conditional structure.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  MULTI_FAMILY:
    transportation_impact_fee:
      depends_on: bedroom_count
      values:
        0: 1250
        1: 2100
        2: 3150
        3: 4200
        4plus: 5250
    park_impact_fee:
      depends_on: bedroom_count
      values:
        0: 600
        1: 950
        2: 1400
        3: 1900
        4plus: 2400
    total_fee: "transportation_impact_fee+park_impact_fee"
```

When evaluating fees, the parser looks up the value of `bedroom_count` from the project data and selects the corresponding fee amount. It is important that the `bedroom_count` values used in practice exactly match the keys defined in `values`.

#### <a name="example6"></a>Example 6 - Multi-Variable Dependencies

Some fees depend on multiple variables simultaneously. For example, school fees may vary by both project type and school district zone.

```yaml
---
metadata:
  effective_date: 2024-01-01
  city_name: "Example City"
  state: CA
  fee_currency: USD

fee_structure:
  MULTI_FAMILY:
    school_fee:
      depends_on:
        - district_zone
        - unit_type
      values:
        elementary|studio: 3.48
        elementary|1_bedroom: 4.21
        elementary|2plus_bedroom: 5.17
        unified|studio: 4.12
        unified|1_bedroom: 5.03
        unified|2plus_bedroom: 6.31
    total_fee: school_fee
```

Values are encoded as pipe-delimited combination keys, matching the order of variables listed in `depends_on`. This format mirrors the OWRS convention for multi-variable dependencies and is straightforward for parsers to evaluate.

---

## <a name="city-examples"></a>Full City Fee Schedule Files

See the [fee schedules for California cities](full_fee_schedules/California/) in this repository for complete examples of city fee structures in OHFS format.

---

## <a name="fee-components"></a>Fee Components Reference

Standard fee component names used across OHFS files:

| Field | Description |
|-------|-------------|
| `building_permit_fee` | Core building permit fee, typically tiered by project valuation |
| `plan_check_fee` | Plan review fee, typically a percentage of the building permit fee |
| `planning_application_fee` | Planning department application or entitlement fee |
| `school_fee` | School impact fee (state-mandated under Education Code) |
| `park_impact_fee` | Parks and recreation impact fee |
| `transportation_impact_fee` | Transportation/traffic impact fee |
| `fire_impact_fee` | Fire department impact or service fee |
| `water_connection_fee` | Water connection or capacity fee |
| `sewer_connection_fee` | Sewer/wastewater connection or capacity fee |
| `strong_motion_fee` | State SMIP strong motion instrumentation fee |
| `green_building_fee` | CalGreen standards compliance or inspection fee |
| `affordable_housing_fee` | In-lieu affordable housing fee |
| `general_plan_fee` | General plan maintenance surcharge |
| `technology_fee` | Technology surcharge or permit software fee |
| `total_fee` | Final total: formula summing all applicable components |

---

## <a name="input-variables"></a>Input Variables Reference

Standard input variable names expected from project data when calculating fees:

| Variable | Description |
|----------|-------------|
| `project_valuation` | Total construction valuation in dollars |
| `gross_sq_ft` | Gross floor area in square feet |
| `net_sq_ft` | Net habitable floor area in square feet |
| `bedroom_count` | Number of bedrooms per unit |
| `unit_count` | Number of dwelling units in the project |
| `lot_sq_ft` | Lot area in square feet |
| `stories` | Number of above-grade stories |
