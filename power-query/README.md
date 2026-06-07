# Power Query Function

## LoadWFile Function

Parameterized Power Query function that consolidates shop load reports into a single fact table.

### Function Signature

```m
LoadWFile = (FileName as text, StartWeek as number, EndWeek as number) =>
```

### Parameters

- **FileName:** Excel file name (e.g., "W1_to_W6.xlsx")
- **StartWeek:** Starting week number (e.g., 0)
- **EndWeek:** Ending week number (e.g., 5)

### What It Does

1. Loads Excel file from SharePoint
2. Extracts ShopLoadReport sheet
3. Selects relevant columns
4. Filters to Load rows
5. Replicates rows for each week (StartWeek to EndWeek)
6. Adds week column (W0, W1, W2, etc.)
7. Returns consolidated table

### Output Columns

- `Resource_Grp`: Machine/Resource Group ID
- `Resource_Grp_Desc`: Resource group description
- `Load_Hours`: Actual load hours (numeric)
- `Week`: Week label (W0, W1, W2, ..., W84)

### Example Usage

```m
W0_to_W6 = LoadWFile("W1 to W6.xlsx", 0, 5)
W7_to_W12 = LoadWFile("W7 to W12.xlsx", 6, 11)
Combined = Table.Combine({W0_to_W6, W7_to_W12, ...})
```

### Key Achievement

One reusable function called 12 times = **95% code reduction** vs. 84 separate queries

### For Detailed Implementation

See [POWER_QUERY_IMPLEMENTATION.md](../docs/POWER_QUERY_IMPLEMENTATION.md) for:
- Line-by-line code explanation
- 13-step transformation walkthrough
- Customization guide
- Troubleshooting

### For Setup Instructions

See [SETUP_INSTRUCTIONS.md](../docs/SETUP_INSTRUCTIONS.md) for step-by-step implementation in your environment.
