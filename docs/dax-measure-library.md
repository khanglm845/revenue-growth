# Retail Revenue Growth & Profitability Diagnostic

## DAX Formula Library

**Project:** Personal Power BI project — Starttrain, The Next Analyst Challenge Season 1  
**Period:** 2016–2018 | **Rows:** 72,743 | **Tools:** Power BI, Power Query, DAX

> This document is a reviewed reference implementation reconstructed from the
> project logic and validated dashboard outputs. The original PBIX measure
> definitions were not exported. Before copying measures into the model, confirm
> the actual table/column names and validate every checkpoint against the final
> PBIX. The PVM section below defines an explicit, internally consistent bridge
> convention so that the interpretation is reproducible and defensible.

## 0. Model assumptions

| Table | Expected fields / role |
| --- | --- |
| `Sales` | Fact: `Date`, `Quantity`, `Price`, `Cost per Unit`, `ProductKey`, `Order method type`, `Retailer country` |
| `Calendar` | Date dimension: `Date`, `Year` |
| `DimProduct` | `ProductKey`, `Product line`, `Product type` |
| `Region` | `Country`, `Region` |
| `SalesManager` | Manager / territory dimension |
| `Growth Driver` | Disconnected field parameter |
| `Revenue Bridge Steps` | Disconnected waterfall axis |
| `Comparison Period` | Disconnected period selector |

Assume one-to-many, single-direction relationships from dimensions to `Sales`.
Mark `Calendar` as the date table.

---

## 1. Base measures

### Revenue (if Price is unit selling price)

```dax
Revenue =
SUMX ( Sales, Sales[Quantity] * Sales[Price] )
```

If the source already contains extended transaction revenue, use
`SUM(Sales[Revenue])` instead.

```dax
Quantity =
SUM ( Sales[Quantity] )
```

```dax
COGS =
SUMX ( Sales, Sales[Quantity] * Sales[Cost per Unit] )
```

```dax
Gross Profit =
[Revenue] - [COGS]
```

```dax
Gross Margin % =
DIVIDE ( [Gross Profit], [Revenue] )
```

```dax
ASP =
DIVIDE ( [Revenue], [Quantity] )
```

```dax
Cost per Unit =
DIVIDE ( [COGS], [Quantity] )
```

---

## 2. Time intelligence

```dax
Revenue LY =
CALCULATE ( [Revenue], DATEADD ( 'Calendar'[Date], -1, YEAR ) )
```

```dax
Quantity LY =
CALCULATE ( [Quantity], DATEADD ( 'Calendar'[Date], -1, YEAR ) )
```

```dax
Gross Profit LY =
CALCULATE ( [Gross Profit], DATEADD ( 'Calendar'[Date], -1, YEAR ) )
```

```dax
ASP LY =
DIVIDE ( [Revenue LY], [Quantity LY] )
```

```dax
Gross Margin LY =
DIVIDE ( [Gross Profit LY], [Revenue LY] )
```

```dax
Revenue Change = [Revenue] - [Revenue LY]
Revenue YoY % = DIVIDE ( [Revenue Change], [Revenue LY] )
Quantity Change = [Quantity] - [Quantity LY]
Quantity YoY % = DIVIDE ( [Quantity Change], [Quantity LY] )
Gross Profit Change = [Gross Profit] - [Gross Profit LY]
Gross Profit YoY % = DIVIDE ( [Gross Profit Change], [Gross Profit LY] )
ASP Change = [ASP] - [ASP LY]
ASP YoY % = DIVIDE ( [ASP Change], [ASP LY] )
Gross Margin Change pp = [Gross Margin %] - [Gross Margin LY]
```

`Gross Margin Change pp` stores a decimal difference. For example,
0.0007 = 0.07 percentage points. Format or create a display measure
accordingly.

```dax
Selected Year =
SELECTEDVALUE ( 'Calendar'[Year], MAX ( 'Calendar'[Year] ) )
```

---

## 3. Revenue Price--Volume--Mix (PVM)

### Convention

At product-type grain, let prior/current quantity be Q0/Q1 and
prior/current ASP be P0/P1. The bridge uses a fixed prior-year portfolio
ASP as the volume baseline and then separates within-product ASP movement
from the remaining product-mix / assortment effect.

-   **Volume effect:** quantity change valued at the prior-year portfolio ASP.
    The product filters are removed from the baseline ASP so that the volume
    component remains additive when the bridge is broken down by Product Line
    or Product Type. Non-product filters such as year, country, or channel remain
    active.
-   **Within-product ASP effect:** product-type ASP movement weighted by the
    midpoint of prior/current quantity.
-   **Product mix / assortment effect:** the residual required to reconcile to
    actual revenue change. This residual can include changes in relative product
    weights and the effect of product types appearing/disappearing between periods.

Keep the product grain consistent across all components.

```dax
PVM Baseline ASP LY =
CALCULATE (
    [ASP LY],
    REMOVEFILTERS ( 'DimProduct' )
)
```

```dax
Revenue Volume Effect =
( [Quantity] - [Quantity LY] ) * [PVM Baseline ASP LY]
```

```dax
Revenue Within-Product ASP Effect =
SUMX (
    VALUES ( 'DimProduct'[Product type] ),
    VAR Q0 = CALCULATE ( [Quantity LY] )
    VAR Q1 = CALCULATE ( [Quantity] )
    VAR P0 = CALCULATE ( [ASP LY] )
    VAR P1 = CALCULATE ( [ASP] )
    RETURN
        IF (
            NOT ISBLANK ( P0 ) && NOT ISBLANK ( P1 ),
            ( P1 - P0 ) * DIVIDE ( Q0 + Q1, 2 )
        )
)
```

```dax
Revenue Product Mix Effect =
[Revenue Change]
    - [Revenue Volume Effect]
    - [Revenue Within-Product ASP Effect]
```

```dax
Revenue PVM Residual =
[Revenue Change]
    - [Revenue Volume Effect]
    - [Revenue Within-Product ASP Effect]
    - [Revenue Product Mix Effect]
```

Expected PVM residual: 0, apart from negligible floating-point rounding.

> **Interpretation note:** because the mix term is defined as the residual after
> the fixed-baseline volume effect and midpoint within-product ASP effect, a zero
> residual confirms the bridge arithmetic. It does not prove that the selected
> decomposition is the only valid economic attribution.

### ASP component checks

```dax
ASP Within-Product Effect =
DIVIDE ( [Revenue Within-Product ASP Effect], [Quantity] )
```

```dax
ASP Product Mix Effect =
DIVIDE ( [Revenue Product Mix Effect], [Quantity] )
```

```dax
ASP Decomposition Residual =
[ASP Change]
    - [ASP Within-Product Effect]
    - [ASP Product Mix Effect]
```

At the **full product-portfolio context**, where `PVM Baseline ASP LY` equals
`[ASP LY]`, the two ASP components reconcile algebraically to `[ASP Change]`
when divided by current-period quantity:

`ASP Change = ASP Within-Product Effect + ASP Product Mix Effect`

Do not assume that identity will hold inside a Product Line / Product Type row,
because the revenue PVM intentionally uses a fixed portfolio baseline ASP to
preserve additivity across the product hierarchy. Use these ASP component
measures for portfolio-level diagnostics unless a separate local ASP
decomposition is implemented. The mix component remains a residual mix /
assortment effect rather than a causal estimate.

---

## 4. Waterfall bridge

Create a disconnected `Revenue Bridge Steps` table with `Step` and
`Sort Order`:

| Step | Sort Order |
| --- | ---: |
| Revenue LY | 1 |
| Volume | 2 |
| Within-Product ASP | 3 |
| Product Mix | 4 |
| Revenue CY | 5 |

```dax
Revenue Bridge Value =
SWITCH (
    SELECTEDVALUE ( 'Revenue Bridge Steps'[Step] ),
    "Revenue LY", [Revenue LY],
    "Volume", [Revenue Volume Effect],
    "Within-Product ASP", [Revenue Within-Product ASP Effect],
    "Product Mix", [Revenue Product Mix Effect],
    "Revenue CY", [Revenue],
    BLANK ()
)
```

Visual setup: Category = `Step`, Y-axis = `[Revenue Bridge Value]`; sort
by `Sort Order`. Mark Revenue LY and Revenue CY as totals if the native
Waterfall visual exposes **Set as total**.

---

## 5. Product contribution and mix

```dax
Growth Contribution % =
DIVIDE (
    [Revenue Change],
    CALCULATE (
        [Revenue Change],
        REMOVEFILTERS ( 'DimProduct' )
    )
)
```

```dax
Product Revenue Share % =
DIVIDE (
    [Revenue],
    CALCULATE ( [Revenue], REMOVEFILTERS ( 'DimProduct' ) )
)
```

Both measures above use the full product portfolio as the denominator while
preserving non-product filters. If a visual instead needs contribution within a
selected subset of products, create a separate `ALLSELECTED` version rather than
changing the company-level measure.

```dax
Quantity Mix =
DIVIDE (
    [Quantity],
    CALCULATE (
        [Quantity],
        REMOVEFILTERS ( 'DimProduct'[Product type] )
    )
)
```

```dax
Quantity Mix LY =
DIVIDE (
    [Quantity LY],
    CALCULATE (
        [Quantity LY],
        REMOVEFILTERS ( 'DimProduct'[Product type] )
    )
)
```

```dax
Quantity Mix Shift pp =
[Quantity Mix] - [Quantity Mix LY]
```

For product-type mix *within a selected product line*, remove Product
Type but preserve Product Line. For portfolio-wide mix, remove both
levels as appropriate.

```dax
Mix Offset % (Display) =
IF (
    [Revenue Volume Effect] > 0
        && [Revenue Product Mix Effect] < 0,
    DIVIDE (
        ABS ( [Revenue Product Mix Effect] ),
        [Revenue Volume Effect]
    )
)
```

Format as percentage (e.g. 0.512 → 51.2%).

### Product Line selection guard

```dax
Show Product Type Diagnostic =
IF ( HASONEVALUE ( 'DimProduct'[Product line] ), 1, 0 )
```

Apply as a visual-level filter = 1 to Product Type diagnostics; do not
apply it to the portfolio scatter.

```dax
Product Diagnostic Title =
IF (
    HASONEVALUE ( 'DimProduct'[Product line] ),
    "Product Type Mix Diagnostics | "
        & SELECTEDVALUE ( 'DimProduct'[Product line] ),
    "Select a Product Line to inspect mix drivers"
)
```

---

## 6. Field parameter: composite-key-safe label

Create the parameter using **Modeling → New parameter → Fields**. Power
BI-generated field parameters commonly contain display, fields, and
order columns that form a composite key. A direct `SELECTEDVALUE` on
only the display column may produce a composite-key error.

Replace the column names below with the exact generated names:

```dax
Selected Growth Driver =
VAR ParameterRows =
    SUMMARIZE (
        'Growth Driver',
        'Growth Driver'[Growth Driver],
        'Growth Driver'[Growth Driver Fields],
        'Growth Driver'[Growth Driver Order]
    )
RETURN
    CONCATENATEX (
        ParameterRows,
        'Growth Driver'[Growth Driver],
        ", "
    )
```

```dax
Growth Driver Chart Title =
"Incremental Revenue by "
    & COALESCE ( [Selected Growth Driver], "Selected Dimension" )
```

Use the actual field parameter in the visual's axis/legend; the title
measure only changes the title text.

---

## 7. Market and channel

```dax
Country Growth Contribution % =
DIVIDE (
    [Revenue Change],
    CALCULATE (
        [Revenue Change],
        REMOVEFILTERS ( 'Region'[Country] )
    )
)
```

```dax
Channel Revenue Share % =
DIVIDE (
    [Revenue],
    CALCULATE (
        [Revenue],
        REMOVEFILTERS ( Sales[Order method type] )
    )
)
```

```dax
Channel Revenue Share LY % =
DIVIDE (
    [Revenue LY],
    CALCULATE (
        [Revenue LY],
        REMOVEFILTERS ( Sales[Order method type] )
    )
)
```

```dax
Channel Mix Shift pp =
[Channel Revenue Share %] - [Channel Revenue Share LY %]
```

```dax
Channel Growth Contribution % =
DIVIDE (
    [Revenue Change],
    CALCULATE (
        [Revenue Change],
        REMOVEFILTERS ( Sales[Order method type] )
    )
)
```

Negative channel contributions are valid. Positive channels can
individually exceed 100% when other channels decline; total
contributions should reconcile to approximately 100%.

```dax
Top Growth Market =
VAR T =
    TOPN (
        1,
        ADDCOLUMNS (
            ALLSELECTED ( 'Region'[Country] ),
            "__Change", CALCULATE ( [Revenue Change] )
        ),
        [__Change], DESC,
        'Region'[Country], ASC
    )
RETURN
    CONCATENATEX ( T, 'Region'[Country], ", " )
```

---

## 8. Dynamic management text

```dax
Growth Quality Insight =
"Units grew "
    & FORMAT ( [Quantity YoY %], "0.0%" )
    & ", faster than revenue at "
    & FORMAT ( [Revenue YoY %], "0.0%" )
    & ", while ASP changed "
    & FORMAT ( [ASP YoY %], "0.0%" )
    & "."
```

```dax
Mix Share of ASP Decline % =
VAR Residual = [ASP Decomposition Residual]
RETURN
    IF (
        [ASP Change] < 0
            && ABS ( Residual ) < 0.0001,
        DIVIDE (
            ABS ( [ASP Product Mix Effect] ),
            ABS ( [ASP Change] )
        )
    )
```

```dax
Mix Insight =
VAR MixShare = [Mix Share of ASP Decline %]
RETURN
    IF (
        NOT ISBLANK ( MixShare ),
        FORMAT ( MixShare, "0%" )
            & " of the ASP decline is attributable to product-mix effects."
    )
```

Do not cap `Mix Share of ASP Decline %` at 100%. It can legitimately exceed
100% when the mix effect and within-product ASP effect move in opposite
directions.

---

## 9. QA measures

```dax
Revenue PVM QA Status =
IF ( ABS ( [Revenue PVM Residual] ) < 0.01, "PASS", "CHECK" )
```

```dax
ASP Decomposition QA Status =
IF (
    ABS ( [ASP Decomposition Residual] ) < 0.0001,
    "PASS",
    "CHECK"
)
```

```dax
Gross Profit Definition Check =
[Gross Profit] - ( [Revenue] - [COGS] )
```

`Gross Profit Definition Check` is expected to be 0 by construction. It is a
definition sanity check, not an independent source reconciliation. If the source
contains an authoritative gross-profit field, add a separate reconciliation
measure against that source field.

---

## 10. Visual setup quick reference

| Page | Visual | Main fields / measures | Setup note |
| --- | --- | --- | --- |
| 1 Executive | KPI cards | Revenue, Quantity, GP, GM%, ASP | Secondary labels show YoY / pp |
| 1 Executive | Waterfall | Bridge Step + `[Revenue Bridge Value]` | Sort by order; mark LY/CY totals |
| 1 Executive | PVM by product line | Product Line + Volume / Within ASP / Mix | Components use the fixed portfolio baseline and should add back to the company bridge |
| 2 Growth Drivers | Dynamic bar | Growth Driver parameter + `[Revenue Change]` | Include negative contributors |
| 2 Growth Drivers | Decomposition Tree | Analyze `[Revenue Change]`; explain by dimensions | Drill from total to segment |
| 3 Portfolio | Scatter | X: Growth Contribution; Y: Mix impact; Details: Product Line | Percentage axes, zero reference |
| 3 Portfolio | Product diagnostics | Product Type, Quantity Mix, Mix Shift, ASP | Filter `[Show Product Type Diagnostic] = 1` |
| 4 Market | Country scatter | X: Revenue YoY; Y: Gross Margin | Percentage axes; company margin reference |
| 4 Market | Channel mix | Order Method + prior/current share | 100% stacked bar, consistent colors |

---

## 11. Formatting

| Measure | Format |
| --- | --- |
| Revenue / GP / PVM effects | `$#,0.0,,"M";($#,0.0,,"M");-` |
| Quantity | `#,0.00,,"M"` |
| ASP / unit cost | `$#,0.00` |
| YoY | `+0.0%;-0.0%;0.0%` |
| Gross Margin | `0.00%` |
| Mix Offset | `0.0%` |
| Mix Shift / Margin Change | Display in percentage points (`pp`) |

For percentage-point display, multiply the underlying decimal difference by 100
in a display measure and append `" pp"`; do not apply percentage formatting to
that multiplied value.

---

## 12. 2018 validation checkpoints

Values were transcribed from the supplied project outputs. Confirm the exact
figures against the final PBIX before publication.

| Metric | Checkpoint |
| --- | ---: |
| Revenue | $424,420,140 |
| Revenue YoY | +17.4% |
| Revenue Change | +$62,832,570 |
| Quantity YoY | +20.9% |
| ASP YoY | -2.93% |
| Gross Profit | $179,298,821 |
| Gross Profit YoY | +17.6% |
| Gross Margin | 42.25% |
| Margin change | +0.07 pp |
| Volume Effect | +$74,527,338 |
| Within-Product ASP Effect | -$2,648,906 |
| Product Mix Effect | -$9,045,863 |
| PVM Residual | $0 |
| ASP Within-Product Effect | -$0.57 |
| ASP Product Mix Effect | -$1.95 |
| ASP Change | -$2.52 |
| Computer mix offset | 51.24% |
| Video Games + Mobile growth contribution | ~72.8% |
| Web revenue share | ~88.8% |
| Web net growth contribution | ~99.5% |

---

## 13. Technical review notes

The formulas in this library are designed to be internally consistent and
interview-defensible, but the file remains a reconstructed reference because the
original PBIX measure definitions were not exported. The 2018 checkpoints should
therefore be treated as acceptance tests for the final implementation.

## 14. Final implementation checklist

-   [ ] Confirm actual source column names and whether price/cost are
    unit-level.
-   [ ] Mark Calendar as the date table.
-   [ ] Verify one-to-many, single-direction dimension-to-fact relationships.
-   [ ] Test LY measures across 2017 and 2018.
-   [ ] Reconcile company-level PVM effects and portfolio-level ASP components independently.
-   [ ] Verify PVM component additivity across Product Line / Product Type views.
-   [ ] Confirm that `PVM Baseline ASP LY` removes product filters only and
    preserves intended non-product slicers.
-   [ ] Format ratios, percentage points, currency, and quantities
    correctly.
-   [ ] Hide Product Type diagnostics until one Product Line is
    selected.
-   [ ] Test field parameter selection without composite-key errors.
-   [ ] Check country/channel contribution totals.
-   [ ] Revalidate all 2018 checkpoints before publishing README/CV
    claims.

### Design cautions

1.  A zero PVM residual confirms arithmetic reconciliation, not causality.
2.  Keep the PVM grain consistent across measures. The fixed portfolio ASP
    baseline is what makes the volume allocation additive across product levels.
3.  `Revenue Product Mix Effect` is a residual mix / assortment effect. If a
    product type exists in only one period, its entry/exit impact is absorbed by
    this residual rather than by the within-product ASP effect.
4.  Do not describe aggregate ASP decline as pure price erosion.
5.  Use "net incremental revenue contribution" when channel contributions
    include negative channels.
6.  Manager performance is not normalized for targets, territory potential,
    or market difficulty.
