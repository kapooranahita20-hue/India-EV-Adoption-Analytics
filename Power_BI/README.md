# Power BI - what is in this folder and how to finish the correction

## The short version

`EV_India_Analytics.pbix` in this folder is the **original** file. It still
contains the three defects listed below, because a `.pbix` is a binary package
whose semantic model is compressed inside an Analysis Services backup (XPress9
compression) - that part cannot be rewritten outside Power BI Desktop itself.

Everything needed to correct it **is** in this project:

| File | What it gives you |
|---|---|
| `../powerbi/DAX_Measures.md` | the corrected measure library, with the values each measure must return |
| `../powerbi/data_model.md` | the recommended star schema, relationships and data types |
| `../powerbi/rebuild_guide.md` | page-by-page rebuild instructions with expected results |
| `../powerbi/project/Model/**` | the **corrected semantic model** as TMDL (text) - 6 tables, 26 measures, 3 relationships, built from the validated data |
| `../powerbi/project/Report/definition/**` | the **corrected report** as PBIR (text) - 5 pages, 33 visuals |
| `../powerbi/EV_India_Analytics_v2.pbip` | the project (model + report) ready to open in Power BI Desktop |
| `../outputs/dashboard/ev_india_dashboard.html` | the corrected dashboard, working today, no Power BI needed |

## The three defects, with the exact fix

### 1. The penetration measure divides by India's national population

In the model:

```dax
measure 'EV Registrations per 1000 People' =
    DIVIDE(
        SUM('EV_State_Analysis'[EV_Registration_BOV]),
        [Total Population 2023],       -- <-- national population, for every state
        0
    ) * 1000
```

Live DAX against the file confirms it:

```
[EV Registrations per 1000 People] for Uttar Pradesh = 0.723494164242605
```

which is exactly the number the dashboard published. The correct value is
**4.2613** (`1,004,330 / 235,687,000 x 1,000`).

**Fix in Desktop:** replace the measure with

```dax
EV per 1,000 Population =
    DIVIDE ( [Total EV Registrations], [Total Population] ) * 1000
```

and point the page-4 visual at it. The file already contains a correct
measure, `EV per 1000 Corrected` - it was simply never used by any visual.

### 2. `Dim_State` has 38 rows for 36 territories, and the fact tables carry a junk row

`Dim_State` is a calculated table built as a `UNION` of `ev_clean`,
`Income_Data` and `Total_Clean`, which spell the same territory three
different ways:

| Spelling | Present in |
|---|---|
| `Andaman & Nicobar Island` | ev_clean, Total_Clean |
| `Andaman & Nicobar Islands` | Income_Data, Sheet1 (population) |
| `Jammu and Kashmir` | ev_clean, Total_Clean |
| `Jammu & Kashmir` | Income_Data, Sheet1 (population) |

So `Dim_State` resolves to 38 distinct labels, the population table only
matches 36 of them, and the state-name drift silently drops Andaman & Nicobar
Islands, Jammu & Kashmir and DNH & DD from every charging visual (33 of 36
states joined). The fact tables also keep the workbook's trailing row, which
is why `DISTINCTCOUNT(State)` returns **37** on the KPI card.

**Fix in Desktop:** replace the `Dim_State` calculated table with the
canonical dimension in `../powerbi/project/Model/tables/Dim_State.tmdl` -
one row per territory, keyed on `State_Code` - and repoint the relationships
to `State_Code` instead of the free-text name.

### 3. Two contradictory forecasts

The notebook published a rising linear trend (1,238,387 to 1,448,530 by
FY2030-31) while the dashboard table published a falling series (1,235,853
to 1,061,051) for the same years. Both fitted through the part-year
observation FY2025-26 (727,648 registrations = 42% of the FY2023-24 peak).

**Fix:** the corrected model carries `Fact_Annual` (actuals, with
`Is_Complete_Year` marking the part year) and `Fact_Forecast` (the
Low/Base/High band), and the forecast page reads from those.

## The 10-minute route

1. Open `EV_India_Analytics.pbix` in Power BI Desktop.
2. In **Home > Enter data** / **Model view**, delete the old tables
   (`ev_clean`, `Total_Clean`, `Sheet1`, `Income_Data`,
   `EV_Charging_Stations`, `India_EV_Forecast_2030 (1)`,
   `EV_Forecast_Combined`, `EV_State_Analysis`).
3. Use **TMDL view** (Model view > TMDL view) and paste
   `../powerbi/project/Model/_full_model.tmdl`, then
   **Update model with changes**. The tables are DAX calculated tables, so
   there is no data source to configure and no refresh to run.
4. Use **Report > Transform file** / the PBIR folder: replace
   `Report/definition/**` with `../powerbi/project/Report/definition/**`.
   (In Desktop 2.159 you can also open `../powerbi/EV_India_Analytics_v2.pbip`
   directly - it contains both the model and the report.)
5. Verify with the DAX query view:
   `EVALUATE ROW("UP", CALCULATE([EV per 1,000 Population], 'Dim_State'[State] = "Uttar Pradesh"))`
   must return **4.2613**, and `COUNTROWS('Dim_State')` must return **36**.

## Why the .pbix could not be rewritten automatically

* The semantic model lives in a compressed Analysis Services backup inside the
  package; it can only be produced by the engine itself.
* Editing any part of the package invalidates the `SecurityBindings`
  signature, so a patched file has to be re-signed through Power BI's own
  packager (`Microsoft.PowerBI.Packaging.PowerBIPackager.Save`) - which is
  what `../tools/rewrite_package.ps1` does. That part works (see
  `../tools/rewrite_package.ps1`), but the model still has to come from the
  engine.
* `pbi-tools` 1.2.0 (the only release) calls the pre-2.159
  `PowerBIPackager.Save` overload and fails with `MissingMethodException`.

The corrected numbers are already published and verifiable in
`../outputs/dashboard/ev_india_dashboard.html` and
`../outputs/report/EV_India_Analytics_Report.pdf`, so the project is
presentable regardless of the `.pbix` binary.
