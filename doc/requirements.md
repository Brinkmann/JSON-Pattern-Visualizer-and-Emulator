EyeQ Drill Validator & Visualiser – Software Requirements

Purpose and scope

The goal is to build a simple web-based tool that helps an exercise designer visualize their drill setup and see the EyeQ JSON pattern play out on a 2D pitch. The tool does not generate or create EyeQ JSON patterns from scratch. Both the drill description and its corresponding JSON pattern are provided by the editor as pre-existing inputs.

The tool's primary function is to interpret the drill description to determine the geometric placement of nodes on a pitch, and then animate the provided JSON pattern on those positioned nodes. The editor can then visually assess whether the pattern behavior makes sense for their drill. If adjustments are needed, the editor manually edits the JSON code and re-runs the animation.

The tool is for single-user, ad hoc use. Nothing needs to be saved on the server; the editor keeps content on screen so the user can copy and paste it into their own systems.

User and usage overview

Primary user: single role called "editor", typically an exercise designer or coach.

Usage flow in plain terms: the editor pastes a drill description and its JSON pattern into one page; the system uses an AI service to interpret the setup description and determine where nodes should be placed on the pitch; once the geometric layout is established, the system shows a 2D pitch with numbered nodes and animates the JSON pattern; the editor watches the animation to verify the pattern behavior matches their drill intent; if the pattern needs adjustment, the editor manually edits the JSON and re-runs the animation.

Inputs

The main screen exposes two text inputs:

Drill text, which is a plain text description that includes at minimum a title, objective, setup and instructions. It may also include coaching points and variations. Examples are "Pass to the cone matching the pass-from colour", "Dribble with the ball through chaos", and "Scan to strike". The setup section is critical as it describes the physical arrangement of cones.

Pattern JSON, which is a single JSON object in the same style as the examples. The tool assumes a valid JSON structure and does not need to perform deep schema validation beyond basic parsing.

Pattern JSON structure (lightweight contract)

The system assumes the following general pattern, based on the examples:

Top-level object with a comment field and one pattern object, for example pattern_1 or pattern_custom.

Inside the pattern object:
– duration: total nominal duration in seconds (integer).
– phases: array of phase objects.

Each phase object:
– phase: phase number (integer).
– nodes: array of node objects.

Each node object:
– node: node id (integer, starting at 1).
– color: colour name string (for example Red, Blue, Green, Yellow, Black).
– secs: duration in seconds for which that node is active in this phase (integer).

The tool must support up to 18 distinct node ids. Behaviour for patterns using more than 18 nodes does not need to be specified in detail and can be treated as invalid input.

The tool does not need to validate that the top-level duration field matches the sum of secs values. However, the tool must validate that each node has the same total duration across all phases. For example, if node 1 appears in three phases with secs values of 4, 6, and 2 (totaling 12 seconds), then every other node must also total 12 seconds when summing its secs values across all phases. If this validation fails, the system should display a clear error message indicating which nodes have mismatched totals and prevent visualization until the JSON is corrected.

High-level flow

Single page or simple two-step flow; implementation can choose what is easiest.

At minimum, the flow is:

1. Input and AI-assisted interpretation of setup description.
2. Clarification of geometric details (if needed).
3. 2D visualisation and animation, with live editing.

It is acceptable to keep all three stages on a single page with clearly separated areas, or to split into an "Input and interpret" view and a "Visualise and edit" view. Visual complexity should be kept low; outcome and clarity are more important than a sophisticated user interface.

AI-assisted interpretation of setup and layout

When the editor presses a "Generate Layout" or similar action, the system sends both text blocks to an AI service:

Drill text as one string.

Pattern JSON as a string (not as parsed data) so the AI has access to raw node ids, colours and phases.

The AI is prompted to return a single, well-formed JSON object with at least these elements:

totalNodesFromText: integer or null; editor's intended number of physical cones if clearly stated in the drill text, otherwise null.

totalNodesFromJSON: integer; number of distinct node ids found in the pattern.

nodeMappings: structured mapping between zones or functional groups and node ids, for example "Zone 1" with nodes [1,2,3], or "Sideline North" with nodes [1–3]. When no zones or groupings are described, this may be an empty array.

geometricPlacement: description of layout rules such as arrangement type (for example square, line, perimeter, behind goals), spacing, pitch dimensions and any zone boundaries.

nodeCoordinates: array of objects that assign each node id an (x, y) coordinate suitable for 2D plotting on a pitch. Coordinates are in an arbitrary but consistent unit, with origin at one corner of the pitch.

clarificationPrompts: array of strings with human-readable questions when the AI cannot confidently determine geometric details from the setup description.

The AI is allowed to infer reasonable geometric placement when the description gives enough hints, for example:

Four cones arranged in a square with 10 metres between each cone, plus a central player, mapped to nodes 1 to 4.

Twelve cones numbered 1 to 12 distributed around the perimeter of the pitch in order.

Four cones behind four mini-goals on a 35 x 25 metre pitch.

When the drill text is explicit about pitch dimensions or arrangement ("35m x 25m", "arrange four cones in a square, 10 metres apart"), the system should prefer those details and use them deterministically for coordinate generation where possible, rather than relying purely on AI guesswork.

The AI's role is to interpret the setup description and produce a sensible geometric layout. The AI does not validate whether the JSON pattern is "correct" or "appropriate" for the drill - that judgment is left to the human editor who will watch the animation.

Automatic validation checks and editor interaction

After an AI response is received, the system applies automatic validation checks that must pass before visualization can proceed.

Node duration consistency

The system calculates the total duration for each node by summing all secs values for that node across all phases.

If the totals differ between nodes, the system stops and shows a clear error message listing which nodes have inconsistent totals. For example: "Node duration mismatch: Node 1 totals 12 seconds, but Node 3 totals 15 seconds. All nodes must have the same total duration across all phases."

The editor cannot proceed to visualisation until the pattern JSON has been manually corrected in the text area. There is no option to "ignore and continue" here.

Node count consistency

The system compares totalNodesFromText (when present) and totalNodesFromJSON.

If both values are present and differ, the system stops and shows a clear message that node counts do not match. The editor cannot proceed to visualisation until they manually correct the pattern JSON in the text area to align with the intended number of cones, or update the drill text. There is no option to "ignore and continue" here.

If totalNodesFromText is null but totalNodesFromJSON is present, the system may proceed, optionally mentioning that the count could not be determined from the description.

Setup clarification and refinement

If the AI cannot confidently produce nodeCoordinates from the drill text, or if nodeMappings is incomplete while the drill text references zones or special roles, the system displays clarification prompts.

The purpose of these prompts is to help the editor improve their setup description so that the geometric layout can be accurately determined. This is not about validating the JSON pattern, but about ensuring the nodes are placed correctly on the canvas to match the intended physical setup.

Typical queries might be:

"Your drill mentions 12 EyeQ SmartCones placed around the field. Please confirm which node numbers correspond to each side or zone, for example: Nodes 1–3 north sideline, 4–6 east sideline, 7–9 south sideline, 10–12 west sideline."

"The exact positions of the four cones are unclear. Please describe their arrangement, for example: 'Cones 1–4 form a square, 10 metres apart, centred in the pitch'."

The editor can answer these questions in a simple text input. The implementation can choose whether to re-call the AI with this extra context or apply a small local helper to generate or adjust nodeCoordinates.

The goal of this interaction is to iterate on the setup description until the AI can confidently place all nodes on the canvas in positions that match the editor's physical drill setup.

Visualisation and animation

Once node counts, node duration consistency and geometric placement are established, the system shows a 2D visualisation area.

Pitch and zones

The pitch is represented as a plain 2D rectangle, typically corresponding to dimensions found in the drill (for example 35 x 25 metres, 40 x 15 metres) but scaled to fit the screen.

Zones or key areas mentioned in the drill (such as "four zones", "sidelines", "behind goals") are drawn as simple boundaries or labels where the AI or deterministic logic can infer them.

Nodes and labelling

Each node is drawn as a small symbol, such as a triangle or circle, positioned at its (x, y) coordinate.

Each node must display its node id (1, 2, 3, etc.) clearly enough to be readable on desktop and tablet screens.

The symbol's fill colour is driven by the currently active phase in the pattern JSON.

Animation of phases

The system loops through phases in the order given in the JSON.

For each phase, it reads nodes and applies the colour for each node; nodes where colour is "Black" can be rendered as off, dark or neutral.

Each phase runs for an effective duration based on secs and the current speed multiplier. A simple approach is to use the maximum secs value in that phase as the phase length, or to use an average or sum, as long as the behaviour is consistent and documented for development.

After the final phase, the animation loops back to phase one.

The editor watches this animation to assess whether the pattern behavior makes sense for their drill. If it does not, the editor manually edits the JSON pattern and re-runs the animation.

Interactive controls

Controls near the visualisation include at least:

A play or pause toggle to start or stop the animation loop.

A speed control that applies a global multiplier to all secs values when calculating phase timings, with supported multipliers 0.5, 1.0, 1.5 and 2.0. For example, at 2.0, a node with secs 4 will be displayed for the equivalent of 2 seconds.

The speed control should not change the JSON itself; it only affects the visual playback.

Live editing behaviour

On the visualisation screen, the current pattern JSON is always visible in a text area. The editor can change values directly and then trigger an update.

Two types of change are supported.

Simple live tweak

Simple tweaks include changes to colour values and secs values, without adding or removing nodes, phases or pattern keys.

When the editor makes such tweaks and presses "Update animation" (or similar), the system:

Parses the JSON and, if successful, validates node duration consistency.

If validation passes, applies the new values, keeps the existing nodeCoordinates and layout, and restarts the animation from phase one using the new secs and colours.

If node duration consistency validation fails, displays an error message without updating the animation.

No AI call is required for this re-run. Errors in basic JSON parsing can be shown as a simple message ("Cannot parse JSON, please check commas and brackets") without extra behaviour.

Structural change

Structural changes include adding or removing nodes, adding or removing phases, or changing anything that affects node count or mapping, including changes to drill text.

When the editor indicates that a structural change has been made, or presses a "Re-interpret setup" button, the system:

Re-parses the JSON.

Calls the AI again with the latest drill text and pattern JSON.

Re-applies the automatic validation checks and clarification process described earlier.

Potentially generates new nodeCoordinates and may ask for further clarifications about the setup.

Only after this process completes does the system return to the visualisation step.

The user interface must make it clear which action is a quick animation update and which action triggers a full re-interpretation of the setup. It is acceptable to label the two buttons differently, for example "Update animation only" and "Re-interpret setup and layout".

Stretch goal: Additional visual elements for documentation

Purpose and priority

This feature is a stretch goal and secondary to the primary objective of visualizing drill setup and animating JSON patterns. The primary goal is to interpret the setup description, place nodes correctly, and animate the pattern so the editor can assess whether it makes sense. This stretch goal enhances the visualization for documentation purposes only.

Requirement

The editor should be able to manually place additional visual elements on the visualization canvas to create a more complete drill setup diagram. These elements complement the numbered node triangles but are not part of the setup interpretation or JSON validation process.

Element types

The system should support drag-and-drop placement of at least the following element types:

Black round markers or dots, to represent additional reference points, boundary markers or non-EyeQ cones.

Goal icons, to represent mini-goals or full-size goals.

Player icons, to represent starting positions or key positions for players.

The exact visual design of these icons is flexible, but they must be clearly distinguishable from the numbered node triangles that represent EyeQ SmartCones.

Interaction model

The editor uses a simple toolbar or palette to select an element type, then clicks or drags to place it on the canvas.

Placed elements can be repositioned by dragging them to a new location.

Placed elements can be deleted, either by dragging them off the canvas, clicking a delete button, or using a similar simple mechanism.

These elements are not inferred from the drill text or JSON pattern. The editor places them manually based on their own knowledge of the drill setup.

Relationship to JSON and animation

These additional elements have no relationship to the JSON pattern or the setup interpretation process. They exist purely as visual documentation aids.

Adding, moving or removing these elements does not trigger re-interpretation or affect the animation in any way.

These elements remain static on the canvas during animation; they do not change colour or respond to phases.

Screenshot and documentation use case

The primary use case for this feature is to allow the editor to create a comprehensive setup diagram that shows both the interpreted EyeQ cone layout (with numbered nodes) and other drill elements such as goals and player positions.

The editor can then capture a screenshot of this enhanced visualization to use alongside the drill description in documentation, training materials or drill libraries.

Implementation notes

This feature is explicitly marked as a stretch goal. If time or resources are limited, the core setup interpretation and animation functionality takes absolute priority.

The placement and storage of these elements can be handled entirely in-browser memory. There is no requirement to persist or save these placements; if the editor refreshes the page, the additional elements can be cleared without issue, as the primary output is a screenshot taken by the editor.

If implementing this feature proves straightforward, it adds significant practical value. If it introduces complexity or delays the core functionality, it should be deferred to a future version.

Non-functional and technical notes

The tool must run in a modern browser. Desktop and tablet use are the primary targets; mobile optimisation is not required.

The pitch and nodes can be rendered using canvas, SVG or any other simple 2D approach. Choice of technology is up to the implementation team.

Back end functionality is minimal. It must accept text inputs, call the AI parsing service and return the AI's structured response to the front end. No server-side persistence is required.

Any debug logging of AI calls or parsing behaviour is optional and is intended purely for development and support. It does not need to be exposed to the editor in the user interface.

Out-of-scope features

The following features are explicitly out of scope for this version:

3D visualisation or camera views.

Audio cues or sound effects.

Multi-user collaboration, accounts or role management.

Persistent drill storage, saving patterns to a server or file download. Keeping the editor fields on screen for copy and paste is sufficient.

Automatic validation of whether a JSON pattern is "appropriate" or "correct" for a given drill. The tool interprets setup descriptions and animates patterns; the human editor judges whether the pattern behavior makes sense.

Automatic inference or AI-assisted placement of additional visual elements such as goals or players from the drill text. The stretch goal feature allows manual placement only.

Generation or creation of EyeQ JSON patterns from drill descriptions. The tool only visualizes pre-existing JSON patterns provided by the editor.

Example drills for reference

The following drills serve as realistic examples of inputs for this tool and can be used for test cases:

Pass to the cone matching the pass-from colour, with four cones arranged in a square, a central player and a JSON pattern that cycles colours across nodes 1 to 4 in several phases.

Dribble with the ball through chaos, with 12 EyeQ SmartCones numbered 1 to 12 placed around the perimeter of a larger pitch, and JSON phases that light different cones using colours such as Red, Green, Yellow, Blue and Black.

Scan to strike, with four mini-goals and four EyeQ cones positioned behind each goal on a 35 x 25 metre area, and a pattern where one cone lights up for a period while others remain dark, reflecting which goal is active at that moment.

The system should be able to take the text and JSON from each of these examples, interpret the setup description to generate a node layout, and display a simple animated 2D view that allows an editor to visually assess whether the pattern behavior matches their drill intent.
