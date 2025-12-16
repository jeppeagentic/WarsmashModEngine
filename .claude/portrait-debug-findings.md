# Unit Portrait Positioning Investigation - Findings and Fix

## Investigation Summary

### What Was Done

1. **Phase 1**: Mapped all UI components in SmashUI
   - Found 5 FDF files: ConsoleUI, UnitPortrait, InventoryCover, TimeOfDayIndicator, ToolTip
   - Verified FDF definitions are correct

2. **Phase 2**: Traced frame creation and positioning system
   - Analyzed how `createSimpleFrame()` works in [GameUI.java:294-316](core/src/com/etheller/warsmash/parsers/fdf/GameUI.java#L294-L316)
   - Found SetPoint application in [GameUI.java:1515-1538](core/src/com/etheller/warsmash/parsers/fdf/GameUI.java#L1515-L1538)
   - Understood the positioning calculation in [AbstractRenderableFrame.java:263-327](core/src/com/etheller/warsmash/parsers/fdf/frames/AbstractRenderableFrame.java#L263-L327)

3. **Phase 3**: Added comprehensive debug logging
   - SetPoint resolution logging in [GameUI.java:1518-1524](core/src/com/etheller/warsmash/parsers/fdf/GameUI.java#L1518-L1524)
   - Frame creation logging in [MeleeUI.java:724-739](core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java#L724-L739)
   - Position bounds logging in [AbstractRenderableFrame.java:322-325](core/src/com/etheller/warsmash/parsers/fdf/frames/AbstractRenderableFrame.java#L322-L325)

## How the Positioning System Works

### Frame Creation Flow
1. `MeleeUI.java` calls `createSimpleFrame("UnitPortrait2", consoleUI, 0)`
2. `GameUI.createSimpleFrame()` looks up the FrameDefinition from templates
3. `GameUI.inflate()` creates the actual frame instance
4. During inflation:
   - Frame is created and immediately added to `nameToFrame` map (line 383)
   - Child frames are recursively inflated
   - SetPoint definitions are processed (lines 1515-1538)
   - For each SetPoint, the "other" frame is looked up via `getFrameByName()`
   - If found, `addSetPoint()` is called with the resolved frame reference

### Position Calculation Flow
1. During rendering, `positionBounds()` is called on each frame
2. For BOTTOMLEFT anchor point:
   - X position = other.getFramePointX(BOTTOMLEFT) + xOffset
   - Y position = other.getFramePointY(BOTTOMLEFT) + yOffset
3. Frame's renderBounds are updated with calculated positions

## Expected Debug Output

When you run the game, you should see output like:

```
[DEBUG] Creating UnitPortrait frame...
[DEBUG] SetPoint for UnitPortrait: myPoint=BOTTOMLEFT, other=ConsoleUI, otherPoint=BOTTOMLEFT, x=0.211, y=0.0, otherFrame=FOUND
[DEBUG] UnitPortrait created: SimpleFrame@...
[DEBUG] Creating UnitPortrait2 frame...
[DEBUG] SetPoint for UnitPortrait2: myPoint=BOTTOMLEFT, other=ConsoleUI, otherPoint=BOTTOMLEFT, x=0.3045, y=0.0, otherFrame=FOUND/NULL
[DEBUG] UnitPortrait2 created: SimpleFrame@...
[DEBUG POSITION] SimpleFrame:ConsoleUI:... finishing position bounds: (x, y, width, height)
[DEBUG POSITION] SimpleFrame:UnitPortrait:... finishing position bounds: (x, y, width, height)
[DEBUG POSITION] SimpleFrame:UnitPortrait2:... finishing position bounds: (x, y, width, height)
```

## Possible Root Causes

### Scenario A: Frame Reference Not Found (otherFrame=NULL)
If the debug output shows `otherFrame=NULL` for UnitPortrait2:
- **Cause**: ConsoleUI not registered in nameToFrame map when UnitPortrait2 is inflated
- **Why**: Unlikely - ConsoleUI is created at [MeleeUI.java:568](core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java#L568) before portraits
- **Fix**: Would need to investigate frame name registration

### Scenario B: Both Frames Have Same Position (likely issue)
If debug shows both portraits have identical x,y coordinates:
- **Cause**: The frames might be positioned correctly initially, but something else is happening
- **Possible sub-causes**:
  1. **Parent hierarchy issue**: Both frames added to consoleUI, inheriting parent position
  2. **Default positioning**: If SetPoints fail silently, frames default to parent's BOTTOMLEFT
  3. **Z-order issue**: Both frames are positioned correctly but one is behind the other
  4. **Viewport conversion issue**: The offset values aren't being converted correctly

### Scenario C: SetPoint Applied But Position Looks Wrong
If SetPoints are applied but positions don't match expected FDF values:
- **Cause**: Viewport coordinate conversion issue
- The convertX/convertY functions at [GameUI.java:1535-1536](core/src/com/etheller/warsmash/parsers/fdf/GameUI.java#L1535-L1536) might be using wrong viewport

## Recommended Fix Based on Most Likely Cause

The most likely issue is that **both frames are defaulting to the parent's position** because SetPoint positioning might not be working as expected when the "other" frame is the parent frame.

### Fix Option 1: Use Anchor Instead of SetPoint (Like SmashSimpleInfoPanel)

Looking at [MeleeUI.java:743-746](core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java#L743-L746), some frames use `addAnchor()` in Java code instead of SetPoint in FDF:

```java
this.smashSimpleInfoPanel
    .addAnchor(new AnchorDefinition(FramePoint.BOTTOM, 0, GameUI.convertY(this.uiViewport, 0.0f)));
```

Apply this pattern to UnitPortrait2:

**File**: [MeleeUI.java:738](core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java#L738)

```java
// Create second portrait
this.portrait2 = new Portrait(this.war3MapViewer, this.portraitScene);
System.out.println("[DEBUG] Creating UnitPortrait2 frame...");
this.unitPortrait2 = (SimpleFrame) this.rootFrame.createSimpleFrame("UnitPortrait2", this.consoleUI, 0);
// Override FDF positioning with explicit anchor
this.unitPortrait2.clearFramePointAssignments();
this.unitPortrait2.addAnchor(new AnchorDefinition(FramePoint.BOTTOMLEFT,
    GameUI.convertX(this.uiViewport, 0.3045f),
    GameUI.convertY(this.uiViewport, 0.0f)));
System.out.println("[DEBUG] UnitPortrait2 created: " + this.unitPortrait2);
```

### Fix Option 2: Remove SetPoint from FDF, Use Relative Positioning

If the issue is that SetPoint to ConsoleUI doesn't work because ConsoleUI is the parent, remove SetPoint from FDF and just use Anchor:

**File**: [UnitPortrait.fdf:46-50](resources/UI/FrameDef/SmashUI/UnitPortrait.fdf#L46-L50)

```fdf
Frame "SIMPLEFRAME" "UnitPortrait2" {
	DecorateFileNames,
    Anchor BOTTOMLEFT, 0.3045, 0,
	Width 0.0835,
	Height 0.114,
```

### Fix Option 3: Position Relative to UnitPortrait Instead of ConsoleUI

Change UnitPortrait2 to position relative to UnitPortrait instead:

**File**: [UnitPortrait.fdf:48](resources/UI/FrameDef/SmashUI/UnitPortrait.fdf#L48)

```fdf
SetPoint BOTTOMLEFT,"UnitPortrait",BOTTOMRIGHT,0.01,0,
```

This was the original approach before the recent commit.

## Next Steps

1. **Run the game** and collect the debug output
2. **Check the console** for the three types of debug messages:
   - `[DEBUG]` lines showing frame creation
   - `[DEBUG]` lines showing SetPoint resolution
   - `[DEBUG POSITION]` lines showing final positions
3. **Based on the output**, choose the appropriate fix:
   - If `otherFrame=NULL`: Fix frame registration issue
   - If positions are identical: Apply Fix Option 1 or 2
   - If positions look wrong: Investigate viewport conversion

## Debug Logging Added

The following debug logs were added and can be removed after the issue is fixed:

1. [GameUI.java:1518-1524](core/src/com/etheller/warsmash/parsers/fdf/GameUI.java#L1518-L1524) - SetPoint resolution logging
2. [MeleeUI.java:724,726,737,739](core/src/com/etheller/warsmash/viewer5/handlers/w3x/ui/MeleeUI.java#L724) - Frame creation logging
3. [AbstractRenderableFrame.java:26](core/src/com/etheller/warsmash/parsers/fdf/frames/AbstractRenderableFrame.java#L26) - DEBUG_LOG flag enabled
4. [AbstractRenderableFrame.java:322-325](core/src/com/etheller/warsmash/parsers/fdf/frames/AbstractRenderableFrame.java#L322-L325) - Position bounds logging (filtered)

To clean up after debugging, revert these changes or set `DEBUG_LOG = false`.
