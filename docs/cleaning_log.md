## transactions.csv

# cleaning test results:

- No column with a NULL entry
- No duplicate rows
- Dates already standardised

# price violation findings
- 3,417 rows (6.4%) have 'selling price' < 'cost price'
- These are likely discount-driven: selling below cost during promos
- Flagged with 'price violation' = 'True'
- Retained in dataset for EDA and mispricing analysis
- Will be EXCLUDED from elasticity model fitting in Phase 3
- Conclusion: flagged and retained for the mispricing analysis
