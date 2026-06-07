# RCCP Capacity Planning: Complete Setup Instructions

## Overview

This guide walks you through implementing the RCCP Capacity Planning Lakehouse solution in your environment.

**Time Estimate:** 2-3 hours  
**Prerequisites:** Microsoft Fabric, Power BI, SharePoint access

---

## Part 1: Prepare Your Data

### Step 1.1: Organize SharePoint Folder

**Goal:** Have all your weekly shop load report files in one SharePoint folder

**Action:**
1. Go to your SharePoint site
2. Create folder (if not exists): `/Shared Documents/RCCP/`
3. Upload all W files:
   - W1_to_W6.xlsx
   - W7_to_W12.xlsx
   - W13_to_W18.xlsx
   - ... (continue for all batches)
   - W77_to_W84.xlsx

**Verify:**
- All files in same folder
- Consistent naming (use .xlsx or .csv, not mixed)
- Each file has "ShopLoadReport" sheet

### Step 1.2: Verify File Structure

**For each Excel file:**
1. Open W1_to_W6.xlsx
2. Look at ShopLoadReport sheet
3. Verify you have:
   - Column with Resource Group ID (e.g., "MOLD-01")
   - Column with Resource Group Description (e.g., "Mold Processing")
   - Column with Load Hours (e.g., "156.5")

**Note column positions:**
- If Resource_Grp is in Column4 ✓
- If Resource_Grp_Desc is in Column11 ✓
- If Load_Hours is in Column22 ✓
- If positions differ, you'll need to update the function (see POWER_QUERY_IMPLEMENTATION.md)

---

## Part 2: Create Fabric Workspace & Lakehouse

### Step 2.1: Create Workspace

1. Open **Fabric** (fabric.microsoft.com)
2. Click **+ New Workspace**
3. Name: `RCCP_Analytics`
4. Description: `RCCP Capacity Planning Lakehouse`
5. Click **Create**

### Step 2.2: Create Lakehouse

1. In your workspace, click **+ New**
2. Select **Lakehouse**
3. Name: `RCCP_Analytics`
4. Click **Create**

**Your Lakehouse is now ready.**

---

## Part 3: Create Dataflow Gen2

### Step 3.1: Create Dataflow

1. In workspace, click **+ New**
2. Select **Dataflow Gen2**
3. Name: `Load_RCCP_Data`
4. Click **Create**

### Step 3.2: Add SharePoint Source

1. In Dataflow editor, click **Get data**
2. Select **SharePoint folder**
3. Paste your SharePoint URL:
   ```
   https://yourcompany.sharepoint.com/sites/manufacturing/Shared Documents/RCCP/
   ```
4. Click **Next**
5. Select your RCCP folder
6. Click **Load**

**You now have a list of all files in the folder.**

---

## Part 4: Create the LoadWFile Function

### Step 4.1: Create Function Query

1. In Power Query editor, click **New source** (bottom left)
2. Select **Blank query**
3. Name the query: `LoadWFile`

### Step 4.2: Paste Function Code

1. In the formula bar, paste the function code (from power-query/LoadWFile.m)
2. **BEFORE SAVING:** Update these placeholders:

**Find this:**
```m
Source = SharePoint.Files("https://#######################/", [ApiVersion = 15]),
```

**Replace with your URL:**
```m
Source = SharePoint.Files("https://yourcompany.sharepoint.com/sites/manufacturing/", [ApiVersion = 15]),
```

**Find this:**
```m
Navigation = #"Filtered rows"{[Name = FileName, #"Folder Path" = "https://##############################################/"]}[Content],
```

**Replace with your folder path:**
```m
Navigation = #"Filtered rows"{[Name = FileName, #"Folder Path" = "https://yourcompany.sharepoint.com/sites/manufacturing/Shared Documents/RCCP/"]}[Content],
```

### Step 4.3: Verify Function Syntax

1. Click outside the formula bar
2. Check for red errors
3. If no errors, you're good to go

---

## Part 5: Create Function Calls

### Step 5.1: Create First Query (W0_to_W6)

1. Click **New source** → **Blank query**
2. Name: `W0_to_W6`
3. Paste:
```m
LoadWFile("W1 to W6.xlsx", 0, 5)
```
4. Press Enter

**Result:** Should load and display data with columns: Resource_Grp, Resource_Grp_Desc, Load_Hours, Week

### Step 5.2: Create Remaining Queries

Repeat for each batch:

```m
// Weeks 7-12
W7_to_W12 = LoadWFile("W7 to W12.xlsx", 6, 11)

// Weeks 13-18
W13_to_W18 = LoadWFile("W13 to W18.xlsx", 12, 17)

// Weeks 19-24 (if you have this file)
W19_to_W24 = LoadWFile("W19 to W24.xlsx", 18, 23)

// ... continue for all batches
```

**Note:** Update file names to match your actual file names in SharePoint

### Step 5.3: Verify Each Query

For each query, check:
- ✓ Rows returned (should be ~300-500 per batch, depending on machines)
- ✓ Columns: Resource_Grp, Resource_Grp_Desc, Load_Hours, Week
- ✓ Week column: W0, W1, W2, etc.
- ✓ Load_Hours: Numbers, not null

---

## Part 6: Combine Results

### Step 6.1: Create Combine Query

1. Click **New source** → **Blank query**
2. Name: `fact_Load_Hours`
3. Paste:

```m
Table.Combine({
    W0_to_W6,
    W7_to_W12,
    W13_to_W18,
    // Add all other queries here
})
```

**Include all your W queries in the list.**

### Step 6.2: Verify Combined Data

Check:
- ✓ Total rows = sum of all W queries (should be ~840)
- ✓ No duplicate rows
- ✓ All 4 columns present
- ✓ Week values span W0-W84 (or your range)

---

## Part 7: Load to Lakehouse

### Step 7.1: Configure Load Settings

1. Select `fact_Load_Hours` query (right-click)
2. Click **Configure load settings**
3. Select: **Load to new table**
4. Table name: `fact_Load_Hours`
5. Lakehouse: `RCCP_Analytics` (created in Step 2.2)
6. Click **Next**

### Step 7.2: Configure Columns

1. Verify column types:
   - Resource_Grp: Text
   - Resource_Grp_Desc: Text
   - Load_Hours: Decimal
   - Week: Text
2. Click **Create**

### Step 7.3: Publish Dataflow

1. Click **Publish** (top right)
2. Wait for dataflow to publish (should take 2-5 minutes)
3. Check **Success** notification

---

## Part 8: Verify Data in Lakehouse

### Step 8.1: View Table

1. Go to Lakehouse: `RCCP_Analytics`
2. Under **Tables**, click `fact_Load_Hours`
3. Preview data

**Expected:**
```
Resource_Grp    Resource_Grp_Desc    Load_Hours    Week
MOLD-01         Mold Processing      156.5         W0
MOLD-01         Mold Processing      156.5         W1
MOLD-01         Mold Processing      156.5         W2
... (continues for all weeks)
LOGOCELL-A      Logo Cell A          89.2          W0
LOGOCELL-A      Logo Cell A          89.2          W1
```

### Step 8.2: Validate Data Quality

**Check:**
- ✓ 840 rows (110 machines × ~7.6 weeks)
- ✓ No null values in key columns
- ✓ Load_Hours are numbers (not text)
- ✓ Week column has format W0-W84
- ✓ No duplicate rows (same machine + week combination appears once)

---

## Part 9: Create Power BI Connection

### Step 9.1: Open Power BI Desktop

1. Launch **Power BI Desktop**
2. Click **Get data** → **Lakehouse**
3. Sign in with your Microsoft account
4. Select workspace: `RCCP_Analytics`
5. Select lakehouse: `RCCP_Analytics`
6. Click **Load**

### Step 9.2: Select Tables

1. In Navigator, select:
   - ✓ `fact_Load_Hours`
   - Optional: Other dimension tables (if you have them)
2. Click **Load**

**Your data is now in Power BI.**

---

## Part 10: Create Dashboard

### Step 10.1: Create Measures

In Power BI, create DAX measures:

```dax
// Total Load
Total Load = SUM('fact_Load_Hours'[Load_Hours])

// Past Due (example: if Load > Capacity)
Past Due = 
  VAR TotalLoad = SUM('fact_Load_Hours'[Load_Hours])
  VAR TotalCapacity = [Total Capacity] // from your capacity table
  RETURN IF(TotalLoad > TotalCapacity, TotalLoad - TotalCapacity, 0)

// C/D Ratio (Capacity/Demand)
C/D Ratio = 
  DIVIDE(
    [Total Capacity],
    [Total Load],
    0)

// Count of Resource Groups
Resource Group Count = DISTINCTCOUNT('fact_Load_Hours'[Resource_Grp])
```

### Step 10.2: Create Visualizations

**KPI Cards:**
- Past Due Hours
- Total Load Hours
- # of Resource Groups
- Latest Load Week

**Charts:**
- Combo chart: Total Load (orange) vs Total Capacity (blue) by week
- Bar chart: Past Due by Resource Group
- Matrix: Weekly detail (Week × Metrics)

**Filters:**
- Department slicer
- Work Center slicer
- Week range slicer

### Step 10.3: Reference Design

See dashboard screenshot in assets/ folder for layout example.

---

## Part 11: Set Up Refresh Schedule

### Step 11.1: Schedule Dataflow Refresh

1. Go to Fabric workspace
2. Right-click dataflow: `Load_RCCP_Data`
3. Click **Settings**
4. Go to **Scheduled refresh**
5. Toggle: **On**
6. Frequency: **Weekly** (recommended)
7. Day: Choose day after you get new data
8. Time: Choose time (e.g., 2 AM)
9. Save

### Step 11.2: Schedule Power BI Refresh

1. In Power BI Service (app.powerbi.com)
2. Go to your workspace
3. Select dataset: `RCCP_Analytics`
4. Settings → **Scheduled refresh**
5. Frequency: **Daily** (to refresh from Lakehouse)
6. Time: 1 hour after Dataflow refresh
7. Save

**Result:** Your dashboard updates automatically!

---

## Troubleshooting

### Problem: "File not found" Error

**Cause:** File name doesn't match SharePoint exactly

**Solution:**
1. Go to SharePoint folder
2. Check exact file name (with spaces, capitalization)
3. Update function call:
   ```m
   LoadWFile("Exact File Name.xlsx", 0, 5)
   ```

### Problem: "Column does not exist" Error

**Cause:** Column positions are different in your files

**Solution:**
1. Open W1_to_W6.xlsx manually
2. Count columns: 1, 2, 3, 4, 5, ...
3. Identify which column has Load_Hours
4. Update this line in LoadWFile function:
   ```m
   // If Load_Hours is in Column25 (not 22):
   {"Column1", "Column4", "Column11", "Column16", "Column25"}
   ```

### Problem: No Data Loaded

**Cause:** Row markers don't match your data

**Solution:**
1. Open W1_to_W6.xlsx manually
2. Find a row with load hours
3. Check what text appears in Column16 (should be "Load:")
4. Check what text appears in Column1 (should be "RG:")
5. If different, update filter:
   ```m
   each [Column16] = "Your Text:" and [Column1] = "Your Marker:"
   ```

### Problem: Dataflow Takes Too Long

**Cause:** Loading all 12 files with full unpivoted dataset

**Solution:**
1. Call function only for recent weeks:
   ```m
   // Instead of all 12 files (W1-W84), load only last batch:
   W77_to_W84 = LoadWFile("W77 to W84.xlsx", 77, 84)
   ```
2. Or split into separate dataflows:
   - Dataflow 1: W0-W27 (historical)
   - Dataflow 2: W28-W55 (recent)
   - Dataflow 3: W56-W84 (current)

### Problem: Dashboard Slow to Load

**Cause:** Too much data in single visualization

**Solution:**
1. Limit weeks shown by default:
   - Add filter: Last 26 weeks only
   - Keep detail table to top 20 resources
2. Use aggregations in Power BI

---

## Success Checklist

**Before declaring "Done", verify:**

- [ ] Lakehouse created: `RCCP_Analytics`
- [ ] Dataflow created: `Load_RCCP_Data`
- [ ] Function created: `LoadWFile`
- [ ] All W queries created (W0_to_W6, W7_to_W12, etc.)
- [ ] Combined query created: `fact_Load_Hours`
- [ ] Data loaded to Lakehouse (840 rows)
- [ ] Power BI connected to Lakehouse
- [ ] Dashboard created with key measures
- [ ] Refresh schedule configured
- [ ] Dashboard working correctly

---

## Next Steps

1. **Create additional dimensions** (if needed):
   - Planned Hours table (for capacity)
   - Machine Master (for attributes)
   - Department/Work Center mappings

2. **Add SQL queries** for:
   - Data quality validation
   - Historical trending
   - Anomaly detection

3. **Expand dashboard** with:
   - Predictive capacity forecasting
   - Multi-week trend analysis
   - Resource utilization trends

---

## Support

**Questions about specific steps?**
- See [POWER_QUERY_IMPLEMENTATION.md](POWER_QUERY_IMPLEMENTATION.md) for function details
- See [ARCHITECTURE.md](../docs/ARCHITECTURE.md) for system design
- Check Troubleshooting section above

**Need to modify the function?**
- Column positions: [POWER_QUERY_IMPLEMENTATION.md](POWER_QUERY_IMPLEMENTATION.md#if-your-column-positions-are-different)
- Row markers: [POWER_QUERY_IMPLEMENTATION.md](POWER_QUERY_IMPLEMENTATION.md#if-your-row-markers-are-different)
- Sheet names: [POWER_QUERY_IMPLEMENTATION.md](POWER_QUERY_IMPLEMENTATION.md#if-your-sheet-name-is-different)

---

**Estimated time to complete:** 2-3 hours  
**Difficulty level:** Intermediate (requires Fabric + Power Query knowledge)  
**Value:** 60% reduction in report generation time

Good luck! 🚀
