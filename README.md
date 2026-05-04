# UV Texture Variation Offset — Houdini

A procedural UV approach in Houdini for instancing varied planks without multi-UDIM.

**Full article:** [zenray.vercel.app/reels/uv-texture-variation-offset](https://zenray.vercel.app/reels/uv-texture-variation-offset)

---

## The Problem

How to get varied planks without using a multi-UDIM that would take at least 4 tiles at 4K just to have varied planks or tiles?

In my previous workflow, I was using 5 different tile materials and 5 different static meshes that I was copying onto my mesh, with the planks in multi-UDIM, which required an extra material and 4 UDIM tiles, for a total texture weight of around 64 MB. That weight is now down to 16 MB, a 75% reduction, with better texel density and padding. UV Project does not work beyond 1,000 planks and does not offer a satisfying texel density either, so a method where the plank UV space is shared was necessary to optimize that space.

> Disclaimer: this method is not the definitive solution, but it does allow in Houdini to repeat a mesh with a varied texture.

---

## Development

I installed RizomUV, but I quickly realized that for a repeatable tileable surface, RizomUV was not the right solution and was more suited to unique surfaces, especially since it creates a proxy, which is not ideal when working inside Houdini.

I then tried to focus on UV Unwrap and UV Layout nodes placed before Copy to Points, because I told myself that if the UVs were computed beforehand, I could then distribute a more relevant 0-1 space across each copy.

I then started writing VEX, specifically using modulo, because I remembered that targeting elements inside a For Each Primitive loop allowed me to target specific elements precisely. I created a string attribute:

```vex
s@name = (@ptnum % 2 == 0) ? "A1" : "A2";
```

I then started wondering whether there was a more built-in node than my VEX expressions, telling myself that over time I might not remember exactly what I had written. That is when I discovered the Name node, which allows writing directly to a prim or point in the Geometry Spreadsheet, with a logic close to groups. I finally used a Group Range to select every other point with an offset.

---

## Result

This project started from the need to instance different tiles for a game-ready asset. The result resembles the trim sheet pipeline, but within a 0-1 UV space.

**Key rule — copytopoints name matching:**
- `s@name` on packed prims must be assigned **after** the Pack SOP
- `s@name` on target points via Group Range + Name SOP (no VEX needed)
- `useidattrib = 1` + `idattrib = "name"` on copytopoints::2.0
- Input 0 = templates (packed) / Input 1 = named points

---

## Files

| File | Description |
|------|-------------|
| `UV_TextureVariationOffset1.hiplc` | Full Houdini scene — test network `/obj/test_planks` (V1) |
| `UV_TestPlanks.hdalc` | HDA V2.1 — UV_TestPlanks node (zenray.dev) |
| `UV_TestPlanks.3.1.hdalc` | HDA V3.1 — procedural naming + foreach UV variation (zenray.dev) |

---

## V2 — Procedural Naming + UV Variation (2026-05-04)

The second iteration replaces the manual Name node chain with a single VEX wrangle and adds per-instance UV Y randomization via a For Each loop.

**Key formulas:**

```vex
// On packed templates (Run Over: Primitives, after pack)
s@name = sprintf("A%d", @primnum + 1);

// On destination points (Run Over: Points)
s@name = sprintf("A%d", @ptnum % 40 + 1);

// Inside For Each loop — UV offset (Run Over: Vertices)
int iter = detail(1, "iteration", 0);
@uv.y += rand(iter + 42);
```

**Graph:** `/obj/uv_testplanks_dev/`

**Node count:** V1 ~25 nodes → V2 ~15 nodes

---

## Author

Matthieu — [zenray.vercel.app](https://zenray.vercel.app) — [@mhzenray](https://x.com/mhzenray)
