# Ursus engraving object model
## Foundation draft · schema 0.1

**Purpose.** Define the editable graphic model for an engraving plate: what objects exist, how they are contained, how items attach to one another, and how the renderer resolves notation glyphs and styles.

**Scope.** Ursus is a visual engraving system. The model stores placed shapes, glyphs, text, and their graphic relationships. It has no playback, MIDI, rhythmic timeline, or audio behavior.

## 1. Working terms

- **Plate** — the editable, paper-sized working surface and the root of the object tree.
- **Page** — a finalized rendering of a plate, such as an SVG or PDF export. A page is an output artifact; the editable source remains the plate.
- **Object tree** — parent/child ownership. It controls containment, selection scope, and lifecycle.
- **Attachment** — a reference from one object to an anchor on another object. Attachments are not ownership and may cross staves.
- **Anchor** — a named or explicit point used to position an attached item or a span endpoint.

A system can contain one or more staves. A plate can contain one or more systems and also items positioned directly on the paper.

## 2. Hierarchical tree

```text
Plate
├── Systems [1..n]
│   ├── Staff [1..n]
│   │   ├── Staff-attached items
│   │   │   ├── Clef
│   │   │   ├── KeySignature
│   │   │   ├── TimeSignature
│   │   │   ├── RehearsalMark / StaffText / Dynamic
│   │   │   └── NoteCluster
│   │   │       ├── Notehead [1..n]
│   │   │       │   ├── Accidental (local to this notehead)
│   │   │       │   ├── Articulation / Ornament
│   │   │       │   └── LaissezVibrerTie (one endpoint, left or right)
│   │   │       └── Stem (optional; may serve several noteheads in a chord)
│   │   │           └── Flag (optional; normally omitted when beamed)
│   │   └── Staff-local vector graphics
│   ├── StaffGroup [0..n] — brace or bracket over one or more staves
│   ├── SystemBarline [0..n] — owns its staff-span configuration
│   └── System spans [0..n]
│       ├── BeamGroup
│       ├── Slur / Tie
│       ├── Hairpin
│       ├── Glissando
│       ├── OttavaLine / TrillLine / PedalLine / LyricExtender
│       └── VoltaBracket / other line or bracket spans
├── PlateText [0..n]
└── PlateGraphic [0..n]
```

### Why NoteCluster is included

A single note can be represented by a cluster with one notehead and an optional stem. A chord can use several independently editable noteheads with one shared stem. The cluster is only a graphic grouping: it has no duration, pitch, voice, or playback meaning. Each notehead still owns its local accidental, articulation, ornament, or one-ended laissez-vibrer tie.

### Ownership versus attachment

The tree above describes ownership. It does not limit where an object may attach. A beam can be owned by a system while its endpoints refer to stems on different staves; a slur can similarly reference noteheads on different staves. A barline is system-owned because its configured span can cover one or more staves. Store cross-object links as IDs and anchor names, never by moving an attached object under an arbitrary endpoint's parent.

## 3. Object roles and minimum data

| Object | Owns / contains | Useful editable properties |
| --- | --- | --- |
| **Plate** | Systems and plate-level objects | Paper width/height, orientation, units, margins, background, style profile |
| **System** | Staves, staff groups, system barlines, and system spans | Position, width, order, optional local transform |
| **Staff** | Staff-attached items and staff-local graphics | Line count, line spacing, length, line appearance, position |
| **Staff item** | Its own subobjects where needed | Role, position/transform, glyph or text, appearance |
| **NoteCluster** | Noteheads and optional shared stem | Position, local grouping, stem direction/geometry |
| **Notehead** | Local notehead attachments | SMuFL glyph, position, scale, rotation |
| **Stem** | Optional flag | Start anchor(s), endpoint, width, appearance |
| **StaffGroup** | Two endpoint records | Brace/bracket kind, member staff IDs, endpoint staff anchors, appearance |
| **SystemBarline** | No child objects; references staff groups or staves | Horizontal position, barline style, scope mode and scope references |
| **System span** | Start/end and optional intermediate anchors | Span kind, control points or segments, line/taper geometry, appearance |
| **PlateText** | None | String, text font, alignment, bounds, rotation |
| **PlateGraphic** | Optional path segments | Primitive/path geometry, fill, stroke, transform |

Every object should have a stable `id` and `type`. Child nesting implies `parentId`; references such as beam endpoints use `targetId`. Common optional fields are `transform`, `visible`, `locked`, `styleRef`, `styleOverrides`, `zIndex`, and `extensions`.

`locked` and selection state are editing concerns, not engraving semantics. Selection state should not be serialized into a finalized page.

## 4. Attachment and endpoint model

An attachment records **what it points to** and **where on it**. A small v0.1 anchor record can be:

```json
{
  "targetId": "notehead-12",
  "anchorName": "center",
  "offsetMm": [0.4, -0.2]
}
```

For an explicit point on a staff, use a staff anchor instead:

```json
{
  "staffId": "staff-2",
  "positionMm": [72.0, 8.0]
}
```

The values above are relative to the target object's or staff's local coordinates. A later version can add richer named anchors, font-provided glyph anchors, and snapping rules without changing the basic reference model.

| Relationship | Anchors and constraints |
| --- | --- |
| **Brace / bracket** | Two staff anchors. Both may target the same staff; that draws a connector confined to one staff. |
| **BeamGroup** | References two or more stems. References may target stems on different staves. Geometry may be a straight segment or a set of editable beam segments. |
| **Slur / tie** | Start and end anchors commonly target different noteheads. Cross-staff targets are allowed. For v0.1, require distinct notehead IDs for a conventional slur or tie. |
| **Hairpin** | Start and end anchors may target the same note or different notes. |
| **Glissando / other line span** | Start and end anchors may target notes, staff positions, or explicit points, including targets on different staves. |
| **Laissez-vibrer tie** | One notehead anchor plus a direction (`left` or `right`); it is not a two-ended tie. |

A span endpoint can be stored as a small owned record under the span, with an attachment reference to a staff or notation object. It is a non-drawing anchor record; the span owns the visible curve or line.

### Barline span configuration (model support; resolver deferred)

A `SystemBarline` belongs to its system, not to a single staff, because one barline may cover one staff or several. Its horizontal position and visual kind (single, double, repeat, final, and so on) are separate from its vertical scope.

A barline's optional `scope.mode` has three values:

- `followStaffGroups` — derive barline segments from one or more `StaffGroup` objects, which represent braces or brackets. If `staffGroupIds` is omitted, use all groups in the system. `includeUngroupedStaffs` defaults to `true` and can be set to `false` to omit staffs outside those groups.
- `explicitStaffSpans` — use `spans`, an array of independently designated spans. Each span contains an ordered `staffIds` list with one or more staves, so a segment can be limited to one staff or bridge multiple staves.
- `inheritPrevious` — inherit the effective scope mode and its group or staff-span selection from the immediately preceding barline in the system.

If the first, left-edge barline has no scope (or uses `inheritPrevious`), its default is the full system. A later barline with no scope defaults to `inheritPrevious`. An explicit scope overrides that default and becomes the source configuration for a later inherited barline. Inheritance applies to vertical scope only; each barline keeps its own position and visual style. Represent compound strokes such as double or repeat barlines as one `SystemBarline` with a compound style so they share one scope.

This data model makes both grouping modes and predecessor inheritance representable now. The renderer's scope resolution and segment-geometry generation are implementation-stage work; the current prototype only needs to preserve and validate the configuration fields and references.

Example with both scope modes, a multi-staff explicit span, and a barline inheriting its predecessor's configuration:

```json
{
  "id": "system-scope-example",
  "type": "System",
  "children": [
    { "id": "staff-1", "type": "Staff" },
    { "id": "staff-2", "type": "Staff" },
    {
      "id": "group-1",
      "type": "StaffGroup",
      "kind": "brace",
      "staffIds": ["staff-1", "staff-2"]
    },
    {
      "id": "barline-start",
      "type": "SystemBarline",
      "xMm": 0
    },
    {
      "id": "barline-1",
      "type": "SystemBarline",
      "xMm": 80,
      "scope": { "mode": "followStaffGroups", "staffGroupIds": ["group-1"] }
    },
    {
      "id": "barline-2",
      "type": "SystemBarline",
      "xMm": 120,
      "scope": {
        "mode": "explicitStaffSpans",
        "spans": [{ "staffIds": ["staff-1", "staff-2"] }]
      }
    },
    {
      "id": "barline-3",
      "type": "SystemBarline",
      "xMm": 160,
      "scope": { "mode": "inheritPrevious" }
    }
  ]
}
```

### Validation rules for v0.1

1. IDs are unique within a plate and every reference resolves.
2. Parent ownership is acyclic. Attachment references do not create ownership cycles.
3. A brace/bracket has exactly two staff endpoints; same-staff endpoints are valid.
4. A beam has at least two stem references; each referenced object is a stem.
5. A conventional slur or tie has two notehead endpoints; a hairpin permits coincident endpoint targets.
6. A one-ended tie has one notehead anchor and an explicit direction.
7. Cross-staff references are valid. No rhythmic or temporal ordering is inferred from them.
8. A `SystemBarline` belongs to one system and may reference only that system's staves and staff groups.
9. An explicit barline span has at least one valid staff ID; a multi-staff span is valid.
10. A first/left-edge barline with omitted scope resolves to full-system scope; a later `inheritPrevious` scope resolves from the preceding barline. The resolver itself is deferred.

## 5. Coordinates and rendering

- Store page geometry in **millimeters**, with the plate origin at the top-left, x increasing rightward and y increasing downward.
- Each child has a local transform relative to its parent. Resolve the transform chain to plate coordinates for rendering.
- A staff defines its own line count, line spacing, line length, and position. Five lines are the default, not a restriction.
- Convert SMuFL font metrics from staff-space coordinates into plate millimeters using the target staff's line spacing. Preserve the original font metrics for precision; round only at the output rasterization or device stage.
- Keep geometry and appearance separate. For example, a slur owns endpoints and curve control points; its stroke thickness, color, and taper are appearance.
- Render order is explicit through `zIndex` or a stable default ordering: staff lines and system barlines, staff symbols and noteheads, stems/beams, spans, then text and plate-level graphics. Local overrides can change this order.

The prototype can place objects explicitly and render their geometry without attempting automatic collision avoidance, rhythmic spacing, or score reflow. Those can be added later as optional layout passes over the same object tree.

## 6. SMuFL glyph and font model

A notation glyph is stored by **font reference plus canonical glyph name**, not as an unlabelled codepoint:

```json
{ "fontId": "bravura", "glyphName": "noteheadBlack" }
```

The font registry should identify the font family, font version, SMuFL version, and the matching font-specific metadata file. At render time, resolve the canonical glyph name to the font's glyph mapping, then use available font metadata for glyph-specific anchors and metrics. A missing glyph should remain identifiable in the model so it can be repaired or rendered as a visible placeholder.

SMuFL defines canonical glyph names and font-specific JSON metadata; that metadata can carry glyph anchors (including notehead stem positions) and engraving defaults such as staff-line, stem, beam, slur, tie, and hairpin thicknesses. Treat those values as starting defaults that engravers can override at profile or object level. See the references below.

## 7. Customization model

Resolve appearance in this order, from broadest to most specific:

1. **Renderer fallback** — only when no other value is available.
2. **Font metadata defaults** — where a chosen SMuFL font supplies them.
3. **Plate engraving profile** — user-defined defaults for staff spacing, line weights, glyph scale, beam spacing, slur/tie thickness, hairpin spread, text fonts, and other graphic properties.
4. **Object style reference** — a reusable named style, if present.
5. **Per-object overrides** — exact local values; these always win.

Keep every object editable even when it uses a profile default: the renderer resolves the appearance without baking default values into each object's geometry. The profile should support unknown future properties through namespaced extension keys.

## 8. Suggested serialized document

Use **JSON** for the first interchange format: it is readable, easy to validate, supports stable object IDs and cross-references, and maps directly to the nested model. Keep a schema version at the root. A small starter example follows; it intentionally shows a system-owned beam that connects stems on two different staves.

```json
{
  "schemaVersion": "0.1.0",
  "fonts": [
    {
      "id": "bravura",
      "family": "Bravura",
      "fontVersion": "example-font-version",
      "smuflVersion": "1.4",
      "metadata": "Bravura.json"
    }
  ],
  "styles": [
    {
      "id": "engraving-default",
      "type": "EngravingProfile",
      "values": {
        "staffLineThicknessMm": 0.25,
        "stemThicknessMm": 0.35,
        "beamThicknessMm": 1.0
      }
    }
  ],
  "plate": {
    "id": "plate-1",
    "type": "Plate",
    "format": { "widthMm": 210, "heightMm": 297, "orientation": "portrait" },
    "origin": "top-left",
    "styleRef": "engraving-default",
    "children": [
      {
        "id": "system-1",
        "type": "System",
        "transform": { "xMm": 18, "yMm": 32 },
        "children": [
          {
            "id": "staff-1",
            "type": "Staff",
            "lineCount": 5,
            "lineSpacingMm": 2,
            "lengthMm": 174,
            "children": [
              {
                "id": "clef-1",
                "type": "GlyphItem",
                "role": "clef",
                "transform": { "xMm": 4, "yMm": 4 },
                "glyph": { "fontId": "bravura", "glyphName": "gClef" }
              },
              {
                "id": "cluster-1",
                "type": "NoteCluster",
                "transform": { "xMm": 42, "yMm": 4 },
                "children": [
                  {
                    "id": "head-1",
                    "type": "Notehead",
                    "glyph": { "fontId": "bravura", "glyphName": "noteheadBlack" }
                  },
                  {
                    "id": "stem-1",
                    "type": "Stem",
                    "startAnchors": [
                      { "targetId": "head-1", "anchorName": "stemUpSE" }
                    ],
                    "endMm": [0, -7],
                    "widthMm": 0.35,
                    "children": [
                      { "id": "flag-1", "type": "Flag", "glyph": { "fontId": "bravura", "glyphName": "flag8thUp" } }
                    ]
                  }
                ]
              }
            ]
          },
          {
            "id": "staff-2",
            "type": "Staff",
            "transform": { "xMm": 0, "yMm": 36 },
            "lineCount": 5,
            "lineSpacingMm": 2,
            "lengthMm": 174,
            "children": [
              {
                "id": "cluster-2",
                "type": "NoteCluster",
                "transform": { "xMm": 54, "yMm": 4 },
                "children": [
                  {
                    "id": "head-2",
                    "type": "Notehead",
                    "glyph": { "fontId": "bravura", "glyphName": "noteheadBlack" }
                  },
                  {
                    "id": "stem-2",
                    "type": "Stem",
                    "startAnchors": [
                      { "targetId": "head-2", "anchorName": "stemUpSE" }
                    ],
                    "endMm": [0, -7],
                    "widthMm": 0.35
                  }
                ]
              }
            ]
          },
          {
            "id": "beam-1",
            "type": "BeamGroup",
            "anchors": [
              { "targetId": "stem-1", "anchorName": "tip" },
              { "targetId": "stem-2", "anchorName": "tip" }
            ],
            "segments": [
              { "from": 0, "to": 1, "thicknessMm": 1.0, "offsetMm": [0, 0] }
            ]
          }
        ]
      },
      {
        "id": "title-1",
        "type": "PlateText",
        "transform": { "xMm": 105, "yMm": 18 },
        "text": "Ursus engraving prototype",
        "alignment": "center",
        "textStyle": { "fontFamily": "serif", "fontSizePt": 12 }
      }
    ]
  }
}
```

The JSON is an illustrative seed, not a complete formal schema. The first implementation should validate the rules above and preserve unknown fields under `extensions`, so later object types can be introduced without breaking older plates.

## 9. Prototype boundary and growth path

**Represent now:** plate dimensions, systems, configurable staves, glyph items, note clusters, text, vector lines/curves, local styles, stable IDs, attachments, cross-staff spans, and both barline scope modes plus predecessor-inheritance fields.

**Defer to implementation:** barline scope resolution and segment geometry, automatic engraving layout, collision detection, score-wide spacing, semantic pitch/duration, playback, MIDI, and audio. The current prototype should preserve the barline configuration without implementing its rendering behavior.

A useful initial renderer can therefore be built in four layers: (1) parse and validate the JSON tree, (2) resolve IDs and transforms, (3) resolve font glyphs and effective styles, and (4) draw staff geometry, glyphs, paths, and text into an exportable page.

## References

- [SMuFL specification and current editor's draft](https://smufl.formats.music/latest/) — canonical glyph mapping and specification status.
- [SMuFL font-specific metadata](https://smufl.formats.music/latest/specification/font-specific-metadata.html) — font metadata format, canonical glyph names, and staff-space measurements.
- [SMuFL glyph anchors](https://smufl.formats.music/latest/specification/glyphswithanchors.html) — optional glyph-specific anchors and metrics.
- [SMuFL engraving defaults](https://www.w3.org/2021/03/smufl14/specification/engravingdefaults.html) — recommended engraving measurements expressed in staff spaces.
