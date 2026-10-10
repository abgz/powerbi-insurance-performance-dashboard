# Agency Performance Dashboard

A Power BI report for a regional life insurance company's agency distribution team. It shows how agents perform against APE (Annualised Premium Equivalent) targets, which products drive premium income, and how well policies renew.

The project is saved in PBIP format. The wireframe for Page 1 is in [`/mockups/page1_wireframe.html`](mockups/page1_wireframe.html).

## Data model decisions

### Two date relationships on `fact_persistency`

`fact_persistency` has two date keys, `issue_date_key` and `renewal_due_date_key`, and both join to `dim_date`.

- `fact_persistency[issue_date_key]` to `dim_date[date_key]` is the **active** relationship.
- `fact_persistency[renewal_due_date_key]` to `dim_date[date_key]` is **inactive**. Measures that need it can switch to it with `USERELATIONSHIP()`.

**Why issue date is the active one:** the Page 4 cohort analysis groups policies by issue year, so the issue date has to drive the date filter. Making the due date active would group policies by the year they fall due, which changes the persistency results considerably.

### Row-level security (RLS) assumption

The `dim_agent` table has no email address, login name or other identifier that can be matched to a signed-in user. To support the "Territory Manager" role, I assumed a user identifier and created an **email column in `dim_agent`, derived from the existing user name column**. The role uses that column to identify the signed-in user and restrict the data to their territory.

This is an assumption made to make RLS testable with **View as**. A real deployment would need a proper user-to-territory mapping (see the open questions below).

## DAX approach for the hardest measure: Active Policy Count

Active Policy Count is a snapshot, so it is **semi-additive**. It can be added across agents, products and regions, but not across time. A plain `SUM` or row count would count policies issued in a period, not policies in force at a point in time.

The approach is to take the count **as of the last date in the selected period**:

```dax
Active Policy Count =
VAR _asOf = MAX ( dim_date[date] )
VAR _asOfKey = YEAR ( _asOf ) * 10000 + MONTH ( _asOf ) * 100 + DAY ( _asOf )
VAR _issued =
    CALCULATE (
        DISTINCTCOUNT ( fact_persistency[policy_id] ),
        REMOVEFILTERS ( dim_date ),
        fact_persistency[issue_date_key] <= _asOfKey
    )
VAR _lapsed =
    CALCULATE (
        DISTINCTCOUNT ( fact_persistency[policy_id] ),
        REMOVEFILTERS ( dim_date ),
        fact_persistency[issue_date_key] <= _asOfKey,
        fact_persistency[renewal_due_date_key] <= _asOfKey,
        fact_persistency[renewal_status] = "lapsed"
    )
RETURN
    _issued - _lapsed
```

How it works:

1. `_asOf` is the last date in the current filter context, such as the end of the month, quarter or year on a chart axis.
2. `REMOVEFILTERS ( dim_date )` stops the date filter from limiting the count to policies issued in that period. The cut-off comes from `_asOfKey` instead.
3. The result is policies issued up to that date, minus policies that had lapsed by that date.
4. Agent, product, region and territory filters still apply, so the measure adds up across those dimensions and takes a snapshot across time. RLS applies in the same way.

Assumptions behind this definition (see the open question on the business definition):

- A policy is in force at a date if it was issued on or before that date and has not lapsed with a due date on or before that date.
- `grace` is treated as still in force, because the data dictionary describes it as overdue but not yet cancelled.
- `fact_persistency` has no lapse date, so the renewal due date stands in for it.

## What I would improve with more time

**UI/UX.** Refine the design and polish across all pages, including the mobile layout.

**Persistency calculations.** Rework the persistency calculations once the open questions below are answered. Specifically, I would build the cohort view on the renewal history the dataset actually supports, and tighten the definition and the date logic behind Active Policy Count.

## Open questions

### 1. Persistency: single renewal record per policy

The data dictionary describes `fact_persistency` as having one row per policy, with a single `renewal_status` and a single `renewal_due_date_key`. Page 4 asks for a cohort analysis that tracks renewal rates across subsequent years for policies grouped by issue year.

With one renewal record per policy, I can calculate persistency for the recorded renewal outcome, but there does not appear to be enough renewal history to track second, third and later renewals for each cohort.

**Question:** Is the expectation to analyse the available renewal outcome by issue-year cohort, or is the dataset expected to support tracking persistency across multiple renewal years? If multi-year tracking is intended, is additional renewal history available, or should I document this data limitation and proceed with the available data?

### 2. RLS: no user identifier in `dim_agent`

`dim_agent` does not contain an email ID, login name or other unique security identifier that can map an authenticated user to their territory for RLS.

**Question:** Should a user-to-territory mapping table be provided, or should a unique identifier (such as an email ID) be added to `dim_agent` to support dynamic RLS?

### 3. Active Policy Count: definition and reporting date

The requirement says Active Policy Count should use a semi-additive pattern, not a simple `SUM`.

**Question:** What is the expected business definition of an active policy, and which reporting date should the measure use? Should it return the active policy count as of the last available date in the selected reporting period, or as of a specified reporting date?
