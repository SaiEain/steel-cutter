# Comprehensive Beam BBS & 1D Optimization Tool - System Instructions

**Context:**
Act as an expert Web Developer and Structural Detailer. Create a complete, single-page web app (HTML, CSS, JS) for a "Comprehensive Beam BBS, Detailing & 1D Cutting Optimization Tool" (ACI 318). Provide ALL code in ONE single file.

## Part 1: Global Settings & Beam Specifications
- Global material inputs: concrete compressive strength `f'c` in psi and reinforcing steel yield strength `fy` in psi.
- Tension lap splice length MUST be calculated automatically; do not ask the user to enter a design-approved lap length.
- The splice class is fixed as **ACI Class B** (no class selector or user override).
- Use the following data sources:
  - `f'c` and `fy`: Part 1, Global Settings & Material Properties.
  - Main reinforcing bar diameter `db`: each beam's `Main Bar Dia (mm)` in Part 1 (Cont), Beam Specifications (Table 1).
- Calculate tension development length with the simplified ACI expression, in consistent imperial units:
  - Convert Table 1 bar diameter from mm to inches: `db(in) = db(mm) / 25.4`.
  - For No. 6 and smaller bars (`db <= 0.75 in`): `ld = fy * db / (25 * sqrt(f'c))`.
  - For No. 7 and larger bars (`db > 0.75 in`): `ld = fy * db / (20 * sqrt(f'c))`.
  - Fixed Class B tension lap length: `lap = ceil(max(12 in, 1.3 * ld))`, rounded UP to the next whole inch.
- Use the calculated lap length for all splice placement, stock-length checks, cut-length/BBS output, and splice-zone fit checks for the corresponding bar diameter. Runs with different main bar diameters must use their own calculated lap lengths.
- Document the calculation assumptions in the UI and exported JSON: normal-weight concrete, uncoated bars, and modifier factors equal to 1.0. The current inputs do not capture epoxy coating, lightweight concrete, or other modification factors.
- Validate that `f'c`, `fy`, and each main bar diameter are positive before calculating.
- Table 1 (Beams): Beam Label, Width, Depth, Clear Cover, Main Bar Dia(mm), Stirrup Dia(mm), Top Sup/Mid Qty, Bot Sup/Mid Qty, Stirrup 2h Spacing, Stirrup Mid Spacing.
- Other global settings: "Override stirrup spacing at lap splices?" checkbox with custom spacing in inches; hook options (configured 90° allowance or custom inches). Round UP hook lengths to whole inches.
- Initial material defaults in the current app: `f'c = 3000 psi` and `fy = 50000 psi`; both remain editable and must be validated before calculation.
- CRITICAL: Initialize Table 1 with ZERO default rows. Add a "Load from CSV" button to parse a CSV file and auto-populate this table.

## Part 2: Main Grid Builder (ONLY METHOD)
- Build grids through a row-based Main Grid Builder; manual sequence building is STRICTLY FORBIDDEN.
- Inputs: Grid Name; Number of Spans; one editable row per span with Length in ft, Length in in, and a Beam selector populated from Table 1; and one positive Column Size (in) input for each column (span count + 1).
- Start with three spans and four columns. Support adding and deleting spans, while requiring at least one span. Keep the displayed column count synchronized with the span count and preserve existing values when the span count changes where possible.
- Validate each span as positive, with nonnegative feet and inches in `[0, 12)`. Every span must have a beam selected and its label must match a Table 1 beam. Require every column size to be positive.
- Buttons: "Generate Grid" (adds data to Table 2) and "Clear Wizard". Table 2 (Sequence List) contains generated grids as editable blocks with "Delete Grid" buttons.

## Part 3: BBS Calculation Logic (STRICT ACI ZONES & CONTINUITY)
- Loop through all grids in Table 2. Max cut piece = 472 inches (39' 4").
- For each continuous main-bar run, use the largest Table 1 main bar diameter in that run to control the continuing bars and calculate its Class B lap length from Part 1. Use that controlling diameter and lap length in every splice-zone and stock-length calculation.
- Top Continuous Bars: Continue through adjacent spans when Beam Width is identical, regardless of depth. Bottom Continuous Bars: Continue when both adjacent Width AND Depth are identical. If the required geometry differs, terminate the run with 90-degree hooks. In each geometrically continuous run, continue the minimum bar quantity present across the run; terminate surplus bars as hooked runs over only the spans that require them (for example, quantities 6-4-4 continue 4, and 4-3-3 continue 3). For differing bar diameters, use the largest diameter in the continuous run to control the continuing bars and their lap/cut calculations. Lap splices must land strictly between `Ln/4` and `3*Ln/4` for top bars, and between `left_face + 2h` and `right_face - 2h` for bottom bars.
- Use 3 in clear cover from a column's outside face to the end of any hooked main bar. Keep the beam clear-cover input for beam/stirrup cover calculations. Reject a grid with a column too narrow to provide the hooked-bar end cover. Assign a bar mark to each continuous cut piece and show the matching piece marks connected at every lap splice in the BBS and Excel output.
- Count each lap once: the next bar piece starts one lap length before the prior piece's end, so do not add the lap length again to the prior piece's cut length. Include the overlap in the 472 in stock-length check.
- Algorithm: Calculate span-by-span. Dynamically shorten cut lengths to force lap splices safely into valid ACI zones. NEVER cut at 472" if it lands in a prohibited zone. Calculate cutoffs (Neg/Pos) safely anchored into supports/spans.

## Part 4: 1D Cutting Optimization
- Group all required pieces (Main Bars and Stirrups) by Bar Diameter. Stock = 472". Use First Fit Decreasing bin packing.
- Output: Display aggregated patterns (e.g., "Qty: 10 Bars | Pattern: [120", 120", 112"] | Waste: X%") and explicitly state the "Total Number of 472-inch Stock Bars Required" per bar diameter.

## Part 5: Unified 2D Grid Elevation (SCALED & ABSOLUTE ANTI-OVERLAP)
- Provide a user choice to draw the beam elevation or skip drawing it. Drawing is enabled by default. When drawings are skipped, hide the drawing panels and prevent DXF export with a clear message; BBS calculation, schedules, JSON, and spreadsheet exports must remain available.
- Generate a 2D Canvas per grid.
- Grid Name: Render the Grid Name text (e.g., "GRID A") at the top-left of the canvas.
- Scale: Beam Depth must be drawn proportional to the span length.
- Anti-Overlap: Hardcode distinct Y-coordinates for Top Continuous, Top Neg Cutoffs, Bottom Continuous, and Bottom Pos Cutoffs. They MUST NEVER overlap visually.
- Outside-Beam: Offset rebar lines entirely OUTSIDE the beam outline (Top bars above, Bottom bars below) if the drawn depth is too narrow.
- Polylines: Draw rebars visually as continuous paths (straight + hooks).

## Part 6: Output & DXF Export
- Render Main BBS Table, Stirrup Schedule Table, and 1D Optimization Report. Provide Excel-compatible `.xls` downloads for the Main BBS and Stirrup Schedule tables, including their visible schedule data; include bar marks and piece-to-piece splice connections in the Main BBS export.
- Show the applied lap length in the Main BBS output for each splice. Include `f'c`, `fy`, fixed splice class `B`, the calculation assumptions, and the calculated lap length for each used main-bar diameter in the final JSON.
- DXF Export: Add a "Download All Drawings as DXF" button. The JS must convert the calculated grid drawings to a valid ASCII `.dxf` file.
  - CRITICAL: Rebars MUST be exported as `LWPOLYLINE` entities so straight runs and hooks are physically connected in AutoCAD.
  - CRITICAL: Include Grid Names as `TEXT` entities. Layout all grids vertically spaced in the DXF model space.
- "Save Final Data" button to log the final JSON structure.
