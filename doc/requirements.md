## 1. Purpose and Scope

### 1.1 Primary Goal
Build a simple web-based tool that helps exercise designers visualize their drill setup and see the EyeQ JSON pattern play out on a 2D pitch.

### 1.2 Key Principles
- **Does not generate JSON patterns** - Both drill description and JSON pattern are provided by the editor as pre-existing inputs
- **AI interprets text only** - AI service reads drill text to determine geometric placement; it never receives or processes JSON
- **Application validates JSON** - All JSON parsing, validation, and animation logic handled by application code
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
2. System sends drill text to AI service to determine node placement
3. Application code parses JSON to extract node IDs and pattern data
4. System displays 2D pitch with numbered nodes at AI-determined coordinates
5. Application code animates the JSON pattern on positioned nodes
6. Editor watches animation to verify pattern matches drill intent
7. If needed, editor manually edits JSON and re-runs animation

---

## 3. Inputs

### 3.1 Drill Text
**Description:** Plain text description of the drill

**Required elements:**
- Title
- Objective
- Setup (critical - describes physical arrangement of cones with cone numbers/IDs)
- Instructions

**Optional elements:**
- Coaching points
- Variations

**Examples:**
- "Pass to the cone matching the pass-from colour"
- "Dribble with the ball through chaos"
- "Scan to strike"

**Important for AI interpretation:** The drill text should reference specific cone numbers or IDs (e.g., "Cones 1-4", "Node 1", "Cone number 12") to help the AI assign coordinates to the correct node IDs.

### 3.2 Pattern JSON
**Description:** Single JSON object defining the EyeQ cone pattern

**Format:** Valid JSON structure (basic parsing only, no deep schema validation)

**Important:** This JSON is NEVER sent to the AI service. It is processed entirely by application code.

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

**All JSON validation is performed by application code, not by AI service.**

---

## 5. High-Level Flow

### 5.1 Three Main Stages
1. **Input and AI-assisted interpretation** of drill text (text only, no JSON to AI)
2. **Application validation and clarification** of JSON and geometric details
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
- **Drill text ONLY** - as one string

**Critical:** The Pattern JSON is NEVER sent to the AI service. The AI's role is strictly limited to interpreting the drill text to determine physical geometric layout.

### 6.3 AI Output
AI returns well-formed JSON object with:

| Field | Type | Description |
|-------|------|-------------|
| `totalNodesFromText` | integer or null | Number of cones mentioned in drill text, or null if not clearly stated |
| `nodeMappings` | array | Mapping between zones/groups and node IDs (e.g., "Zone 1": [1,2,3]) |
| `geometricPlacement` | string | Description of layout rules (type, spacing, dimensions, zones) |
| `nodeCoordinates` | array | Objects assigning each node ID an (x, y) coordinate |
| `clarificationPrompts` | array | Questions when AI cannot determine geometric details |

**How AI determines node IDs:**
The AI searches the drill text for references to cone numbers or IDs (e.g., "Cones 1-4", "Node 5", "Cone number 12", "twelve cones numbered 1 to 12"). The AI uses these references to assign coordinates to specific node IDs. If the text doesn't explicitly number the cones, the AI may ask for clarification.

**What AI does NOT know:**
The AI has no knowledge of the Pattern JSON contents, structure, or which node IDs are used in the JSON. The application code is responsible for bridging the AI's geometric layout with the JSON's node references.

### 6.4 Geometric Placement Examples
AI can infer reasonable placement when description provides hints:

- "Four cones arranged in a square with 10 metres between each cone, cones numbered 1-4" → nodes 1-4 in square formation
- "Twelve cones numbered 1 to 12 distributed around the perimeter" → 12 nodes in circle/rectangle
- "Four cones behind four mini-goals on a 35 x 25 metre pitch, numbered 1 to 4" → 4 nodes positioned behind goal positions

### 6.5 Deterministic vs. AI Interpretation
- **When explicit dimensions given** (e.g., "35m x 25m", "10 metres apart") → prefer deterministic calculation
- **When description is vague** → AI infers reasonable placement

### 6.6 AI Role Clarification
**AI responsibility:**
- Reads drill text ONLY
- Interprets setup description to produce geometric layout
- Outputs node coordinates for referenced cone numbers/IDs

**Application code responsibility:**
- Parses Pattern JSON
- Extracts node IDs from JSON
- Validates JSON structure and node duration consistency
- Bridges AI coordinates with JSON node IDs by matching node numbers
- Animates pattern using colors and timing from JSON

**AI does NOT:**
- Receive or process Pattern JSON
- Validate whether JSON pattern is "correct" for the drill
- Know which node IDs exist in the JSON

---

## 7. Automatic Validation Checks

**All validation checks are performed by application code, not by AI service.**

### 7.1 Node Duration Consistency ⛔ BLOCKING

**Performed by:** Application code

**Rule:** Each node must have the same total duration across all phases

**Process:**
1. Application code parses the Pattern JSON
2. Application calculates total duration for each node (sum of all `secs` values)
3. Application compares totals across all nodes
4. If totals differ → display error and block visualization

**Error message example:**
```
Node duration mismatch:
- Node 1 totals 12 seconds
- Node 3 totals 15 seconds

All nodes must have the same total duration across all phases.
```

**Resolution:** Editor manually corrects JSON in text area. No "ignore and continue" option.

### 7.2 Node Count Consistency ⛔ BLOCKING

**Performed by:** Application code

**Rule:** Node count mentioned in drill text should match node count in JSON

**Process:**
1. AI service returns `totalNodesFromText` (from drill text interpretation)
2. Application code parses Pattern JSON and calculates `totalNodesFromJSON` (count of distinct node IDs)
3. Application code compares these two values
4. If both present and differ → display error and block visualization
5. If `totalNodesFromText` is null → may proceed (with optional notice)

**Critical distinction:**
- `totalNodesFromText` = from AI service (based on drill text)
- `totalNodesFromJSON` = calculated by application code (from parsing JSON)
- **Comparison performed by application code, not AI**

**Resolution:** Editor manually corrects JSON or updates drill text. No "ignore and continue" option.

### 7.3 Node ID Matching

**Performed by:** Application code

**Rule:** Node IDs in the JSON should have corresponding coordinates from the AI

**Process:**
1. Application extracts distinct node IDs from Pattern JSON
2. Application checks if AI-provided `nodeCoordinates` includes coordinates for each node ID
3. If node IDs in JSON don't have coordinates → request clarification or re-interpretation

**Note:** The AI may not know which node IDs the JSON uses, so it's the application's job to ensure the bridge between text interpretation and JSON structure.

---

## 8. Setup Clarification and Refinement

### 8.1 Purpose
Help editor improve setup description so geometric layout can be accurately determined

**This is NOT about validating JSON pattern** - it's about ensuring nodes are placed correctly on canvas to match intended physical setup

### 8.2 When Clarification Occurs
- AI cannot confidently produce `nodeCoordinates`
- AI cannot determine which cone numbers/IDs correspond to which positions
- `nodeMappings` is incomplete while drill text references zones/special roles
- Application detects node IDs in JSON that don't have corresponding coordinates from AI

### 8.3 Example Clarification Prompts

**For cone numbering:**
```
"Your drill mentions cones but doesn't specify which cone numbers go where.
Please update your drill text to include cone numbers, for example:
'Place cones numbered 1-4 in a square, 10 metres apart'"
```

**For zone mapping:**
```
"Your drill mentions 12 EyeQ SmartCones placed around the field. 
Please confirm which cone numbers correspond to each side or zone, for example:
- Cones 1–3 north sideline
- Cones 4–6 east sideline
- Cones 7–9 south sideline
- Cones 10–12 west sideline"
```

**For geometric placement:**
```
"The exact positions of the four cones are unclear. 
Please describe their arrangement, for example:
'Cones 1–4 form a square, 10 metres apart, centred in the pitch'"
```

### 8.4 Editor Response
- Editor answers in simple text input or updates drill text
- System re-calls AI service with updated drill text ONLY (never with JSON)
- Application may apply local helper logic to adjust mappings

### 8.5 Iteration Goal
Iterate on setup description until:
1. AI can confidently place all referenced cone numbers in correct positions
2. Application can match all JSON node IDs with AI-provided coordinates

---

## 9. Visualization and Animation

### 9.1 Pitch Display

**Representation:** Plain 2D rectangle

**Dimensions:**
- Derived from drill description (e.g., "35 x 25 metres", "40 x 15 metres")
- Scaled to fit screen

**Zones:**
- Key areas drawn as simple boundaries or labels (e.g., "four zones", "sidelines", "behind goals")
- Inferred by AI from drill text or applied by deterministic logic

### 9.2 Node Display

**Symbol:** Small triangle or circle at (x, y) coordinate

**Positioning:** Uses coordinates from AI's `nodeCoordinates` output

**Labeling:** 
- Each node displays its node ID (1, 2, 3, etc.)
- Must be readable on desktop and tablet screens

**Color:**
- Fill color driven by currently active phase in JSON (processed by application code)
- Color = "Black" → rendered as off, dark, or neutral

**Bridge between AI and JSON:**
Application code matches node IDs from JSON with coordinates from AI output to position and animate each node correctly.

### 9.3 Animation Behavior

**All animation logic performed by application code based on Pattern JSON.**

**Phase Loop:**
1. Application reads phases from JSON in order
2. For each phase, application applies color to each node based on JSON
3. Phase duration = calculated by application based on `secs` and current speed multiplier
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
Start or stop the animation loop (controlled by application code)

### 10.2 Speed Control

**Multipliers supported:** 0.5×, 1.0×, 1.5×, 2.0×

**Behavior:**
- Applies global multiplier to all `secs` values for phase timing calculation
- Example: At 2.0×, a node with `secs: 4` displays for equivalent of 2 seconds
- Implemented by application code

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
1. Application code parses JSON
2. Application validates node duration consistency
3. If valid:
   - Application applies new values
   - Keep existing AI-provided `nodeCoordinates` and layout
   - Application restarts animation from phase one with new `secs` and colors
4. If invalid:
   - Display error message
   - Do not update animation

**Error handling:**
- JSON parsing errors → simple message: "Cannot parse JSON, please check commas and brackets"

**No AI involvement** - all processing done by application code.

#### 11.2.2 Structural Change (Re-interpretation Required)

**What qualifies:**
- Adding or removing nodes
- Adding or removing phases
- Changing anything affecting node count or mapping
- Changes to drill text

**Process when editor presses "Re-interpret Setup":**
1. Application re-parses JSON locally
2. **System calls AI again with latest Drill Text ONLY** (no JSON sent to AI)
3. Application re-applies all automatic validation checks
4. Application checks if new node IDs in JSON have corresponding coordinates from AI
5. AI may return new `clarificationPrompts` if setup details unclear
6. Application potentially receives new `nodeCoordinates` from AI
7. Application may prompt editor for further setup clarifications
8. Only after complete → return to visualization

**UI Requirement:**
Interface must clearly distinguish between:
- "Update animation only" (simple tweak - no AI call)
- "Re-interpret setup and layout" (structural change - AI called with text only)

**Critical:** Even during structural changes, the AI service receives ONLY the drill text, never the JSON.

---

## 12. Stretch Goal: Additional Visual Elements

### 12.1 Priority
🎯 **STRETCH GOAL** - Secondary to primary objective

**Primary objective:** AI interprets drill text for geometric layout, application code validates JSON and animates pattern

**Stretch goal:** Enhance visualization for documentation purposes

### 12.2 Purpose
Allow editor to manually place additional visual elements on canvas to create complete drill setup diagram

**Not part of:**
- AI interpretation process
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

**Manual only:** These elements are NOT inferred from drill text - editor places based on their own knowledge

### 12.5 Relationship to Core Functionality

**No relationship to:**
- AI interpretation process
- JSON pattern
- Animation behavior

**Behavior:**
- Adding/moving/removing elements does NOT trigger re-interpretation
- Elements remain static during animation
- Do NOT change color or respond to phases

### 12.6 Use Case
**Primary use case:** Screenshot for documentation

**Workflow:**
1. AI interprets drill text and provides node coordinates
2. Application places numbered nodes at coordinates
3. Application animates JSON pattern
4. Editor manually adds goals, players, markers
5. Editor captures screenshot
6. Screenshot used in documentation, training materials, drill libraries

### 12.7 Implementation Notes

**Priority:** Core functionality (AI text interpretation + application JSON validation + animation) takes absolute priority

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

### 13.3 Backend and AI Service

**AI Service responsibilities:**
- Accept drill text input ONLY (no JSON)
- Interpret setup description
- Return geometric layout data (`nodeCoordinates`, `nodeMappings`, etc.)

**Application code responsibilities:**
- Parse and validate Pattern JSON
- Calculate `totalNodesFromJSON`
- Validate node duration consistency
- Match JSON node IDs with AI-provided coordinates
- Handle all animation logic
- Bridge AI output with JSON data

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
| ❌ AI processing of JSON data | AI only interprets drill text for geometry |
| ❌ AI validation of JSON patterns | Application code handles all JSON validation |
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
- **Setup text:** "Four cones numbered 1 to 4 arranged in a square, 10 metres apart, plus a central player"
- **AI output:** Coordinates for nodes 1-4 in square formation
- **JSON:** Pattern cycles colors across nodes 1-4 in several phases
- **Application:** Matches JSON nodes 1-4 with AI coordinates and animates

### 15.2 Dribble with the Ball Through Chaos
- **Setup text:** "12 EyeQ SmartCones numbered 1 to 12 placed around the perimeter of a 40m x 30m pitch"
- **AI output:** Coordinates for nodes 1-12 around perimeter
- **JSON:** Phases light different cones using Red, Green, Yellow, Blue, Black
- **Application:** Matches JSON nodes 1-12 with AI coordinates and animates

### 15.3 Scan to Strike
- **Setup text:** "Four mini-goals on a 35 x 25 metre area, with EyeQ cones numbered 1-4 positioned behind each goal"
- **AI output:** Coordinates for nodes 1-4 behind goal positions
- **JSON:** One cone lights up per phase while others remain dark (indicating active goal)
- **Application:** Matches JSON nodes 1-4 with AI coordinates and animates

### 15.4 Testing Objective
System should be able to:
1. Send drill text to AI (never JSON)
2. Receive geometric layout from AI
3. Parse JSON with application code
4. Match JSON node IDs with AI-provided coordinates
5. Display animated 2D view
6. Allow editor to visually assess if pattern behavior matches drill intent

---

## 16. Architecture Summary

### 16.1 Clear Separation of Concerns

**AI Service:**
- Input: Drill text ONLY
- Output: Geometric layout data (coordinates, mappings, placement descriptions)
- Role: Interpret written setup descriptions to determine physical cone positions

**Application Code:**
- Input: Pattern JSON (never sent to AI)
- Processing: Parse JSON, extract node IDs, validate structure and duration consistency
- Bridge: Match JSON node IDs with AI-provided coordinates
- Output: Animated visualization with colors and timing from JSON

**Human Editor:**
- Provides both drill text and JSON as inputs
- Watches animation to judge if pattern makes sense
- Manually edits JSON if pattern needs adjustment

### 16.2 Data Flow

```
1. Editor inputs Drill Text + Pattern JSON
2. Drill Text → AI Service → nodeCoordinates
3. Pattern JSON → Application Parser → node IDs, colors, timing
4. Application matches node IDs with coordinates
5. Application animates pattern on positioned nodes
6. Editor assesses animation
7. If needed: Editor edits JSON → repeat from step 3
```

**Critical:** JSON never flows to AI. Text never needs to describe colors or timing.


**Status:** Draft for development handover

**Major revision:** Architecture clarified - AI processes text only, application code handles all JSON processing
