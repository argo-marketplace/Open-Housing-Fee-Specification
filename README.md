
# Open Housing Fee Specification

This repository documents the **Open Housing Fee Specification** (OHFS), a machine-readable format for specifying and sharing housing permit fee information.

OHFS is designed for planners, housing advocates, developers, and researchers interested in analyzing housing permit fees across California jurisdictions. OHFS attempts to fully encode a jurisdiction's permit fee structure in a form that is easy to store, share, modify, and apply programmatically.

> **Status:** spec version `0.2` (April 2026). This release introduces typed
> fee components (see [Fee Component Types](#fee-component-types)). It is a
> breaking change relative to the earlier untyped/`Tiered`/`depends_on`
> shape.

### Table of Contents

* [Why a Standard for Sharing Housing Fees?](#why)
   - [Benefits of OHFS](#benefits)
* [Getting Started](#getting-started)
   - [Example 1 - Simple Flat Fee](#example1)
   - [Example 2 - Linear (per-square-foot) Fee](#example2)
   - [Example 3 - Marginal Tiered Building Permit Fee](#example3)
   - [Example 4 - Step Fee by Square-Footage Threshold](#example4)
   - [Example 5 - Proportional Plan Check Fee](#example5)
   - [Example 6 - Categorical Impact Fee by Bedroom Count](#example6)
   - [Example 7 - Multi-Variable Categorical Fee](#example7)
   - [Example 8 - Conditional Fee with `applies_if`](#example8)
* [Fee Component Types](#fee-component-types)
* [Modifiers](#modifiers)
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
2. Every fee component declares an explicit `type` (or uses one of two scalar shorthands), so parsers don't have to guess from sibling keys.
3. Tier, step, categorical, and formula structures are first-class — covering the full range of California fee structures from flat fees to per-bedroom impact fees to coastal-zone surcharges.

### <a name="benefits"></a>Benefits of OHFS

* Cross-jurisdictional analysis of permit fee levels and structures
* Quantifying the impact of fees on housing production costs and affordability
* Identifying fee structures that may disproportionately burden affordable or smaller-scale projects
* Integration with permitting estimation tools and development pro forma models
* Tracking fee changes over time as cities update their fee schedules

---

## <a name="getting-started"></a>Getting Started

The best way to understand OHFS is through examples. Files use the `.ohfs` extension.

Each fee component is either:
- a **bare number** (shorthand for `{type: flat, amount: N}`),
- a **bare string containing operators** (shorthand for `{type: formula, formula: "..."}`), or
- an **object with an explicit `type` field** and the keys appropriate to that type.

#### <a name="example1"></a>Example 1 - Simple Flat Fee

The simplest OHFS file represents a jurisdiction that charges a single fixed fee for ADU applications.

```yaml
---
metadata:
  ohfs_version: "0.2"
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

In Example 1, `planning_application_fee: 850` is the bare-number shorthand for a `flat` component, and `total_fee` is set to that component's name.

#### <a name="example2"></a>Example 2 - Linear (per-square-foot) Fee

Many cities charge building permit fees on a per-square-foot basis, sometimes with a base fee.

```yaml
fee_structure:
  SINGLE_FAMILY:
    building_permit_fee:
      type: linear
      driver: gross_sq_ft
      base: 200
      rate: 0.38
    total_fee: building_permit_fee
```

A `linear` component computes `base + rate * driver`. Here `gross_sq_ft` is supplied from the project data; the result is `200 + 0.38 * gross_sq_ft`.

#### <a name="example3"></a>Example 3 - Marginal Tiered Building Permit Fee

Most cities use a tiered fee structure based on project valuation, where successive bands of valuation are charged at decreasing rates and the total is the sum across bands.

```yaml
fee_structure:
  SINGLE_FAMILY:
    building_permit_fee:
      type: marginal_tiered
      driver: project_valuation
      tiers:
        - {from: 0,       rate: 0.0200}
        - {from: 25000,   rate: 0.0175}
        - {from: 50000,   rate: 0.0150}
        - {from: 100000,  rate: 0.0120}
        - {from: 500000,  rate: 0.0090}
        - {from: 1000000, rate: 0.0060}
    total_fee: building_permit_fee
```

Each tier's `from` is the lower bound of the band; the `rate` applies to the portion of the driver that falls within that band. On a $75,000 project, the first $25,000 is billed at $0.0200/$ ($500), the next $25,000 at $0.0175/$ ($437.50), and the remaining $25,000 at $0.0150/$ ($375), for a total of $1,312.50.

If you want a non-marginal version where the entire driver value is charged at a single tier's rate selected by which band the driver falls in, use `type: block_tiered` with the same `tiers` shape.

#### <a name="example4"></a>Example 4 - Step Fee by Square-Footage Threshold

Some jurisdictions charge a flat amount that depends on which size band the project falls into — common for ADU permits, where the fee is one number for ADUs ≤ 750 sq ft, another for 751–1000 sq ft, and another above that.

```yaml
fee_structure:
  ADU:
    building_permit_fee:
      type: step
      driver: gross_sq_ft
      steps:
        - {up_to: 750,  amount: 1850}
        - {up_to: 1000, amount: 2400}
        - {up_to: null, amount: 3200}
    total_fee: building_permit_fee
```

`up_to` is the inclusive upper bound of each band. The final entry's `up_to: null` represents "no upper bound." Unlike `marginal_tiered`, the result is a single flat amount selected by which band the driver value falls in — there is no summation across bands.

#### <a name="example5"></a>Example 5 - Proportional Plan Check Fee

Plan check (plan review) fees are typically calculated as a percentage of the building permit fee.

```yaml
fee_structure:
  SINGLE_FAMILY:
    building_permit_fee:
      type: marginal_tiered
      driver: project_valuation
      tiers:
        - {from: 0, rate: 0.0200}
        - {from: 100000, rate: 0.0120}
    plan_check_fee:
      type: proportional
      of: building_permit_fee
      rate: 0.65
    strong_motion_fee:
      type: linear
      driver: project_valuation
      base: 0
      rate: 0.00010
    total_fee: "building_permit_fee+plan_check_fee+strong_motion_fee"
```

`proportional` reads another fee component named in `of` and multiplies it by `rate`. `total_fee` is a string formula summing components by name; bare component names sum without parentheses.

#### <a name="example6"></a>Example 6 - Categorical Impact Fee by Bedroom Count

Impact fees often vary by a discrete project attribute. Use `categorical` with a single `driver`.

```yaml
fee_structure:
  MULTI_FAMILY:
    transportation_impact_fee:
      type: categorical
      driver: bedroom_count
      scope: per_unit
      values:
        "0":     1250
        "1":     2100
        "2":     3150
        "3":     4200
        "4plus": 5250
    park_impact_fee:
      type: categorical
      driver: bedroom_count
      scope: per_unit
      values:
        "0":     600
        "1":     950
        "2":     1400
        "3":     1900
        "4plus": 2400
    total_fee: "transportation_impact_fee+park_impact_fee"
```

The `scope: per_unit` modifier multiplies the looked-up amount by `unit_count` to produce the contribution to `total_fee`. See [Modifiers](#modifiers) for the full set.

#### <a name="example7"></a>Example 7 - Multi-Variable Categorical Fee

Some fees depend on multiple variables simultaneously. Use `categorical` with `drivers` (plural) and pipe-delimited keys, ordered to match `drivers`.

```yaml
fee_structure:
  MULTI_FAMILY:
    school_fee:
      type: categorical
      drivers: [district_zone, unit_type]
      scope: per_sqft
      values:
        "elementary|studio":         3.48
        "elementary|1_bedroom":      4.21
        "elementary|2plus_bedroom":  5.17
        "unified|studio":            4.12
        "unified|1_bedroom":         5.03
        "unified|2plus_bedroom":     6.31
    total_fee: school_fee
```

`scope: per_sqft` multiplies the looked-up rate by `gross_sq_ft`.

#### <a name="example8"></a>Example 8 - Conditional Fee with `applies_if`

Some fees apply only when a project meets a boolean condition — for example, the Coastal Development Permit fee in coastal-zone jurisdictions.

```yaml
fee_structure:
  SINGLE_FAMILY:
    coastal_development_permit_fee:
      type: flat
      amount: 2400
      applies_if: in_coastal_zone
    building_permit_fee: 1500
    total_fee: "building_permit_fee+coastal_development_permit_fee"
```

When `in_coastal_zone` is false (or absent), the component contributes 0. `applies_if` accepts a bare project-variable name or a comparison expression (e.g., `"unit_count >= 10"`).

---

## <a name="fee-component-types"></a>Fee Component Types

Every long-form component declares a `type`. The set is small and intentionally separated into fee shapes whose output is *fixed* (independent of any continuous project variable) and shapes whose output is *variable*.

### Fixed shapes

| `type` | What it computes | Required keys |
|---|---|---|
| `flat` | a constant dollar amount | `amount` |
| `categorical` | a value looked up by one or more discrete project variables | `driver` or `drivers`, `values` |
| `step` | a flat amount selected by which band of a continuous variable the project falls in (whole amount, not marginal) | `driver`, `steps` (each `{up_to, amount}`; final entry's `up_to: null` means unbounded) |

### Variable shapes

| `type` | What it computes | Required keys |
|---|---|---|
| `linear` | `base + rate * driver` | `driver`, `rate` (and optional `base`, default 0) |
| `marginal_tiered` | sum of `rate_i * portion_in_tier_i` across all tiers up to the driver value | `driver`, `tiers` (each `{from, rate}`) |
| `block_tiered` | `rate_i * driver`, where tier `i` is the band the driver value falls in | `driver`, `tiers` (each `{from, rate}`) |
| `proportional` | `rate * <another fee component>` | `of`, `rate` |
| `formula` | the value of an arbitrary expression over project variables and other fee components | `formula` |

### Scalar shorthands

| Shorthand | Equivalent long form |
|---|---|
| `fee_name: 850` | `{type: flat, amount: 850}` |
| `fee_name: "0.65*building_permit_fee"` | `{type: formula, formula: "0.65*building_permit_fee"}` |

A bare string is treated as `formula` only if it contains an operator (`+`, `-`, `*`, `/`, parentheses) or whitespace; otherwise it is treated as a reference to another component (used in `total_fee`).

---

## <a name="modifiers"></a>Modifiers

Modifiers are optional keys that apply to any component regardless of `type`.

| Modifier | Effect |
|---|---|
| `scope` | How the type's output is scaled into `total_fee`. One of `per_project` (default), `per_unit` (× `unit_count`), `per_sqft` (× `gross_sq_ft`), `per_bedroom` (× `bedroom_count`). |
| `cap` | Upper bound applied to the per-component result after type evaluation but before `scope` scaling. |
| `floor` | Lower bound, applied the same way as `cap`. |
| `applies_if` | Boolean predicate — bare variable name or comparison expression. When false, the component contributes 0. |
| `notes` | Free-text annotation, ignored by parsers. |

### Evaluation order

For a single component the parser:
1. Evaluates `applies_if`. If false, the component value is 0, skip the rest.
2. Evaluates the type (`flat`, `linear`, `step`, etc.).
3. Applies `floor` then `cap`.
4. Applies `scope` (multiplies by the scope variable, default ×1).

`total_fee` is then evaluated as a formula over component names. Components referenced by name in `total_fee` use their post-modifier value.

---

## <a name="city-examples"></a>Full City Fee Schedule Files

See the [fee schedules for California cities](full_fee_schedules/California/) in this repository for complete examples of city fee structures in OHFS format.

> **Note on accuracy:** the city files in this repository carry an
> `UNVERIFIED` flag in `metadata.notes` if their fee amounts have not yet
> been confirmed against the official source document. Treat the structure
> as illustrative of the format, not as a substitute for the city's
> current fee resolution.

---

## <a name="fee-components"></a>Fee Components Reference

Standard fee component names used across OHFS files:

| Field | Description |
|-------|-------------|
| `building_permit_fee` | Core building permit fee |
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
| `affordable_housing_fee` | In-lieu affordable housing or linkage fee |
| `general_plan_fee` | General plan maintenance surcharge |
| `technology_fee` | Technology surcharge or permit software fee |
| `coastal_development_permit_fee` | Coastal Development Permit (coastal-zone jurisdictions only) |
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
| `in_coastal_zone` | Boolean — whether the project lies in the Coastal Zone |
| `district_zone` | School district zone identifier (used by `school_fee`) |
| `planning_area` | Community plan / planning area identifier (San Diego, others) |
| `market_area` | Market area or geography tier (LA Build Better LA, others) |
| `tenure_type` | `rental` or `ownership` (SF inclusionary, others) |
| `unit_type` | `studio`, `1_bedroom`, `2plus_bedroom`, etc. |
