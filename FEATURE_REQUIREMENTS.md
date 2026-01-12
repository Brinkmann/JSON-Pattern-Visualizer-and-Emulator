# JSON Pattern Visualizer - Feature Requirements

## Overview
These features improve pattern validation and testing for exercise designers creating cone light patterns for ESP32-powered smart cones.

---

## 1. Per-Node Duration Validator

### Purpose
Ensure all nodes have the same total duration across all phases so the pattern loops correctly. If nodes have different totals, they won't synchronize when the pattern restarts at Phase 1.

### Requirements

**1.1 Calculate per-node totals**
- For each node (1-18), add up all `secs` values across all phases
- Include `"Black"` or missing colors (these count as OFF time)
- Display the total for each node

**1.2 Validate consistency**
- Check if all nodes have the **same total duration**
- Flag any nodes with different totals as errors
- Pattern is only valid when all node totals match

**Example - VALID Pattern:**
```
Phase 1: Node 1 = Red 5s,   Node 2 = Blue 3s
Phase 2: Node 1 = Black 3s, Node 2 = Yellow 5s

Node 1 total: 5s + 3s = 8s ✓
Node 2 total: 3s + 5s = 8s ✓
Valid! ✓
```

**Example - INVALID Pattern:**
```
Phase 1: Node 1 = Red 5s,   Node 2 = Blue 3s
Phase 2: Node 1 = Black 2s, Node 2 = Yellow 5s

Node 1 total: 5s + 2s = 7s ✗
Node 2 total: 3s + 5s = 8s ✗
Invalid - Node 1 has 7s, Node 2 has 8s
```

**1.3 Display validation results**

Show a summary table on the **Editor Page** (after clicking Validate):

```
Node Duration Validation:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Node | Phase 1 | Phase 2 | Phase 3 | Total | Status
-----|---------|---------|---------|-------|--------
  1  |   5s    |   3s    |   4s    |  12s  |   ✓
  2  |   3s    |   5s    |   4s    |  12s  |   ✓
  3  |   4s    |   4s    |   5s    |  13s  |   ✗ Mismatch!
```

**Success message:**
```
✓ All nodes have consistent 12s duration
```

**Error message:**
```
✗ Duration mismatch detected:
  • Nodes 1-2: 12s total
  • Node 3: 13s total

Fix: Adjust Node 3 timings to total 12s across all phases.
```

---

## 2. Node Sequence Validator

### Purpose
Ensure nodes are numbered sequentially without gaps (1, 2, 3, 4...). Gaps indicate a configuration error or unnecessary complexity.

### Requirements

**2.1 Check for gaps**
- Find all unique node numbers used across all phases
- Sort them numerically
- Check if they form a continuous sequence starting from 1

**2.2 Display warnings**

**Valid - No gaps:**
```
✓ Nodes used: 1, 2, 3, 4, 5, 6 (sequential)
```

**Invalid - Has gaps:**
```
⚠️ Node numbering has gaps: 1, 2, 3, 4, 6, 7, 8, 9
   Missing: Node 5

Recommendation: Renumber Node 6→5, Node 7→6, Node 8→7, Node 9→8
```

**Invalid - Doesn't start at 1:**
```
✗ Nodes must start at 1
  Found: 2, 3, 4, 5
  Missing: Node 1
```

---

## 3. Slow-Motion Playback Speeds

### Purpose
Allow designers to test patterns in slow motion to verify precise timing and color transitions.

### Requirements

**3.1 Add slower speed options**

Current speeds: 1x, 1.5x, 2x, 3x, 4x

**Add these slower speeds:**
- 0.25x (quarter speed)
- 0.5x (half speed)
- 0.75x (three-quarter speed)

**Full speed list:**
```
0.25x | 0.5x | 0.75x | 1x | 1.5x | 2x | 3x | 4x
```

**3.2 Update the visualization page**
- Add speed chip buttons for 0.25x, 0.5x, 0.75x
- Highlight the active speed
- Update the speed label (e.g., "0.5x")
- Slow-motion should work with the existing animation loop

**Location:** Visualization Page (side panel controls)

---

## 4. Metadata-Enriched JSON Export

### Purpose
Include pattern summary information in the exported JSON to help the main platform display pattern details without parsing the entire structure.

### Requirements

**4.1 Calculate metadata from pattern**
- **duration_seconds:** Total loop duration (sum of all phase durations)
- **nodes_required:** Number of unique nodes used
- **phases_count:** Number of phases
- **colors_used:** Array of unique colors (exclude "Black")
- **recommended_layout:** The layout selected in editor (circle/square/triangle/rectangle)
- **validated:** True if pattern passed all validation checks
- **created_timestamp:** ISO 8601 timestamp (e.g., "2026-01-12T10:30:00Z")

**4.2 Add metadata to JSON export**

When clicking "Copy JSON" button, include a `_metadata` object at the top level:

```json
{
  "_metadata": {
    "duration_seconds": 40,
    "nodes_required": 4,
    "phases_count": 8,
    "colors_used": ["Red", "Blue", "Green", "Yellow"],
    "recommended_layout": "circle",
    "validated": true,
    "created_timestamp": "2026-01-12T10:30:00Z"
  },
  "_comment": "Pass to the cone matching the pass-from color",
  "pattern_1": {
    "duration": 40,
    "phases": [...]
  }
}
```

**Important:** The existing pattern structure (`pattern_1`, `duration`, `phases`) must remain unchanged - this is what the ESP32 hardware expects.

**4.3 Update copy confirmation**
```
✓ JSON copied to clipboard (with metadata)
```

---

## 5. Smart Error Messages

### Purpose
Replace generic error messages with specific, actionable guidance that helps designers fix problems.

### Requirements

**5.1 Duration mismatch errors**

**Bad (generic):**
```
❌ Validation failed
```

**Good (specific):**
```
❌ Duration mismatch detected:
   • Node 1: 12s total
   • Node 2: 12s total
   • Node 3: 13s total ← Problem!

Fix: Adjust Node 3 timings to total 12s across all phases.
Suggestion: Reduce one of Node 3's phase durations by 1s
```

**5.2 Node gap errors**

**Bad:**
```
❌ Invalid node configuration
```

**Good:**
```
⚠️ Node numbering has gaps: 1, 2, 3, 5, 6
   Missing: Node 4

Fix: Either add Node 4, or renumber Node 5→4 and Node 6→5
```

**5.3 JSON syntax errors**

**Bad:**
```
❌ Invalid JSON
```

**Good:**
```
❌ JSON syntax error at line 15:
   Expected comma or closing brace

   14 |     "color": "Red"
   15 |     "secs": 5        ← Missing comma here
   16 |   },

Fix: Add comma after "Red" on line 14
```

**5.4 Missing required fields**

**Bad:**
```
❌ Invalid pattern structure
```

**Good:**
```
❌ Missing required field: "duration"

Pattern must include:
{
  "pattern_1": {
    "duration": <number>,  ← Missing!
    "phases": [...]
  }
}
```

**5.5 Color validation**

**Bad:**
```
❌ Invalid color
```

**Good:**
```
❌ Invalid color "Orange" for Node 3 in Phase 2

Allowed colors: Red, Blue, Green, Yellow, Black
Did you mean: "Red"?
```

---

## Technical Notes

### JSON Structure Requirements
The pattern JSON must follow this exact structure (ESP32 requirement):

```json
{
  "_comment": "Optional description",
  "pattern_1": {
    "duration": <total_seconds>,
    "phases": [
      {
        "phase": <number>,
        "nodes": [
          { "node": <1-18>, "color": "<Red|Blue|Green|Yellow|Black>", "secs": <number> }
        ]
      }
    ]
  }
}
```

### Valid Colors
- Red
- Blue
- Green
- Yellow
- Black (means node is OFF)

### Validation Order
Run validations in this order:
1. JSON syntax validation
2. Required fields check
3. Node sequence check (sequential 1, 2, 3...)
4. Per-node duration consistency check
5. Color validity check

Stop at first error and show specific message.

---

## Implementation Checklist

### Editor Page (Edit Page_v6.2.html)
- [ ] Add per-node duration calculator
- [ ] Add node sequence validator
- [ ] Display validation results table
- [ ] Show smart error messages
- [ ] Add metadata to JSON export

### Visualization Page (Visualization Page_6.2.html)
- [ ] Add 0.25x speed chip button
- [ ] Add 0.5x speed chip button
- [ ] Add 0.75x speed chip button
- [ ] Test slow-motion playback works correctly

### Testing
- [ ] Test valid pattern (all nodes same duration, sequential)
- [ ] Test invalid pattern (duration mismatch)
- [ ] Test invalid pattern (node gaps)
- [ ] Test slow-motion speeds (0.25x, 0.5x, 0.75x)
- [ ] Test metadata export contains correct values
- [ ] Test error messages are helpful and specific

---

## Success Criteria

**A student has successfully completed this work when:**

1. ✅ Editor shows per-node duration totals for each node
2. ✅ Editor blocks visualization if nodes have mismatched durations
3. ✅ Editor warns if node numbering has gaps
4. ✅ Visualization page has working 0.25x, 0.5x, 0.75x speeds
5. ✅ Copy JSON button includes accurate `_metadata` object
6. ✅ Error messages tell designers exactly what's wrong and how to fix it
