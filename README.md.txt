□	README.md containing: data model decisions, DAX approach for the hardest measure, what you would improve with more time

This table has two date keys — issue_date_key and renewal_due_date_key — both joining to dim_date. Only one relationship can be active. You must use USERELATIONSHIP() in DAX for the other. Decide which is your default and document the reason in your README.


RLS assumption

todoes:

3.  Not all agent_ids in fact_sales exist in dim_agent. Explore the data first. Decide how to label or exclude these records — do not let them silently disappear.


