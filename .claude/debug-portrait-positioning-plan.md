# Debug Plan: Unit Portrait #2 Positioning Issue

## Problem Statement
UnitPortrait2 is appearing in the same position as UnitPortrait, despite having different positioning coordinates in the FDF file. Both portraits are overlapping instead of appearing side by side.

## Investigation Plan

### Phase 1: Isolate the UI Components
**Goal:** Show only the portraits to eliminate visual clutter and clearly see if they're actually overlapping or if one is hidden.

1. **Find all UI component definitions in SmashUI**
   - Search for all `.fdf` files in `resources/UI/FrameDef/SmashUI/`
   - List all Frame definitions to understand what's being rendered
   - Identify non-portrait UI elements

2. **Comment out non-portrait UI elements**
   - In the main UI definition files, comment out frames that aren't the portraits
   - Keep only: UnitPortrait, UnitPortrait2, and ConsoleUI (their parent)
   - This will make it visually obvious if the portraits are overlapping

3. **Test visibility**
   - Run the game and verify only portraits are visible
   - Take screenshots to document the current state

### Phase 2: Trace the Frame Creation and Positioning System
**Goal:** Understand how the FDF parser and UI system handle frame positioning, especially SetPoint directives.

1. **Understand the FDF parsing system**
   - Read `fdfparser/src/com/etheller/warsmash/fdfparser/FDFParser.java`
   - Find how `SetPoint` is parsed and stored
   - Find how frame references (like "UnitPortrait" in SetPoint) are resolved

2. **Trace frame creation in MeleeUI.java**
   - Find where `createSimpleFrame("UnitPortrait", ...)` is called (around line 724)
   - Find where `createSimpleFrame("UnitPortrait2", ...)` is called (around line 735)
   - Understand the order of creation and whether order matters

3. **Find the frame positioning implementation**
   - Search for files that handle `SetPoint` positioning logic
   - Look for classes related to: AnchorDefinition, FramePoint, positioning calculation
   - Understand how frames resolve their anchors relative to other frames

4. **Check if frame lookups work correctly**
   - Find the code that resolves frame names (e.g., "UnitPortrait" reference in SetPoint)
   - Verify if `getFrameByName()` can find frames created with `createSimpleFrame()`
   - Check if there's a timing issue where UnitPortrait2 is positioned before UnitPortrait is registered

### Phase 3: Add Debug Logging
**Goal:** Add logging to trace exactly what's happening during frame creation and positioning.

1. **Add logging to frame creation**
   - In MeleeUI.java, add System.out.println() calls when creating both portraits
   - Log the frame name, parent, and any initial position

2. **Add logging to SetPoint processing**
   - Find where SetPoint directives are applied
   - Log: frame name, anchor point, reference frame name, and calculated position
   - Log if a reference frame cannot be found

3. **Add logging to frame positioning calculations**
   - Log the final X/Y coordinates of both portraits after all positioning is applied
   - This will show if they're actually being positioned differently or ending up at the same coords

### Phase 4: Test Alternative Positioning Approaches
**Goal:** Try different ways to position UnitPortrait2 to identify what works.

1. **Test absolute positioning with different values**
   - Try moving UnitPortrait2 to a completely different location (e.g., 0.5, 0.5)
   - This will confirm if absolute positioning works at all for this frame

2. **Test if the issue is specific to BOTTOMLEFT/BOTTOMRIGHT**
   - Try different anchor points (TOP, CENTER, etc.)
   - See if the problem is with the specific anchor combination

3. **Test creating UnitPortrait2 differently in Java**
   - Instead of using SetPoint in FDF, try using `addAnchor()` in Java code (like SmashSimpleInfoPanel does)
   - This will show if the issue is with FDF parsing or the positioning system itself

4. **Test if frame hierarchy matters**
   - Try creating UnitPortrait2 as a child of UnitPortrait instead of ConsoleUI
   - Try creating it with a different parent frame

### Phase 5: Root Cause Analysis
**Goal:** Based on findings, identify the exact cause.

Possible causes to investigate:
- Frame name resolution fails (UnitPortrait2 can't find ConsoleUI or vice versa)
- SetPoint directives are being applied in wrong order
- createSimpleFrame() doesn't properly apply FDF positioning
- Z-order issue (both portraits exist in correct positions but one is hidden behind the other)
- Coordinate system issue (positions are calculated incorrectly)
- The second portrait's positioning is being overridden somewhere in the Java code

### Phase 6: Implement Fix
**Goal:** Apply the correct fix based on root cause.

Common fixes depending on root cause:
1. If timing issue: Change order of frame creation
2. If FDF parsing issue: Apply positioning in Java code instead
3. If reference resolution issue: Use absolute positioning or fix frame registration
4. If override issue: Remove conflicting positioning code in Java

## Key Files to Examine

### FDF Files
- `resources/UI/FrameDef/SmashUI/UnitPortrait.fdf` - The portrait definitions
- `resources/UI/FrameDef/SmashUI/ConsoleUI.fdf` - Parent frame definition (if exists)
- Other `.fdf` files in SmashUI directory - To understand overall UI structure

### Java Files
- `core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java` - Where portraits are created
- `fdfparser/src/com/etheller/warsmash/fdfparser/FDFParser.java` - FDF parsing logic
- Search for: GameUIFrame, UIFrame, SimpleFrame, SetPoint, AnchorDefinition, FramePoint - Core positioning classes

## Success Criteria
- UnitPortrait appears at position (0.211, 0)
- UnitPortrait2 appears at position (0.3045, 0) or adjusted position that's clearly next to UnitPortrait
- Both portraits are visible and not overlapping
- Understanding of why the original approach didn't work

## Notes
- The recent commit changed SetPoint from relative positioning `SetPoint BOTTOMLEFT,"UnitPortrait",BOTTOMRIGHT,0.01,0` to absolute `SetPoint BOTTOMLEFT,"ConsoleUI",BOTTOMLEFT,0.3045,0`
- This change didn't fix the issue, suggesting the problem is deeper than just the SetPoint values
- Both frames use `createSimpleFrame()` which might not fully honor FDF positioning
