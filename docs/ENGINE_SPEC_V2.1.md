# 0. System Overview

## 0.1 Phases and Responsibilities

The engine is strictly split into four phases:

1. **Phase 1: Scorer (Utility)**
    
    Convert raw business metrics into a stable, robust, per-SKU priority score ( S_i ).
    
2. **Phase 2: Allocator (What)**
    
    Decide for each SKU how many facings it gets in each orientation “role” (visual vs stock), under global space and business constraints.
    
    Output: `F_front[i]`, `F_stock[i]` (stock = side/flat) per SKU.
    
3. **Phase 3: Placer (Where)**
    
    Map facings to shelves and horizontal positions with brand clusters and 2D bin-packing.
    
    Includes **Look-Ahead Backfill** to avoid jagged gaps.
    
4. **Phase 4: Optimizer (Refinement)**
    
    Use simulated annealing to refine layout with swaps, shifts, and **RotateOrientation** operations, without violating hard business constraints.
    

Each phase has clear input/output data contracts.

---

# 0.2 Data Model (Inputs)

## 0.2.1 SKU Model

Per SKU:

- `sku_id`
- `brand`
- `subbrand`
- `variant`
- `segment`
    
    `(Premium | Core+ | Core | Econ | Kids | Bulk | Saver | Tail)`
    
- **Base dimensions:**
    - `width_mm` (front-on width on shelf)
    - `height_mm`
    - `depth_mm` (shelf depth; used for side/flat)
- **Performance:**
    - `daily_sales_velocity` (units/day)
    - `margin_per_unit` (currency)
- **Facing mechanics:**
    - `units_per_facing_front`
    - `units_per_facing_side` (or flat; defaults to front if equal)
- **Strategy:**
    - `strategic_flag(New | Hero | Core | Support | Deprioritized)`
    - `must_stock` (bool)
- **Orientation policy:**
    - `allowed_orientations: [Front, Side, Flat]`
    (subset; typical: `[Front, Side]` or `[Front]`)
    - `min_front_facings` (int; usually 1 for New/Hero/Core, 0–1 for others)
    - `max_side_share` (0–1; per SKU or from segment default)
- **Other flags:**
    - `heavy_pack` (bool)
    - `bulk_pack` (bool)

---

## 0.2.2 Shelf Model

Per shelf:

- `shelf_id`
- `shelf_width_mm`
- `shelf_height_mm` (vertical clearance)
- `shelf_bottom_height_mm` (height of shelf base from floor)
- `tags: set`
e.g. `{EyeLevel, AboveEye, BelowEye, Bottom, KidsBand, ...}`

Global:

- `total_shelf_width = Σ shelf_width_mm` over all shelves in this bay/section.

---

## 0.2.3 Configuration (Per Category / Retailer Template)

- **Scorer weights:**
    - `W_sales`, `W_margin`, `W_turn`, `W_strat`
- **Robust scaling:**
    - `sales_historic_max` (anchored denominator for sales score)
    - Optionally similar anchors for margin and turnover.
- **Velocity & DoS:**
    - `v_min` (minimum effective velocity)
    - `min_DoS_target` (e.g. 2 days)
    - `min_DoS_floor` (e.g. 1 day)
    - `survival_coverage_percent` (e.g. 70% by `S_i` / cumulative sales)
- **Variety:**
    - `variety_width_target` (e.g. 0.85 of total width)
- **Facings limits:**
    - `max_facings_per_sku` (e.g. 6)
    - `max_brand_facing_share` (optional, e.g. 40%)
- **Lift curve:**
    - `lift_base` (e.g. 0.25)
    - `lift_decay` (e.g. 0.7)
- **Orientation visual impact** (per segment, per orientation):
    - `visual_impact[segment]["Front"]` (usually 1.0)
    - `visual_impact[segment]["Side"]` (e.g. Premium 0.3, Core 0.5, Bulk 0.8)
    - `visual_impact[segment]["Flat"]` (if used; often ≤ side)
- **Side / stock rules:**
    - `max_side_share_by_segment[segment]`
    - Category flags: e.g. `category_is_stock_driven` (detergents, rice).
- **Backfill configuration (Phase 3):**
    - `backfill_lookahead_depth` (N, e.g. 5 clusters)
- **Optimizer weights:**
    - `w_space`, `w_brand`, `w_vertical`, `w_DoS`, `w_variety`, `w_strategic`, `w_side`, `w_side_frag`, `w_orientation`
- **Annealing schedule:**
    - `T0`, `T_min`, `cooling_rate`, `max_iterations`

---

## 0.2.4 Pre-Computed Sets & Stats

Before Phase 1:

- For each metric (sales, margin, turnover):
    - `p1`, `p99` (for clipping)
    - Optional: IQR for robust scaling
- For velocity & width:
    - `v_p25` (slow threshold)
    - `width_p90` (massive threshold)
- For each SKU:
    - `fit_shelves[sku_id]` = set of shelves where:
        
        [
        shelf_height_mm height_mm + clearance_gap
        ]
        

---

# Phase 1: Scorer (Robust Utility Computation)

**Goal:** Per-SKU priority score ( S_i ) that is **stable over time** and **robust to new “super hero” SKUs**.

---

## 1.1 Robust Scaling / Anti-Jitter

Instead of pure min-max, use anchored / robust scaling.

### Sales

- Clip outliers using historic / recent data:
    
    [
    sales_clipped[i] = clamp(sales[i], p1_{sales}, p99_{sales})
    ]
    
- Anchor to a **historic category max**:
    
    [
    Z_{sales}[i] = (1.0, )
    ]
    

This avoids entire Z-scale collapsing when a single SKU spikes.

### Margin and Turnover

- Similar approach, either:
    - Anchored to historic maxima, or
    - Robust scaling via IQR (P25–P75) with clipping.

Example for turnover:

- `turn_clipped = clamp(turn[i], p1_turn, p99_turn)`
- `turn_anchor = max(turn_historic_max, some_baseline)`
- `Z_turn[i] = min(1.0, turn_clipped / turn_anchor)`

If a metric is effectively constant across SKUs (after clipping):

- Set `Z_metric[i] = 0.5` for all SKUs (no discrimination).

---

## 1.2 Strategic Term

Map `strategic_flag` to step scores:

- New Launch: `1.0`
- Hero: `0.9`
- Core: `0.7`
- Support / Tail: `0.3`
- Deprioritized: `0.0`

Call this `Z_strat[i]`.

---

## 1.3 Composite Priority Score

For SKU ( i ):

[
S_i = W_{sales} Z_{sales,i} +
W_{margin} Z_{margin,i} +
W_{turn} Z_{turn,i} +
W_{strat} Z_{strat,i}
]

Also derive:

- `high_velocity[i]` flag from `Z_turn[i]` and/or raw `turnover`.
- `high_margin[i]` flag from `Z_margin[i]`.

---

## 1.4 Stability Guardrails

After scoring:

- Check distribution of `S_i`:
    - No single SKU should have `S_i > 0.99` while all others sit `< 0.1` without a conscious configuration choice.
- If a new item appears with massive sales:
    - It reaches `Z_sales ≈ 1`, but others keep their relative differences anchored.

**Output of Phase 1:**

For each SKU:

- `S_i`
- `Z_*` metrics
- Flags (`high_velocity`, `high_margin`)
- Original config fields (unchanged)

---

# Phase 2: Allocator (What – Front vs Stock Facings)

**Goal:** Compute, for each SKU:

- `F_front[i]` = number of front-facing (visual) facings
- `F_stock[i]` = number of non-front “stock” facings (side or flat)

Subject to:

- Global linear width constraint.
- DoS coverage and must-stock rules.
- Variety breadth.
- Profit density.
- Side / stock guardrails.

---

## 2.0 Helpers

- Effective velocity:
    
    [
    V_{eff}[i] = (daily_sales_velocity[i], v_{min})
    ]
    
- Orientation metadata (per SKU):
    - `visual_front[i] = visual_impact[segment[i]]["Front"]` (≈ 1.0)
    - `visual_stock[i]` = chosen orientation’s visual impact:
        - If using Side for stock:`visual_stock[i] = visual_impact[segment[i]]["Side"]`
        - If using Flat:`visual_stock[i] = visual_impact[segment[i]]["Flat"]`
- Width per role:
    - `w_front[i] = width_mm[i]`
    - `w_stock[i] = depth_mm[i]` (for Side) or chosen dimension for Flat
- Units per facing:
    - `u_front[i] = units_per_facing_front[i]`
    - `u_stock[i] = units_per_facing_side[i]` or flat (can equal front)
- Side/stock moderation:
    - `max_side_share[i] = min(max_side_share_by_segment[segment[i]], max_side_share_override_if_any)`
- Total used width:
    
    [
    W_{used} = *i ( F*{front}[i] w_{front}[i] + F_{stock}[i] w_{stock}[i] )
    ]
    

---

## 2.1 Survival Pass (DoS + Must-Stock, with Stock vs Visual Logic)

### 2.1.1 Initialization

For all SKUs:

- `F_front[i] = 0`
- `F_stock[i] = 0`

### 2.1.2 Must-Stock and Minimum Visibility

For each SKU:

- If `must_stock` or `strategic_flag ∈ {New, Hero, Core}`:
    
    [
    F_{front}[i] = (F_{front}[i], min_front_facings[i])
    ]
    

(normally ≥ 1)

### 2.1.3 DoS for High-Priority SKUs

1. Sort SKUs by `S_i` descending.
2. Identify “survival set”:
    - Top `survival_coverage_percent` SKUs by `S_i` and/or cumulative sales.
3. For each `i` in survival set:
    - Compute DoS per facing:
        - Front:
            
            [
            cap_{daily_front}[i] = 
            ]
            
        - Stock (if `allowed_orientations` includes stock orientation):
            
            [
            cap_{daily_stock}[i] = 
            ]
            
    - Target DoS:
        
        [
        inv_{target_days} = min_{DoS_target}
        ]
        
    - Current DoS:
        
        [
        DoS_{current} = F_{front}[i] cap_{daily_front}[i] + F_{stock}[i] cap_{daily_stock}[i]
        ]
        
    - Gap:
        
        [
        DoS_{gap} = (0, inv_{target_days} - DoS_{current})
        ]
        
    - If `DoS_gap > 0`:
        - Candidate extra fronts:
            
            [
            extra_{front_needed} = DoS_{gap} / cap_{daily_front}[i] 
            ]
            
        - Candidate extra stock (if stock allowed):
            
            [
            extra_{stock_needed} = DoS_{gap} / cap_{daily_stock}[i] 
            ]
            
        - Evaluate width cost of options:
            - Front only:`width_front_only = extra_front_needed * w_front[i]`
            - Stock only:`width_stock_only = extra_stock_needed * w_stock[i]`
            - Mixed solutions (if desired) to respect `min_front_facings` and `max_side_share`.
        - Choose the option with:
            - Minimal added width
            - Respecting:
                - `F_front[i] ≥ min_front_facings[i]`
                - `(F_stock[i] + extra_stock) / (total_facings + extra_front + extra_stock) ≤ max_side_share[i]`
                - `F_front[i] + F_stock[i] + extras ≤ max_facings_per_sku`
4. Massive slow movers constraint:
    - Define:
        - `massive[i] = width_mm[i] ≥ width_p90`
        - `slow[i] = V_eff[i] ≤ v_p25`
    - If both true:
        - Cap total facings:
            
            [
            F_{front}[i] + F_{stock}[i] 
            ]
            
            while still:
            
            [
            F_{front}[i] min_{front_facings}[i]
            ]
            
5. Global cap:
- For every SKU:
    
    [
    F_{front}[i] + F_{stock}[i] max_{facings_per_sku}
    ]
    

---

### 2.1.4 Feasibility Adjustment (Space)

Compute `W_used`.

If `W_used > total_shelf_width`:

1. Relax `min_DoS_target → min_DoS_floor`, recompute increments.
2. If still infeasible:
    - Trim in this order:
        1. Extra **stock** facings of lowest-priority SKUs (ascending `S_i`), ensuring DoS stays ≥ floor where possible.
        2. Extra **front** facings beyond `min_front_facings[i]` starting from lowest `S_i`.
    
    Under no circumstance:
    
    - Drop `F_front[i] < min_front_facings[i]`.
    - Drop `F_front[i] + F_stock[i] < 1` for must-stock SKUs.

At the end of Survival:

- High-priority SKUs have at least `min_DoS_floor` coverage where space allows.
- All important SKUs have at least minimum front visibility.
- Side/stock facings are used only where capacity pressures justify them.

---

## 2.2 Variety Pass (Assortment Breadth; Front-Only)

1. Compute:
    - `W_target_variety = variety_width_target * total_shelf_width`
    - Recompute `W_used`.
2. Build list of SKUs with **zero facings**:
    - `F_front[i] + F_stock[i] == 0`
3. Sort this list by `S_i` descending.
4. For each SKU `i`:
    - If `W_used ≥ W_target_variety`, stop.
    - If `W_used + w_front[i] ≤ total_shelf_width`:
        - Set `F_front[i] = 1`, `F_stock[i] = 0`
        - `W_used += w_front[i]`

Variety is defined in terms of front-facing facings (visual assortment).

---

## 2.3 Profit Density Pass (MPROS, with Gap Awareness)

Remaining width:

[
W_{free} = total_shelf_width - W_{used}
]

Loop while `W_free ≥ min_candidate_width`:

1. For each SKU that can accept more facings
    
    (`F_front + F_stock < max_facings_per_sku` and brand share OK):
    
    - Let:
        
        [
        n = F_{front}[i] + F_{stock}[i]
        ]
        
    - Compute **diminishing lift**:
        
        [
        lift = lift_{base} e^{-lift_{decay} n}
        ]
        
    - **Front candidate:**
        - If `w_front[i] ≤ W_free`:
            
            [
            MPROS_{front}[i] = 
            ]
            
    - **Stock candidate** (if allowed and under side share cap):
        - If `w_stock[i] ≤ W_free`:
            - Raw profit per cm:
                
                [
                raw_MPROS_{stock} = 
                ]
                
            - Visual penalty:
                
                [
                visual_penalty =  (0 < penalty < 1)
                ]
                
            - Final:
                
                [
                MPROS_{stock}[i] = raw_MPROS_{stock} visual_penalty
                ]
                
2. Pick the **best candidate**:
    - `(sku_id, orientation)` with maximum MPROS that fits in `W_free` and side share limit.
3. If no candidate fits (`W_free` is smaller than any facing width), stop.
4. Apply the facing:
    - If front:`F_front[i] += 1`, `W_free -= w_front[i]`.
    - If stock:`F_stock[i] += 1`, `W_free -= w_stock[i]`.

---

### Side-Pivot Triggers (Implicit in the Allocator)

The allocator **naturally** pivots to stock (side/flat) facings only under:

1. **DoS infeasibility**
    
    Front-only cannot hit DoS targets within `total_shelf_width`.
    
2. **Velocity–width mismatch**
    
    High churn, wide pack; side orientation vastly better profit per cm.
    
3. **Visual saturation**
    
    Extra fronts have negligible lift; stock facings are more space-efficient.
    
4. **Variety preservation**
    
    To avoid dropping SKUs, compress stock using stock facings for heroes.
    
5. **Gap absorption**
    
    Later phases see small gaps; stock facings fill them better.
    

**Guardrails:**

- `F_front[i] ≥ min_front_facings[i]`.
- `(F_stock[i] / (F_front[i] + F_stock[i])) ≤ max_side_share[i]`.
- Must-stock SKUs always have at least one front facing.

**Phase 2 Output:**

For each SKU:

- `F_front[i]`
- `F_stock[i]`
- Reason tags such as:
    - `side_reason = DoS_Pressure | Variety_Preservation | Velocity_Width_Mismatch | Gap_Absorption | Policy_Override`

---

# Phase 3: Placer (Where – Brand Blocks + Backfill Packing)

**Input:** `(F_front[i], F_stock[i])`, shelves, `fit_shelves`

**Output:** facings with `(sku_id, orientation, shelf_id, x_start_mm, x_end_mm)`

---

## 3.1 Cluster Construction (Brand Blocks)

1. Build clusters:
    - Group SKUs by `(brand, subbrand)`.
2. For each cluster:
    - Build ordered SKU list:
        - Sort SKUs by:
            - Segment priority:`Premium → Core+ → Core → Econ/Bulk/Saver/Tail`
            - Then `S_i` descending.
    - Expand to a **sequence of facings**:
        
        For each SKU `i` in cluster order:
        
        - Append `F_front[i]` front-facing units.
        - Append `F_stock[i]` stock facings (side/flat) immediately after its front units.
    - Compute:
        
        [
        cluster_width = facing_width
        ]
        
        using `w_front` or `w_stock` per facing.
        

Clusters are the primary units for vertical & horizontal placement.

---

## 3.2 Vertical Assignment (Which Shelf)

### 3.2.1 Ideal Zone per Cluster

Based on dominant segment / strategy within cluster:

- Premium: `EyeLevel → AboveEye → BelowEye`
- Core / Hero: `EyeLevel → BelowEye`
- Bulk / Heavy: `Bottom → BelowEye`
- Kids: shelves intersecting the [0.8m, 1.2m] height band.

### 3.2.2 Candidate Shelves

For each cluster:

- Feasible shelves = shelves where **all SKUs** in that cluster can fit height:
    
    [
    shelf_id fit_shelves[sku]
    ]
    
    or at least support all SKUs via splitting logic later.
    
- Rank shelves by:
    - Match to ideal zone tags
    - Remaining target capacity (rough even distribution per shelf)

### 3.2.3 Assignment

Sort clusters by:

- Segment importance (Premium, Hero clusters first)
- Then max `S_i` within cluster

For each cluster:

- Assign to the best shelf in its candidate list with enough provisional capacity.
- If no ideal shelf left:
    - Assign to next best feasible shelf and record a vertical deviation for Phase 4.

---

## 3.3 Horizontal Packing with Look-Ahead Backfill

Within each shelf, pack assigned clusters.

### 3.3.1 Initial Ordering

For a given shelf:

- Take all clusters assigned to it.
- Sort by descending `cluster_width` (first-fit decreasing).

### 3.3.2 Placement with Backfill

Maintain:

- `x_pos = 0`
- `remaining_gap = shelf_width - x_pos`

For each `current_cluster` in `sorted_clusters`:

1. **If `current_cluster.width ≤ remaining_gap`:**
    - Place it at `x_pos`.
    - `x_pos += current_cluster.width`
    - `remaining_gap = shelf_width - x_pos`
    - Continue.
2. **If `current_cluster.width > remaining_gap`:**
    
    ### 2.1 Look-Ahead Backfill
    
    - Initialise `found_filler = False`.
    - Look ahead up to `backfill_lookahead_depth` clusters in `sorted_clusters` (not yet placed):
        
        For each `candidate_cluster`:
        
        - If `candidate_cluster.width ≤ remaining_gap`:
            - Place candidate in the gap:
                - `place(candidate_cluster, x_pos)`
                - `x_pos += candidate_cluster.width`
                - `remaining_gap = shelf_width - x_pos`
            - Remove candidate from `sorted_clusters`.
            - Set `found_filler = True`
            - Break look-ahead loop.
    - If `found_filler = True`:
        - Re-evaluate `remaining_gap` and re-try `current_cluster` (do not advance the main cluster pointer yet).
    
    ### 2.2 If No Filler Found
    
    - Trigger **Split/Spill** logic for `current_cluster`:
        - If `current_cluster.width > shelf_width`:
            - Split into:
                - “Head” containing highest-priority SKUs / facing units that fit into `remaining_gap`.
                - “Tail” with remaining units to be placed on the next shelf (preferably just below, to form a vertical block).
        - Else:
            - Leave `remaining_gap` as a gap and move `current_cluster` to next shelf (or mark for Phase 4 adjustments).

---

### 3.3.3 Within-Cluster Layout

Once a cluster is assigned to a shelf at `[x_start, x_end]`:

- Place SKUs left → right according to cluster order.
- For each SKU:
    - Place all its front facings first.
    - Then place its stock facings to the right, forming a “stock tail”.

Orientation here is as decided in Phase 2: front vs stock.

**Output of Phase 3 (per facing):**

- `sku_id`
- `orientation` (`Front` or `Stock`)
- `shelf_id`
- `x_start_mm`
- `x_end_mm`

Also record:

- Gaps per shelf.
- Brand block structure.

---

# Phase 4: Optimizer (Refinement – SA + Orientation Rotations)

**Goal:** Refine layout to reduce global cost while preserving hard constraints.

---

## 4.1 Hard Constraints (Never Break)

- **Shelf capacity:**
    
    [
    shelf: facing_widths shelf_width_mm
    ]
    
- **Height feasibility:**
    - SKU only on shelves in `fit_shelves[sku_id]`.
- **Allocation integrity:**
    - Total facings per SKU = `F_front[i] + F_stock[i]` from Phase 2 (count).
    Orientation may be rotated *incrementally* but must respect DoS and min fronts.
- **Visibility:**
    
    [
    F_{front_visible}[i] min_{front_facings}[i]
    ]
    
    (total front-oriented facings across shelves)
    
- **Must-stock:**
    - For must-stock SKUs: total facings ≥ 1.

If a move would break any of these, it is rejected immediately.

---

## 4.2 Orientation State and RotateOrientation

Introduce **orientation per facing**:

- `orientation ∈ {Front, Side, Flat}`
(Stock facings generally use Side/Flat as per config.)

Phase 2 conceptually distinguishes `F_front` vs `F_stock`; Phase 4 refines **which facings** are front vs stock, within:

- `min_front_facings[i]`
- `max_side_share[i]`
- DoS constraints.

### RotateOrientation Move

- Pick a SKU `i` and one of its facings on a given shelf.
- If SKU’s `allowed_orientations` includes another orientation (e.g., Front → Side or Side → Front):
    1. Tentatively change orientation:
        - From Front → Stock orientation (Side/Flat)
        or Stock → Front.
    2. Recompute:
        - Facing width `w_eff`
        - Units per facing `u_eff`
        - Shelf width usage on that shelf
        - SKU’s overall DoS:
            
            [
            DoS_i = (  )
            ]
            
    3. Check constraints:
        - Shelf width not exceeded.
        - `DoS_i ≥ min_DoS_floor` (or doesn’t worsen beyond allowed slack).
        - New `front_facings_count[i] ≥ min_front_facings[i]`.
        - Side share:
            
            [
             max_{side_share}[i]
            ]
            
    4. If all checks pass:
        - Accept the orientation change as a **candidate state** for SA (not final yet).

This lets the optimizer exploit orientation flexibility to solve local space or vertical issues while obeying business rules.

---

## 4.3 Cost Function

All components normalized to [0,1]:

- `C_space`:
    
    Total gap length across all shelves / total shelf width.
    
- `C_brand`:
    
    Brand splits – number and severity of non-contiguous brand blocks.
    
- `C_vertical`:
    
    Weighted distance of clusters from their ideal vertical zones.
    
- `C_DoS`:
    
    Residual DoS shortfalls vs `min_DoS_floor` (penalize any SKU falling short).
    
- `C_variety`:
    
    Penalty if any SKU falls below 1 facing where it was present in Phase 2 (used if optional trimming allowed).
    
- `C_strategic`:
    
    Penalties if New/Hero/Core SKUs are off ideal shelves, below min fronts, or in low-visibility zones.
    
- `C_side`:
    
    Penalty for overuse of stock orientations, especially for Premium/Hero SKUs.
    
- `C_sideFrag`:
    
    Penalty when stock facings for a SKU are scattered rather than adjacent to its front block.
    
- `C_orientation`:
    
    Penalty for “non-front” dominant orientations when not justified by DoS/space.
    
    e.g., if a Premium SKU has most facings in Side/Flat, cost increases.
    

Total cost:

[
C_{total} =
w_{space} C_{space} +
w_{brand} C_{brand} +
w_{vertical} C_{vertical} +
w_{DoS} C_{DoS} +
w_{variety} C_{variety} +
w_{strategic} C_{strategic} +
w_{side} C_{side} +
w_{sideFrag} C_{sideFrag} +
w_{orientation} C_{orientation}
]

Weights set so:

- `w_DoS`, `w_variety`, `w_strategic` are large (hard-ish constraints).
- Orientation and aesthetic terms lower, guiding but not overriding.

---

## 4.4 Move Operators

All moves must respect hard constraints.

1. **SwapNeighbors**
    - Pick a shelf, swap two adjacent clusters or blocks.
    - Helps reduce brand splits and improve symmetry.
2. **VerticalShift**
    - Move a cluster up/down one shelf:
        - Check height fit and shelf capacity.
    - Used to align clusters with ideal vertical zones.
3. **IntraClusterReorder**
    - Reorder SKUs inside a cluster (or segment group) to improve left-to-right flow, brand storytelling, or gap metrics.
4. **SideTailReorder**
    - Rearrange stock tails within cluster, e.g., move them further right or to lower shelves while keeping adjacency.
5. **SideBlockShift**
    - Move a contiguous block of stock facings for SKU `i` downwards to a less visible shelf if it reduces `C_side` and fits.
6. **RotateOrientation**
    - As described above; swap a facing between Front and Stock modes when justified.
7. **FacingTrim** (optional, strictly guarded)
    - Only if, after all moves, a shelf still marginally overflows width.
    - Identify SKU with:
        - Lowest `S_i`
        - `total_facings[i] > minimum_allowed[i]` (from Phase 2 survival)
    - Prefer trimming a stock facing.
    - Recompute DoS and guard constraints. If any violation, revert.

---

## 4.5 Annealing Schedule

- Initialize:
    - `T = T0` (e.g. 1.0)
    - `best_layout = current_layout`
    - `best_cost = C_total`
- For iteration in `1..max_iterations`:
    - Randomly pick a move type (weighted).
    - Propose a move:
        - If it violates constraints, discard and sample another.
    - Compute `C_new`.
    - `ΔC = C_new - C_old`.
    - Acceptance rule:
        - If `ΔC < 0`: accept.
        - Else: accept with probability `exp(-ΔC / T)`.
    - If `C_new < best_cost`: update `best_layout`.
    - Update temperature:
        
        [
        T = T cooling_rate
        ]
        
    - Stop early if:
        - `T < T_min`, or
        - No improvements for N steps.

Final output: `best_layout`.

---

# Guardrails, Trickle-Down Checks, and Explainability

## After Each Phase

### After Phase 1

- Validate:
    - `S_i` distribution reasonable, no jitter spikes.
    - Metrics with no variance set to neutral 0.5.

---

### After Phase 2

- Strict checks:
    - `W_used ≤ total_shelf_width`.
    - Must-stock SKUs: `F_front + F_stock ≥ 1`.
    - For all SKUs:
        - `F_front ≥ min_front_facings`.
        - `F_front + F_stock ≤ max_facings_per_sku`.
        - If stock used:
            - Side share ≤ `max_side_share`.
    - No DoS below `min_DoS_floor` for survival set unless flagged.
- KPIs:
    - Variety (% SKUs with `F_front ≥ 1`).
    - DoS coverage distribution.
    - Total profit index vs baseline.
    - Side/stock usage summary.

---

### After Phase 3

- Validate:
    - No shelf overflow.
    - All SKUs respect `fit_shelves`.
- Record:
    - Gaps by shelf (for `C_space`).
    - Brand blocks and splits.

---

### After Phase 4

- Revalidate all hard constraints.
- Final KPIs:
    - Variety
    - DoS
    - Profit index
    - Side usage
    - Brand block quality
- Explainability per SKU:
    - `F_front`, `F_stock`
    - Orientation distribution
    - Side/stock reason tags
    - Shelf and X positions
    - Whether placed at ideal vertical zone or nearest feasible

---

# Benefits of this v2.1 CSP-Style Heuristic Engine

- **Computable CSP**
    
    Business rules are expressed as constraints and objectives, not slogans.
    
- **Phase separation**
    - Phase 2 does pure math allocation of facings under global constraints.
    - Phase 3 converts that to geometry with smart packing (look-ahead backfill).
    - Phase 4 refines with SA and orientation tweaks, bounded by hard rules.
- **Stability under real data**
    - Robust, anchored scaling avoids planogram jitter from new “monster” SKUs.
    - Outliers clipped and normalized properly.
- **Space efficiency**
    - Look-ahead backfill typically improves effective linear utilization by several points vs naive placement.
- **Orientation intelligence**
    - Front vs side/flat facings treated as distinct roles (selling vs stocking).
    - Clear, defensible triggers for when stock orientations are allowed.
    - Orientation rotations under SA fix edge cases without breaking DoS or visibility.
- **Explainable outcomes**
    - Every side facing, every rotation, every extra facing can be traced back to a concrete rule:
        - DoS pressure
        - Variety preservation
        - Profit density
        - Policy override
