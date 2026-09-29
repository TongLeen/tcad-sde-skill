---
name: tcad-sde
description: Generate Sentaurus SDE scripts (Scheme dialect) as TypeScript with the @tongleen/tcad-sde wrapper. Use when defining 2D TCAD device structures — drawing geometry/regions, adding contacts/electrodes, constant or Gaussian doping, and mesh refinement — and emitting a Sentaurus SDE (.cmd) script.
---

# Sentaurus SDE Script Generation with TypeScript

This skill generates Sentaurus **SDE** scripts (`.cmd`, a Scheme dialect). The
wrapper turns idiomatic TypeScript calls into SDE Scheme forms and writes them to
a file. Always emit a script the user can run; only execute it when asked.

## When to use

Use this skill whenever the user wants to build a 2D device structure for
Sentaurus SDE: drawing geometry/regions, defining contacts, applying doping, and
refining the mesh. The library lives in `src/` and is published as
`@tongleen/tcad-sde`.

## How it works

- The wrapper accumulates Scheme command strings in memory and emits them as text.
- `useSde<M, D>()` returns `{ draw, contact, dop, mesh, save, run, runAndExit }`.
- Generic `M` is a union of **material name strings**; `D` is a union of **dopant
  name strings**. They only provide type safety — the strings are emitted verbatim
  into the script, so they must match the material/dopant names Sentaurus knows.
- `save(filename)` writes all accumulated commands to `filename` (Node `fs`).
- **Default behavior: call only `save()`.** Call `run()` / `runAndExit()` only when
  the user explicitly asks to execute the script (they require `sde` on `PATH`).

## Setup

```ts
import { useSde, position } from "@tongleen/tcad-sde";

type Material = "Silicon" | "Oxide" | "Metal";
type Dopant = "BoronActiveConcentration" | "ArsenicActiveConcentration";

const { draw, contact, dop, mesh, save, run, runAndExit } = useSde<
    Material,
    Dopant
>();
```

### Coordinates — `position`

`position(x, y, z = 0)` returns a `Position`. The class is also exported directly.
Useful helpers: `shift`, `shiftX`, `shiftY`, `add`, `midpoint`.

```ts
const p = position(0, 0);
const q = p.shiftX(5).shiftY(3);
```

## `draw` — geometry, regions, global settings

### `setCoordMode(mode)`

```ts
draw.setCoordMode("y_down"); // or "x_down"
```

- `"y_down"` → process up-direction `+z` (default choice for process simulation).
- `"x_down"` → `-x`.

Call this before drawing. With `y_down`, the device surface is at small `y` and the
substrate at large `y`; negative `y` is above the surface.

### `setOverlapBehavior(behavior)`

```ts
draw.setOverlapBehavior("replace"); // ABA: new geometry overwrites old
draw.setOverlapBehavior("keep"); // BAB: new geometry only fills empty area
```

Use `"replace"` for the main device stack and switch to `"keep"` when drawing
passivation/insulator layers that must not overwrite existing structures.

### `rectangle({ name, material, p0, p1, round? })`

Creates a rectangle from `p0` to `p1`. **The region is auto-named `${name}.region`.**

```ts
draw.rectangle({
    name: "Substrate",
    material: "Silicon",
    p0: position(0, 0),
    p1: position(10, 5),
    // round: [[position(0, 0), 0.2]], // optional corner fillets (radius in µm)
});
```

### `polygon({ name, material, positions, round? })`

`positions` is a non-empty list (use ≥ 3 points for a valid polygon).

```ts
draw.polygon({
    name: "Trench",
    material: "Oxide",
    positions: [position(0, 0), position(1, 0), position(0.5, 1)],
});
```

### `round` option (both shapes)

`round` is `[Position, number][]`: each entry is `[vertex position, radius µm]`. The
position must be an actual vertex of the shape being filleted (`sdegeo:fillet-2d`).

### `unite({ positions, name? })`

Merges bodies into one. Select **exactly one point inside each region** to unite,
and pick points that are in each region's own area (not on an overlap boundary).
If `name` is given, the merged region is renamed `${name}.region`.

```ts
draw.unite({
    positions: [position(5, 1), position(5, 3), position(5, 5)],
    name: "Merged",
});
```

### `vertex(position)`

Inserts an extra vertex at `position` (rarely needed).

### `saveModel(filename)`

`(sde:save-model "...")` — saves the structure (typically `n@node@_str.tdr`).

## `contact` — electrodes / contacts

### `add({ name, contacts })`

`contacts` is a list of `{ position, shape: "body" | "edge", remove?: boolean }`.
The contact set is auto-defined the first time a `name` is used.

```ts
// Body contact that is removed (typical for an electrode drawn as "Metal")
draw.rectangle({
    name: "AnodeMetal",
    material: "Metal",
    p0: position(0, 0),
    p1: position(5, -1),
});
contact.add({
    name: "Anode",
    contacts: [{ position: position(2.5, -0.5), shape: "body", remove: true }],
});

// Edge contact on a boundary
contact.add({
    name: "Cathode",
    contacts: [{ position: position(5, 5), shape: "edge" }],
});
```

**Important:** a body contact with `remove: true` deletes the metal body and marks
the electrode material as `"Contact"`. In mesh statements referring to that
electrode, use `"Contact"` (not `"Metal"`).

## `dop` — doping and mole fractions

`dop.constant` and `dop.gaussian` cover the common cases.

### `dop.constant({ ... })`

```ts
type Constant = {
    name: string;
    dopant: D;
    concentration: number; // cm^-3 (or mole fraction for xMoleFraction)
    decay?: { distribute: "Erf" | "Gauss"; factor: number };
    replace?: "Replace" | "LocalReplace" | "NoReplace"; // default "NoReplace"
} & (
    | { kind: "position"; rectangle: [Position, Position] }
    | { kind: "material"; material: M }
    | { kind: "region"; region: string }
);
```

- `kind: "region"` — uniform doping across a named region. `region` is the bare
  name (no `.region`); the API appends the suffix.
- `kind: "material"` — all regions of a material.
- `kind: "position"` — a rectangular window.

**Caveat:** `decay`/`replace` are only forwarded for `position` and `material`
kinds. They are **ignored** when `kind: "region"`.

```ts
dop.constant({
    name: "AlGaN_Mole",
    dopant: "xMoleFraction",
    concentration: 0.18,
    kind: "region",
    region: "AlGaN_Barrier",
});
```

### `dop.gaussian({ ... })`

```ts
type Gaussian = {
    name: string;
    start: Position;
    end: Position; // start->end defines a baseline window (Line)
    peak_depth: number; // peak distance along the window normal
    dopant: D;
    polar: "Positive" | "Negative" | "Both"; // right-hand side of start->end = Positive
    lateral?: { dist: "Gauss" | "Erf"; factor: number };
    eval_at_baseline?: boolean; // default true
    replace?: "Replace" | "LocalReplace" | "NoReplace"; // default "NoReplace"
} & (
    | {
          kind: "conc";
          peak_conc: number;
          another_conc: number;
          another_depth: number;
      }
    | { kind: "dose"; dose: number; std_dev: number }
);
```

```ts
dop.gaussian({
    name: "Anode",
    dopant: "BoronActiveConcentration",
    start: position(0, 0),
    end: position(5, 0),
    peak_depth: 0.1,
    polar: "Positive",
    lateral: { dist: "Erf", factor: 0.4 },
    kind: "conc",
    peak_conc: 1e19,
    another_conc: 1e14,
    another_depth: 1,
});
```

**Caveat:** be careful with `replace` on Gaussian profiles — it overwrites values
in the placement window and can corrupt concentrations at the profile edge. Use
`replace` only for in-situ doping defined directly.

## `mesh` — refinement

### `refine({ name, dx, dy, func?, ...kind })`

```ts
type Refine = {
    name: string;
    dx: [number, number]; // x step bounds [a, b]; order is normalized internally
    dy: [number, number];
    func?: RefinementFunction[];
} & (
    | { kind: "position"; rectangle: [Position, Position] }
    | { kind: "material"; material: M }
    | { kind: "region"; region: string }
);
```

`RefinementFunction` is one of:

```ts
type RefinementFunction =
    | { func: "MaxTransDiff"; value: number } // limit concentration gradient
    | {
          func: "MaxLenInt";
          interface: [string, string];
          value: number; // interface step (µm)
          factor: number; // growth factor away from the interface
          double_side?: boolean; // refine both sides
          use_region_names: true; // interface uses region names (suffix added)
      }
    | {
          func: "MaxLenInt";
          interface: [Material, Material]; // when use_region_names is false/omitted
          value: number;
          factor: number;
          double_side?: boolean;
          use_region_names?: false;
      };
```

For `MaxLenInt`, pass bare region names when `use_region_names: true`; the API
appends `.region`.

```ts
mesh.refine({
    name: "2DEG",
    dx: [0.001, 0.1],
    dy: [0.001, 0.1],
    kind: "material",
    material: "GaN",
    func: [
        {
            func: "MaxLenInt",
            interface: ["GaN", "AlGaN"],
            value: 0.001,
            factor: 1.4,
            double_side: true,
        },
    ],
});
```

### `offset({ maxlevel, kind, material|region, targets })`

```ts
mesh.offset({
    maxlevel: 5,
    kind: "material",
    material: "GaN", // refinement applied on this side
    targets: [{ target: "Contact", value: 0.001, factor: 1.5 }],
});
```

`offset` follows curved/rounded interfaces better than `MaxLenInt`.

**Caveat:** for `kind: "region"` the source `region` gets `.region` appended, but
`targets[].target` is passed through **as-is**. So a region target must be given as
the full name including the `.region` suffix. Prefer `kind: "material"` for offset.

### `buildMesh(filename)`

`(sde:build-mesh "...")` — outputs the mesh TDR (typically `n@node@_msh.tdr`).

## Naming rules (critical)

- `draw.rectangle`/`polygon` auto-name the region `${name}.region`.
- **Never add `.region` yourself** — the API appends it wherever it is needed
  (`region` in `dop`, `mesh.refine`, `mesh.offset`, and `MaxLenInt.interface` when
  `use_region_names: true`).
- All `name` values (profiles, refinements, regions, contacts) should be **unique**
  within the script.

## Best practices

- **Coordinates:** use `draw.setCoordMode("y_down")` for process-style structures
  (surface at top, substrate at bottom). Anchor a reference plane (e.g. a hetero
  interface) at `y = 0` and measure layer thicknesses relative to it.
- **Overlap:** call `draw.setOverlapBehavior("replace")` before drawing the device
  stack; switch to `"keep"` only for passivation/insulator fills, then restore.
- **Regions:** one region per material per phase. Multiple regions of the same
  material can break `MaxTransDiff`. Define regions that carry bulk traps
  separately.
- **Contacts:** draw electrode geometry as `"Metal"`, then `contact.add` with
  `remove: true`; reference `"Contact"` afterwards in mesh statements.
- **Mesh:** refine interfaces with `MaxLenInt` (not a `position` rectangle), refine
  doping gradients with `MaxTransDiff`, and use `mesh.offset` only at rounded
  corners. Keep insulator/substrate bulk coarse and densify only at interfaces.

## Execution

```ts
draw.saveModel("n@node@_str.tdr");
mesh.buildMesh("n@node@_msh.tdr");
save("n@node@_str.cmd"); // always do this

// Only when the user asks to run:
// run();          // spawns `sde -e -r`, returns exit status
// runAndExit();   // same, then process.exit()
```

## Complete example: silicon diode

```ts
import { useSde, position } from "@tongleen/tcad-sde";

type Material = "Silicon" | "Oxide" | "Metal";
type Dopant = "BoronActiveConcentration" | "ArsenicActiveConcentration";

const { draw, contact, dop, mesh, save } = useSde<Material, Dopant>();

draw.setCoordMode("y_down");
draw.setOverlapBehavior("replace");

// --- geometry ---
draw.rectangle({
    name: "Substrate",
    material: "Silicon",
    p0: position(0, 0),
    p1: position(10, 5),
});
draw.rectangle({
    name: "Oxide",
    material: "Oxide",
    p0: position(0, 0),
    p1: position(10, -1),
});
draw.rectangle({
    name: "AnodeMetal",
    material: "Metal",
    p0: position(0, 0),
    p1: position(5, -1),
});

// --- contacts ---
contact.add({
    name: "Anode",
    contacts: [{ position: position(2.5, -0.5), shape: "body", remove: true }],
});
contact.add({
    name: "Cathode",
    contacts: [{ position: position(5, 5), shape: "edge" }],
});

// --- doping ---
dop.constant({
    name: "Substrate",
    dopant: "ArsenicActiveConcentration",
    concentration: 1e14,
    kind: "region",
    region: "Substrate",
});
dop.gaussian({
    name: "Anode",
    dopant: "BoronActiveConcentration",
    start: position(0, 0),
    end: position(5, 0),
    peak_depth: 0.1,
    polar: "Positive",
    lateral: { dist: "Erf", factor: 0.4 },
    kind: "conc",
    peak_conc: 1e19,
    another_conc: 1e14,
    another_depth: 1,
});

// --- mesh ---
mesh.refine({
    name: "Sub",
    dx: [0.01, 0.4],
    dy: [0.01, 0.4],
    kind: "position",
    rectangle: [position(0, 0), position(10, 5)],
    func: [{ func: "MaxTransDiff", value: 1.2 }],
});
mesh.offset({
    maxlevel: 5,
    kind: "material",
    material: "Silicon",
    targets: [{ target: "Oxide", value: 0.001, factor: 1.5 }],
});

mesh.buildMesh("n@node@");
draw.saveModel("n@node@");
save("n@node@_cod.cmd");
```

## Common materials / dopants (generic unions)

- Materials: `Silicon`, `Oxide`, `Metal`, `PolySilicon`, `Nitride`; GaN-family:
  `GaN`, `AlGaN`, `AlN`, `Al2O3`, `Si3N4`, `PI`, `Contact`.
- Dopants: `BoronActiveConcentration`, `ArsenicActiveConcentration`,
  `PhosphorusActiveConcentration`, `xMoleFraction`,
  `pMagnesiumActiveConcentration`.

## Reference

See `ref/gan-hemt-ref.md` for a detailed p-GaN HEMT structure: coordinate
conventions, layer stack, X-layout, per-material mesh strategy, 2DEG refinement,
and a full script template.
