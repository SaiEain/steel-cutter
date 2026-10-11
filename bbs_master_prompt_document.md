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
- CRITICAL: Initialize Table 1 with ZERO default rows. Add a "Load from CSV" button to parse a CSV file and auto-populate this table.

## Part 2: Tekla-Style Main Grid Builder Wizard (ONLY METHOD)
- Create a sticky "Main Grid Builder Wizard". (Manual sequence building is STRICTLY FORBIDDEN).
- Inputs:
  1. Grid Name (e.g., Grid A).
  2. Spans (Tekla-Style ft&in syntax, e.g., `3*15'6", 20'0"`).
  3. Beam Assignments: Comma-separated text (e.g., `B1, B2, B1`). MUST validate exact match with Table 1 labels. Count MUST equal 1 OR the exact number of parsed spans.
  4. Column Sizes (in): Comma-separated text (e.g., `14, 16, 14, 14`). Count MUST equal 1 OR the exact number of parsed spans + 1.
- Buttons: "Generate Grid" (adds data to Table 2) and "Clear Wizard".
- Table 2 (Sequence List): Contains generated grids as editable blocks with "Delete Grid" buttons.

## Part 3: BBS Calculation Logic (STRICT ACI ZONES & CONTINUITY)
- Loop through all grids in Table 2. Max cut piece = 472 inches (39' 4").
- For each continuous main-bar run, obtain the run's Table 1 main bar diameter and calculate its Class B lap length from Part 1. Use that run-specific lap length in every splice-zone and stock-length calculation; never reuse one global lap length for bars of different diameters.
- Top Continuous Bars: Continue into next span ONLY if adjacent Beam Width is identical. If different, terminate with a 90° hook. Lap splices must land strictly between `Ln/4` and `3*Ln/4` (Middle Half).
- Bottom Continuous Bars: Continue ONLY if adjacent Width AND Depth are BOTH identical. If either differs, terminate with a 90° hook. Lap splices must land strictly between `left_face + 2h` and `right_face - 2h`.
- Algorithm: Calculate span-by-span. Dynamically shorten cut lengths to force lap splices safely into valid ACI zones. NEVER cut at 472" if it lands in a prohibited zone. Calculate cutoffs (Neg/Pos) safely anchored into supports/spans.

## Part 4: 1D Cutting Optimization
- Group all required pieces (Main Bars and Stirrups) by Bar Diameter. Stock = 472". Use First Fit Decreasing bin packing.
- Output: Display aggregated patterns (e.g., "Qty: 10 Bars | Pattern: [120", 120", 112"] | Waste: X%") and explicitly state the "Total Number of 472-inch Stock Bars Required" per bar diameter.

## Part 5: Unified 2D Grid Elevation (SCALED & ABSOLUTE ANTI-OVERLAP)
- Generate a 2D Canvas per grid.
- Grid Name: Render the Grid Name text (e.g., "GRID A") at the top-left of the canvas.
- Scale: Beam Depth must be drawn proportional to the span length.
- Anti-Overlap: Hardcode distinct Y-coordinates for Top Continuous, Top Neg Cutoffs, Bottom Continuous, and Bottom Pos Cutoffs. They MUST NEVER overlap visually.
- Outside-Beam: Offset rebar lines entirely OUTSIDE the beam outline (Top bars above, Bottom bars below) if the drawn depth is too narrow.
- Polylines: Draw rebars visually as continuous paths (straight + hooks).

## Part 6: Output & DXF Export
- Render Main BBS Table, Stirrup Schedule Table, and 1D Optimization Report.
- Show the applied lap length in the Main BBS output for each splice. Include `f'c`, `fy`, fixed splice class `B`, the calculation assumptions, and the calculated lap length for each used main-bar diameter in the final JSON.
- DXF Export: Add a "Download All Drawings as DXF" button. The JS must convert the calculated grid drawings to a valid ASCII `.dxf` file.
  - CRITICAL: Rebars MUST be exported as `LWPOLYLINE` entities so straight runs and hooks are physically connected in AutoCAD.
  - CRITICAL: Include Grid Names as `TEXT` entities. Layout all grids vertically spaced in the DXF model space.
- "Save Final Data" button to log the final JSON structure.
