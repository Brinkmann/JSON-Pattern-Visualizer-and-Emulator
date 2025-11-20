# EyeQ Drill Validator & Visualiser
## Software Requirements Document

---

## 1. Purpose and Scope

### 1.1 Primary Goal
Build a simple web-based tool that helps exercise designers visualize their drill setup and see the EyeQ JSON pattern play out on a 2D pitch.

### 1.2 Key Principles
- **Does not generate JSON patterns** - Both drill description and JSON pattern are provided by the editor as pre-existing inputs
- **Interprets setup descriptions** - Uses AI to determine geometric placement of nodes on the pitch
- **Animates existing patterns** - Plays the provided JSON pattern on positioned nodes
- **Human validation** - Editor visually assesses whether pattern behavior makes sense for their drill

### 1.3 Usage Context
- Single-user, ad hoc use
- No server-side persistence required
- Content remains on screen for copy/paste into other systems

---

## 2. User Overview

### 2.1 Primary User
**Role:** "Editor" - typically an exercise designer or coach

### 2.2 Typical Workflow
1. Editor pastes drill description and JSON pattern
2. System interprets setup description using AI to determine node placement
3. System displays 2D pitch with numbered nodes
4. System animates the JSON pattern on positioned nodes
5. Editor watches animation to verify pattern matches drill intent
6. If needed, editor manually edits JSON and re-runs animation

---

## 3. Inputs

### 3.1 Drill Text
**Description:** Plain text description of the drill

**Required elements:**
- Title
- Objective
- Setup (critical - describes physical arrangement of cones)
- Instructions

**Optional elements:**
- Coaching points
- Variations

**Examples:**
- "Pass to the cone matching the pass-from colour"
- "Dribble with the ball through chaos"
- "Scan to strike"

### 3.2 Pattern JSON
**Description:** Single JSON object defining the EyeQ cone pattern

**Format:** Valid JSON structure (basic parsing only, no deep schema validation)

---

## 4. Pattern JSON Structure

### 4.1 Overall Structure
```
Top-level object
├── comment: (string)
└── pattern object (e.g., pattern_1, pattern_custom)
    ├── duration: (integer) total nominal duration in seconds
    └── phases: (array) of phase objects
```

### 4.2 Phase Object
```
phase object
├── phase: (integer) phase number
└── nodes: (array) of node objects
```

### 4.3 Node Object
```
node object
├── node: (integer) node id, starting at 1
├── color: (string) color name (Red, Blue, Green, Yellow, Black)
└── secs: (integer) duration in seconds for this phase
```

### 4.4 Constraints
- **Maximum nodes:** 18 distinct node IDs
- **Duration field:** Top-level `duration` does not need to match sum of `secs` values
- **Node duration consistency:** All nodes MUST have equal total duration across all phases

**Example validation:**
- Node 1: phases with secs [4, 6, 2] = 12 seconds total
- Node 2: MUST also total 12 seconds
- Node 3: MUST also total 12 seconds
- If any node has a different total → error, visualization blocked

---

## 5. High-Level Flow

### 5.1 Three Main Stages
1. **Input and AI-assisted interpretation** of setup description
2. **Clarification** of geometric details (if needed)
3. **2D visualization and animation** with live editing

### 5.2 Implementation Options
- **Option A:** Single page with clearly separated areas
- **Option B:** Two views - "Input and interpret" → "Visualise and edit"

**Design principle:** Visual complexity should be kept low; outcome and clarity are more important than sophisticated UI

---

## 6. AI-Assisted Setup Interpretation

### 6.1 Trigger
Editor presses "Generate Layout" or similar button

### 6.2 AI Input
System sends to AI service:
- **Drill text** - as one string
- **Pattern JSON** - as string (not parsed) so AI has access to raw node IDs, colors, phases

### 6.3 AI Output
AI returns well-formed JSON object with:

| Field | Type | Description |
|-------|------|-------------|
| `totalNodesFromText` | integer or null | Number of cones from drill text, or null if not clearly stated |
| `totalNodesFromJSON` | integer | Number of distinct node IDs in the pattern |
| `nodeMappings` | array | Mapping between zones/groups and node IDs (e.g., "Zone 1": [1,2,3]) |
| `geometricPlacement` | string | Description of layout rules (type, spacing, dimensions, zones) |
| `nodeCoordinates` | array | Objects assigning each node ID an (x, y) coordinate |
| `clarificationPrompts` | array | Questions when AI cannot determine geometric details |

### 6.4 Geometric Placement Examples
AI can infer reasonable placement when description provides hints:

- "Four cones arranged in a square with 10 metres between each cone" → nodes 1-4 in square formation
- "Twelve cones numbered 1 to 12 distributed around the perimeter" → 12 nodes in circle/rectangle
- "Four cones behind four mini-goals on a 35 x 25 metre pitch" → 4 nodes positioned behind goal positions

### 6.5 Deterministic vs. AI Interpretation
- **When explicit dimensions given** (e.g., "35m x 25m", "10 metres apart") → prefer deterministic calculation
- **When description is vague** → AI infers reasonable placement

### 6.6 AI Role Clarification
**AI interprets setup description to produce geometric layout**

**AI does NOT validate whether JSON pattern is "correct" for the drill** - that judgment is left to human editor watching the animation

---

## 7. Automatic Validation Checks

### 7.1 Node Duration Consistency ⛔ BLOCKING

**Rule:** Each node must have the same total duration across all phases

**Process:**
1. System calculates total duration for each node (sum of all `secs` values)
2. Compare totals across all nodes
3. If totals differ → display error and block visualization

**Error message example:**
```
Node duration mismatch:
- Node 1 totals 12 seconds
- Node 3 totals 15 seconds

All nodes must have the same total duration across all phases.
```

**Resolution:** Editor manually corrects JSON in text area. No "ignore and continue" option.

### 7.2 Node Count Consistency ⛔ BLOCKING

**Rule:** Node count in drill text must match node count in JSON (when both are available)

**Process:**
1. Compare `totalNodesFromText` with `totalNodesFromJSON`
2. If both present and differ → display error and block visualization
3. If `totalNodesFromText` is null → may proceed (with optional notice)

**Resolution:** Editor manually corrects JSON or updates drill text. No "ignore and continue" option.

---

## 8. Setup Clarification and Refinement

### 8.1 Purpose
Help editor improve setup description so geometric layout can be accurately determined

**This is NOT about validating JSON pattern** - it's about ensuring nodes are placed correctly on canvas to match intended physical setup

### 8.2 When Clarification Occurs
- AI cannot confidently produce `nodeCoordinates`
- `nodeMappings` is incomplete while drill text references zones/special roles

### 8.3 Example Clarification Prompts

**For zone mapping:**
```
"Your drill mentions 12 EyeQ SmartCones placed around the field. 
Please confirm which node numbers correspond to each side or zone, for example:
- Nodes 1–3 north sideline
- Nodes 4–6 east sideline
- Nodes 7–9 south sideline
- Nodes 10–12 west sideline"
```

**For geometric placement:**
```
"The exact positions of the four cones are unclear. 
Please describe their arrangement, for example:
'Cones 1–4 form a square, 10 metres apart, centred in the pitch'"
```

### 8.4 Editor Response
- Editor answers in simple text input
- Implementation choice: re-call AI with extra context OR apply local helper to adjust coordinates

### 8.5 Iteration Goal
Iterate on setup description until AI can confidently place all nodes in positions matching editor's physical drill setup

---

## 9. Visualization and Animation

### 9.1 Pitch Display

**Representation:** Plain 2D rectangle

**Dimensions:**
- Derived from drill description (e.g., "35 x 25 metres", "40 x 15 metres")
- Scaled to fit screen

**Zones:**
- Key areas drawn as simple boundaries or labels (e.g., "four zones", "sidelines", "behind goals")
- Inferred by AI or deterministic logic

### 9.2 Node Display

**Symbol:** Small triangle or circle at (x, y) coordinate

**Labeling:** 
- Each node displays its node ID (1, 2, 3, etc.)
- Must be readable on desktop and tablet screens

**Color:**
- Fill color driven by currently active phase in JSON
- Color = "Black" → rendered as off, dark, or neutral

### 9.3 Animation Behavior

**Phase Loop:**
1. System loops through phases in JSON order
2. For each phase, apply color to each node
3. Phase duration = based on `secs` and current speed multiplier
4. After final phase → loop back to phase one

**Phase Duration Calculation:**
- Use maximum `secs` value in that phase, OR
- Use average or sum of `secs` values
- Must be consistent and documented for development

**Editor Assessment:**
- Editor watches animation to assess if pattern behavior makes sense for their drill
- If not → editor manually edits JSON and re-runs animation

---

## 10. Interactive Controls

### 10.1 Play/Pause Toggle
Start or stop the animation loop

### 10.2 Speed Control

**Multipliers supported:** 0.5×, 1.0×, 1.5×, 2.0×

**Behavior:**
- Applies global multiplier to all `secs` values for phase timing calculation
- Example: At 2.0×, a node with `secs: 4` displays for equivalent of 2 seconds

**Important:** Speed control does NOT modify JSON itself - only affects visual playback

---

## 11. Live Editing Behavior

### 11.1 JSON Visibility
Current pattern JSON always visible in text area on visualization screen

### 11.2 Two Edit Types

#### 11.2.1 Simple Live Tweak (No AI Call)

**What qualifies:**
- Changes to `color` values
- Changes to `secs` values
- NO adding/removing nodes, phases, or pattern keys

**Process when editor presses "Update Animation":**
1. Parse JSON
2. Validate node duration consistency
3. If valid:
   - Apply new values
   - Keep existing `nodeCoordinates` and layout
   - Restart animation from phase one with new `secs` and colors
4. If invalid:
   - Display error message
   - Do not update animation

**Error handling:**
- JSON parsing errors → simple message: "Cannot parse JSON, please check commas and brackets"

#### 11.2.2 Structural Change (Full Re-interpretation)

**What qualifies:**
- Adding or removing nodes
- Adding or removing phases
- Changing anything affecting node count or mapping
- Changes to drill text

**Process when editor presses "Re-interpret Setup":**
1. Re-parse JSON
2. Call AI again with latest drill text and pattern JSON
3. Re-apply all automatic validation checks
4. Re-apply clarification process if needed
5. Potentially generate new `nodeCoordinates`
6. May ask for further setup clarifications
7. Only after complete → return to visualization

**UI Requirement:**
Interface must clearly distinguish between:
- "Update animation only" (simple tweak)
- "Re-interpret setup and layout" (structural change)

---

## 12. Stretch Goal: Additional Visual Elements

### 12.1 Priority
🎯 **STRETCH GOAL** - Secondary to primary objective

**Primary objective:** Interpret setup description, place nodes correctly, animate pattern for editor assessment

**Stretch goal:** Enhance visualization for documentation purposes

### 12.2 Purpose
Allow editor to manually place additional visual elements on canvas to create complete drill setup diagram

**Not part of:**
- Setup interpretation process
- JSON validation process

### 12.3 Element Types

| Element | Purpose |
|---------|---------|
| **Black round markers/dots** | Additional reference points, boundary markers, non-EyeQ cones |
| **Goal icons** | Mini-goals or full-size goals |
| **Player icons** | Starting positions or key player positions |

**Design requirement:** Must be clearly distinguishable from numbered node triangles (EyeQ SmartCones)

### 12.4 Interaction Model

**Placement:**
1. Editor selects element type from toolbar/palette
2. Editor clicks or drags to place on canvas

**Repositioning:**
- Drag element to new location

**Deletion:**
- Drag off canvas, OR
- Click delete button, OR
- Similar simple mechanism

**Manual only:** These elements are NOT inferred from drill text or JSON - editor places based on their own knowledge

### 12.5 Relationship to Core Functionality

**No relationship to:**
- JSON pattern
- Setup interpretation process
- Animation behavior

**Behavior:**
- Adding/moving/removing elements does NOT trigger re-interpretation
- Elements remain static during animation
- Do NOT change color or respond to phases

### 12.6 Use Case
**Primary use case:** Screenshot for documentation

**Workflow:**
1. System interprets setup and places numbered nodes
2. System animates JSON pattern
3. Editor manually adds goals, players, markers
4. Editor captures screenshot
5. Screenshot used in documentation, training materials, drill libraries

### 12.7 Implementation Notes

**Priority:** Core functionality (setup interpretation + animation) takes absolute priority

**Storage:** 
- In-browser memory only
- No persistence required
- Page refresh clears additional elements (acceptable - output is screenshot)

**Decision criteria:**
- ✅ Straightforward to implement → adds significant practical value
- ❌ Introduces complexity or delays core functionality → defer to future version

---

## 13. Technical Requirements

### 13.1 Platform
**Browser-based:** Must run in modern browsers

**Target devices:**
- Desktop (primary)
- Tablet (primary)
- Mobile optimization NOT required

### 13.2 Rendering
**Technology options:** Canvas, SVG, or any simple 2D approach

**Choice:** Left to implementation team

### 13.3 Backend
**Minimal functionality required:**
- Accept text inputs
- Call AI parsing service
- Return AI's structured response to front end

**Not required:**
- Server-side persistence
- User accounts
- Session management

### 13.4 Debug Logging
**Optional:** AI calls and parsing behavior

**Purpose:** Development and support only

**Not required:** Expose to editor in UI

---

## 14. Out of Scope

The following features are **explicitly excluded** from this version:

| Feature | Reason |
|---------|--------|
| ❌ 3D visualization or camera views | Complexity beyond requirements |
| ❌ Audio cues or sound effects | Not needed for primary objective |
| ❌ Multi-user collaboration | Single-user tool |
| ❌ Accounts or role management | No persistence required |
| ❌ Persistent drill storage | Copy/paste sufficient |
| ❌ Server-side pattern saving | Copy/paste sufficient |
| ❌ File download functionality | Screenshot sufficient |
| ❌ Automatic JSON pattern validation | Human editor judges if pattern makes sense |
| ❌ AI validation of pattern "correctness" | Editor watches animation and decides |
| ❌ Automatic placement of goals/players | Stretch goal is manual placement only |
| ❌ JSON pattern generation from descriptions | Tool only visualizes pre-existing JSON |

---

## 15. Example Drills for Testing

### 15.1 Pass to the Cone Matching the Pass-From Colour
- **Setup:** Four cones arranged in a square, plus a central player
- **Nodes:** 1 to 4
- **JSON:** Pattern cycles colors across nodes 1-4 in several phases

### 15.2 Dribble with the Ball Through Chaos
- **Setup:** 12 EyeQ SmartCones numbered 1-12 around perimeter of larger pitch
- **Nodes:** 1 to 12
- **JSON:** Phases light different cones using Red, Green, Yellow, Blue, Black

### 15.3 Scan to Strike
- **Setup:** Four mini-goals with four EyeQ cones behind each goal on 35 x 25 metre area
- **Nodes:** 4 nodes (one per goal)
- **JSON:** One cone lights up per phase while others remain dark (indicating active goal)

### 15.4 Testing Objective
System should be able to:
1. Take drill text and JSON from each example
2. Interpret setup description to generate node layout
3. Display simple animated 2D view
4. Allow editor to visually assess if pattern behavior matches drill intent

---

## Document Version
**Last updated:** [Date to be inserted]

**Status:** Draft for development handover
