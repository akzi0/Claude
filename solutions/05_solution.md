# Solution: CFG to SSA Walkthrough (Hard Exercise — Lesson 05)

## The Question

Given CFG:
- **B0** (entry): `x = 1`, `y = 2`, `if cond goto B1 else B2`
- **B1**: `x = x + y`, `goto B3`
- **B2**: `y = x * 2`, `goto B3`
- **B3** (exit): `z = x + y`, `return z`

CFG edges: `B0→B1`, `B0→B2`, `B1→B3`, `B2→B3`

**Questions:**
- (a) Dominator tree
- (b) Dominance frontiers
- (c) Phi function placement
- (d) Full SSA renaming
- (e) Constant folding when `cond = true`
- (f) Live variables and interference graph

---

## (a) Dominator Tree

**Definition:** Node `d` dominates node `n` if every path from the entry to `n` passes through `d`. The immediate dominator (`idom`) is the closest strict dominator.

**Computing dominators by inspection:**

- **B0:** Entry node. Dominates itself and everything reachable (B0 is the only entry). `idom(B0) = B0` (self).
- **B1:** Only reachable via B0 (the edge B0→B1 is the only path). `idom(B1) = B0`.
- **B2:** Only reachable via B0 (the edge B0→B2 is the only path). `idom(B2) = B0`.
- **B3:** Reachable via B0→B1→B3 and B0→B2→B3. Both paths pass through B0. B1 is NOT on all paths (the B0→B2→B3 path skips B1). B2 is NOT on all paths. So `idom(B3) = B0`.

```
Dominator Tree:
         B0          (idom = self)
       /  |  \
      B1  B2  B3     (all have idom = B0)
```

B0 strictly dominates B1, B2, and B3. No other dominance relationships exist between siblings.

---

## (b) Dominance Frontiers

**Definition:** The dominance frontier of `n` (`DF(n)`) is the set of blocks `y` such that `n` dominates a predecessor of `y` but does NOT strictly dominate `y`.

Equivalently, `y ∈ DF(n)` if:
- There exists an edge `p→y` in the CFG, and
- `n` dominates `p`, and
- `n` does not strictly dominate `y`.

**Algorithm (Cytron et al.):** For each join point `y` with multiple predecessors, traverse from each predecessor up the dominator tree until we reach `idom(y)`, adding `y` to the DF of each node visited.

**Applying to each block:**

`B3` is the only join point (has predecessors B1 and B2):

From predecessor **B1**:
- B1 ≠ idom(B3) = B0, so add B3 to DF(B1): `DF(B1) = {B3}`
- Move up: next is idom(B1) = B0 = idom(B3), stop.

From predecessor **B2**:
- B2 ≠ idom(B3) = B0, so add B3 to DF(B2): `DF(B2) = {B3}`
- Move up: next is idom(B2) = B0 = idom(B3), stop.

`B1`, `B2` have only one predecessor each (B0), so they have no join points:
- `DF(B0) = {}` (B0 dominates everything — nothing is on a frontier)
- `DF(B3) = {}` (no successors; or: B3 has no path that doesn't dominate itself as entry has none)

**Result:**
```
DF(B0) = {}
DF(B1) = {B3}
DF(B2) = {B3}
DF(B3) = {}
```

**Intuition:** B3 is in DF(B1) because B1 dominates its predecessor (itself, via B1→B3), but B1 does not dominate B3 (the B0→B2→B3 path avoids B1). B3 is a "convergence point" that B1 influences but doesn't control exclusively.

---

## (c) Phi Function Placement

**Rule:** Insert `φ` (phi) functions at the dominance frontier of each variable's definition sites.

**Variables and definition sites:**
- `x`: defined in B0 (x=1) and B1 (x=x+y) → `defs(x) = {B0, B1}`
- `y`: defined in B0 (y=2) and B2 (y=x*2) → `defs(y) = {B0, B2}`
- `z`: defined in B3 → `defs(z) = {B3}` (no join point in DF)
- `cond`: defined in B0 → `defs(cond) = {B0}` (no phi needed)

**IDF (Iterated Dominance Frontier):** Compute DF of all definition sites, then if a phi is inserted it becomes a new definition site (requiring another DF pass). Iterate until no new phis needed.

**For `x`:**
- IDF({B0, B1}) = DF(B0) ∪ DF(B1) = {} ∪ {B3} = **{B3}**
- Insert `x = φ(...)` at B3. Is B3 a new definition site? Yes, but DF(B3) = {}, so no further phis.
- **φ for x placed at: B3** ✓

**For `y`:**
- IDF({B0, B2}) = DF(B0) ∪ DF(B2) = {} ∪ {B3} = **{B3}**
- Insert `y = φ(...)` at B3.
- **φ for y placed at: B3** ✓

**For `z`:** DF(B3) = {} → no phi needed.

---

## (d) Full SSA Renaming

SSA renaming gives each variable definition a unique subscript. The renaming algorithm does a DFS over the dominator tree, maintaining a stack of current versions for each variable.

### Renaming pass (DFS order: B0 → B1 → B3 → back to B0 → B2 → B3)

**Processing B0:**
```
x_0 = 1           ← new definition, push x_0 onto stack[x]
y_0 = 2           ← new definition, push y_0 onto stack[y]
cond_0 = ...      ← new definition (where cond comes from)
if cond_0 goto B1 else B2
```
Stack state after B0: `stack[x]=[x_0]`, `stack[y]=[y_0]`

**Processing B1 (child of B0 in dom tree):**
```
x_1 = x_0 + y_0  ← uses top of stack[x]=x_0, top of stack[y]=y_0; new def x_1
goto B3
```
Fill B3's phi for x from B1: argument = x_1 (top of stack[x])
Fill B3's phi for y from B1: argument = y_0 (top of stack[y])
Stack state: `stack[x]=[x_0, x_1]`, `stack[y]=[y_0]`

**Processing B3 (successor, DFS visits after B1):**
- B3 has phi nodes. At this point we've visited from B1 (will also need to visit from B2).
- Phis get one argument from each predecessor in order.

**Backtrack to B0, then process B2:**
Pop x_1 from stack: `stack[x]=[x_0]`, `stack[y]=[y_0]`

**Processing B2:**
```
y_1 = x_0 * 2    ← uses top of stack[x]=x_0; new def y_1
goto B3
```
Fill B3's phi for x from B2: argument = x_0 (top of stack[x])
Fill B3's phi for y from B2: argument = y_1 (top of stack[y])
Stack state: `stack[x]=[x_0]`, `stack[y]=[y_0, y_1]`

**B3 with phi arguments filled:**
```
x_2 = φ(x_1 [from B1], x_0 [from B2])
y_2 = φ(y_0 [from B1], y_1 [from B2])
z_0 = x_2 + y_2
return z_0
```

### Complete SSA Form

```
B0:  x_0 = 1
     y_0 = 2
     cond_0 = <input>
     if cond_0 goto B1 else B2

B1:  x_1 = x_0 + y_0
     goto B3

B2:  y_1 = x_0 * 2
     goto B3

B3:  x_2 = φ(x_1 [from B1], x_0 [from B2])
     y_2 = φ(y_0 [from B1], y_1 [from B2])
     z_0 = x_2 + y_2
     return z_0
```

**Verification:** Every use of a variable (x_0, y_0, x_1, y_1, x_2, y_2) reaches exactly one definition. Each variable is defined exactly once. The phi functions have exactly 2 arguments (one per predecessor of B3: B1 and B2).

---

## (e) Constant Folding When `cond = true`

When `cond_0 = true` at compile time, SCCP (Sparse Conditional Constant Propagation) marks the edge `B0→B2` as **not executable** and prunes B2.

**Step 1: Prune unreachable edge B0→B2**
- B2 is no longer reachable.
- The phi argument from B2 is removed: single-argument phis collapse.

**Step 2: Phi collapse**
```
x_2 = φ(x_1)  →  x_2 = x_1   (only B1 predecessor remains)
y_2 = φ(y_0)  →  y_2 = y_0   (only B1 predecessor remains)
```

**Step 3: Constant propagation (substituting known values)**
- `x_0 = 1` → substitute x_0=1 everywhere
- `y_0 = 2` → substitute y_0=2 everywhere
- `x_1 = x_0 + y_0 = 1 + 2 = 3` → constant fold → `x_1 = 3`
- `x_2 = x_1 = 3`
- `y_2 = y_0 = 2`
- `z_0 = x_2 + y_2 = 3 + 2 = 5`

**Step 4: Dead Code Elimination**
- `x_0 = 1`: dead after inlining (no uses remain)
- `y_0 = 2`: dead after inlining
- `x_1 = 3`: dead (x_2 = 3 directly)
- `cond_0 = true`: dead
- All of B2 is dead

**Final optimized output:**
```
B0:  (unconditional) goto B1
B1:  (empty — all computations folded)
B3:  return 5
```

Which simplifies to just: `return 5`. The entire function is a constant.

---

## (f) Live Variables and Interference Graph

### Live Variable Analysis (backward dataflow)

For each block, compute `LiveOut` = variables live at exit, `LiveIn` = variables live at entry.

**Transfer function:** `LiveIn[B] = Gen[B] ∪ (LiveOut[B] − Kill[B])`
- `Gen[B]`: variables used in B before being defined
- `Kill[B]`: variables defined in B

Working in the SSA form (general case, cond unknown):

**B3 (exit):**
- Uses: x_2, y_2 (for `z_0 = x_2 + y_2`)
- Defines: z_0
- LiveOut[B3] = {} (exit block)
- LiveIn[B3] = {x_2, y_2} ∪ (phi inputs from predecessors)

**B1:**
- Uses: x_0, y_0 (for `x_1 = x_0 + y_0`)
- Defines: x_1
- LiveOut[B1] = phi-function arguments from B1 = {x_1, y_0} (these feed B3's phis)
- LiveIn[B1] = {x_0, y_0}

**B2:**
- Uses: x_0 (for `y_1 = x_0 * 2`)
- Defines: y_1
- LiveOut[B2] = phi-function arguments from B2 = {x_0, y_1}
- LiveIn[B2] = {x_0}

**B0:**
- Defines: x_0, y_0, cond_0
- LiveOut[B0] = LiveIn[B1] ∪ LiveIn[B2] = {x_0, y_0} ∪ {x_0} = {x_0, y_0}
- LiveIn[B0] = {cond} (cond is the only input from outside)

### Interference Graph

Two variables **interfere** if they are simultaneously live at any program point.

**At end of B1** (before goto B3): `{x_1, y_0}` are live → edge x_1 ─── y_0

**At end of B2** (before goto B3): `{x_0, y_1}` are live → edge x_0 ─── y_1

**At B3 (after phis):**
- After `x_2 = φ(...)` and `y_2 = φ(...)`: both x_2 and y_2 live → edge x_2 ─── y_2
- During `z_0 = x_2 + y_2`: x_2 and y_2 still live

**Interference graph:**
```
x_0 ────── y_1         (both live at end of B2)
x_1 ────── y_0         (both live at end of B1)
x_2 ────── y_2         (both live during z_0 computation)
```

The graph has 3 independent edges — it's 3 disjoint pairs. This means it's 2-colorable (2 registers suffice).

### 2-Register Allocation

| Variable | Register | Justification |
|----------|----------|---------------|
| x_0      | R0       | First def |
| y_0      | R1       | Interferes with x_0 indirectly (via x_1/y_0 edge → must be R1) |
| x_1      | R0       | Interferes with y_0 (R1) → must be R0 |
| y_1      | R1       | Interferes with x_0 (R0) → must be R1 |
| x_2      | R0       | x_1 dies before x_2 is live → can reuse R0 |
| y_2      | R1       | Interferes with x_2 (R0) → must be R1 |
| z_0      | R0       | x_2/y_2 die after the add → R0 free for z_0 |

No spills needed. The CFG structure (B1 and B2 are independent paths) ensures no 3-way simultaneous liveness.

---

## Python Implementation

```python
"""
CFG → SSA construction.
Builds the CFG from the exercise, runs dominator analysis, 
places phi functions, and prints the SSA form.
"""
from dataclasses import dataclass, field
from typing import List, Dict, Set, Optional, Tuple


@dataclass
class Instruction:
    """Single SSA instruction."""
    dest: Optional[str]   # None for branches/returns
    op: str
    args: List[str] = field(default_factory=list)
    is_phi: bool = False

    def __repr__(self):
        if self.is_phi:
            args_str = ", ".join(f"{v}[from {b}]" for v, b in zip(self.args[::2], self.args[1::2]))
            return f"{self.dest} = φ({args_str})"
        if self.dest:
            return f"{self.dest} = {self.op}({', '.join(self.args)})"
        return f"{self.op}({', '.join(self.args)})"


@dataclass
class BasicBlock:
    label: str
    instructions: List[Instruction] = field(default_factory=list)
    successors: List[str] = field(default_factory=list)
    predecessors: List[str] = field(default_factory=list)


class CFG:
    def __init__(self):
        self.blocks: Dict[str, BasicBlock] = {}
        self.entry: str = ""
    
    def add_block(self, label: str) -> BasicBlock:
        b = BasicBlock(label)
        self.blocks[label] = b
        if not self.entry:
            self.entry = label
        return b
    
    def add_edge(self, src: str, dst: str):
        self.blocks[src].successors.append(dst)
        self.blocks[dst].predecessors.append(src)


def build_exercise_cfg() -> CFG:
    """Build the CFG from the hard exercise."""
    cfg = CFG()
    
    b0 = cfg.add_block("B0")
    b1 = cfg.add_block("B1")
    b2 = cfg.add_block("B2")
    b3 = cfg.add_block("B3")
    
    b0.instructions = [
        Instruction("x", "const", ["1"]),
        Instruction("y", "const", ["2"]),
        Instruction(None, "branch", ["cond", "B1", "B2"]),
    ]
    b1.instructions = [
        Instruction("x", "add", ["x", "y"]),
        Instruction(None, "goto", ["B3"]),
    ]
    b2.instructions = [
        Instruction("y", "mul", ["x", "2"]),
        Instruction(None, "goto", ["B3"]),
    ]
    b3.instructions = [
        Instruction("z", "add", ["x", "y"]),
        Instruction(None, "return", ["z"]),
    ]
    
    cfg.add_edge("B0", "B1")
    cfg.add_edge("B0", "B2")
    cfg.add_edge("B1", "B3")
    cfg.add_edge("B2", "B3")
    
    return cfg


def compute_dominators(cfg: CFG) -> Dict[str, str]:
    """Compute immediate dominators using Cooper's algorithm."""
    labels = list(cfg.blocks.keys())
    idom: Dict[str, Optional[str]] = {l: None for l in labels}
    idom[cfg.entry] = cfg.entry
    
    # Reverse postorder (RPO) traversal
    def rpo(label: str, visited: Set[str], order: List[str]):
        visited.add(label)
        for succ in cfg.blocks[label].successors:
            if succ not in visited:
                rpo(succ, visited, order)
        order.insert(0, label)
    
    order = []
    rpo(cfg.entry, set(), order)
    label_to_rpo = {l: i for i, l in enumerate(order)}
    
    def intersect(b1: str, b2: str) -> str:
        while b1 != b2:
            while label_to_rpo[b1] > label_to_rpo[b2]:
                b1 = idom[b1]
            while label_to_rpo[b2] > label_to_rpo[b1]:
                b2 = idom[b2]
        return b1
    
    changed = True
    while changed:
        changed = False
        for label in order:
            if label == cfg.entry:
                continue
            preds_with_idom = [p for p in cfg.blocks[label].predecessors if idom[p] is not None]
            if not preds_with_idom:
                continue
            new_idom = preds_with_idom[0]
            for p in preds_with_idom[1:]:
                new_idom = intersect(new_idom, p)
            if idom[label] != new_idom:
                idom[label] = new_idom
                changed = True
    
    return idom


def compute_dominance_frontiers(cfg: CFG, idom: Dict[str, str]) -> Dict[str, Set[str]]:
    """Compute dominance frontiers."""
    df: Dict[str, Set[str]] = {l: set() for l in cfg.blocks}
    
    for label, block in cfg.blocks.items():
        if len(block.predecessors) >= 2:  # join point
            for pred in block.predecessors:
                runner = pred
                while runner != idom[label]:
                    df[runner].add(label)
                    runner = idom[runner]
    
    return df


def insert_phi_functions(cfg: CFG, df: Dict[str, Set[str]]):
    """Insert phi function placeholders at dominance frontier."""
    # Collect definitions per variable
    defs: Dict[str, Set[str]] = {}
    for label, block in cfg.blocks.items():
        for instr in block.instructions:
            if instr.dest:
                defs.setdefault(instr.dest, set()).add(label)
    
    # Insert phis
    for var, def_blocks in defs.items():
        worklist = list(def_blocks)
        has_phi: Set[str] = set()
        while worklist:
            block_label = worklist.pop()
            for frontier_label in df[block_label]:
                if frontier_label not in has_phi:
                    # Insert phi at beginning of frontier block
                    preds = cfg.blocks[frontier_label].predecessors
                    phi = Instruction(var, "phi", [], is_phi=True)
                    phi.args = ["?"] * (2 * len(preds))  # placeholder
                    cfg.blocks[frontier_label].instructions.insert(0, phi)
                    has_phi.add(frontier_label)
                    if frontier_label not in def_blocks:
                        worklist.append(frontier_label)


def rename_variables(cfg: CFG, idom: Dict[str, str]):
    """SSA renaming: assign unique subscripts to each definition."""
    counters: Dict[str, int] = {}
    stacks: Dict[str, List[str]] = {}
    
    def new_name(var: str) -> str:
        i = counters.get(var, 0)
        counters[var] = i + 1
        name = f"{var}_{i}"
        stacks.setdefault(var, []).append(name)
        return name
    
    def top(var: str) -> str:
        return stacks[var][-1] if stacks.get(var) else var
    
    # Build dom tree children
    children: Dict[str, List[str]] = {l: [] for l in cfg.blocks}
    for label in cfg.blocks:
        if label != cfg.entry:
            children[idom[label]].append(label)
    
    def rename(label: str, push_counts: Dict[str, int]):
        block = cfg.blocks[label]
        local_pushes: Dict[str, int] = {}
        
        for instr in block.instructions:
            if not instr.is_phi:
                # Rename uses first
                instr.args = [top(a) if a in stacks or a.isalpha() else a
                              for a in instr.args]
            if instr.dest:
                new = new_name(instr.dest)
                local_pushes[instr.dest] = local_pushes.get(instr.dest, 0) + 1
                instr.dest = new
        
        # Fill phi arguments in successors
        for succ_label in block.successors:
            succ = cfg.blocks[succ_label]
            pred_idx = succ.predecessors.index(label)
            for instr in succ.instructions:
                if instr.is_phi:
                    orig_var = instr.dest.rsplit('_', 1)[0] if '_' in (instr.dest or '') else instr.dest
                    # Find original variable name before renaming
                    # (Use the placeholder we set during phi insertion)
                    instr.args[pred_idx * 2] = top(orig_var) if stacks.get(orig_var) else '?'
                    instr.args[pred_idx * 2 + 1] = label
        
        for child in children[label]:
            rename(child, {})
        
        # Pop local definitions
        for var, count in local_pushes.items():
            for _ in range(count):
                stacks[var].pop()
    
    rename(cfg.entry, {})


def print_ssa(cfg: CFG):
    """Print the SSA form."""
    for label, block in cfg.blocks.items():
        print(f"{label}:")
        for instr in block.instructions:
            print(f"  {instr}")
        print()


def main():
    cfg = build_exercise_cfg()
    
    print("=== Original CFG ===")
    print_ssa(cfg)
    
    idom = compute_dominators(cfg)
    print("=== Immediate Dominators ===")
    for label, dom in idom.items():
        print(f"  idom({label}) = {dom}")
    
    df = compute_dominance_frontiers(cfg, idom)
    print("\n=== Dominance Frontiers ===")
    for label, frontier in df.items():
        print(f"  DF({label}) = {sorted(frontier)}")
    
    insert_phi_functions(cfg, df)
    rename_variables(cfg, idom)
    
    print("\n=== SSA Form ===")
    print_ssa(cfg)


if __name__ == "__main__":
    main()
```

---

## Common Mistakes and Traps

1. **Confusing dom tree levels with idom.** B3's idom is B0, not B1 or B2, because neither B1 nor B2 is on ALL paths to B3.

2. **DF(B3) = {} in this example.** B3 has no successors in the CFG (it's the exit), so no block y exists where B3 dominates a predecessor but not y.

3. **Phi argument order matters.** The i-th phi argument must correspond to the i-th predecessor. If B3's predecessors are [B1, B2], then `φ(x_1, x_0)` means x_1 comes from B1 and x_0 from B2.

4. **Single-argument phis after DCE.** When cond=true, B2 is dead, and `φ(x_1)` becomes just `x_2 = x_1`. This trivial phi is immediately eliminated.

5. **Live variables ≠ SSA variables defined.** A variable's SSA subscript tells you WHICH definition it refers to, not whether it's currently live. You still need backward dataflow analysis for liveness.

6. **Interference = simultaneous liveness, not just both defined.** `x_0` and `x_1` are NOT necessarily interfering (x_0 might be dead by the time x_1 is defined). Check liveness, not definition sites.
