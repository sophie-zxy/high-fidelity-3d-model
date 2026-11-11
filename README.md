A practical guide for prompting an AI agent

# How to create a better, high-fidelity 3D model

Why AI-built models come out as stacked spheres with googly eyes — and the exact pipeline, numbers, and wording to give an agent instead, so what comes back is cute, smooth, and actually finished. Written to be handed to an agent as-is: the copy-paste prompt is in §9.

For: anyone prompting Claude / an AI agent to build in BlenderWorks for: any cute character — the running example is a sitting catResearch checked: 10 Nov 2026

**Left:** assembled primitives — every part keeps its own perfect shape, so every seam shows. **Right:** one fused form — no seams, no circles, the silhouette reads as a single animal. That difference is the whole article. Illustration, not a render.

Contents

1 · Why blobs look ugly 2 · The 5-stage pipeline 3 · Pick a technique 4 · What makes it cute 5 · Fix the 4 classic mistakes 6 · Fur & colour 7 · Driving the AI agent 8 · Start from a base mesh 9 · The copy-paste prompt 10 · Honest limits 11 · Done checklist Jump to the prompt ↓

## The short version

**Don't stack spheres. Fuse them.** Build one low-poly base mesh, fuse it into a single continuous skin (voxel remesh or metaballs), shape it with subdivision and sculpting in three passes — big forms, then anatomy, then detail — retopologise to clean quads if it will move, and drive all of it from a versioned Python script that your agent re-renders and _actually looks at_ after every step.

If your agent does only three things differently, make it these: **(1)** fuse the blockout into one mesh, **(2)** render a front / side / top / three-quarter contact sheet after every stage and judge the silhouette before adding detail, **(3)** never call a grey blockout “finished.”

## 1 · Why primitive blobs look ugly

Stacking a sphere for the head, a sphere for the body, cylinders for the legs and a cone for each ear feels efficient — each part is “done” in one line of code. It fails for three visible reasons:

  * **Every seam shows.** Two intersecting spheres meet in a hard pinch line that real skin never has. Your eye reads the joins, not the animal.
  * **Every part keeps a perfect silhouette.** Real heads aren't spheres — skulls are slightly flattened at the sides, cheeks are wider, muzzles project. A perfect circle in the outline is a giveaway.
  * **Parts don't pull on each other.** In a real body, the neck blends into the chest and the cheek blends into the muzzle. Separate objects can't blend.

> “A cat composed of geometric shapes, or a cat wearing plate mail armor” — the classic failure is disparate primitives that were never turned into a single unified skin, where vertices pull on each other like real skin.

— Robert Peake, on early cat walk-cycle modelling · [robertpeake.com](https://www.robertpeake.com/archives/149-early-cat-walk-cycle.html)

The test is simple and brutal: **look at the silhouette from front, side, and top. If you can still draw a circle around the head and another around the body, with a visible pinch between them, the fusion failed** — no amount of fur, texture, or lighting will rescue it.

## 2 · The five-stage pipeline

This is the pattern that human tutorials and the Claude-agent skill repos independently converge on. Each stage has a gate: don't move on until the render proves it.

1

### Block out proportions — one base mesh

Start from a **single** low-poly mesh: a round cube with a Subdivision Surface modifier at level 2, extruding and insetting faces for legs and ears and moving vertices with proportional editing (`O`). Or trace a side-view reference: extrude one plane along the silhouette, add Solidify at offset 0, then subdivide. Or extrude a single-vertex skeleton and give it volume with the Skin modifier (scale each vertex's radius with `Ctrl+A`). Add a Mirror modifier _before_ Subsurf for symmetry. This stage is only about proportions against reference photos — front and side orthos on screen, grey material, nothing else.

2

### Fuse it into one continuous mesh

The step that kills the “stacked spheres” look. Join the overlapping volumes (`Ctrl+J`) and run **Voxel Remesh** (`bpy.ops.object.voxel_remesh()`) — it rebuilds everything as one watertight, uniform mesh. Raise the voxel resolution by no more than ~1.5× per step. Alternative: build the blockout as **metaballs** — head, cheeks, muzzle, and body as one signed-distance field that blends smoothly by definition — then Convert to Mesh and remesh. Either way, you end this stage with one skin, zero seams.

3

### Shape it: primary → secondary → tertiary

Work big to small, and never detail a form that isn't right yet — “most bad sculpts are detailed too early on forms that were never right.” **Primary:** the big masses with Grab / Clay / Snake Hook, or scripted `bmesh` vertex offsets for cheeks, muzzle, chest, and haunches. **Secondary:** anatomy — muscle, folds, the muzzle-and-chin volume. **Tertiary:** fine detail only at the end. Increase resolution (Multires levels) only when the current density blocks the form you're trying to make.

4

### Retopologise to clean quads — if it will move

A voxel or Dyntopo mesh is triangles: fine for a still render, unusable for animation — triangles and n-gons pinch, break loop cuts, and deform badly. Build a clean quad mesh over the sculpt: QuadRemesh (`bpy.ops.object.quadriflow_remesh()`) for a first pass, then manual cleanup, or RetopoFlow (paid). Targets from the sources: **base mesh ≤5,000 quads with ≤4 Multires levels** for games; **5–15k quads** for a game hero character, **20–50k** for film. Place loops deliberately: **≥3 concentric loops around each eye socket, ≥3 elliptical loops around the mouth,** and loops perpendicular to every joint's rotation axis. Subdivision (Catmull-Clark) multiplies faces by 4 per level — it amplifies whatever topology you give it, good or bad.

5

### Drive it from a script — and verify every stage

One deterministic entry script (e.g. `build_cat.py`) with fixed seeds and every dimension as a named constant at the top. The script is the source of truth; Blender is just the runtime. After _every_ stage, render a multi-angle contact sheet and look at it before continuing. Full detail on this workflow is in §7 — it's where AI agents succeed or fail.

Pipeline sources: Creative Bloq cat tutorial; Yarsa Labs; RobLe3/cc-blender-skill sculpting & retopo overview; expert-pack sculpting / multires notes; 73k-y 3D workflow topology reference — full links at the end.

## 3 · Pick a technique, not a pile of shapes

Each row below beats stacking primitives for a different reason. For a scripted / agent-built character, the most “code-shaped” route is the metaball blockout in row 5, fused and remeshed — distance fields blend without ever failing the way mesh booleans do.

Box modelling
    One cube → Subsurf → extrude / inset / proportional-edit. The mesh stays **one connected surface** , so there are no sphere-on-sphere seams to hide.

Silhouette extrusion
    Trace a reference photo, extrude a plane along the outline. Gets the **profile right in one stroke** — the single most important thing for recognisability. A blockout stage, not final topology.

Single-vertex + Skin
    Extrude a skeleton of single vertices, Skin gives each a radius, Subsurf smooths it. A fast, armature-like blockout for body, legs, and tail.

Sculpting
    Primary → secondary → tertiary forms (see Stage 3). The honest caveat, repeated by several sources: **an agent cannot sculpt by brush judgement headlessly** — it should script the setup, base mesh, and proportions, and leave fine sculpting to a human pass or a good base mesh.

SDF / metaballs
    Shapes defined by distance functions, blended without boolean failures. Blender's metaballs are a live-meshed SDF; Blender 5.1+ also adds SDF Smooth remesh. Best scripted route to a **unified organic form**.

Voxel Remesh / Multires
    Voxel Remesh fuses; Multires preserves clean quad base topology while detail lives on levels — bake a normal map from it. **Never ship a raw Dyntopo or voxel mesh for animation.**

Procedural: bmesh + Geometry Nodes
    Excellent for **parametric, repeatable** parts: whisker and fur scattering, tail taper, patches, ears and paws via `bmesh` lathe / extrude. Its ceiling is stated explicitly in the sources: scripting “struggles with bespoke organic forms (creatures, faces…). That's Sculpt Mode territory… with no real headless path.” Script the scaffold — don't expect nodes alone to produce a cute face.

Lattice deformation
    A non-destructive squash / stretch that survives rotation — the standard trick for flattening an eye sphere front-to-back while it stays riggable. Parent the lattice to the head bone.

## 4 · What actually makes it cute

Cute is mostly proportions and restraint, decided at the blockout stage — not detail added later. For a sitting character (the cat is the worked example):

Body

### Wide base, upright chest

Sitting = haunches wide at the base, chest upright, front legs as straight columns, paws flat on the floor. The head overlaps the chest _deeply_ — merge or sculpt the neck transition so there's no neck-sphere seam.

Head

### Not a sphere

Flatten the sides slightly (real skulls are squarer than you think), widen the cheeks, then build the muzzle as a _separate smaller fused volume_ — short, wide, pushed forward — with a small chin bump below it. Eyelids can start as spheres, then get merged and sculpted into the face (the Blender Studio “Rain” workflow).

Ears

### Extrude, taper, line up

Extrude the head's top faces, scale on Y, taper to a rounded triangle — or use separate meshes with Solidify + Subsurf. Inner ear: a slightly inset, thinner duplicate in pink. Placement rule: **top of the ear lines up with the eyebrows, bottom with the nose.**

Paws

### Fused, with grooves — not toe spheres

Extrude or inset the body's bottom faces, or fuse short tapered cylinders into the chest and flatten them at the floor. Add 2–3 shallow toe grooves with creases or loop cuts. Hind paws tuck at the sides of the haunches.

**An honest caveat on numbers:** no single authoritative “cute cat proportions” source was found in the research. The figures in §5 (five eye-widths, “scale the eyes down by half”) are triangulated from general character-art rules and tutorial practice — not a measured model sheet. Treat every proportion as a named constant in the script, and verify it against reference images via the contact sheet rather than trusting any one number.

## 5 · Fix the four classic mistakes

These are the four defects that make an agent-built character read as “ugly” at a glance. Each has a root cause — fix the cause, not the symptom.

Mistake 1 · Eyes

### Too large, googly, protruding

**Root cause:** scale + protrusion + the whole ball being visible. **Fixes, all sourced:**

  * **Scale them down — even by half.** Blender Studio's critique of its own character Rain: “The most noticeable element is the big round eyes. I think it would help a lot to scale them down as far as even half the size.”
  * **Five-eye-widths rule:** five eye-widths should fit across the face. Measure eye diameter against head width in the script — and assert it.
  * **Don't show the whole ball.** Sink the sphere into its socket so only a lens-shaped, almond portion shows, and overlap it with eyelids top and bottom.
  * **Flatten the ball** with a Lattice, front-to-back — non-destructive, still rotatable.
  * **Build the eye in layers:** sclera (barely visible on a cute cat), coloured iris disc, pupil slightly proud of the iris, tiny white highlight as its own mesh or shader. Recess the iris and pupil into the eye — never paint them on the bulge.

Mistake 2 · Whiskers

### Thin sticks in close-up

**Root cause:** constant-radius cylinders or curves with hard ends and no droop. **Fixes:**

  * **Taper to a point:** Bézier curve + bevel, per-point radius (`Alt+S`) — about **1.5–2  mm thick at the root** at cat scale, to ~0 at the tip. Or a teardrop taper object (Taper Start 0.90, Radius Max 0.10).
  * **Curve them:** 3-point Béziers with a gentle downward / outward arc and a slight S, fanned 3–4 per side in two rows — never straight radial spikes.
  * **Soften the material:** off-white with slight translucency, not pure-white hard plastic. In close-up, a thin curve _with a profile_ reads soft; a thin cylinder reads as a stick.
  * Alternatives: Hair Curves with a set profile, or flattened tapered mesh ribbons. Don't use particle hair for whiskers.

Mistake 3 · Tail

### Stiff, straight, sticking up

**Root cause:** a straight primitive with uniform radius and no curve. **Fixes:**

  * **Build it as a Bézier curve, not a cylinder:** 4–6 control points in an **S-curve** — out low from the back, sweeping along the ground, curling up or inward at the tip. A sitting cat wraps its tail around its paws or curls it beside its body. Never a vertical pole.
  * **Taper it:** thick at the base, soft point at the tip, via per-point radius on the curve.
  * **Rig it, don't freeze it:** a chain of **5–8 bones with Spline IK** fitted to that curve (Blender's manual: Spline IK “fits a chain of bones to the shape of a Curve… particularly well suited for… tails”), plus a hook / empty at the tip as a control. Advice for cat animation runs to 8–12 spine / tail bones with Spline IK or bendy bones — 2–4 bones produce a visible fold.
  * **Parametrise it:** in the script, the tail is a list of (position, radius) control points, so the curl is tuned by numbers and re-rendered.

Mistake 4 · Whole body

### Reads as stacked spheres — or worse, the wrong animal

**Root cause:** visible seams, plus every part keeping its own perfect-sphere silhouette. A long narrow torso, small round ears, and no muzzle will read as a ferret or a weasel no matter how smooth the shading is — silhouette _is_ species. **Fixes:**

  * **Fuse** head, body, cheeks, and muzzle with voxel remesh or metaballs (Stage 2).
  * **Break every circle:** flatten the skull sides, widen the cheeks, add chest and haunch volumes so no cross-section is a circle. If the subject should be short and wide, check the side view: **if it's longer than it is tall, send it back.**
  * **Add the transitional forms** — neck-to-chest and cheek-to-muzzle blends — with sculpting or proportional editing.
  * **Verify by silhouette** on the contact sheet, every stage (§7).

## 6 · Fur and colour, without ruining the form

### Fur — three options, in increasing cost

(a) No fur — texture only
    Clean stylised look: fur suggested by silhouette, a slightly fuzzy Principled BSDF with sheen, and painted colour variation. **Cheapest and most reliable for animation.** Start here.

(b) Shells & tufts
    Duplicate the body mesh scaled slightly outward, or add sculpted tufts at cheeks, chest, and tail tip. Reads well in stylised renders; parent / weight-paint to the same bones so it deforms with the rig.

(c) Strand fur — Hair Curves
    Blender 3.3+'s modern system, recommended over legacy particle hair: groomed guides + interpolated children, Geometry Nodes for density / clump / length / curl, Principled Hair BSDF. Short dense clumped guides for a short-haired character. **Don't weight-paint fur as if it were skin** — parent the Curves object to the body or use Surface Deform so it follows deformation.

### Colour patches (e.g. calico) — best first

(a) Texture paint ✓
    UV-unwrap the fused mesh and paint patches directly in Texture Paint mode: soft brush, irregular organic blobs, patches crossing seams on purpose — real markings ignore anatomy. Highest fidelity, survives rigging, one material.

(b) Vertex paint as a mask
    Paint a colour attribute and use it as the Mix factor between coloured Principled BSDFs. Fast and procedural, but resolution is limited by vertex density — good as a _mask driving a texture_ , blotchy as final colour.

(c) Separate patch meshes
    Only for bold graphic patches. Risk: floating, seams, and gaps when the body deforms, unless identically parented and weight-painted. Avoid on a first build.

### If it needs to animate: rigging

Use **Rigify** (bundled): Add → Armature → **Cat** metarig (also Basic Quadruped / Wolf / Horse). Rigify only creates the controls — it does not skin the mesh. You still parent with automatic weights and fix the painting, especially at the tail base, haunches, and jaw. Scale the character and apply scale first (Rigify assumes 1 unit = 1 m). Add the Spline-IK tail chain from §5, eye bones with Damped Track to a control in front of each eye, and jaw / ear bones. Validate with extreme poses — sit, stretch, tail curl, head turn — rendered as a contact sheet before calling the rig done.

## 7 · How to drive the AI agent (this is where it usually fails)

The modelling above only works if the agent's _working loop_ is right. Two MCP bridges connect an LLM to Blender — don't confuse them:

Community: mcp-for-blender ✓ default
    **Maintained, most-used.** MIT, ~32,052 stars / 2,904 forks and PyPI `mcp-for-blender` 2.2.2 (checked 20–23 Oct 2026). Blender 3.0+. Install: `claude mcp add blender uvx mcp-for-blender`. Best for building scenes, executing code, and downloading assets (Poly Haven, Sketchfab, Poly Pizza) plus AI text / image-to-3D (Meshy / Tripo / Hyper3D / Hunyuan3D — some premium). [github.com/ahujasid/mcp-for-blender](https://github.com/ahujasid/mcp-for-blender/blob/main/README.md)

Official: Blender Lab MCP
    **Experimental** — v1.0.3 (11 Sep 2026), Blender **5.1+ only** , and officially outside Blender's roadmap. Blender warns it executes LLM code with no guards: run it in a VM or a machine with no sensitive data. Best for inspecting / debugging an existing scene and searching docs / API. [blender.org/lab/mcp-server](https://www.blender.org/lab/mcp-server/)

Both share one architecture: a Blender add-on opens a local TCP socket (`localhost:9876`, no auth) and an MCP server relays the agent's calls to it. The workhorse tool is `execute_blender_code` / `execute_code` — arbitrary `bpy` Python. Each call is isolated, so the agent must re-import `bpy, bmesh, mathutils` every time; calls time out at roughly **15–30 seconds** , so long renders and exports belong in headless runs.

### The loop every good skill repo converges on

1

### Read first

`get_scene_info` / `get_objects_summary` / `get_object_info`, or a small inspection script, _before_ mutating anything. “Never assume a step worked; screenshot and check, because errors compound.”

2

### One small step at a time

Blockout → parts → detail → material → light → render. Roughly 90% of the work is Python sent through `execute_code`. Prefer the data API (`bpy.data` / `bmesh` / modifiers) over `bpy.ops`, which is context-dependent and often fails headless with `poll()` errors.

3

### Verify visually, every step

Render, read the PNG back, and plan the next fix from what you actually see (“roof too flat → re-skew”). An agent that doesn't look at its renders is guessing.

4

### Judge from a contact sheet, not one angle

Front / right / top / three-quarter in one labelled image. One angle hides floating and intersecting parts; this single check catches most of §5's four mistakes. Keep the camera and crop fixed while judging a geometry edit, and use complementary angles to resolve depth rather than flattening the model to win one projection.

5

### Draft first, hero render last

Fast previews while iterating — EEVEE or viewport renders at low resolution (e.g. 256 px), Workbench / matcap for pure silhouette — then a full Cycles / EEVEE render only at milestones. Background render jobs let the agent keep working; snapshot / diff tools and undo checkpoints go before any risky edit.

6

### Script is the source of truth

“Create the asset as source-controlled Python plus generated `.blend` and `.glb` outputs. Treat the script as the durable source and Blender as the execution runtime.” Regenerate after every change; never patch generated outputs by hand; never add final materials to disguise unresolved proportions.

### Headless alternative & gotchas
    
    
    blender --background --factory-startup --python build_cat.py

  * Use MCP for live interactive iteration; headless for reproducible batch builds, renders, and turntables.
  * `shade_auto_smooth` silently no-ops headless; `use_auto_smooth` was removed in Blender 4.1 and crashes on 4.1+ / 5.x.
  * The `bpy` pip module pins an exact Python per Blender version — always confirm the running version with `bpy.app.version` before choosing tools.
  * **Pin one Blender version for the whole project first.** The version split is real: the community MCP needs 3.0+, the official MCP needs 5.1+, the Blender Studio Cat rig needs exactly 3.6, and several skills target 5.2 LTS.

Workflow sources: blender-agent-studio (verified live 10 Nov 2026); minihellboy/claude-blender; DreamLab agentbox Blender skill; garyhtou blender-procedural headless notes.

## 8 · Don't start from scratch if a clean base exists

The official Blender Human Base Meshes (CC0) set the pattern even though they aren't animals: start from a clean quad base mesh instead of primitives. For a cat specifically:

Blender Studio — Cat rig
    **CC-BY** (credit: “Cat Rig (CC-BY) Blender Foundation | studio.blender.org”). A full production rig by Demeter Dzadik — the best _reference_ for rig and topology, not a cute stylised base. Caveat: **Blender 3.6 LTS exactly** ; it may break on lower or higher versions. [studio.blender.org/characters/cat/v1](https://studio.blender.org/characters/cat/v1/)

Sketchfab — Cartoon Cat, Rigged (solajibs)
    **CC-BY.** Rigged, UV-unwrapped, 4K textures, .blend + .fbx + .obj + .stl; Blender 2.8-era, so expect material / version fixes on import. [sketchfab.com](https://sketchfab.com/3d-models/cartoon-cat-rigged-5e48a26825f34e37a161c9b412a81730)

Sketchfab — Blue cartoon cat (cassielouiseam)
    **CC-BY.** Rigged, 11.6k tris, Blender-made. [sketchfab.com](https://sketchfab.com/3d-models/blue-cartoon-cat-e4a9373d937141a49b73f523c998ed63)

Sketchfab — Chubby Pallas Cat (DreamNoms)
    **CC-BY.** 934 tris, textured, rigged with a walk cycle — very low-poly; rig reference only. [sketchfab.com](https://sketchfab.com/3d-models/chubby-pallas-cat-rigged-and-animated-9f9122ec0c2340c5a5ce9db4aef8b3f7)

Meshy — free cat models
    **CC0 per page.** 5,671+ community AI-generated cats in GLB / FBX / OBJ / STL / BLEND. Mostly static, variable quality and topology; auto-rig is humanoid-only, so you'd rig one yourself. [meshy.ai/tags/cat](https://www.meshy.ai/tags/cat)

Poly Haven / ambientCG / Kenney
    **CC0** , no attribution. Textures, HDRIs, and models — useful for materials and lighting, not cat meshes.

**No clean, CC0, cute-stylised, rigged calico-cat base mesh was found.** Honest options: use the Blender Studio Cat for rig / topology study and a CC-BY Sketchfab cartoon cat as a proportion / topology starting point (with attribution), or build the fused blockout from scratch per §2. Don't present a Meshy AI download as a clean base mesh. And **re-check the licence badge on the asset page at download time** — listings change, and some “free” cats are CC-BY-NC (non-commercial) or AI-generated with unclear provenance.

## 9 · The copy-paste prompt for your agent

This is the deliverable. Paste it to the agent, fill in the three bracketed bits, and attach reference photos. It encodes the whole article — pipeline, numbers, verification loop, and completion gate — so the agent can't drift back to stacking spheres.

Copy the prompt
    
    
    Build me a cute, high-fidelity stylised 3D character in Blender: [SUBJECT — e.g. “a short-haired calico cat, sitting”]. Reference photos are attached — match their proportions and markings; do not invent markings.
    
    HARD RULES
    1. Do NOT build the character by stacking visible spheres, cubes, cylinders, or cones. Primitives may only be a temporary blockout. The final body must be ONE fused, continuous mesh: block out overlapping volumes, then fuse them with Voxel Remesh (raise resolution ≤1.5× per step) or build the blockout as metaballs and convert to mesh. If I can draw a circle around the head and another around the body with a pinch between them, it has failed.
    2. Work in stages, and do not skip ahead: (1) blockout proportions against the reference, (2) fuse into one mesh, (3) shape primary → secondary → tertiary forms with Subdivision Surface + sculpting / proportional editing — never detail a form that isn't right yet, (4) retopologise to clean quads if it will be animated (≥3 edge loops around each eye socket and around the mouth; no triangles in deforming areas), (5) separate objects ONLY for things that are truly separate: eyes, nose, inner ears, whisker curves, tail curve.
    3. The script is the source of truth. Write one deterministic build script (fixed seed, every dimension a named constant at the top: HEAD_W, EYE_D, BODY_L, BODY_H, TAIL_POINTS, …). Regenerate the .blend and .glb from the script after every change. Never patch the generated files by hand, and never call a grey blockout “finished” or hide bad proportions under materials.
    
    CUTE, NOT UGLY — CHECK THESE NUMBERS
    - Eyes: five eye-widths must fit across the face — assert EYE_D × 5 ≈ HEAD_W in the script. If in doubt, scale the eyes down (as far as half). Sink each eye into its socket so only an almond / lens shape shows, overlap it with eyelids, flatten it front-to-back, and build it in layers (sclera, iris, pupil, highlight). No protruding marbles, no paint on a bulge.
    - Head: not a sphere. Flatten the sides slightly, widen the cheeks, add a short wide muzzle as a fused volume with a chin bump below it.
    - Ears: rounded triangles, tapered. Top of ear ≈ eyebrow line, bottom ≈ nose line. Inner ear inset and thinner.
    - Tail / limbs / appendages: never a straight uniform cylinder. Use a Bézier curve with 4–6 control points in a natural curve (for a sitting cat: low S-curve along the ground, curling at the tip), tapered from thick base to soft point via per-point radius. Parametrise it as a list of (position, radius) points in the script.
    - Whiskers / fine strands: tapered Bézier curves, ~1.5–2 mm at the root to 0 at the tip, gentle downward arc, 3–4 per side in two rows, off-white soft material. No constant-radius sticks.
    - Silhouette is species: keep the body proportions true to the reference in SIDE view as well as front. If it reads as a different animal (long and narrow when it should be short and round), fix the blockout before anything else.
    
    VERIFY — AFTER EVERY STAGE
    Render a contact sheet (front, side, top, three-quarter) at a fixed camera, plus low-res drafts (256 px, EEVEE / Workbench) while iterating. Open the images and actually look at them. Report what you see wrong and fix it before continuing. Prefer bpy.data / bmesh over bpy.ops. Confirm bpy.app.version at the start and tell me which Blender version and which bridge (mcp-for-blender) you are using.
    
    DONE MEANS
    The script reruns cleanly from scratch, the .blend reopens, the .glb passes a fresh-import check, and no part reads as floating, intersecting, or blockout-like from any view in the final contact sheet. Show me that contact sheet when you claim it's done.

### The script scaffold behind it

A good agent's build script looks like this in outline — constants first, stages in order, a render at the end of each:
    
    
    # build_character.py — deterministic, rerun from scratch every time
    import bpy, bmesh, mathutils  # re-import in every MCP call; calls are isolated
    
    # 1. CONSTANTS — all proportions named, all in one place
    HEAD_W = 0.42; EYE_D = HEAD_W / 5.0   # five-eye-widths rule, asserted below
    BODY_L = 0.55; BODY_H = 0.48          # side view: not longer than tall
    TAIL_POINTS = [(pos, radius), ...]    # S-curve, tapering to ~0 at the tip
    assert abs(EYE_D * 5 - HEAD_W) < 0.01
    
    # 2. clean scene → blockout (round cube / metaballs / traced silhouette)
    # 3. fuse: join → voxel_remesh()  → ONE watertight mesh
    # 4. shape: Subsurf + bmesh offsets (cheeks, muzzle, chest, haunches)
    # 5. details as separate objects: eyes (socketed), nose, inner ears,
    #    whisker curves (tapered), tail curve (tapered)
    # 6. retopo / QuadRemesh if animating → UV → texture-painted markings
    # 7. render contact sheet: front / side / top / ¾ — and LOOK at it

## Skills & repos worth giving your agent

blender-agent-studio (ifBars) ✓ workflow
    The most complete workflow reference found: 10 skills (modelling, character, procedural, animation, asset validation, rendering…), staged gates, script-as-source doctrine, validated on Blender 5.2 LTS. Use the canonical `ifBars` repo — beware forks under other usernames. [github.com/ifBars/blender-agent-studio](https://github.com/ifBars/blender-agent-studio/blob/HEAD/plugins/blender-agent-studio/skills/blender-modeling-workflow/SKILL.md)

minihellboy/claude-blender
    Claude Code + MCP, 34 tools: multi-view contact sheet, scene diff / snapshot, background renders, slash commands `/blender-iterate`, `/blender-studio-shot`, `/blender-turntable`. MIT, small / experimental (10 stars, Jan 2026) — copy its contact-sheet + diff + iterate pattern even if you don't adopt the repo. [github.com/minihellboy/claude-blender](https://github.com/minihellboy/claude-blender)

MartinRapcan/blender-claude-skill
    Teaches an agent to drive Blender via headless `bpy`; prefers `bpy.data` / `bmesh` over `bpy.ops`. Tested on 4.5 LTS + 5.x, MIT. [github.com/martinrapcan/blender-claude-skill](https://github.com/martinrapcan/blender-claude-skill/blob/HEAD/README.md)

CypherPoet — blender-mastery
    Official MCP + headless CLI, Blender 5.2 API, `bpy` patterns, rigging / Geometry Nodes / rendering. [github.com/cypherpoet/custom-agent-skills](https://github.com/cypherpoet/custom-agent-skills/blob/HEAD/plugins/blender-kit/skills/blender-mastery/SKILL.md)

RobLe3/cc-blender-skill
    Knowledge base whose sculpting + retopo overview is the best single-page pipeline summary found. [github.com/roble3/cc-blender-skill](https://github.com/roble3/cc-blender-skill/blob/HEAD/knowledge/03-sculpting-retopo/00-overview.md)

DreamLab-AI/agentbox — Blender skill
    Meta-skill for driving Blender via BlenderMCP: decompose → `execute_code` → screenshot → correct. [github.com/dreamlab-ai/agentbox](https://github.com/dreamlab-ai/agentbox/blob/HEAD/skills/blender/SKILL.md)

majidmanzarpour/blender-game-skills
    Game-asset skills including `blender-image-to-3d` (reference → rigged GLB / FBX, gated on rendered evidence). MIT. [github.com/majidmanzarpour/blender-game-skills](https://github.com/majidmanzarpour/blender-game-skills)

Scenario blender-plugin
    Local MCP with a `capture_views` contact-sheet tool and `blender_api_help` to inspect the `bpy` API before scripting. Experimental vendor plugin. [github.com/scenario-labs/blender-plugin](https://github.com/scenario-labs/blender-plugin/blob/main/docs/MCP.md)

MorphoMesh (DuongNAD)
    AI creature pipeline + FastMCP bridge to Blender 5.2.1: quadruped base meshes, QuadriFlow retopo, Rigify metarigs, Hair Curves dual-coat fur. Experimental / research — reference architecture, not a drop-in character builder. [github.com/duongnad/morphomesh](https://github.com/duongnad/morphomesh/blob/HEAD/README.md)

**Practical pick:** mcp-for-blender for live iteration + blender-agent-studio's modelling / character workflow for the staged gates + blender-claude-skill or blender-mastery for `bpy` / `bmesh` idioms. Treat the official Lab MCP as a second, inspection-oriented bridge, and the small repos as patterns to borrow, not dependencies. Note that no widely-adopted standalone Python modelling library for organic characters was found — _the skill repos are the library_.

## 10 · Honest limits — read before you promise anyone a result

  * **Sculpting cannot be fully scripted.** No source shows a reliable headless sculpt pass. The agent scripts setup and proportions; fine sculpting is a human step, or you start from a base mesh (§8). Budget for one of those.
  * **“Cute proportions” have no single authority** (§4) — triangulated rules of thumb, to be checked against reference photos, not gospel.
  * **Whisker and fur close-up quality wasn't render-tested** in the underlying research — the taper / profile advice comes from artist tutorials, not from a verified render in that session. Check your own close-ups.
  * **Licences move.** Re-check Sketchfab / Meshy badges at download time (§8).
  * **Version soup is real** (§7) — pin a Blender version before choosing a bridge or a base rig.
  * **The official MCP has no sandbox.** It executes generated code with no guards — use a VM or a spare machine.

## 11 · The “done” checklist

From blender-agent-studio's completion gate. If any line fails, it isn't finished — it's a blockout with good PR.

  * The build script reruns cleanly from scratch, with proportions as named constants.
  * The body is one fused mesh — no visible seams, no circle-around-the-head silhouette in any view.
  * Final contact sheet checked: front, side, top, and three-quarter — no part floating, intersecting, or reading as a primitive.
  * Eyes pass the five-eye-widths check and sit in sockets behind eyelids, not on the surface.
  * Tail / appendages are tapered curves, not uniform cylinders; whiskers taper to a point.
  * The side silhouette matches the reference's proportions — right animal, right posture.
  * The .blend reopens and the .glb passes a fresh-import check.
  * If rigged: extreme poses (sit, stretch, tail curl, head turn) rendered and checked before calling the rig done.

**Sources** — research checked 10 Nov 2026. “Verified live” pages were opened that day; others are from search-result text, as noted in the research. Key sources:

  * [mcp-for-blender README](https://github.com/ahujasid/mcp-for-blender/blob/main/README.md) — verified live
  * [blender-agent-studio modelling workflow skill](https://github.com/ifBars/blender-agent-studio/blob/HEAD/plugins/blender-agent-studio/skills/blender-modeling-workflow/SKILL.md) — verified live
  * [Blender Studio — Cat rig (CC-BY)](https://studio.blender.org/characters/cat/v1/) — verified live
  * [minihellboy/claude-blender](https://github.com/minihellboy/claude-blender) — verified live
  * [Blender Lab MCP server (official)](https://www.blender.org/lab/mcp-server/)
  * [cc-blender-skill — sculpting & retopo overview](https://github.com/roble3/cc-blender-skill/blob/HEAD/knowledge/03-sculpting-retopo/00-overview.md)
  * [3D workflow pipeline — topology reference](https://github.com/73k-y/3d-workflow-pipeline/blob/HEAD/TOPOLOGY_REFERENCE.md)
  * [garyhtou — Blender procedural / headless notes](https://github.com/garyhtou/claude-plugins/blob/HEAD/plugins/high-fidelity-web/skills/building-3d-for-web/references/blender-procedural.md)
  * [Creative Bloq — model a cute cat in Blender](https://www.creativebloq.com/3d/heres-how-to-model-a-cute-cat-or-anything-else-in-blender-in-just-a-few-easy-steps)
  * [Yarsa Labs — model 3D cats in Blender](https://blog.yarsalabs.com/model-3d-cats-in-blender/)
  * [Robert Peake — early cat walk cycle / unified skin](https://www.robertpeake.com/archives/149-early-cat-walk-cycle.html)
  * [Blender Manual — Spline IK](https://docs.blender.org/manual/it/latest/animation/constraints/tracking/spline_ik.html)
  * [Blender Manual — Rigify](https://docs.blender.org/manual/en/2.81/addons/rigging/rigify.html)
  * [CGI Academy Hub — Hair Curves grooming](https://cgiacademyhub.com/blog/hair-curves-blender-grooming)

Confidence: moderate–high for workflows and general techniques (multiple independent skill repos plus Blender docs agree); moderate for character-specific proportions (mostly tutorials, not one authoritative guide).
