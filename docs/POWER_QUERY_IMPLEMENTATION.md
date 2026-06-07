# RCCP Capacity Planning: Power Query Implementation Guide

## Overview

This guide explains the `LoadWFile` parameterized function that consolidates 84 weekly shop load reports into a single fact table.

**Key Achievement:** One reusable function called 12 times = 95% code reduction vs. 84 separate queries.

---

## The LoadWFile Function: Complete Breakdown

### Function Signature

```m
LoadWFile = (FileName as text, StartWeek as number, EndWeek as number) =>
```

**Parameters:**
- `FileName`: Name of the Excel file (e.g., "W1_to_W6.xlsx")
- `StartWeek`: Starting week number (e.g., 0 for W0)
- `EndWeek`: Ending week number (e.g., 5 for W5)

**Returns:** Consolidated table with columns: Resource_Grp, Resource_Grp_Desc, Load_Hours, Week

---

## Step-by-Step Code Walkthrough

### STEP 1: Load from SharePoint

```m
Source = SharePoint.Files("https://#######################/", [ApiVersion = 15]),

#"Filtered rows" = Table.SelectRows(Source, each Text.Contains([Folder Path], "RCCP")),

Navigation = #"Filtered rows"{[Name = FileName, #"Folder Path" = "https://##############################################/"]}[Content]
```

**What it does:**
1. Connect to SharePoint folder
2. Filter to files in RCCP path
3. Select the specific file by name

**Why this matters:**
- `Text.Contains([Folder Path], "RCCP")` finds the RCCP folder
- Uses dynamic `FileName` parameter (e.g., "W1_to_W6.xlsx")
- Supports any file in the SharePoint directory

**You need to update:**
```m
// BEFORE:
Source = SharePoint.Files("https://#######################/", [ApiVersion = 15])

// AFTER: Replace with YOUR SharePoint URL
Source = SharePoint.Files("https://yourcompany.sharepoint.com/sites/manufacturing/", [ApiVersion = 15])

// BEFORE:
Navigation = #"Filtered rows"{[Name = FileName, #"Folder Path" = "https://##############################################/"]}[Content]

// AFTER: Replace with YOUR folder path
Navigation = #"Filtered rows"{[Name = FileName, #"Folder Path" = "https://yourcompany.sharepoint.com/sites/manufacturing/Shared Documents/RCCP/"]}[Content]
```

---

### STEP 2: Import Excel Workbook

```m
#"Imported Excel workbook" = Excel.Workbook(Navigation, null, true),

#"Navigation 1" = #"Imported Excel workbook"{[Item = "ShopLoadReport", Kind = "Sheet"]}[Data]
```

**What it does:**
1. Read the Excel file content from SharePoint
2. Extract the "ShopLoadReport" sheet
3. Load all data from that sheet

**Why this matters:**
- `Excel.Workbook()` reads the entire workbook
- `[Item = "ShopLoadReport"]` selects specific sheet
- `[Data]` gets the actual table content

**Assumption:** Your Excel files must have a sheet named "ShopLoadReport"

---

### STEP 3: Select Relevant Columns

```m
#"Selected Columns" = Table.SelectColumns(
    #"Navigation 1",
    {"Column1", "Column4", "Column11", "Column16", "Column22"})
```

**What it does:**
Keep only 5 columns, discard the rest.

**Column mapping:**
| Column | Purpose | Example |
|--------|---------|---------|
| Column1 | Marker (should be "RG:") | "RG:" |
| Column4 | Machine/Resource Group ID | "MOLD-01" |
| Column11 | Resource Group Description | "Mold Processing" |
| Column16 | Load Marker (should be "Load:") | "Load:" |
| Column22 | Load Hours (actual hours) | 156.5 |

**Why this matters:**
- Removes unnecessary columns (operations, dates, etc.)
- Reduces data size
- Focuses on what we need

**If your columns are different:**
```m
// Count from left (Column1 is first column)
// Column1, Column2, Column3, Column4, Column5, Column6, ...
// If Machine is in Column5 instead of Column4, change to:
Table.SelectColumns(#"Navigation 1", {"Column1", "Column5", "Column11", "Column16", "Column22"})
```

---

### STEP 4: Fill Down Empty Cells

```m
#"Filled Down" = Table.FillDown(
    #"Selected Columns",
    {"Column1", "Column4", "Column11"})
```

**What it does:**
Propagate values down cells that are empty.

**Example (Before):**
```
Column1    Column4        Column11
RG:        MOLD-01        Mold Processing
           (empty)        (empty)
           (empty)        (empty)
RG:        LOGOCELL       Logo Cell
```

**Example (After):**
```
Column1    Column4        Column11
RG:        MOLD-01        Mold Processing
RG:        MOLD-01        Mold Processing
RG:        MOLD-01        Mold Processing
RG:        LOGOCELL       Logo Cell
```

**Why this matters:**
- Excel files have merged cells for readability
- Power Query can't work with merged cells
- Fill down recreates the structure

---

### STEP 5: Filter Empty Rows

```m
#"Filtered Rows1" = Table.SelectRows(
    #"Filled Down",
    each ([Column22] <> null and [Column22] <> ""))
```

**What it does:**
Remove rows where Column22 (Load_Hours) is empty or null.

**Why this matters:**
- Data often has blank rows (spacing)
- We only want rows with actual load hours
- Removes noise

---

### STEP 6: Keep Only "Load:" Rows

```m
#"Filtered Rows" = Table.SelectRows(
    #"Filtered Rows1",
    each [Column16] = "Load:" and [Column1] = "RG:" and [Column22] <> "")
```

**What it does:**
Keep only rows where:
- Column16 = "Load:" (identifies load rows)
- Column1 = "RG:" (identifies resource group marker)
- Column22 has a value (has load hours)

**Why this matters:**
- Excel sheets have multiple row types (headers, operations, descriptions)
- We only want load rows
- "Load:" and "RG:" markers ensure we get correct rows

**Example:**
```
Column1    Column16      Column22       KEEP?
RG:        Load:         156.5          YES ✓
RG:        Operations:   text           NO (Column16 ≠ "Load:")
(empty)    Load:         123            NO (Column1 ≠ "RG:")
RG:        Load:                        NO (Column22 empty)
```

---

### STEP 7: Rename Columns

```m
#"Renamed Columns" = Table.RenameColumns(
    #"Filtered Rows",
    {
        {"Column4", "Resource_Grp"},
        {"Column11", "Resource_Grp_Desc"},
        {"Column22", "Load_Hours"}
    })
```

**What it does:**
Replace generic column names with meaningful names.

**Mapping:**
- Column4 → Resource_Grp (machine ID)
- Column11 → Resource_Grp_Desc (machine description)
- Column22 → Load_Hours (the metric we care about)

**Result:**
```
Resource_Grp    Resource_Grp_Desc    Load_Hours
MOLD-01         Mold Processing      156.5
LOGOCELL-A      Logo Cell A          89.2
```

---

### STEP 8: Set Data Types

```m
#"Changed Type" = Table.TransformColumnTypes(
    #"Renamed Columns",
    {
        {"Resource_Grp", type text},
        {"Resource_Grp_Desc", type text},
        {"Load_Hours", type number}
    })
```

**What it does:**
Specify correct data types for each column.

**Why this matters:**
- Resource_Grp: Text (not to be calculated)
- Resource_Grp_Desc: Text (description)
- Load_Hours: Number (can sum, average, etc.)

**Impact:**
- Enables calculations on Load_Hours
- Prevents Excel interpreting numbers as text

---

### STEP 9: Remove Unnecessary Columns

```m
#"Removed Columns" = Table.SelectColumns(
    #"Changed Type",
    {"Resource_Grp", "Resource_Grp_Desc", "Load_Hours"})
```

**What it does:**
Keep only the 3 renamed columns, drop Column1 and Column16.

**Result:**
Clean table with just what we need.

---

### STEP 10: Replicate Rows for Each Week (THE MAGIC)

```m
WeekList = List.Generate(
    () => StartWeek,
    each _ <= EndWeek,
    each _ + 1)
```

**What it does:**
Create a list of week numbers from StartWeek to EndWeek.

**Example (StartWeek=0, EndWeek=5):**
```
{0, 1, 2, 3, 4, 5}
```

**Why this matters:**
- Generates all week numbers programmatically
- No hardcoding (flexible)
- Foundation for replication

---

### STEP 11: Add Week Column for Each Week

```m
ReplicatedTables = List.Transform(
    WeekList,
    (WeekNum) =>
        let
            WeekLabel = "W" & Text.From(WeekNum),
            Duplicated = #"Removed Columns",
            #"Added Week" = Table.AddColumn(
                Duplicated,
                "Week",
                each WeekLabel,
                type text)
        in
            #"Added Week")
```

**What it does:**
For each week number, create a new table with week label added.

**Example (for W1_to_W6.xlsx with 50 rows):**
- Input: 50 rows (Resource_Grp, Resource_Grp_Desc, Load_Hours)
- Process:
  - Week 0: Duplicate 50 rows, add "W0" to each
  - Week 1: Duplicate 50 rows, add "W1" to each
  - Week 2: Duplicate 50 rows, add "W2" to each
  - ... continues through Week 5
- Output: 300 rows (50 rows × 6 weeks)

**Result:**
```
Resource_Grp    Resource_Grp_Desc    Load_Hours    Week
MOLD-01         Mold Processing      156.5         W0
MOLD-01         Mold Processing      156.5         W1
MOLD-01         Mold Processing      156.5         W2
... (continues through W5)
LOGOCELL-A      Logo Cell A          89.2          W0
LOGOCELL-A      Logo Cell A          89.2          W1
... (continues through W5)
```

**Why this is clever:**
- Same source data appears for each week
- Week column allows filtering/analysis by week
- No duplicate data (same hours reported for each week's schedule)

---

### STEP 12: Combine All Week Tables

```m
#"Combined Tables" = Table.Combine(ReplicatedTables)
```

**What it does:**
Stack all week tables vertically into one.

**Example:**
- Input: List of 6 tables (one per week)
- Output: Single table with all 6 week's data stacked

---

### STEP 13: Remove Duplicates

```m
#"Removed Duplicates" = Table.Distinct(#"Combined Tables")
```

**What it does:**
Remove any duplicate rows (safety check).

**Why this matters:**
- Data might have duplicate entries
- Ensures data integrity
- One row per machine per week

---

## How to Implement in Your Environment

### 1. Get Your SharePoint Information

```
Navigate to your SharePoint folder in browser
URL: https://yourcompany.sharepoint.com/sites/manufacturing/Shared Documents/RCCP/

Copy the URL (you'll need it)
```

### 2. Create a New Dataflow in Fabric

```
Fabric → Data Engineering → Dataflow Gen2 → New Dataflow
Name: RCCP_LoadAndReplicateByWeek
```

### 3. Add Power Query Source

```
Get Data → SharePoint Folder
Paste your SharePoint URL
Select your folder
```

### 4. Create the Function

```
In Power Query Editor:
New Source → Blank query
Name it: LoadWFile
Paste the function code

Replace placeholders:
- SharePoint URL
- Folder path
```

### 5. Call the Function

```
Create new queries:
W0_to_W6 = LoadWFile("W1 to W6.xlsx", 0, 5)
W7_to_W12 = LoadWFile("W7 to W12.xlsx", 6, 11)
W13_to_W18 = LoadWFile("W13 to W18.xlsx", 12, 17)
... continue for all files
```

### 6. Combine Results

```
Combined = Table.Combine({W0_to_W6, W7_to_W12, W13_to_W18, ...})

Final = Combined
```

### 7. Load to Lakehouse

```
Home → Load → Load to new table
Table name: fact_Load_Hours
Lakehouse: RCCP_Analytics
```

---

## Customization Guide

### If Your Column Positions Are Different

**Find the correct columns:**
1. Open one of your Excel files
2. Count from left: Column1, Column2, Column3, etc.
3. Identify which column has:
   - Resource Group ID
   - Resource Group Description
   - Load Hours

**Update the function:**
```m
// ORIGINAL:
Table.SelectColumns(#"Navigation 1", {"Column1", "Column4", "Column11", "Column16", "Column22"})

// IF Column5 has Load Hours instead of Column22:
Table.SelectColumns(#"Navigation 1", {"Column1", "Column4", "Column11", "Column16", "Column5"})
```

### If Your Sheet Name Is Different

**Update this line:**
```m
// ORIGINAL:
#"Navigation 1" = #"Imported Excel workbook"{[Item = "ShopLoadReport", Kind = "Sheet"]}[Data]

// IF your sheet is called "Shop Load":
#"Navigation 1" = #"Imported Excel workbook"{[Item = "Shop Load", Kind = "Sheet"]}[Data]
```

### If Your Row Markers Are Different

**Update the filter:**
```m
// ORIGINAL (looking for "Load:" and "RG:"):
each [Column16] = "Load:" and [Column1] = "RG:"

// IF your markers are different:
each [Column16] = "Your Marker:" and [Column1] = "Your Row Marker"
```

---

## Troubleshooting

### Error: "File not found"

**Cause:** FileName parameter doesn't match actual file name

**Solution:** 
```
Check exact file name in SharePoint
Note exact spelling (case-sensitive)
Pass correct name to LoadWFile function
```

### Error: "Column does not exist"

**Cause:** Selected columns are wrong

**Solution:**
1. Open source file manually
2. Count actual column positions
3. Update Table.SelectColumns to use correct columns

### Error: "No rows returned"

**Cause:** Row markers ("Load:", "RG:") don't match your data

**Solution:**
1. Open source file
2. Check what text appears in Column16
3. Update filter criteria

### Error: "Week column is blank"

**Cause:** WeekList is empty (EndWeek < StartWeek)

**Solution:**
```
Check function call parameters:
W0_to_W6 = LoadWFile("W1 to W6.xlsx", 0, 5)
                                        ↑   ↑
                                    Start End
Make sure EndWeek > StartWeek
```

---

## Performance Tips

### Reduce Query Load Time

**Problem:** Function takes too long to run

**Solution:** Call function only for weeks you need
```m
// Load only recent weeks (W77-W84)
W77_to_W84 = LoadWFile("W77 to W84.xlsx", 77, 84)

// Don't call all 12 batches if you only need last 8 weeks
```

### Refresh Scheduling

```
Fabric Dataflow Gen2 can be scheduled to refresh:
- Daily
- Weekly (recommended for shop load reports)
- On-demand
```

---

## Summary

The `LoadWFile` function is powerful because it:

✅ **Reusable:** One definition, 12+ calls  
✅ **Parameterized:** StartWeek/EndWeek are flexible  
✅ **Automated:** No manual consolidation  
✅ **Scalable:** Add more weeks without code changes  
✅ **Maintainable:** Fix logic once, applies everywhere  

**Result:** 95% code reduction vs. 84 separate queries.

---

**Need help?** Check the troubleshooting section or review specific steps above.
