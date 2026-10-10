# Agency Performance Dashboard

A Power BI report for a regional life insurance company's agency distribution team. It shows how agents perform against APE (Annualised Premium Equivalent) targets, which products drive premium income, and how well policies renew.

The project is saved in PBIP format. The wireframe for Page 1 is in [`/mockups/page1_wireframe.html`](mockups/page1_wireframe.html).

## Data files and setup

The source CSV files are **not included in this repository**. They are excluded through `.gitignore`, so the supplied data stays out of version control. If you clone or pull this repository, the report will open but will show no data until you add the files yourself.

To run the report:

1. Get the CSV files supplied with the assessment: `dim_date`, `dim_agent`, `dim_product`, `fact_sales`, `fact_target` and `fact_persistency`.
2. Place them in the `data/` folder at the root of the repository, keeping the original file names.
3. Open the project in Power BI Desktop. If the files are somewhere else, go to **Transform data > Data source settings** and point each source to the new location.
4. Select **Refresh**. The model, measures and visuals then load against your copy of the data.

## Data model decisions

### Two date relationships on `fact_persistency`

`fact_persistency` has two date keys, `issue_date_key` and `renewal_due_date_key`, and both join to `dim_date`.

- `fact_persistency[issue_date_key]` to `dim_date[date_key]` is the **active** relationship.
- `fact_persistency[renewal_due_date_key]` to `dim_date[date_key]` is **inactive**. Measures that need it can switch to it with `USERELATIONSHIP()`.

**Why issue date is the active one:** the Page 4 cohort analysis groups policies by issue year, so the issue date has to drive the date filter. Making the due date active would group policies by the year they fall due, which changes the persistency results considerably.

### Row-level security (RLS) assumption

The `dim_agent` table has no email address, login name or other identifier that can be matched to a signed-in user. To support the "Territory Manager" role, I assumed a user identifier and created an **email column in `dim_agent`, derived from the existing user name column**. The role uses that column to identify the signed-in user and restrict the data to their territory.

This is an assumption made to make RLS testable with **View as**. A real deployment would need a proper user-to-territory mapping (see the open questions below).

## What I would improve with more time

**UI/UX.** Refine the design and polish across all pages, including the mobile layout.

**Persistency calculations.** Rework the persistency calculations once the open questions below are answered. Specifically, I would build the cohort view on the renewal history the dataset actually supports.

## Open questions

### 1. Persistency: single renewal record per policy

The data dictionary describes `fact_persistency` as having one row per policy, with a single `renewal_status` and a single `renewal_due_date_key`. Page 4 asks for a cohort analysis that tracks renewal rates across subsequent years for policies grouped by issue year.

With one renewal record per policy, I can calculate persistency for the recorded renewal outcome, but there does not appear to be enough renewal history to track second, third and later renewals for each cohort.

**Question:** Is the expectation to analyse the available renewal outcome by issue-year cohort, or is the dataset expected to support tracking persistency across multiple renewal years? If multi-year tracking is intended, is additional renewal history available, or should I document this data limitation and proceed with the available data?

### 2. RLS: no user identifier in `dim_agent`

`dim_agent` does not contain an email ID, login name or other unique security identifier that can map an authenticated user to their territory for RLS.

**Question:** Should a user-to-territory mapping table be provided, or should a unique identifier (such as an email ID) be added to `dim_agent` to support dynamic RLS?
