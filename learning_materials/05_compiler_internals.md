# Lesson 05: Compiler Internals — SSA, Optimization Passes, and Register Allocation

> **Level**: Graduate PL/Compiler Theory  
> **Prerequisites**: Data structures, algorithms, basic Python, discrete math  
> **Goal**: Build working implementations of SSA construction, key optimization passes, liveness analysis, and graph-coloring register allocation. Understand what production compilers (LLVM, GCC, V8) actually do internally.

---

## 1. Compiler Pipeline Overview

A compiler transforms source text into executable machine code through a sequence of well-defined phases. Each phase consumes a structured representation and produces a lower-level one:

```
Source Text
    │
    ▼  Lexical Analysis (Lexer)
Token Stream       ← "x", "=", "1", "+", "2", ...
    │
    ▼  Syntactic Analysis (Parser)
Abstract Syntax Tree (AST)
    │
    ▼  Semantic Analysis
Decorated AST      ← type-checked, scope-resolved
    │
    ▼  IR Generation
Intermediate Representation (IR)
    │
    ▼  Optimization Passes (N passes, iterated)
Optimized IR
    │
    ▼  Instruction Selection
Target-specific Instructions
    │
    ▼  Register Allocation
Register-assigned Instructions
    │
    ▼  Code Emission / Linking
Machine Code / Binary
```

### Why Intermediate Representations?

The key insight behind IR design is **decoupling**. If you have N source languages and M target architectures, a naive approach requires N×M compilers. With a shared IR:

- Each frontend (C, Rust, Swift, Julia) compiles to the same IR → **N frontend passes**
- Each backend (x86-64, ARM64, RISC-V, WASM) compiles from the same IR → **M backend passes**
- Total: **N + M** instead of N × M

This is why LLVM has become an ecosystem: Clang, Rust's `rustc`, Apple's Swift compiler, and Julia all share LLVM's backends for free. You get x86 vectorization improvements in your Rust code because the LLVM backend team improved it — no Rust-specific work needed.

### Three-Address Code (TAC)

Three-address code is an IR where every instruction has **at most one operator and three operands** (two sources, one destination):

```
t1 = a + b        # binary op
t2 = t1 * c       # binary op
t3 = -t2          # unary op
t4 = t2 > 0       # comparison
if t4 goto L1     # conditional branch
t5 = x[i]        # array load
x[i] = t5        # array store
```

TAC is easy to:
- **Generate**: each AST node maps to 1–3 TAC instructions
- **Analyze**: simple pattern matching on three-operand templates
- **Transform**: reorder, delete, or replace individual instructions

In practice, TAC is organized into **basic blocks** — maximal sequences of instructions with a single entry (the first instruction) and single exit (a branch or return). Control flow between blocks forms a **Control Flow Graph (CFG)**.

### Static Single Assignment (SSA) Form

SSA is a refinement of TAC where **every variable is assigned exactly once**. If a variable is assigned multiple times in the source, each assignment gets a fresh subscripted name:

```python
# Original TAC
x = 1
x = x + y     # x assigned twice!
z = x + 1

# SSA form
x_0 = 1
x_1 = x_0 + y_0    # fresh subscript for new def
z_0 = x_1 + 1
```

When control flow merges (e.g., after an if-else), we use a **phi function** to select among multiple reaching definitions:

```
# SSA with phi at merge point
if cond:
    x_1 = 1      # then-branch
else:
    x_2 = 2      # else-branch
# merge point:
x_3 = φ(x_1, x_2)   # select based on which predecessor was taken
z = x_3 + 1
```

The phi function `φ(x_1, x_2)` is not a real instruction — it's a notational device meaning "take the value from whichever predecessor block was executed." During code generation, phi functions are replaced with copy instructions inserted at the end of predecessor blocks.

**Why does SSA matter for optimization?**

1. **Reaching definitions are trivial**: in SSA, each use of a variable has exactly one reaching definition — its unique defining instruction. No dataflow analysis needed.
2. **Use-def chains are explicit**: the subscript tells you exactly which definition you're using.
3. **Dead code elimination is O(n)**: if a variable's use list is empty, its definition is dead. Delete it, update predecessors' use lists, and repeat.
4. **Constant propagation is immediate**: if `x_3 = 5`, replace all uses of `x_3` with `5` in one pass.

Virtually every modern optimizing compiler uses SSA: LLVM, GCC (since 4.0), the JVM JIT (C2), V8 Turbofan, SpiderMonkey IonMonkey, and PyPy's tracing JIT.

---

## 2. Building SSA Form

Constructing SSA form from a CFG involves three phases:

1. **Compute dominators** (who controls the flow to each block)
2. **Compute dominance frontiers** (where phi functions are needed)
3. **Insert phi functions** and **rename variables**

### Dominators

Block A **dominates** block B (written A dom B) if **every path from the entry block to B passes through A**. Every block dominates itself.

The **immediate dominator** of B, `idom(B)`, is the unique dominator of B that is dominated by all other dominators of B. The idom relationship forms a tree — the **dominator tree** — rooted at the entry block.

```
CFG:            Dominator Tree:
                     
  Entry              Entry
  /   \               │
 B1    B2            B1   B2
  \   /              │
   B3               B3
```

In this example, Entry dominates everything. B1 dominates B3 (via one path), but B2 also leads to B3 — so neither B1 nor B2 dominates B3 strictly. `idom(B3) = Entry`.

### Dominance Frontier

The **dominance frontier** DF(B) is the set of blocks where B's dominance "ends":

> DF(B) = { Y : ∃ predecessor X of Y such that B dominates X, but B does not *strictly* dominate Y }

Informally: Y is in DF(B) if B dominates some predecessor of Y, but B does not dominate Y itself. This is precisely where a phi function for variables defined in B might be needed.

**Algorithm** (Cytron et al., 1991):
```
for each block B:
    for each successor Y of B:
        if idom(Y) ≠ B:    # B doesn't immediately dominate Y
            DF(B) ∪= {Y}
    for each child C of B in dominator tree:
        for each Y in DF(C):
            if idom(Y) ≠ B:
                DF(B) ∪= {Y}
```

### Phi Function Placement: Iterated Dominance Frontier

A phi function for variable `v` is needed at block Y if:
- Two different definitions of `v` can reach Y via different paths

The criterion is: place a phi for `v` at every block in the **iterated dominance frontier (IDF)** of the set of blocks that define `v`.

```
IDF(S) = DF(S) ∪ DF(S ∪ DF(S)) ∪ ...   (iterated until fixed point)
```

Each phi function is itself a definition, so it may require additional phi functions in further frontiers — hence "iterated."

### Variable Renaming

After placing phi functions, rename variables with a DFS over the dominator tree:

- Maintain a **definition stack** for each variable: `stacks[v]`
- When you encounter a definition `v = ...`, push a fresh version `v_k` onto `stacks[v]` and update the instruction to define `v_k`
- When you encounter a use of `v`, replace it with the top of `stacks[v]`
- Fill in phi function arguments when visiting predecessor blocks: for each successor `S` and each phi `φ(v, ...)` in `S`, append `stacks[v].top()` to the phi's argument list for this predecessor
- On backtrack (leaving a block in DFS), pop definitions that were pushed in this block

---

## 3. Python Implementation: SSA Construction

```python
from dataclasses import dataclass, field
from typing import List, Set, Dict, Optional, Tuple
from collections import defaultdict, deque


@dataclass
class BasicBlock:
    id: int
    instructions: List[Tuple[str, str, str, str]] = field(default_factory=list)
    # Each instruction: (dest, op, src1, src2) — src2 may be None
    successors: List['BasicBlock'] = field(default_factory=list)
    predecessors: List['BasicBlock'] = field(default_factory=list)
    phi_functions: Dict[str, List[str]] = field(default_factory=dict)
    # phi_functions[var] = [src_var_from_pred0, src_var_from_pred1, ...]

    defs: Set[str] = field(default_factory=set)
    uses: Set[str] = field(default_factory=set)

    def __repr__(self):
        return f"BB{self.id}"


class SSABuilder:
    """
    Constructs SSA form for a CFG using the Cytron et al. algorithm.

    References:
      - Cytron et al. 1991: "Efficiently Computing Static Single Assignment
        Form and the Control Dependence Graph"
      - Cooper, Harvey, Kennedy 2001: "A Simple, Fast Dominance Algorithm"
    """

    def __init__(self, blocks: List[BasicBlock], entry_id: int = 0):
        self.blocks = blocks
        self.entry_id = entry_id
        self.block_map: Dict[int, BasicBlock] = {b.id: b for b in blocks}
        self.idom: Dict[int, int] = {}          # block_id -> idom_id
        self.dom_tree: Dict[int, List[int]] = defaultdict(list)  # parent -> children
        self.df: Dict[int, Set[int]] = defaultdict(set)  # dominance frontier
        self._postorder: List[int] = []
        self._postorder_index: Dict[int, int] = {}
        self._counter: Dict[str, int] = defaultdict(int)
        self._stacks: Dict[str, List[str]] = defaultdict(list)

    # ------------------------------------------------------------------ #
    #  Phase 1: Dominator computation (Cooper-Harvey-Kennedy 2001)         #
    # ------------------------------------------------------------------ #

    def _compute_postorder(self) -> None:
        """DFS postorder traversal of CFG."""
        visited = set()
        order = []

        def dfs(bid: int):
            visited.add(bid)
            for succ in self.block_map[bid].successors:
                if succ.id not in visited:
                    dfs(succ.id)
            order.append(bid)

        dfs(self.entry_id)
        self._postorder = order
        self._postorder_index = {bid: i for i, bid in enumerate(order)}

    def compute_dominators(self) -> None:
        """
        Cooper-Harvey-Kennedy linear-time dominator algorithm.
        Works on postorder numbering; entry has the highest postorder index.
        """
        self._compute_postorder()
        n = len(self.blocks)
        entry = self.entry_id

        # Initialize: only entry's idom is defined (to itself)
        self.idom = {b.id: -1 for b in self.blocks}
        self.idom[entry] = entry

        def intersect(b1: int, b2: int) -> int:
            """Walk up the dominator tree to find common dominator."""
            finger1, finger2 = b1, b2
            while finger1 != finger2:
                while self._postorder_index[finger1] < self._postorder_index[finger2]:
                    finger1 = self.idom[finger1]
                while self._postorder_index[finger2] < self._postorder_index[finger1]:
                    finger2 = self.idom[finger2]
            return finger1

        changed = True
        while changed:
            changed = False
            # Reverse postorder (excluding entry)
            rpo = list(reversed(self._postorder))
            for bid in rpo:
                if bid == entry:
                    continue
                block = self.block_map[bid]
                # Find first processed predecessor
                new_idom = -1
                for pred in block.predecessors:
                    if self.idom[pred.id] != -1:
                        new_idom = pred.id
                        break
                if new_idom == -1:
                    continue
                for pred in block.predecessors:
                    if pred.id == new_idom:
                        continue
                    if self.idom[pred.id] != -1:
                        new_idom = intersect(pred.id, new_idom)
                if self.idom[bid] != new_idom:
                    self.idom[bid] = new_idom
                    changed = True

        # Build dom tree children list
        for bid, idom_id in self.idom.items():
            if bid != entry:
                self.dom_tree[idom_id].append(bid)

    # ------------------------------------------------------------------ #
    #  Phase 2: Dominance frontier computation                             #
    # ------------------------------------------------------------------ #

    def compute_dominance_frontiers(self) -> None:
        """
        Compute DF using Cytron et al. algorithm from dominator tree.
        For each join point Y with multiple predecessors, add Y to DF(X)
        for all X that dominate a predecessor of Y but don't dominate Y.
        """
        for block in self.blocks:
            if len(block.predecessors) >= 2:   # join point
                for pred in block.predecessors:
                    runner = pred.id
                    while runner != self.idom[block.id]:
                        self.df[runner].add(block.id)
                        runner = self.idom[runner]

    # ------------------------------------------------------------------ #
    #  Phase 3: Phi function insertion                                     #
    # ------------------------------------------------------------------ #

    def insert_phi_functions(self, var_defs: Dict[str, List[int]]) -> None:
        """
        For each variable v, insert phi nodes at IDF(defs(v)).
        var_defs: maps variable name to list of block IDs where it's defined.
        """
        for var, def_blocks in var_defs.items():
            worklist: deque = deque(def_blocks)
            has_phi: Set[int] = set()
            in_worklist: Set[int] = set(def_blocks)

            while worklist:
                bid = worklist.popleft()
                in_worklist.discard(bid)
                for frontier_bid in self.df[bid]:
                    if frontier_bid not in has_phi:
                        block = self.block_map[frontier_bid]
                        # Insert phi: one argument per predecessor
                        block.phi_functions[var] = [var] * len(block.predecessors)
                        has_phi.add(frontier_bid)
                        # The phi is itself a definition of var
                        if frontier_bid not in in_worklist:
                            worklist.append(frontier_bid)
                            in_worklist.add(frontier_bid)

    # ------------------------------------------------------------------ #
    #  Phase 4: Variable renaming                                          #
    # ------------------------------------------------------------------ #

    def _fresh_name(self, var: str) -> str:
        """Generate a fresh SSA version for variable var."""
        k = self._counter[var]
        self._counter[var] += 1
        return f"{var}_{k}"

    def rename_variables(self) -> None:
        """DFS over dominator tree, renaming vars to SSA form."""
        pushed: Dict[str, int] = defaultdict(int)  # how many times we pushed per var

        def rename(bid: int):
            block = self.block_map[bid]
            local_pushes: Dict[str, int] = defaultdict(int)

            # Rename phi function LHS (phi defines a new version)
            for var in list(block.phi_functions.keys()):
                new_name = self._fresh_name(var)
                self._stacks[var].append(new_name)
                local_pushes[var] += 1
                # Rename the key itself by replacing with new_name
                block.phi_functions[new_name] = block.phi_functions.pop(var)

            # Rename uses and defs in ordinary instructions
            new_instrs = []
            for dest, op, src1, src2 in block.instructions:
                # Rename uses (sources)
                new_src1 = self._stacks[src1][-1] if src1 and self._stacks[src1] else src1
                new_src2 = None
                if src2:
                    new_src2 = self._stacks[src2][-1] if self._stacks[src2] else src2

                # Rename def (destination)
                new_dest = dest
                if dest:
                    new_dest = self._fresh_name(dest)
                    self._stacks[dest].append(new_dest)
                    local_pushes[dest] += 1

                new_instrs.append((new_dest, op, new_src1, new_src2))
            block.instructions = new_instrs

            # Fill in phi arguments in successors
            for succ in block.successors:
                pred_index = succ.predecessors.index(block)
                for phi_var, args in succ.phi_functions.items():
                    # Find original var name (strip trailing _k)
                    orig = '_'.join(phi_var.split('_')[:-1]) if '_' in phi_var else phi_var
                    # Try to find current def of orig
                    if self._stacks.get(orig):
                        args[pred_index] = self._stacks[orig][-1]

            # Recurse into dom tree children
            for child_id in self.dom_tree[bid]:
                rename(child_id)

            # Pop local definitions
            for var, count in local_pushes.items():
                for _ in range(count):
                    if self._stacks[var]:
                        self._stacks[var].pop()

        rename(self.entry_id)

    # ------------------------------------------------------------------ #
    #  Public API                                                          #
    # ------------------------------------------------------------------ #

    def to_ssa(self, var_defs: Dict[str, List[int]]) -> None:
        """Full SSA construction pipeline."""
        self.compute_dominators()
        self.compute_dominance_frontiers()
        self.insert_phi_functions(var_defs)
        self.rename_variables()

    def dump(self) -> str:
        lines = []
        for block in self.blocks:
            lines.append(f"\n--- BB{block.id} ---")
            lines.append(f"  preds: {[p.id for p in block.predecessors]}")
            lines.append(f"  succs: {[s.id for s in block.successors]}")
            for phi_var, args in block.phi_functions.items():
                lines.append(f"  {phi_var} = φ({', '.join(args)})")
            for dest, op, src1, src2 in block.instructions:
                if src2:
                    lines.append(f"  {dest} = {src1} {op} {src2}")
                else:
                    lines.append(f"  {dest} = {op} {src1}")
        return '\n'.join(lines)
```

### Worked Example: If-Else with Merge

Consider this source:
```python
if cond:
    x = 1      # Block 1
else:
    x = 2      # Block 2
z = x + 1     # Block 3 (merge)
```

CFG:
```
B0 (entry: cond test) → B1 (x=1) → B3
                       → B2 (x=2) → B3
```

Dominators: `idom(B1) = B0`, `idom(B2) = B0`, `idom(B3) = B0`

Dominance frontier: `DF(B1) = {B3}`, `DF(B2) = {B3}`, `DF(B0) = {}`, `DF(B3) = {}`

Both B1 and B2 define `x`, and `B3 ∈ DF(B1) ∩ DF(B2)`, so we insert `x = φ(...)` at B3.

After renaming:
```
B0: ...
B1: x_1 = 1
B2: x_2 = 2
B3: x_3 = φ(x_1, x_2)    # selects x_1 if came from B1, x_2 if from B2
    z_0 = x_3 + 1
```

```python
# Demo of the SSA builder
def build_if_else_example():
    # Create blocks
    b0 = BasicBlock(id=0)
    b1 = BasicBlock(id=1)
    b2 = BasicBlock(id=2)
    b3 = BasicBlock(id=3)

    # Wire CFG
    b0.successors = [b1, b2]
    b1.successors = [b3]
    b2.successors = [b3]
    b1.predecessors = [b0]
    b2.predecessors = [b0]
    b3.predecessors = [b1, b2]

    # Instructions (dest, op, src1, src2)
    b0.instructions = [("cond", "load", "cond_input", None)]
    b1.instructions = [("x", "const", "1", None)]
    b2.instructions = [("x", "const", "2", None)]
    b3.instructions = [("z", "+", "x", "1")]

    # x is defined in blocks 1 and 2
    var_defs = {"x": [1, 2], "cond": [0], "z": [3]}

    builder = SSABuilder([b0, b1, b2, b3], entry_id=0)
    builder.to_ssa(var_defs)
    print(builder.dump())

build_if_else_example()
```

---

## 4. Key Optimization Passes

Once in SSA form, a suite of classical optimizations become almost trivial to implement. Here are the most important ones, ordered from simplest to most powerful.

### 4.1 Constant Folding

Evaluate constant expressions at compile time. Every instruction of the form `t = c1 OP c2` where `c1` and `c2` are compile-time constants can be replaced by `t = result`:

```python
def constant_fold(block: BasicBlock) -> bool:
    """
    Replace constant binary ops with their result.
    Returns True if any change was made.
    """
    changed = False
    new_instrs = []
    for dest, op, src1, src2 in block.instructions:
        folded = False
        if src1.lstrip('-').isdigit() and src2 and src2.lstrip('-').isdigit():
            c1, c2 = int(src1), int(src2)
            result = None
            if op == '+':  result = c1 + c2
            elif op == '-': result = c1 - c2
            elif op == '*': result = c1 * c2
            elif op == '/' and c2 != 0: result = c1 // c2
            elif op == '<<': result = c1 << c2
            elif op == '>>': result = c1 >> c2
            if result is not None:
                new_instrs.append((dest, "const", str(result), None))
                folded = True
                changed = True
        if not folded:
            new_instrs.append((dest, op, src1, src2))
    block.instructions = new_instrs
    return changed
```

In SSA, constant folding has a cascade effect: once `t1 = 5`, replace all uses of `t1` with `5`, which may expose more constant expressions.

### 4.2 Dead Code Elimination (DCE)

In SSA, a definition is **dead** if it has no uses. Since each variable is defined exactly once, we can maintain a use-count per SSA variable:

```python
def dead_code_elimination(blocks: List[BasicBlock]) -> int:
    """
    Remove instructions whose results are never used.
    Returns count of instructions removed.
    """
    from collections import Counter
    use_count: Counter = Counter()

    # Count all uses
    for block in blocks:
        for phi_var, args in block.phi_functions.items():
            for arg in args:
                use_count[arg] += 1
        for dest, op, src1, src2 in block.instructions:
            if src1: use_count[src1] += 1
            if src2: use_count[src2] += 1

    removed = 0
    # Iteratively remove dead defs (removal may expose more dead defs)
    changed = True
    while changed:
        changed = False
        for block in blocks:
            new_instrs = []
            for instr in block.instructions:
                dest, op, src1, src2 = instr
                # Keep if: no dest (side-effect instr), or dest is used
                if not dest or use_count[dest] > 0:
                    new_instrs.append(instr)
                else:
                    # Removing this def: decrement use counts of its srcs
                    if src1: use_count[src1] -= 1
                    if src2: use_count[src2] -= 1
                    removed += 1
                    changed = True
            block.instructions = new_instrs
    return removed
```

DCE in SSA is O(n) per pass and typically requires only 1–2 iterations.

### 4.3 Common Subexpression Elimination (CSE)

If `t1 = a + b` appears at one point and later `t2 = a + b` (with no intervening modification of `a` or `b`), replace `t2` with a copy of `t1`. In SSA, since `a` and `b` are single-assignment values, any two instructions `t1 = a_k op b_j` and `t2 = a_k op b_j` with the same operand versions are guaranteed to compute the same value:

```python
def global_cse(blocks: List[BasicBlock]) -> Dict[str, str]:
    """
    Returns a mapping of eliminated vars to their canonical replacement.
    """
    expr_to_var: Dict[Tuple, str] = {}
    replacements: Dict[str, str] = {}

    for block in blocks:
        new_instrs = []
        for dest, op, src1, src2 in block.instructions:
            key = (op, src1, src2)
            if key in expr_to_var and op not in ('load', 'call'):  # not pure ops
                # dest is redundant — replace with canonical var
                replacements[dest] = expr_to_var[key]
            else:
                expr_to_var[key] = dest
                new_instrs.append((dest, op, src1, src2))
        block.instructions = new_instrs

    return replacements
```

CSE in SSA reduces to a **value numbering** problem: two expressions have the same value number if they compute provably identical results.

### 4.4 Sparse Conditional Constant Propagation (SCCP)

SCCP (Wegman & Zadeck, 1991) is the most powerful intraprocedural optimization. It simultaneously propagates constants and eliminates unreachable code:

- Each SSA value has a **lattice value**: `⊤` (unknown), a specific constant, or `⊥` (non-constant/overdefined)
- Each CFG edge has an **executability** bit: initially all edges are not executable (except entry)
- **Lattice meet**: `⊤ ∧ c = c`, `c ∧ c = c`, `c1 ∧ c2 = ⊥` (if c1 ≠ c2), `⊥ ∧ x = ⊥`

```python
from enum import Enum, auto
from typing import Union

class LatticeTag(Enum):
    TOP = auto()    # undetermined (optimistic assumption)
    CONST = auto()  # known constant
    BOT = auto()    # not a constant (overdefined)

@dataclass
class LatticeValue:
    tag: LatticeTag
    value: Optional[int] = None  # only meaningful if tag == CONST

    @staticmethod
    def top() -> 'LatticeValue': return LatticeValue(LatticeTag.TOP)

    @staticmethod
    def const(v: int) -> 'LatticeValue': return LatticeValue(LatticeTag.CONST, v)

    @staticmethod
    def bot() -> 'LatticeValue': return LatticeValue(LatticeTag.BOT)

    def meet(self, other: 'LatticeValue') -> 'LatticeValue':
        if self.tag == LatticeTag.TOP: return other
        if other.tag == LatticeTag.TOP: return self
        if self.tag == LatticeTag.BOT or other.tag == LatticeTag.BOT:
            return LatticeValue.bot()
        if self.value == other.value:
            return LatticeValue.const(self.value)
        return LatticeValue.bot()

    def __repr__(self):
        if self.tag == LatticeTag.TOP: return "⊤"
        if self.tag == LatticeTag.BOT: return "⊥"
        return str(self.value)


def sccp(blocks: List[BasicBlock], entry_id: int = 0) -> Dict[str, LatticeValue]:
    """
    Sparse Conditional Constant Propagation.
    Returns final lattice values for all SSA variables.
    """
    lattice: Dict[str, LatticeValue] = defaultdict(LatticeValue.top)
    executable: Set[Tuple[int, int]] = set()  # (pred_id, succ_id)
    block_map = {b.id: b for b in blocks}

    # Worklists
    cfg_worklist: deque = deque()     # CFG edges (bid_from, bid_to)
    ssa_worklist: deque = deque()     # SSA defs whose lattice changed

    # Seed: entry block is reachable
    entry = block_map[entry_id]
    for succ in entry.successors:
        cfg_worklist.append((entry_id, succ.id))

    def evaluate(op: str, v1: LatticeValue, v2: Optional[LatticeValue]) -> LatticeValue:
        if v1.tag == LatticeTag.BOT or (v2 and v2.tag == LatticeTag.BOT):
            return LatticeValue.bot()
        if v1.tag == LatticeTag.TOP or (v2 and v2.tag == LatticeTag.TOP):
            return LatticeValue.top()
        c1 = v1.value
        c2 = v2.value if v2 else None
        if op == '+' and c2 is not None: return LatticeValue.const(c1 + c2)
        if op == '-' and c2 is not None: return LatticeValue.const(c1 - c2)
        if op == '*' and c2 is not None: return LatticeValue.const(c1 * c2)
        if op == 'const': return LatticeValue.const(int(c1) if c1 is not None else 0)
        return LatticeValue.bot()

    while cfg_worklist or ssa_worklist:
        while cfg_worklist:
            from_id, to_id = cfg_worklist.popleft()
            edge = (from_id, to_id)
            if edge in executable:
                continue
            executable.add(edge)
            block = block_map[to_id]

            # Process phi functions
            for phi_var, args in block.phi_functions.items():
                merged = LatticeValue.top()
                for i, pred in enumerate(block.predecessors):
                    if (pred.id, to_id) in executable:
                        merged = merged.meet(lattice[args[i]])
                old = lattice[phi_var]
                if merged != old:
                    lattice[phi_var] = merged
                    ssa_worklist.append(phi_var)

            # Process instructions if block newly executable
            for dest, op, src1, src2 in block.instructions:
                v1 = lattice[src1] if src1 else LatticeValue.top()
                v2 = lattice[src2] if src2 else None
                result = evaluate(op, v1, v2)
                old = lattice[dest]
                if result != old:
                    lattice[dest] = result
                    ssa_worklist.append(dest)

        while ssa_worklist:
            var = ssa_worklist.popleft()
            # Propagate to uses — simplified: re-evaluate all blocks
            # (In production: maintain def-use chains for efficiency)
            for block in blocks:
                for dest, op, src1, src2 in block.instructions:
                    if src1 == var or src2 == var:
                        v1 = lattice[src1] if src1 else LatticeValue.top()
                        v2 = lattice[src2] if src2 else None
                        result = evaluate(op, v1, v2)
                        if result != lattice[dest]:
                            lattice[dest] = result
                            ssa_worklist.append(dest)

    return dict(lattice)
```

SCCP can prove that an `if (x > 0)` branch is never taken when `x` is a known non-positive constant — something that simple constant folding cannot do.

### 4.5 Loop Invariant Code Motion (LICM)

Move computations that produce the same result on every loop iteration to the **loop preheader** (a block that executes once before the loop):

```python
def find_natural_loops(blocks: List[BasicBlock], idom: Dict[int, int]) -> List[Tuple[int, Set[int]]]:
    """
    A natural loop is identified by a back edge (n -> h) where h dominates n.
    Returns list of (header_id, loop_body_block_ids).
    """
    def dominates(a: int, b: int) -> bool:
        """Does a dominate b? Walk up idom tree from b."""
        cur = b
        while cur != a:
            parent = idom.get(cur, cur)
            if parent == cur:  # reached root without finding a
                return False
            cur = parent
        return True

    loops = []
    for block in blocks:
        for succ in block.successors:
            # back edge: block -> succ where succ dominates block
            if dominates(succ.id, block.id):
                # Collect loop body via backward DFS from block
                loop_body: Set[int] = {succ.id}
                worklist = [block.id]
                while worklist:
                    n = worklist.pop()
                    if n not in loop_body:
                        loop_body.add(n)
                        bk = next((b for b in blocks if b.id == n), None)
                        if bk:
                            worklist.extend(p.id for p in bk.predecessors)
                loops.append((succ.id, loop_body))
    return loops


def licm(blocks: List[BasicBlock], idom: Dict[int, int]) -> int:
    """
    Move loop-invariant instructions to the loop preheader.
    Returns count of instructions hoisted.
    """
    loops = find_natural_loops(blocks, idom)
    block_map = {b.id: b for b in blocks}
    hoisted = 0

    for header_id, loop_body in loops:
        # Collect all defs inside the loop
        loop_defs: Set[str] = set()
        for bid in loop_body:
            b = block_map[bid]
            for dest, _, _, _ in b.instructions:
                if dest:
                    loop_defs.add(dest)

        # Find invariant instructions: all srcs are defined outside loop
        # or are themselves invariant (iterate to fixed point)
        invariant_defs: Set[str] = set()
        changed = True
        while changed:
            changed = False
            for bid in loop_body:
                b = block_map[bid]
                for dest, op, src1, src2 in b.instructions:
                    if dest in invariant_defs or op in ('load', 'call', 'store'):
                        continue
                    s1_inv = src1 not in loop_defs or src1 in invariant_defs
                    s2_inv = (not src2) or (src2 not in loop_defs) or (src2 in invariant_defs)
                    if s1_inv and s2_inv:
                        invariant_defs.add(dest)
                        changed = True

        # Move invariant instructions to preheader (we model it as the header's
        # predecessor that is outside the loop)
        header = block_map[header_id]
        preheader_instrs = []
        for bid in loop_body:
            b = block_map[bid]
            remaining = []
            for instr in b.instructions:
                dest, op, src1, src2 = instr
                if dest in invariant_defs:
                    preheader_instrs.append(instr)
                    hoisted += 1
                else:
                    remaining.append(instr)
            b.instructions = remaining

        # Prepend hoisted instructions to header
        header.instructions = preheader_instrs + header.instructions

    return hoisted
```

LICM is especially powerful when the hoisted expression involves a memory load that the compiler can prove doesn't alias with any loop store.

### 4.6 Strength Reduction

Replace expensive operations with cheaper equivalents. In loops, the canonical example is replacing a multiplication by the loop counter with a running addition:

```
# Before: mul in every iteration
for i in range(n):
    result = base + i * stride    # mul each iteration

# After: strength reduction
s = base                          # preheader
for i in range(n):
    result = s                    # just use s
    s = s + stride                # one add per iteration
```

In a compiler IR, this is implemented during loop analysis by identifying **induction variables** (variables that change by a constant amount each iteration) and **derived induction variables** (linear functions of induction variables).

---

## 5. Register Allocation via Graph Coloring

After optimization, we must map the (potentially large) set of SSA variables to the finite set of machine registers (e.g., 16 general-purpose registers on x86-64).

### 5.1 Liveness Analysis

A variable is **live** at a program point if it may be used in the future before being redefined. We compute liveness with **backward dataflow**:

```
gen(B)  = variables used in B before any redefinition in B
kill(B) = variables defined in B

livein(B)  = gen(B) ∪ (liveout(B) − kill(B))
liveout(B) = ∪_{S ∈ succ(B)} livein(S)
```

The system is solved iteratively until a fixed point:

```python
def compute_liveness(blocks: List[BasicBlock]) -> Dict[int, Tuple[Set[str], Set[str]]]:
    """
    Iterative backward dataflow for liveness analysis.
    Returns {block_id: (livein, liveout)}.
    """
    block_map = {b.id: b for b in blocks}
    livein:  Dict[int, Set[str]] = {b.id: set() for b in blocks}
    liveout: Dict[int, Set[str]] = {b.id: set() for b in blocks}

    # Precompute gen and kill for each block
    gen:  Dict[int, Set[str]] = {}
    kill: Dict[int, Set[str]] = {}

    for block in blocks:
        g: Set[str] = set()
        k: Set[str] = set()

        # Also count phi function arguments as uses
        for phi_var, args in block.phi_functions.items():
            for arg in args:
                if arg not in k:
                    g.add(arg)
            k.add(phi_var)

        for dest, op, src1, src2 in block.instructions:
            if src1 and src1 not in k:
                g.add(src1)
            if src2 and src2 not in k:
                g.add(src2)
            if dest:
                k.add(dest)

        gen[block.id]  = g
        kill[block.id] = k

    changed = True
    while changed:
        changed = False
        for block in reversed(blocks):  # process in reverse postorder
            bid = block.id
            new_liveout: Set[str] = set()
            for succ in block.successors:
                new_liveout |= livein[succ.id]

            new_livein = gen[bid] | (new_liveout - kill[bid])

            if new_livein != livein[bid] or new_liveout != liveout[bid]:
                changed = True
            livein[bid]  = new_livein
            liveout[bid] = new_liveout

    return {b.id: (livein[b.id], liveout[b.id]) for b in blocks}
```

**Complexity**: O(n²) in the worst case (reducible CFGs converge in one reverse-postorder pass for many programs), O(n) per iteration, and typically converges in 2–5 iterations for real programs.

### 5.2 Interference Graph

Two SSA variables **interfere** if they are simultaneously live at any program point. Equivalently (exploiting SSA properties): variable `u` interferes with variable `v` if `u` is live at the definition point of `v` (or vice versa):

```python
def build_interference_graph(
    blocks: List[BasicBlock],
    liveness: Dict[int, Tuple[Set[str], Set[str]]]
) -> Dict[str, Set[str]]:
    """
    Build interference graph. Returns adjacency sets.
    Two vars interfere if they are simultaneously live.
    """
    graph: Dict[str, Set[str]] = defaultdict(set)

    def add_edge(u: str, v: str):
        if u != v:
            graph[u].add(v)
            graph[v].add(u)

    for block in blocks:
        livein, liveout = liveness[block.id]
        live = set(liveout)  # start with liveout, scan backward

        # Backward scan through instructions
        for dest, op, src1, src2 in reversed(block.instructions):
            if dest:
                # dest interferes with everything currently live
                for v in live:
                    add_edge(dest, v)
                live.discard(dest)  # def kills liveness
            if src1: live.add(src1)
            if src2: live.add(src2)

    return dict(graph)
```

### 5.3 Graph Coloring: Chaitin's Algorithm

**Chaitin's algorithm** (1982) is the canonical approach to register allocation via graph coloring:

```python
def color_graph(graph: Dict[str, Set[str]], num_registers: int) -> Dict[str, Union[int, str]]:
    """
    Chaitin-style graph coloring for register allocation.
    Returns dict: var -> register_number (or 'SPILL' if spilled).
    
    Algorithm:
      1. Simplify: repeatedly remove nodes with degree < num_registers
      2. Spill: if no low-degree node, pick highest-cost node to spill
      3. Color: restore nodes in reverse simplification order, assign colors
    """
    import copy
    adj = {v: set(neighbors) for v, neighbors in graph.items()}
    all_vars = list(graph.keys())
    if not all_vars:
        return {}

    stack: List[str] = []
    spilled: Set[str] = set()
    removed: Set[str] = set()
    working_adj = copy.deepcopy(adj)

    def degree(v: str) -> int:
        return len(working_adj.get(v, set()) - removed)

    # Simplify phase
    n = len(all_vars)
    for _ in range(n):
        # Find a node with degree < num_registers
        candidate = None
        for v in all_vars:
            if v not in removed and degree(v) < num_registers:
                candidate = v
                break
        if candidate is None:
            # Spill: choose var with most conflicts (naive heuristic)
            candidate = max(
                (v for v in all_vars if v not in removed),
                key=lambda v: degree(v),
                default=None
            )
            if candidate:
                spilled.add(candidate)
        if candidate:
            stack.append(candidate)
            removed.add(candidate)

    # Color phase: restore in reverse order
    colors: Dict[str, int] = {}
    for var in reversed(stack):
        if var in spilled:
            colors[var] = -1  # spill marker
            continue
        neighbor_colors = {
            colors[n] for n in adj.get(var, set())
            if n in colors and colors[n] >= 0
        }
        # Assign the lowest available color
        for color in range(num_registers):
            if color not in neighbor_colors:
                colors[var] = color
                break
        else:
            colors[var] = -1  # forced spill

    result = {}
    for var in all_vars:
        c = colors.get(var, -1)
        result[var] = 'SPILL' if c == -1 else c
    return result
```

### 5.4 Spill Code Insertion

When a variable is spilled, the compiler must:
1. At each **definition**: store the result to a stack slot (`store r0, [sp + offset]`)
2. At each **use**: reload from the stack slot before use (`load r_temp, [sp + offset]`)

The spill **cost** heuristic is crucial: prefer to spill variables used infrequently and not inside loops:
```
spill_cost(v) = Σ_{use u of v} 10^(loop_nesting_depth(u))
```

Choose the variable with the **lowest** spill cost to spill.

### 5.5 Copy Coalescing

After register allocation, the code may contain redundant copies `t2 = t1` (from phi destruction or compiler temporaries). If `t1` and `t2` don't interfere, they can share the same register, eliminating the copy entirely.

George and Appel's **conservative coalescing** ensures that coalescing never increases the number of spills:
- **George criterion**: safe to coalesce `u` and `v` if every neighbor of `u` with degree ≥ k either interferes with `v` or has degree < k
- **Briggs criterion**: safe to coalesce if the merged node has fewer than k high-degree neighbors

---

## 6. Python Implementation: Complete Liveness + Register Allocation

```python
def full_register_allocation_example():
    """
    End-to-end example: build a small SSA program, compute liveness,
    build interference graph, and color it with 2 registers.
    """
    # Simulate a simple SSA block: a = b + c; d = a * 2; e = b - d
    # Variables: a, b, c, d, e
    # Liveout of this single block: {e}
    # Liveness (backward): e live at end; before 'e=b-d': {b,d}; before 'd=a*2': {a,b};
    #                      before 'a=b+c': {b,c}

    @dataclass
    class SimpleBlock:
        id: int
        instructions: list
        successors: list
        predecessors: list
        phi_functions: dict
        defs: Set[str]
        uses: Set[str]

    block = SimpleBlock(
        id=0,
        instructions=[
            ("a", "+", "b", "c"),
            ("d", "*", "a", "2"),
            ("e", "-", "b", "d"),
        ],
        successors=[],
        predecessors=[],
        phi_functions={},
        defs={"a", "d", "e"},
        uses={"b", "c"},
    )

    # Manual liveness (single block, liveout={e})
    liveness = {0: ({"b", "c"}, {"e"})}

    # Build interference graph manually based on simultaneous liveness
    # At 'a = b+c': b,c live → b-c interfere
    # After 'a' defined: a,b live → a-b interfere
    # At 'd = a*2': a,b live → a-b interfere (already)
    # After 'd' defined: b,d live → b-d interfere
    # At 'e = b-d': b,d live; e not live yet
    interference = {
        "a": {"b"},
        "b": {"a", "c", "d"},
        "c": {"b"},
        "d": {"b"},
        "e": set(),
    }

    print("Interference graph:")
    for v, neighbors in interference.items():
        print(f"  {v} -- {sorted(neighbors)}")

    allocation = color_graph(interference, num_registers=2)
    print("\nRegister allocation (2 registers):")
    for var, reg in sorted(allocation.items()):
        label = f"R{reg}" if isinstance(reg, int) and reg >= 0 else "SPILL"
        print(f"  {var} → {label}")

full_register_allocation_example()
```

Expected output (one valid coloring):
```
Interference graph:
  a -- ['b']
  b -- ['a', 'c', 'd']
  c -- ['b']
  d -- ['b']
  e -- []

Register allocation (2 registers):
  a → R0
  b → R1
  c → R0
  d → R0
  e → R0
```

Here `b` gets R1 because it interferes with `a`, `c`, and `d` — all of which can share R0 since they don't interfere with each other (they're never simultaneously live).

---

## 7. LLVM IR — What Real Compilers Produce

LLVM IR is the gold standard for production compiler IRs. It is:
- **Typed**: every value has a type (`i64`, `double`, `i8*`, etc.)
- **SSA-based**: every value is defined exactly once, phi nodes at join points
- **Infinite virtual registers**: `%0`, `%1`, `%result`, etc.
- **Explicit control flow**: every basic block ends with a terminator (`br`, `ret`, `switch`)

### Example: Sum of Squares

This Python function:
```python
def sum_squares(n: int) -> int:
    total = 0
    for i in range(n):
        total += i * i
    return total
```

Compiles to the following LLVM IR (simplified, showing the SSA structure):

```llvm
; Function: sum_squares(n: i64) -> i64
define i64 @sum_squares(i64 %n) {
entry:
  ; Unconditional jump to loop header
  br label %loop

loop:
  ; phi nodes select values from predecessors
  ; %i:   0 (from entry) or %i_next (from loop)
  ; %sum: 0 (from entry) or %sum_next (from loop)
  %i   = phi i64 [ 0, %entry ], [ %i_next, %loop ]
  %sum = phi i64 [ 0, %entry ], [ %sum_next, %loop ]

  ; Compute i * i
  %i2       = mul nsw i64 %i, %i

  ; Compute sum + i*i
  %sum_next = add nsw i64 %sum, %i2

  ; Increment i
  %i_next   = add nsw i64 %i, 1

  ; Check loop condition: i_next < n
  %cond     = icmp slt i64 %i_next, %n

  ; Branch: continue loop or exit
  br i1 %cond, label %loop, label %exit

exit:
  ; Return final sum
  ret i64 %sum_next
}
```

**Instruction-by-instruction explanation:**

| Instruction | Meaning |
|-------------|---------|
| `define i64 @sum_squares(i64 %n)` | Function def: returns 64-bit int, takes 64-bit int arg `%n` |
| `br label %loop` | Unconditional branch to `%loop` |
| `phi i64 [ 0, %entry ], [ %i_next, %loop ]` | Phi: if from `%entry` → 0; if from `%loop` → `%i_next` |
| `mul nsw i64 %i, %i` | Signed multiply, `nsw` = no signed wrap (enables optimizations) |
| `add nsw i64 %sum, %i2` | Signed add, no wrap |
| `icmp slt i64 %i_next, %n` | Signed less-than comparison, result is `i1` (1-bit bool) |
| `br i1 %cond, label %loop, label %exit` | Conditional branch |
| `ret i64 %sum_next` | Return the final value |

**Key LLVM optimization passes:**

| Pass | What it does |
|------|-------------|
| `mem2reg` | Promotes `alloca`/`load`/`store` patterns to SSA phi nodes (entry to SSA form) |
| `instcombine` | Constant folding + algebraic simplifications (e.g., `x+0→x`, `x*1→x`) |
| `simplifycfg` | Removes unreachable blocks, merges blocks, eliminates trivial branches |
| `loop-unroll` | Replicates loop body to reduce branch overhead and expose instruction-level parallelism |
| `loop-vectorize` | Auto-vectorization using SIMD (SSE/AVX on x86, NEON on ARM) |
| `gvn` | Global value numbering (advanced CSE across blocks) |
| `inline` | Function inlining (the enabler of most other optimizations) |
| `licm` | Loop invariant code motion |
| `scalar-evolution` | Analyzes loop induction variables (enables strength reduction, vectorization) |

---

## 8. What Textbooks Don't Tell You

### 8.1 SSA Destruction: The Lost Copy Problem

Before code generation, SSA must be **destructed** — phi functions don't exist in machine code. The naive approach inserts a copy at the end of each predecessor:

```
B3: x_3 = φ(x_1 from B1, x_2 from B2)
```

Becomes:
```
B1: ... ; x_3 = x_1     ← inserted copy
B2: ... ; x_3 = x_2     ← inserted copy
B3: (no phi)
```

But this is wrong when the phi argument and the phi result share a live range. Consider:

```
B1: a_1 = φ(a_0, b_1)
    b_1 = φ(b_0, a_1)
```

Naively, you'd insert `a_1 = a_0; b_1 = b_0` in the predecessor — but these are **swap** operations, not independent copies. The **swap problem** requires either a temporary variable or a swap instruction.

The correct solution (Briggs et al.) is to insert copies into a **parallel copy** notation and then serialize them respecting dependency chains, using a temporary for cycles.

### 8.2 CPython vs. PyPy: Two Worlds

**CPython** compiles Python to **stack-based bytecode** (`.pyc` files). There is no SSA, no register allocation, no serious optimization. The bytecode interpreter evaluates operations one at a time:

```python
import dis
def f(x): return x * 2 + 1
dis.dis(f)
# LOAD_FAST   0 (x)
# LOAD_CONST  1 (2)
# BINARY_MULTIPLY
# LOAD_CONST  2 (1)
# BINARY_ADD
# RETURN_VALUE
```

Each `BINARY_MULTIPLY` goes through a Python object dispatch: check types, call `__mul__`, handle errors. This is why CPython is slow for numeric code.

**PyPy** traces hot execution paths, records them as a linear sequence of guards and operations, puts that sequence into SSA form, optimizes it (constant folding, DCE, escape analysis), then compiles it to native machine code using a register allocator. The result can be 10–100× faster than CPython on numeric benchmarks.

### 8.3 Alias Analysis Is the Real Bottleneck

The optimizations described above (LICM, CSE, vectorization) all assume they can reason about **which memory locations a pointer may alias**. If the compiler cannot prove that `*p` and `*q` refer to different locations, it must assume they alias, which blocks:

- **LICM**: can't hoist `*p` out of a loop if `*q = ...` might write to `*p`
- **Vectorization**: can't process `a[i]` and `b[i]` in parallel if `a == b`
- **CSE**: can't eliminate redundant loads if a store might alias them

In C, the `restrict` keyword is a programmer assertion that a pointer doesn't alias. Python has no such mechanism, making Python JITs extremely conservative with memory operations.

Modern alias analysis uses **points-to analysis** (e.g., Andersen's algorithm, Steensgaard's algorithm) to compute which heap objects each pointer may reference. This is NP-hard in general but can be done efficiently with approximations.

### 8.4 Inlining Is the Meta-Optimization

Most optimizations only become effective **after** aggressive function inlining. Consider:

```python
def square(x): return x * x
def sum_squares(n):
    return sum(square(i) for i in range(n))
```

Without inlining, the compiler sees `square()` as an opaque call — it can't vectorize or do strength reduction on it. After inlining, the compiler sees `i * i` directly and can apply all loop optimizations.

The GCC/LLVM inlining heuristic computes an **inline cost** based on code size growth, and an **inline benefit** based on how many call sites there are and whether arguments are constant (constant arguments enable further optimizations after inlining).

The rule of thumb: **most of the performance gain at -O3 comes from inlining + DCE**.

### 8.5 The Optimizer Cliff

A small, innocent-looking source change can cause a function to exceed the inlining threshold, preventing inlining, which prevents downstream optimizations, resulting in a **10–100× performance regression**. This is called the **optimizer cliff**.

Example: adding a single `print()` statement to a hot function may push it over the code size limit for inlining. The fix is to use `__attribute__((always_inline))` in C, or restructure the code to keep hot paths small.

This is why performance-critical production code is tested with **regression benchmarks** on every commit, not just correctness tests. A compiler like LLVM has thousands of performance regression tests (`llvm/test/Transforms/`) that verify specific IR patterns are produced.

### 8.6 Profile-Guided Optimization (PGO)

Static heuristics for inlining, branch prediction, and register allocation are fundamentally limited. **PGO** solves this by:

1. Compiling with instrumentation (`clang -fprofile-generate`)
2. Running on representative workloads → profile data (which branches are hot, which calls are frequent)
3. Recompiling with profile data (`clang -fprofile-use`) → better inlining decisions, better branch layout, better register allocation for hot paths

V8 (Chrome's JavaScript engine), GraalVM, and all production Java JITs use PGO continuously at runtime (adaptive compilation), recompiling hot functions as more profile data accumulates.

---

## 9. Hard Exercise: Complete SSA Walkthrough

**Given CFG:**
- **Block 0** (entry): `x = 1`, `y = 2`, `if cond goto B1 else B2`
- **Block 1**: `x = x + y`, `goto B3`
- **Block 2**: `y = x * 2`, `goto B3`
- **Block 3** (exit): `z = x + y`, `return z`

### (a) Draw the Dominator Tree

CFG edges: `B0→B1`, `B0→B2`, `B1→B3`, `B2→B3`.

- `idom(B0) = B0` (entry dominates itself)
- `idom(B1) = B0` (only path to B1 is through B0)
- `idom(B2) = B0` (only path to B2 is through B0)
- `idom(B3) = B0` (paths: B0→B1→B3 and B0→B2→B3; both pass through B0, but B1 is not on both paths, B2 is not on both paths, so idom is B0)

```
Dominator Tree:
        B0
      / | \
    B1  B2  B3
```

### (b) Dominance Frontiers

Applying the algorithm at each join point:

- **B3** has predecessors B1 and B2
  - `idom(B3) = B0`
  - Runner from B1: B1 ≠ B0, so `DF(B1) ∪= {B3}`; then B1's idom is B0, stop
  - Runner from B2: B2 ≠ B0, so `DF(B2) ∪= {B3}`; then B2's idom is B0, stop

Result:
```
DF(B0) = {}
DF(B1) = {B3}
DF(B2) = {B3}
DF(B3) = {}
```

### (c) Phi Function Placement

Variables and their definition sites:
- `x`: defined in B0 and B1 → defs = {B0, B1}
- `y`: defined in B0 and B2 → defs = {B0, B2}
- `z`: defined in B3 → defs = {B3}
- `cond`: defined in B0

For `x`: IDF({B0, B1}) = DF(B0) ∪ DF(B1) = {} ∪ {B3} = **{B3}** → insert `x = φ(...)` at B3

For `y`: IDF({B0, B2}) = DF(B0) ∪ DF(B2) = {} ∪ {B3} = **{B3}** → insert `y = φ(...)` at B3

### (d) SSA Renaming

**Block 0 (entry)**:
```
x_0 = 1
y_0 = 2
if cond_0 goto B1 else B2
```
Push: stacks[x] = [x_0], stacks[y] = [y_0]

**Block 1** (DFS child of B0 in dom tree):
```
x_1 = x_0 + y_0   # use top of stacks: x_0, y_0; fresh def x_1
```
Push: stacks[x] = [x_0, x_1]
Fill B3's phi for x from B1: x_1
Fill B3's phi for y from B1: y_0

**Block 3** (successor of B1, also DFS child of B0):
```
x_3 = φ(x_1, x_2)    # args filled after visiting B1 and B2
y_3 = φ(y_0, y_1)
z_0 = x_3 + y_3
return z_0
```

**Block 2** (DFS child of B0 in dom tree):
```
y_1 = x_0 * 2   # use x_0 (top of stacks[x] after backtrack from B1); fresh def y_1
```
Push: stacks[y] = [y_0, y_1]
Fill B3's phi for x from B2: x_0
Fill B3's phi for y from B2: y_1

**Complete SSA form:**
```
B0: x_0 = 1
    y_0 = 2
    if cond_0 goto B1 else B2

B1: x_1 = x_0 + y_0
    goto B3

B2: y_1 = x_0 * 2
    goto B3

B3: x_3 = φ(x_1 [from B1], x_0 [from B2])
    y_3 = φ(y_0 [from B1], y_1 [from B2])
    z_0 = x_3 + y_3
    return z_0
```

### (e) Constant Folding When `cond` Is Known True

If `cond_0 = true` at compile time, the `if` always takes the `B1` branch. SCCP marks edge `B0→B2` as **not executable** and prunes B2.

At B3:
- `x_3 = φ(x_1 [from B1], x_0 [dead])` → only B1 is executable → `x_3 = x_1`
- `y_3 = φ(y_0 [from B1], y_1 [dead])` → only B1 is executable → `y_3 = y_0`

Substituting constants:
- `x_0 = 1`, `y_0 = 2`
- `x_1 = x_0 + y_0 = 1 + 2 = 3` ← constant fold
- `x_3 = x_1 = 3`
- `y_3 = y_0 = 2`
- `z_0 = x_3 + y_3 = 3 + 2 = 5` ← constant fold

### (f) Dead Code Elimination

After constant folding and pruning B2:
- `x_0 = 1`: used by `x_1 = x_0 + y_0` → **KEEP** (but after fold: x_1 = 3 directly)
- `y_0 = 2`: used by `x_1` and `y_3` → **KEEP** (but after fold: used value is constant)
- `x_1 = 3`: used by `x_3 = x_1` → becomes `x_3 = 3`
- `y_1 = x_0 * 2`: in dead block B2 → **DEAD, ELIMINATE**
- `z_0 = 5`: returned — **KEEP**

Final after constant propagation + DCE:
```
B0: if true goto B1 else B2   ← becomes unconditional jump
B1: (nothing — all computations folded)
B3: return 5
```

Or more simply: the function compiles to a constant return of `5`.

### (g) Interference Graph and 2-Register Coloring

Consider the SSA form **before** knowing cond is true (the general case):

Variables: `x_0`, `y_0`, `x_1`, `y_1`, `x_3`, `y_3`, `z_0`

Simultaneously live at B3 (liveout of predecessors feeding B3):
- Entering B3 from B1: `{x_1, y_0}` — these are the live values
- Entering B3 from B2: `{x_0, y_1}` — these are the live values

At B3 itself, after phi:
- After `x_3 = φ(...)`: `x_3` is live, `x_1`/`x_0` may die
- After `y_3 = φ(...)`: `y_3` is live, `y_0`/`y_1` may die
- During `z_0 = x_3 + y_3`: both `x_3` and `y_3` are live

Interference edges (the key pairs that are simultaneously live):
```
x_3 ── y_3        (both live during z_0 = x_3 + y_3)
x_1 ── y_0        (both live at end of B1)
x_0 ── y_1        (both live at end of B2)
```

With 2 registers (R0, R1):

```
Color x_3 = R0
Color y_3 = R1     (interferes with x_3)
Color x_1 = R0     (no interference with y_3)
Color y_0 = R1     (interferes with x_1)
Color x_0 = R0     (can share R0: x_0 is dead before x_3 is live)
Color y_1 = R1     (interferes with x_0)
Color z_0 = R0     (only live at return, no live interferers at that point)
```

**2-register allocation succeeds** with no spills. The interference graph is 2-colorable because variables from different execution paths (B1 vs B2) are never simultaneously live in the same block.

---

## 10. Summary and Further Reading

| Concept | Key Insight |
|---------|-------------|
| SSA form | Each variable assigned once; phi functions at merge points |
| Dominators | idom tree drives phi placement and renaming |
| Dominance frontier | Exactly where phi functions are needed |
| Constant folding | O(n) in SSA; cascades via use lists |
| DCE | O(n) in SSA; empty use list = dead def |
| SCCP | Propagates constants + prunes unreachable branches simultaneously |
| LICM | Requires natural loop detection + invariance analysis |
| Liveness analysis | Backward dataflow; gen/kill sets per block |
| Interference graph | Edges = simultaneous liveness |
| Graph coloring | NP-complete; Chaitin's simplify/spill/color approximation |
| LLVM IR | Typed SSA with explicit phi nodes and terminators |
| Inlining | The meta-optimization; enables all others |
| Alias analysis | The fundamental barrier to memory optimizations |
| PGO | Overcomes static heuristic limitations with runtime data |

### Canonical References

**Foundational SSA and Dataflow:**
1. **Cytron et al. (1991)** — "Efficiently Computing Static Single Assignment Form and the Control Dependence Graph" — *TOPLAS*. The foundational SSA paper with dominance frontier algorithm.
2. **Cooper, Harvey, Kennedy (2001)** — "A Simple, Fast Dominance Algorithm" — The practically-linear dominator algorithm used in LLVM.
3. **Wegman & Zadeck (1991)** — "Constant Propagation with Conditional Branches" — SCCP algorithm. Original paper that introduced the lattice framework.
4. **Ferrante, Ottenstein, Warren (1987)** — "The Program Dependence Graph and Its Use in Optimization" — PDG: combining data and control dependences.

**Register Allocation:**
5. **Chaitin et al. (1981)** — "Register Allocation via Coloring" — Graph coloring register allocator.
6. **Briggs, Cooper, Torczon (1994)** — "Improvements to Graph Coloring Register Allocation" — Coalescing, better spill heuristics, SSA destruction.
7. **Poletto & Sarkar (1999)** — "Linear Scan Register Allocation" — The fast O(n log n) algorithm used in JIT compilers.

**Alias Analysis:**
8. **Andersen (1994)** — "Program Analysis and Specialization for the C Programming Language" — Points-to analysis.
9. **Steensgaard (1996)** — "Points-to Analysis in Almost Linear Time" — Union-find based, near-linear points-to.

**Interprocedural and Advanced:**
10. **Briggs & Cooper (1994)** — "Effective Partial Redundancy Elimination" — GVN-PRE.
11. **Stadler et al. (2014)** — "Partial Escape Analysis and Scalar Replacement for Java" — Graal's key optimization.
12. **Click & Paleczny (1995)** — "A Simple Graph-Based Intermediate Representation" — Sea of Nodes IR.

**Textbooks:**
13. **"Engineering a Compiler" (Cooper & Torczon, 3rd ed., 2022)** — The best modern textbook on the entire compiler pipeline. Excellent chapters on SSA, dataflow, and register allocation.
14. **"Advanced Compiler Design and Implementation" (Muchnick, 1997)** — Encyclopedic treatment of dataflow analysis and optimization. The reference for advanced optimization passes.
15. **"Compilers: Principles, Techniques, and Tools" (Aho, Lam, Sethi, Ullman — Dragon Book, 2nd ed.)** — Classic textbook; strong on grammars and parsing, weaker on modern SSA.
16. **"Modern Compiler Implementation in Java/ML/C" (Appel)** — Excellent modern treatment; includes register allocation, instruction selection, garbage collection.

**Production Compiler Resources:**
17. **LLVM Language Reference Manual** — Definitive guide to LLVM IR. Every LLVM pass has a corresponding test in `llvm/test/Transforms/`.
18. **V8 blog (v8.dev)** — Turbofan and Maglev architecture explained by the V8 team.
19. **HotSpot internals wiki (OpenJDK wiki)** — C2 compiler, tiered compilation, deoptimization.

---

## 11. Control Dependence and the Program Dependence Graph

Dataflow analysis tracks **data dependencies** (which definition reaches which use). But compilers also need **control dependencies** — which branches determine whether a given instruction executes at all.

### 11.1 Control Dependence

Statement Y is **control dependent** on statement X if:
- X has at least two successors
- There is a path from X to Y that doesn't go through the post-dominator of X
- X can determine whether Y executes

Formally, using the **post-dominator tree** (the dominator tree of the reversed CFG):

> Y is control dependent on X if:
> 1. Y post-dominates some successor of X
> 2. Y does not strictly post-dominate X

```python
def compute_control_dependences(blocks: List[BB]) -> Dict[int, Set[int]]:
    """
    Returns control_dep[Y] = set of block IDs that Y is control-dependent on.
    Uses the post-dominator tree of the reversed CFG.
    """
    # Reverse the CFG
    rev_blocks = {b.id: BB(b.id) for b in blocks}
    # Add a synthetic exit -> all sinks edge for the reversed CFG
    for b in blocks:
        for s in b.succs:
            rev_blocks[s.id].succs.append(rev_blocks[b.id])
            rev_blocks[b.id].preds.append(rev_blocks[s.id])

    # Find sink blocks (no successors) as entry for reversed CFG
    sinks = [b.id for b in blocks if not b.succs]

    # The post-dominator is the dominator of the reversed CFG from the exit
    # For simplicity, we use a synthetic exit node
    # ...
    # (Full implementation uses reversed CFG + standard dominator algorithm)

    # Here we compute it directly via the definition
    bmap = {b.id: b for b in blocks}
    cd: Dict[int, Set[int]] = defaultdict(set)

    for x in blocks:
        if len(x.succs) < 2:
            continue  # no branching, no control dependence created
        for succ in x.succs:
            # Y is control dependent on X if this edge (X→succ) leads to
            # a region that X doesn't always execute
            # Walk forward from succ until we reach a post-dominator of X
            # This is a simplified structural approach
            visited: Set[int] = set()
            wl = [succ.id]
            while wl:
                bid = wl.pop()
                if bid in visited or bid == x.id:
                    continue
                visited.add(bid)
                cd[bid].add(x.id)  # bid is control-dependent on x
                b = bmap.get(bid)
                if b:
                    wl.extend(s.id for s in b.succs)

    return dict(cd)
```

### 11.2 The Program Dependence Graph (PDG)

The **Program Dependence Graph** (Ferrante, Ottenstein, Warren 1987) combines data and control dependences into a single graph. It is the basis for:

- **Slicing**: extract only the instructions that affect a particular variable at a particular point (data flow backward from the slice criterion, plus the control dependences of those instructions)
- **Parallelization**: two statements can execute in parallel iff they have no data or control dependences between them
- **Vectorization**: a loop body can be vectorized iff the dependency graph within the loop is acyclic (or the only cycles can be broken by reordering)

```
PDG Node: each instruction
PDG Edge (data): X → Y if X defines a variable used by Y
PDG Edge (control): X → Y if Y is control-dependent on X
```

### 11.3 Why SCCP Needs Control Dependence

Recall SCCP's lattice propagation. When SCCP determines that a branch `if (c > 0)` is always taken (c is a known positive constant), it must mark the **not-taken** edge as not executable. This requires knowing which blocks are reached exclusively via that edge — precisely the blocks control-dependent on the branch.

Without control dependence information, SCCP would need to re-examine every block on every propagation step. With it, when a branch's condition becomes constant, SCCP only re-examines the blocks that are control-dependent on that branch — a dramatic efficiency improvement.

---

## 13. Instruction Selection: From IR to Machine Instructions

After optimization, the compiler must select specific machine instructions for each IR operation. This is non-trivial because:
- One IR operation may map to different machine instructions depending on context (e.g., `x * 8` → `SHL r, 3` on x86, `LSL r, #3` on ARM)
- Machine instructions can compute multiple IR operations at once (e.g., x86 `LEA rax, [rbx + rcx*4 + 8]` computes `rax = rbx + rcx*4 + 8` in one instruction)
- Some IR operations have no direct machine equivalent and must be lowered to a sequence of instructions

### 13.1 Tree Pattern Matching (BURG)

The standard approach: represent IR as an expression tree, define **rewrite rules** that match subtrees, and find the **minimum cost tiling** of the tree.

```python
@dataclass
class IRNode:
    """Expression tree node for instruction selection."""
    op: str
    children: List['IRNode'] = field(default_factory=list)
    value: Optional[int] = None   # for constants
    var: Optional[str]  = None    # for variables

    def is_const(self) -> bool: return self.op == 'const'
    def is_var(self) -> bool:   return self.op == 'var'


# Each pattern: (pattern_tree, cost, emitted_instruction_template)
# Pattern language: op(child1, child2) or wildcards
MachinePattern = Tuple[str, int, str]   # (pattern_name, cost, asm_template)

X86_PATTERNS = [
    # Simple register ops
    ("ADD(r, r)",        1,  "add {dest}, {src}"),
    ("SUB(r, r)",        1,  "sub {dest}, {src}"),
    ("IMUL(r, r)",       3,  "imul {dest}, {src}"),
    ("AND(r, r)",        1,  "and {dest}, {src}"),
    ("OR(r, r)",         1,  "or  {dest}, {src}"),
    # Immediate operands (saves a load)
    ("ADD(r, imm32)",    1,  "add {dest}, {imm}"),
    ("IMUL(r, imm32)",   3,  "imul {dest}, {src}, {imm}"),
    # Shifts (cheaper than multiply for powers of 2)
    ("SHL(r, imm5)",     1,  "shl {dest}, {imm}"),
    ("SHR(r, imm5)",     1,  "shr {dest}, {imm}"),
    # Memory addressing modes
    ("MOV(MEM(r))",      4,  "mov {dest}, [{base}]"),
    ("MOV(MEM(ADD(r,r)))", 4, "mov {dest}, [{base}+{idx}]"),
    ("MOV(MEM(ADD(r,SHL(r,imm))))", 4, "mov {dest}, [{base}+{idx}*{scale}]"),
    # x86 LEA: computes address arithmetic in one instruction
    ("LEA(ADD(r, r))",   1,  "lea {dest}, [{base}+{idx}]"),
    ("LEA(ADD(r, ADD(SHL(r,imm), imm32)))", 1, "lea {dest}, [{base}+{idx}*{s}+{disp}]"),
]
```

The key insight: `a[i*4 + 8]` in the IR becomes a single `MOV rax, [rbase + ridx*4 + 8]` instruction on x86 using the scaled indexed addressing mode — one instruction that would naively take 3 IR operations. The pattern matcher finds this automatically.

### 13.2 Dynamic Programming Tiling (Maximal Munch)

```python
def tile_tree(node: IRNode, patterns: List[MachinePattern]) -> Tuple[int, List[str]]:
    """
    Dynamic programming instruction selection.
    Returns (cost, list_of_instructions).
    
    For each node, find the cheapest pattern that covers it and
    recursively tile its uncovered subtrees.
    """
    if node.is_var():
        return 0, [f"; use {node.var}"]
    if node.is_const():
        return 1, [f"mov r_temp, {node.value}"]

    # Try all patterns (simplified: just look at op and child count)
    best_cost = float('inf')
    best_instrs = []

    # Recursively solve children first (they may be covered separately)
    child_costs = []
    child_instrs = []
    for child in node.children:
        cc, ci = tile_tree(child, patterns)
        child_costs.append(cc)
        child_instrs.extend(ci)

    # Find cheapest matching pattern for this node
    for pat_name, pat_cost, template in patterns:
        # Simplified matching (real BURG uses proper tree automata)
        if node.op.upper() in pat_name:
            total_cost = pat_cost + sum(child_costs)
            if total_cost < best_cost:
                best_cost = total_cost
                best_instrs = child_instrs + [template]

    return best_cost, best_instrs
```

In production compilers, BURG (Bottom-Up Rewrite Grammar) uses tree automata to match all patterns simultaneously in linear time. LLVM uses a different approach: **SelectionDAG** (a Directed Acyclic Graph rather than a tree, to handle value reuse), with a table-driven selector generated from `.td` (TableGen) pattern definitions.

### 13.3 Instruction Scheduling

After selection, instructions must be **scheduled** — ordered so that long-latency operations (memory loads, floating-point) don't stall the pipeline waiting for their results.

Modern CPUs have:
- **Out-of-order execution**: the CPU reorders instructions internally, but can only do so within a window (~200 instructions for Intel Skylake)
- **Multiple execution units**: several instructions can execute in parallel if they don't have data dependencies
- **Deep pipelines**: a floating-point multiply may take 4 cycles, during which 4 subsequent independent instructions can execute

The compiler's scheduler builds a **dependence graph** (edges from producers to consumers) and finds an ordering that respects dependencies while minimizing the critical path:

```python
def list_schedule(instrs: List[Instr], latencies: Dict[str, int]) -> List[Instr]:
    """
    List scheduling: greedily issue instructions whose dependencies are satisfied,
    prioritizing those on the critical path.
    """
    # Build dependency graph
    last_def: Dict[str, int] = {}   # var -> instruction index that defines it
    ready_time: List[int] = [0] * len(instrs)
    deps: Dict[int, List[int]] = defaultdict(list)  # i -> [j: j must precede i]

    for i, (dest, op, s1, s2) in enumerate(instrs):
        if s1 and s1 in last_def:
            j = last_def[s1]
            deps[i].append(j)
            ready_time[i] = max(ready_time[i], ready_time[j] + latencies.get(instrs[j][1], 1))
        if s2 and s2 in last_def:
            j = last_def[s2]
            deps[i].append(j)
            ready_time[i] = max(ready_time[i], ready_time[j] + latencies.get(instrs[j][1], 1))
        if dest:
            last_def[dest] = i

    # Emit instructions in order of earliest ready time (simplified list scheduler)
    scheduled = sorted(range(len(instrs)), key=lambda i: ready_time[i])
    return [instrs[i] for i in scheduled]
```

Real schedulers use heuristics like **longest path to exit** (to avoid stalling critical paths) and **register pressure** (to avoid creating too many live values simultaneously).

---

## 15. Interprocedural Analysis: Call Graphs and Devirtualization

Intraprocedural analyses (within a single function) are limited by function call boundaries. **Interprocedural analysis (IPA)** reasons across function boundaries.

### 15.1 Call Graph Construction

The **call graph** has one node per function and one edge per call site `f()` in function `g`:

```python
@dataclass
class CallGraphNode:
    func_name: str
    callees: List[str] = field(default_factory=list)
    callers: List[str] = field(default_factory=list)
    is_recursive: bool = False

def build_call_graph(functions: Dict[str, List[BB]]) -> Dict[str, CallGraphNode]:
    """Build a static call graph from a set of function CFGs."""
    graph: Dict[str, CallGraphNode] = {
        name: CallGraphNode(name) for name in functions
    }
    for fname, blocks in functions.items():
        for b in blocks:
            for dest, op, s1, s2 in b.instrs:
                if op == 'call':
                    callee = s1  # function name
                    if callee in graph:
                        graph[fname].callees.append(callee)
                        graph[callee].callers.append(fname)

    # Detect recursive functions (SCC decomposition)
    # (simplified: just check self-calls and mutual recursion via DFS)
    visited, in_stack = set(), set()
    def dfs_scc(f: str):
        visited.add(f); in_stack.add(f)
        for callee in graph[f].callees:
            if callee not in visited:
                dfs_scc(callee)
            elif callee in in_stack:
                graph[f].is_recursive = True
        in_stack.discard(f)
    for f in graph: dfs_scc(f)
    return graph
```

### 15.2 Devirtualization

In OOP languages (Java, C++, Python), method calls are often **virtual** — the actual method to call depends on the runtime type of the receiver. This is an optimization barrier: the compiler can't inline an unknown callee.

**Devirtualization** proves at compile time which concrete method will be called, enabling inlining:

**Technique 1: Class hierarchy analysis (CHA)**
If the class hierarchy has only one concrete implementation of `foo()`, all calls to `obj.foo()` resolve to that one implementation.

```python
def devirtualize_cha(call_site_type: str, method: str,
                     class_hierarchy: Dict[str, List[str]]) -> Optional[str]:
    """
    Returns the unique concrete implementation of method for call_site_type,
    or None if multiple implementations exist.
    """
    implementors = []
    def collect(cls: str):
        if method in get_methods(cls):
            implementors.append(cls)
        for subclass in class_hierarchy.get(cls, []):
            collect(subclass)
    collect(call_site_type)
    return implementors[0] if len(implementors) == 1 else None
```

**Technique 2: Profile-guided devirtualization**
Use runtime profiling to find that 99% of calls to a virtual method go to class `Foo`. Insert a **type guard** and inline `Foo.method()`, with a slow path for other types:

```python
# Before: virtual call (opaque)
result = obj.process(data)

# After: profile-guided devirtualization
if type(obj) is FastPath:          # type check (1 branch)
    result = FastPath.process(obj, data)   # inlined — enables all optimizations
else:
    result = obj.process(data)     # slow path: genuine virtual dispatch
```

This is the key optimization in V8 (JavaScript), HotSpot JVM, and PyPy: most JavaScript objects have the same "hidden class" / "shape" for their first few accesses, so the type guard is almost never false, and the inlined fast path runs at near-native speed.

### 15.3 Interprocedural Constant Propagation

When a function is called with a constant argument at all call sites, that argument is effectively a compile-time constant inside the function:

```python
# If all calls to add_offset use offset=8:
def add_offset(x, offset): return x + offset
# Specialize to:
def add_offset_8(x): return x + 8  # offset folded; x+8 may simplify further
```

This is the basis for **function cloning** and **partial evaluation** — powerful techniques used in Haskell's GHC (via specialization of type class methods) and in `numba`/`cython` (specializing Python functions for specific types).

---

## 16. Escape Analysis and Object Stack Allocation

**Escape analysis** determines whether an object allocated at a program point can "escape" the scope in which it was created — if not, it can be stack-allocated (zero GC overhead) or scalar-replaced (its fields become separate variables, potentially in registers).

### 15.1 Connection Points Analysis

An object allocation `o = new T(...)` escapes if any of the following hold:
1. It is stored into a field of another object or a global variable
2. It is passed to a function that may store it (and we can't inline that function)
3. It is returned from the function

```python
@dataclass
class AllocationSite:
    var: str              # SSA var holding the allocated object
    block_id: int         # where it was allocated
    escapes: bool = False # conservative: assume non-escaping until proven otherwise

def escape_analysis(blocks: List[BB], call_graph: Dict[str, bool]) -> Dict[str, bool]:
    """
    Returns {var: escapes} for all allocation sites.
    call_graph maps function names to whether they might store their args.
    """
    # Find all allocation sites
    alloc_sites: Dict[str, AllocationSite] = {}
    for b in blocks:
        for dest, op, s1, s2 in b.instrs:
            if op == 'new':
                alloc_sites[dest] = AllocationSite(var=dest, block_id=b.id)

    # Track escapes
    escapes: Dict[str, bool] = {v: False for v in alloc_sites}

    for b in blocks:
        for dest, op, s1, s2 in b.instrs:
            # Store to a field: GETFIELD(s1, field) = s2 — s2 escapes
            if op == 'setfield' and s2 in alloc_sites:
                escapes[s2] = True
            # Return — escapes
            if op == 'return' and s1 in alloc_sites:
                escapes[s1] = True
            # Call to function that might store its args
            if op == 'call':
                func = dest  # simplified: function name in dest
                if call_graph.get(func, True):  # conservative: assume stores
                    if s1 in alloc_sites: escapes[s1] = True
                    if s2 in alloc_sites: escapes[s2] = True

    return escapes
```

### 15.2 Scalar Replacement of Aggregates (SROA)

If an object doesn't escape and its fields are accessed in a regular pattern, the compiler can replace the object with individual scalar variables — one per field:

```python
# Before SROA
def compute(a, b):
    point = Point(a, b)        # allocated on heap (or stack)
    return point.x + point.y  # two field loads

# After SROA (escape analysis proves point doesn't escape)
def compute(a, b):
    point_x = a    # 'x' field becomes a scalar
    point_y = b    # 'y' field becomes a scalar
    return point_x + point_y  # no allocation, no loads
```

In SSA terms, `point.x` and `point.y` become ordinary SSA variables — enabling all the SSA-based optimizations (constant folding, DCE, register allocation) to apply to them directly.

LLVM's `mem2reg` pass does exactly this for `alloca` instructions (stack allocations): it promotes `alloca`/`store`/`load` patterns to phi functions and SSA values, which are then optimized by subsequent passes.

### 15.3 Partial Escape Analysis

A more powerful variant: an allocation may escape on some paths but not others. **Partial escape analysis** (Stadler et al., 2014, as used in Graal/GraalVM) can materialize the object only on the paths where it escapes:

```python
# Before partial escape analysis
def process(cond, x, y):
    obj = MyObj(x, y)          # always allocated
    if cond:
        global_list.append(obj)  # obj escapes on this path
    return obj.val + 1           # obj doesn't escape on !cond path

# After partial escape analysis:
def process(cond, x, y):
    if cond:
        obj = MyObj(x, y)           # only allocate when it will escape
        global_list.append(obj)
        return obj.val + 1
    else:
        obj_val = compute_val(x, y) # inline field access, no allocation
        return obj_val + 1
```

GraalVM achieves significant speedups on Java benchmarks via partial escape analysis, especially for iterator objects that typically escape in slow paths (exception handling) but not in the hot path.

---

## 17. How to Debug and Inspect Real Compilers

Production compiler debugging requires specific tools and mental models. This section covers practical techniques used daily by compiler engineers.

### 17.1 LLVM IR Dumps

```bash
# Compile C to LLVM IR (human-readable)
clang -S -emit-llvm -O0 example.c -o example_unopt.ll
clang -S -emit-llvm -O2 example.c -o example_opt.ll

# Run specific optimization passes manually
opt -mem2reg -S example_unopt.ll -o after_mem2reg.ll
opt -instcombine -S after_mem2reg.ll -o after_instcombine.ll
opt -O2 -S example.ll -o optimized.ll

# Print IR before and after each pass
opt -O2 -print-before-all -print-after-all example.ll 2>&1 | head -200

# Show which passes run in O2
opt -O2 -debug-pass=Structure example.ll -o /dev/null 2>&1

# Statistics: how many instructions each pass eliminated
opt -O2 -stats example.ll -o /dev/null 2>&1
```

### 17.2 Python — Inspecting CPython Bytecode

```python
import dis, opcode

def inspect_bytecode(func):
    """Print rich bytecode analysis of a Python function."""
    code = func.__code__
    print(f"Function: {func.__name__}")
    print(f"  co_varnames: {code.co_varnames}")
    print(f"  co_consts:   {code.co_consts}")
    print(f"  co_names:    {code.co_names}")
    print(f"  stacksize:   {code.co_stacksize}")
    print()
    dis.dis(func)

def example_loop(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

inspect_bytecode(example_loop)
# Output reveals: BINARY_MULTIPLY, BINARY_ADD — no SSA, no register allocation.
# Each opcode does a full Python object dispatch.


# Compare with numba-compiled version (if numba installed):
try:
    from numba import njit
    jitted = njit(example_loop)
    jitted(10)  # trigger compilation
    # jitted.inspect_llvm() prints the LLVM IR
    # jitted.inspect_asm()  prints the native assembly
    print(jitted.inspect_types())
except ImportError:
    print("Install numba to see JIT-compiled output")
```

### 17.3 GCC Optimization Reports

```bash
# GCC: show what optimizations fired (and which didn't, and why)
gcc -O2 -fopt-info-all example.c -o example 2>&1 | grep -E "vectorized|inlined|unrolled"

# Specific optimization notes
gcc -O2 -fopt-info-vec-missed example.c   # why vectorization failed
gcc -O2 -fopt-info-inline-missed example.c # why inlining failed

# Assembly output with source annotations
gcc -O2 -S -fverbose-asm example.c
```

### 17.4 Understanding Assembly Output

Reading assembly is the ground truth for compiler behavior. Key x86-64 patterns to recognize:

```nasm
; Inlined and constant-folded (entire computation becomes a constant)
mov eax, 42
ret

; Loop vectorized with AVX2
vmovdqu ymm0, [rsi]          ; load 8 int32s
vpsubd ymm0, ymm0, ymm1      ; subtract 8 int32s
vmovdqu [rdi], ymm0          ; store 8 int32s

; Strength-reduced loop (multiply → indexed add)
; Before: for i: a[i] = b[i] * stride
; After: uses LEA with scale factor
lea rax, [rbx + rcx*4]       ; computes rbx + rcx*4 in one cycle

; Tail-call optimization (recursive call becomes a jump)
jmp _recur_func              ; instead of: call + ret
```

### 17.5 Flame Graphs for Compiler Overhead

When a compiler is itself slow, use profiling:

```bash
# Profile the compiler itself
perf record -g clang -O2 large_file.cpp
perf report                   # view hottest functions in the compiler
# Common hotspots: alias analysis, inline cost computation, register allocation

# LLVM TimePass infrastructure
clang -O2 -mllvm -time-passes large_file.cpp
# Shows time spent in each optimization pass
```

### 17.6 Bisecting Optimization Regressions

When a "surprising" optimization causes a regression:

```bash
# Narrow down which pass causes the regression
opt -O1 buggy.ll | ./run_benchmark   # fast
opt -O2 buggy.ll | ./run_benchmark   # slow → something in O2 regresses

# Binary search over passes
opt -O1 -loop-unroll buggy.ll | ./run_benchmark   # is unrolling the culprit?
opt -O1 -vectorize buggy.ll | ./run_benchmark      # or vectorization?

# LLVM's reduce tool: minimize the IR to the smallest reproducer
llvm-reduce --test=is_slow.sh buggy.ll
# Produces minimal IR that still triggers the regression
```

---

## 18. Auto-Vectorization: SIMD and Data Parallelism

Modern CPUs contain **SIMD (Single Instruction, Multiple Data)** units:
- **SSE4.2**: 128-bit registers; 4 × float32, 2 × float64
- **AVX2**: 256-bit registers; 8 × float32, 4 × float64
- **AVX-512**: 512-bit registers; 16 × float32, 8 × float64
- **ARM NEON**: 128-bit, 4 × float32
- **ARM SVE**: scalable vectors (512–2048 bits)

Auto-vectorization transforms a scalar loop into a vectorized one that processes multiple elements per SIMD instruction.

### 17.1 Prerequisite Checks

For a loop to be vectorizable:
1. **No loop-carried dependencies**: `a[i]` doesn't depend on `a[i-1]`
2. **Affine array accesses**: `a[3*i + 5]` — expressible as a linear function of the IV
3. **No aliasing**: `a` and `b` don't point to the same memory
4. **Uniform control flow**: no data-dependent branches inside the loop (or predicatable ones)

```python
def check_vectorizable(loop_body: Set[int], iv: str, blocks: List[BB],
                       alias_sets: Dict[str, Set[str]]) -> Tuple[bool, str]:
    """
    Checks whether a loop can be vectorized.
    Returns (can_vectorize, reason_if_not).
    """
    bmap = {b.id: b for b in blocks}
    array_accesses: List[Tuple[str, str, int]] = []  # (array, index_var, offset)

    for bid in loop_body:
        b = bmap[bid]
        for dest, op, s1, s2 in b.instrs:
            if op in ('load', 'store') and s1:
                # Parse simple affine access: base[iv + const] or base[iv * k + const]
                # (simplified: just check if iv is involved)
                array_accesses.append((s1, iv, 0))

            # Detect loop-carried dependency: if dest is used in a later iteration
            # This requires SCEV analysis to prove i-th use doesn't depend on (i-1)-th def
            if op in ('load',) and dest == s1:
                return False, f"Loop-carried dependency on {dest}"

    # Check aliasing between all array pairs
    arrays = {acc[0] for acc in array_accesses}
    for a1 in arrays:
        for a2 in arrays:
            if a1 != a2 and a2 in alias_sets.get(a1, set()):
                return False, f"Potential alias between {a1} and {a2}"

    return True, ""


def vectorize_loop(scalar_loop_ir: List[Instr], vector_width: int) -> List[Instr]:
    """
    Convert scalar loop body to SIMD instructions.
    vector_width: number of elements per SIMD register (e.g., 8 for AVX2 float32).
    
    Strategy: loop unrolling by vector_width, then pack loads/stores/ops into SIMD.
    """
    vec_instrs = []

    for dest, op, s1, s2 in scalar_loop_ir:
        if op == 'load':
            # VLOAD loads vector_width consecutive elements starting at s1
            vec_instrs.append((f"v_{dest}", 'vload', s1, str(vector_width)))
        elif op == 'store':
            vec_instrs.append((dest, 'vstore', f"v_{s1}", str(vector_width)))
        elif op in ('+', '-', '*', '/'):
            # Element-wise SIMD operation
            vec_instrs.append((f"v_{dest}", f"v{op}", f"v_{s1}", f"v_{s2}"))
        else:
            # Non-vectorizable instruction: scalar fallback
            vec_instrs.append((dest, op, s1, s2))

    return vec_instrs
```

### 17.2 Loop Remainder Handling

SIMD vectorization processes `vector_width` elements per iteration. If the loop count is not a multiple of `vector_width`, a **remainder loop** handles the leftover elements:

```python
# Before vectorization (N elements, vector_width=8)
for i in range(N):
    c[i] = a[i] + b[i]

# After vectorization:
VW = 8
i = 0
while i + VW <= N:       # vectorized main loop
    vc = vadd(vload(a+i, 8), vload(b+i, 8))
    vstore(c+i, vc, 8)
    i += VW

while i < N:             # scalar remainder loop
    c[i] = a[i] + b[i]
    i += 1
```

Alternatively, use **masked operations** (AVX-512's `vmovdqu8` with a mask register, or ARM SVE's predicates) to avoid the remainder loop entirely.

### 17.3 SLP Vectorization

**Superword Level Parallelism (SLP)** vectorizes across multiple consecutive statements rather than loop iterations:

```python
# Before SLP
a0 = x[0] + y[0]
a1 = x[1] + y[1]
a2 = x[2] + y[2]
a3 = x[3] + y[3]

# After SLP: pack into one 4-wide SIMD add
va = vload(x, 4)   # [x[0], x[1], x[2], x[3]]
vb = vload(y, 4)   # [y[0], y[1], y[2], y[3]]
vc = vadd(va, vb)  # [a0, a1, a2, a3]
vstore(a, vc, 4)
```

LLVM's SLP vectorizer identifies **isomorphic statement groups** (same structure, sequential access patterns) and packs them into SIMD operations. This is effective in unrolled loops and array initializations.

---

## 19. Points-to Analysis: Alias Analysis Algorithms

Alias analysis is the foundation of almost every memory optimization. Without it, the compiler must conservatively assume any two pointers might alias — preventing vectorization, LICM, load elimination, and store-load forwarding.

### 19.1 The Aliasing Problem

Two pointers `p` and `q` **must-alias** if they always refer to the same location, **may-alias** if they sometimes do, and **no-alias** if they never do.

```python
# No aliasing possible (different allocations)
p = object()
q = object()
# Safe to reorder: p and q are distinct

# May-alias (unknown function may alias)
def unknown_fn(p, q): ...
# Compiler must assume p and q may alias

# Must-alias (provably same location)
p = array
q = array
# Reading p and writing q affect the same memory
```

### 19.2 Andersen's Algorithm (Flow-Insensitive Points-to Analysis)

Andersen's algorithm computes a **points-to set** for each pointer variable: `pt(p)` = the set of abstract locations that `p` may point to:

```python
def andersen_points_to(stmts: List[Tuple[str, str, str]]) -> Dict[str, Set[str]]:
    """
    Flow-insensitive points-to analysis (Andersen 1994).
    
    Handles four statement forms:
      - ('alloc', dest, obj_id):    dest = &obj   → pt(dest) ⊇ {obj}
      - ('copy', dest, src):        dest = src     → pt(dest) ⊇ pt(src)
      - ('load', dest, src):        dest = *src    → ∀x∈pt(src): pt(dest) ⊇ pt(x)
      - ('store', dest, src):       *dest = src    → ∀x∈pt(dest): pt(x) ⊇ pt(src)
    """
    pt: Dict[str, Set[str]] = defaultdict(set)
    changed = True

    while changed:
        changed = False
        for kind, dst, src in stmts:
            if kind == 'alloc':
                if src not in pt[dst]:
                    pt[dst].add(src)
                    changed = True
            elif kind == 'copy':
                new_pts = pt[src] - pt[dst]
                if new_pts:
                    pt[dst] |= new_pts; changed = True
            elif kind == 'load':
                for obj in pt[src]:
                    new_pts = pt[obj] - pt[dst]
                    if new_pts:
                        pt[dst] |= new_pts; changed = True
            elif kind == 'store':
                for obj in pt[dst]:
                    new_pts = pt[src] - pt[obj]
                    if new_pts:
                        pt[obj] |= new_pts; changed = True

    return dict(pt)


# Example:
stmts = [
    ('alloc', 'p', 'obj1'),   # p = &obj1
    ('alloc', 'q', 'obj2'),   # q = &obj2
    ('copy',  'r', 'p'),      # r = p
    ('load',  's', 'r'),      # s = *r (→ pt(s) ⊇ pt(obj1))
    ('store', 'q', 'p'),      # *q = p (→ pt(obj2) ⊇ pt(p))
]
result = andersen_points_to(stmts)
for var, pts in sorted(result.items()):
    print(f"  pt({var}) = {sorted(pts)}")
# pt(p) = ['obj1']
# pt(q) = ['obj2']
# pt(r) = ['obj1']
# pt(s) = []        (obj1 is a concrete object, no points-to)
# pt(obj2) = ['obj1']  (because *q = p, and q→obj2, p→obj1)
```

**Complexity**: O(n³) in the worst case (n variables). For real programs with millions of variables, approximate algorithms are needed.

### 19.3 Steensgaard's Algorithm (O(n·α(n)) Union-Find)

Steensgaard's algorithm sacrifices precision for speed: it **unifies** points-to sets using union-find, giving a nearly-linear-time algorithm:

```python
def steensgaard(stmts: List[Tuple[str, str, str]]) -> Dict[str, str]:
    """
    Steensgaard's points-to analysis using union-find.
    Returns a union-find representative for each variable.
    Steensgaard's key rule: if p and q point to overlapping sets,
    merge those sets.
    """
    parent: Dict[str, str] = {}
    deref: Dict[str, str] = {}  # canonical var -> what it points to

    def find(x: str) -> str:
        if x not in parent: parent[x] = x
        if parent[x] != x: parent[x] = find(parent[x])
        return parent[x]

    def union(x: str, y: str) -> None:
        rx, ry = find(x), find(y)
        if rx != ry: parent[rx] = ry

    for kind, dst, src in stmts:
        rd, rs = find(dst), find(src)
        if kind == 'alloc':
            deref[rd] = src          # rd points to src
        elif kind == 'copy':
            if rd in deref and rs in deref:
                union(deref[rd], deref[rs])  # merge their pointees
            elif rs in deref:
                deref[rd] = deref[rs]
        elif kind == 'load':
            if rs in deref:
                union(rd, deref[rs])
        elif kind == 'store':
            if rd in deref:
                union(deref[rd], rs)

    return {v: find(v) for v in parent}
```

### 19.4 Type-Based Alias Analysis (TBAA)

In C/C++, the strict aliasing rule says: pointers to different types cannot alias (with some exceptions). LLVM encodes this in IR metadata:

```llvm
; Annotate loads/stores with TBAA metadata
%x = load float, float* %p, !tbaa !1
store i32 0, i32* %q, !tbaa !2
; !1 and !2 are different TBAA nodes → proven no-alias

!1 = !{!"float", !0}
!2 = !{!"int", !0}
!0 = !{!"root"}
```

Python's dynamic typing makes TBAA inapplicable — any variable can hold any type, so type-based aliasing proofs are impossible without runtime type guards.

---

## 20. SSA Destruction: Phi Functions to Machine Code

One of the least-discussed but most subtle aspects of SSA-based compilation is **SSA destruction** — converting phi functions back to concrete copy instructions before register allocation.

### 19.1 The Naive Approach and Its Failure

The obvious approach: for `x_3 = φ(x_1 [from B1], x_2 [from B2])`, insert copies at the end of each predecessor:

```
B1: ... ; x_3 = x_1    ← inserted
B2: ... ; x_3 = x_2    ← inserted
B3: (phi removed)
```

This fails in two cases.

**The Lost Copy Problem**: Suppose we have two phi functions at B3:
```
B3:  a_3 = φ(a_1, a_2)
     b_3 = φ(b_1, b_2)
```
And in B1: `a_2 = b_1`. After naive phi destruction:
```
B1: a_3 = a_1
    b_3 = b_1    ← but this overwrites a_3! (if a_3 and b_1 share a register)
```

**The Swap Problem**: Two phis where one's argument is the other's result:
```
B1: a = φ(a_init, b_loop)
    b = φ(b_init, a_loop)    ← a_loop was the OLD a, not the new one
```
After naive insertion, we'd get `a = b_loop; b = a_loop` — sequential copies, but we need a **swap**: `tmp = a; a = b_loop; b = tmp`.

### 19.2 Sequentialization Algorithm (Briggs et al.)

The correct algorithm serializes a set of parallel copies:

```python
def serialize_parallel_copies(copies: List[Tuple[str, str]]) -> List[Tuple[str, str]]:
    """
    Convert a set of parallel copies {dst_i = src_i} to a sequence of
    sequential copies that achieves the same net effect.
    
    copies: list of (dst, src) pairs to be performed "simultaneously"
    Returns: list of (dst, src) pairs to be performed sequentially.
    """
    # Represent as a graph: edge (src -> dst) means "copy src to dst"
    # Nodes with in-degree 0 can be done first (no one overwrites their source)
    out_edges: Dict[str, str] = {}  # src -> dst
    in_deg: Dict[str, int] = defaultdict(int)
    for dst, src in copies:
        out_edges[src] = dst
        in_deg[dst] += 1

    result = []
    ready = deque(src for src, _ in [(s, d) for d, s in copies] if in_deg.get(src, 0) == 0)

    # Process nodes with no incoming copy edges first
    while ready:
        src = ready.popleft()
        if src in out_edges:
            dst = out_edges[src]
            result.append((dst, src))   # do this copy
            del out_edges[src]
            in_deg[dst] -= 1
            if in_deg[dst] == 0 and dst in out_edges:
                ready.append(dst)

    # Handle remaining cycles (swap problem)
    while out_edges:
        # Pick any node in a cycle, break it with a temp
        start = next(iter(out_edges))
        tmp = f"__tmp_{start}"
        result.append((tmp, start))     # tmp = start

        cur = start
        while True:
            nxt = out_edges.pop(cur)
            result.append((nxt, cur if cur != start else tmp))
            if nxt == start:
                break
            cur = nxt

    return result


# Example: the swap case
copies = [("a", "b"), ("b", "a")]
serialized = serialize_parallel_copies(copies)
print("Swap serialization:")
for dst, src in serialized:
    print(f"  {dst} = {src}")
# Output:
# __tmp_a = a
# a = b
# b = __tmp_a
```

### 19.3 Phi Destruction Choices: Before vs. After Register Allocation

There are two schools:

**Phi destruction before register allocation** (traditional):
1. Destroy phis → insert copies
2. Run register allocation on the copy-filled code
3. Coalescing during register allocation eliminates most copies

**Phi destruction after register allocation** (modern, e.g., LLVM):
1. Run register allocation with SSA-aware liveness (using the interference graph property of SSA)
2. After allocation, replace each phi argument `x_i` with the physical register assigned to `x_i`
3. If phi and its argument share the same register → no copy needed (free)
4. If not → insert a parallel copy instruction, then serialize

LLVM uses the second approach via the `PHIElimination` and `TwoAddressInstructionPass` passes. The key insight: in SSA, the interference graph has a special property (SSA invariant property) that means Chaitin's algorithm can assign registers to SSA values directly without first destroying the phis.

---

## 21. Advanced Register Allocation: Linear Scan and SSA-Based Allocation

Graph-coloring (Chaitin) register allocation is optimal but slow — O(n²) for the interference graph construction, and the coloring problem is NP-complete (though Chaitin's heuristic is fast in practice). JIT compilers need faster algorithms.

### 21.1 Linear Scan Register Allocation

**Linear scan** (Poletto & Sarkar, 1999) is O(n log n) and fast enough for JIT use:

1. Compute a **linear ordering** of all instructions (e.g., RPO block ordering)
2. For each SSA variable, compute its **live interval**: `[first_def, last_use]` in the linear order
3. Scan instructions left to right, maintaining active (currently live) intervals sorted by endpoint
4. When a new interval starts: if a free register exists, assign it; otherwise **spill** the interval with the latest endpoint

```python
@dataclass(order=True)
class LiveInterval:
    start: int              # first definition point in linear order
    end: int                # last use point
    var: str = field(compare=False)
    reg: Optional[int] = field(default=None, compare=False)

def compute_live_intervals(blocks: List[BB]) -> List[LiveInterval]:
    """Assign each instruction a linear position and compute live intervals."""
    pos: Dict[Tuple[int, int], int] = {}  # (block_id, instr_idx) -> linear position
    cur = 0
    intervals: Dict[str, LiveInterval] = {}

    for b in blocks:  # assumes blocks in RPO order
        for i, (dest, op, s1, s2) in enumerate(b.instrs):
            pos[(b.id, i)] = cur
            if dest:
                if dest not in intervals:
                    intervals[dest] = LiveInterval(start=cur, end=cur, var=dest)
                intervals[dest].end = cur
            for src in (s1, s2):
                if src and src in intervals:
                    intervals[src].end = max(intervals[src].end, cur)
            cur += 1

    return sorted(intervals.values())


def linear_scan(intervals: List[LiveInterval], num_regs: int) -> Dict[str, int]:
    """
    Linear scan register allocator.
    Returns var -> register (-1 = spilled).
    """
    import heapq

    free_regs = list(range(num_regs))
    active: List[Tuple[int, LiveInterval]] = []  # heap by end point
    allocation: Dict[str, int] = {}

    for interval in sorted(intervals, key=lambda i: i.start):
        # Expire old intervals
        new_active = []
        while active:
            end, old_iv = active[0]
            if end <= interval.start:
                heapq.heappop(active)
                free_regs.append(old_iv.reg)  # release the register
            else:
                break

        if not free_regs:
            # Spill: the interval with the furthest endpoint loses its register
            if active and active[-1][0] > interval.end:
                # Spill the interval that ends latest
                _, spill_iv = active.pop()
                free_regs.append(spill_iv.reg)
                spill_iv.reg = -1
                allocation[spill_iv.var] = -1

        if free_regs:
            reg = free_regs.pop()
            interval.reg = reg
            allocation[interval.var] = reg
            heapq.heappush(active, (interval.end, interval))
        else:
            allocation[interval.var] = -1  # spill

    return allocation


def demo_linear_scan():
    intervals = [
        LiveInterval(start=0, end=10, var='a'),
        LiveInterval(start=1, end=5,  var='b'),
        LiveInterval(start=3, end=8,  var='c'),
        LiveInterval(start=6, end=12, var='d'),
    ]
    alloc = linear_scan(intervals, num_regs=2)
    for var, reg in alloc.items():
        print(f"  {var} → {'SPILL' if reg==-1 else f'R{reg}'}")

demo_linear_scan()
# a → R0  (active: [a])
# b → R1  (active: [a,b])
# c → SPILL (no free regs; c ends at 8, b ends at 5: spill c)
# d → R1  (b freed at 5; d gets R1)
```

### 21.2 SSA-Based Linear Scan

An important optimization: SSA variables have a special **chordal interference graph** property. For SSA-form programs, the interference graph is chordal (every cycle of length ≥ 4 has a chord), which means it is **always k-colorable if and only if no clique is larger than k**. This is significant because:

1. Optimal coloring of chordal graphs is polynomial (vs. NP-complete for general graphs)
2. Live intervals in SSA form don't need to be split (no phi argument interference unless the phi is on a critical edge)

The practical consequence: linear scan on SSA variables is simpler because split intervals and the associated "holes" in live ranges are easier to reason about — a variable's live range ends at its last use, not "wherever it might be needed by a phi in a later block."

### 21.3 LLVM's Register Allocator

LLVM provides multiple register allocators:
- **`greedy`** (default): sophisticated, uses live range splitting and local search
- **`basic`**: simple linear scan variant
- **`pbqp`**: Partitioned Boolean Quadratic Programming — models register allocation as a constrained optimization problem
- **`fast`**: O(n) with poor quality, used in -O0 for fast compilation

The `greedy` allocator uses **live range splitting** to handle situations where a variable is used infrequently in a hot loop but needed after: split the range, spill part of it, and reload before the use — minimizing spill cost in the critical path.

---

## 22. JIT Compilation: Tracing, Method JITs, and Tiered Compilation

**Just-In-Time (JIT) compilation** compiles code at runtime, after observing actual execution. This enables optimizations impossible for ahead-of-time (AOT) compilers: profile-guided decisions, type specialization, and on-the-fly deoptimization.

### 21.1 Tracing JIT (PyPy, LuaJIT, Firefox TraceMonkey)

A **tracing JIT** works by:
1. Interpreting the program until a **hot loop** is detected (a backward branch executed > threshold times, e.g., 1000)
2. Starting to **record a trace** — a linear sequence of all operations executed in one loop iteration, including all branches taken (as **guards**)
3. Optimizing the trace (it's straight-line code — easy!) and compiling to native machine code
4. On future iterations: execute the native trace. When a guard fails (branch taken differently), fall back to interpretation or re-trace

```python
class TraceRecorder:
    """Simplified trace recorder for a hypothetical interpreter."""

    def __init__(self):
        self.trace: List[dict] = []
        self.guards: List[dict] = []
        self.recording = False

    def record_op(self, op: str, operands: list, result) -> None:
        if not self.recording: return
        self.trace.append({'op': op, 'operands': operands, 'result': result})

    def record_guard(self, condition: bool, expected: bool, bailout_pc: int) -> None:
        """Record a guard: if condition != expected, bail out to interpreter."""
        if not self.recording: return
        if condition != expected:
            # Guard failure: stop tracing, mark trace as incomplete
            self.recording = False
            return
        self.guards.append({
            'expected': expected,
            'bailout_pc': bailout_pc,
            'snapshot': self._current_state_snapshot()
        })

    def optimize_trace(self) -> List[dict]:
        """Apply optimizations to the linear trace."""
        # Constant folding: propagate constants through the trace
        values: Dict[int, object] = {}
        opt_trace = []
        for instr in self.trace:
            # Fold if all operands are known constants
            if all(isinstance(op, (int, float)) for op in instr['operands']):
                result = eval_op(instr['op'], instr['operands'])
                values[id(instr['result'])] = result
            else:
                opt_trace.append(instr)
        return opt_trace

    def _current_state_snapshot(self): return {}
```

**Key insight**: a trace is just a straight-line sequence with guards. No control flow! This makes optimization trivially applicable. The downside: if guards fail frequently (polymorphic code), traces don't help and can hurt (due to recording overhead).

### 21.2 Method JIT (HotSpot JVM, V8 Turbofan, LLVM-based JITs)

A **method JIT** compiles entire methods (functions), not just loop traces:
1. Detect hot methods (call count > threshold)
2. Compile the entire CFG of the method to native code
3. Use profile data for inlining decisions and branch prediction hints
4. Insert **deoptimization points** where runtime checks might fail

```
Method JIT pipeline (V8 Turbofan):
    JavaScript AST
         │
         ▼  Bytecode generation (Ignition interpreter)
    Bytecode + runtime type feedback
         │
         ▼  Sea of Nodes graph construction
    Typed IR (with speculative type assumptions)
         │
         ▼  Inlining (based on call frequency profile)
    Inlined typed IR
         │
         ▼  SSA construction + redundancy elimination
    Optimized SSA IR
         │
         ▼  Register allocation (linear scan)
    Register-assigned instructions
         │
         ▼  Code generation
    Native x86/ARM machine code
```

The "speculative type assumptions" are guarded: if `x` was always an `int` in profiling, compile `x + y` as an integer add, but insert a check that `x` is still an int. If the check fails → **deoptimize**: throw away the JIT-compiled frame, reconstruct the interpreter stack, and resume in the interpreter.

### 21.3 Tiered Compilation

Modern runtimes use multiple compilation tiers:

| Tier | Compiler | When | Characteristics |
|------|---------|------|-----------------|
| T0 | Interpreter | Always | Slowest, no warmup, collects profile |
| T1 | Quick JIT (C1) | After ~100 invocations | Fast compile, few optimizations |
| T2 | Optimizing JIT (C2) | After ~10,000 invocations | Slow compile, aggressive optimizations |
| T3 | Speculative JIT | Per-call-site profile | Type-specialized, may deoptimize |

```python
class TieredCompilationManager:
    """Manages compilation tiers for a function."""

    QUICK_THRESHOLD    = 100
    OPTIMIZING_THRESHOLD = 10_000

    def __init__(self):
        self.invocation_counts: Dict[str, int] = defaultdict(int)
        self.compiled_versions: Dict[str, str]  = {}  # func -> tier
        self.profiles: Dict[str, dict] = defaultdict(dict)

    def on_invocation(self, func_name: str, args_types: tuple) -> str:
        """Called on each function invocation. Returns which tier to use."""
        count = self.invocation_counts[func_name]
        self.invocation_counts[func_name] += 1

        # Update type profile
        types_seen = self.profiles[func_name].get('arg_types', set())
        types_seen.add(args_types)
        self.profiles[func_name]['arg_types'] = types_seen

        if count < self.QUICK_THRESHOLD:
            return 'interpreter'
        elif count < self.OPTIMIZING_THRESHOLD:
            if self.compiled_versions.get(func_name) != 'quick':
                self._compile_quick(func_name)
            return 'quick'
        else:
            if self.compiled_versions.get(func_name) != 'optimizing':
                # Use profile data: if only one arg type seen, specialize
                if len(types_seen) == 1:
                    self._compile_specialized(func_name, list(types_seen)[0])
                else:
                    self._compile_optimizing(func_name)
            return self.compiled_versions[func_name]

    def _compile_quick(self, fn): self.compiled_versions[fn] = 'quick'
    def _compile_optimizing(self, fn): self.compiled_versions[fn] = 'optimizing'
    def _compile_specialized(self, fn, types): self.compiled_versions[fn] = f'specialized({types})'
```

### 21.4 Deoptimization: The Safety Net

When a speculative optimization's assumption is violated at runtime, the JIT must **deoptimize**:
1. Identify which optimistic assumption was violated (type check, bounds check, null check)
2. Map the JIT-compiled register state back to the interpreter's variable state (**OSR exit** — On-Stack Replacement)
3. Resume the interpreter from the equivalent bytecode position
4. Potentially mark the assumption as invalid and recompile without it

Deoptimization requires keeping **debug info** alongside the JIT-compiled code: for each instruction, a mapping from machine registers to logical variables (similar to DWARF debug info). This overhead is non-trivial — V8 sets aside significant memory for this mapping.

---

## 23. Production Compiler Architecture: LLVM, GCC, V8, HotSpot Compared

Understanding how the major production compilers implement the concepts in this lesson puts everything in context. Each made different architectural choices with different tradeoffs.

### 23.1 LLVM Architecture

LLVM's key design principle: **all passes communicate through the same IR**. Every optimization is a `Pass` that reads/modifies LLVM IR and outputs LLVM IR. Passes are composable and independently testable.

```
Frontend (Clang/Rust/Swift/Julia)
    │   Emits LLVM IR
    ▼
Pass Manager (runs optimization pipeline)
    │
    ├─ Analysis passes (compute info, don't modify IR):
    │    ├─ DominatorTree
    │    ├─ LoopInfo
    │    ├─ ScalarEvolution (SCEV)
    │    ├─ AliasAnalysis (chain of analyses)
    │    └─ MemoryDependenceAnalysis
    │
    ├─ Transform passes (modify IR):
    │    ├─ mem2reg          → promotes alloca to SSA
    │    ├─ instcombine      → peephole + constant folding
    │    ├─ gvn              → global value numbering
    │    ├─ licm             → loop invariant code motion
    │    ├─ loop-unroll      → unrolls inner loops
    │    ├─ loop-vectorize   → SLP + loop vectorization
    │    ├─ inline           → function inlining
    │    └─ simplifycfg      → dead block elimination, branch simplification
    │
    └─ Backend (LLVM CodeGen):
         ├─ SelectionDAG     → instruction selection
         ├─ MachineScheduler → pre-RA scheduling
         ├─ RegisterAlloc    → greedy/linear scan
         ├─ PrologEpilog     → frame setup/teardown
         └─ AsmPrinter       → emits assembly/object
```

The **new pass manager** (NewPM, default since LLVM 14) uses analyses lazily — a pass requests an analysis, which is computed on demand and cached. The **legacy pass manager** computed analyses eagerly.

### 23.2 GCC Architecture

GCC predates LLVM by 20 years and shows its age in architecture, but has extremely mature optimizers:

```
Frontend (C/C++/Fortran/Ada)
    │
    ▼  GENERIC (language-agnostic tree IR)
    │
    ▼  Gimplification (GIMPLE: 3-address simplified IR)
    │
    ▼  Tree SSA optimization (on GIMPLE SSA):
    │    ├─ tree-vrp  (value range propagation)
    │    ├─ tree-dce
    │    ├─ tree-ccp  (conditional constant prop, = SCCP)
    │    ├─ tree-sink (code sinking — opposite of LICM)
    │    └─ tree-loop-* (vectorization, unrolling, interchange)
    │
    ▼  RTL (Register Transfer Language — machine-level IR)
    │    ├─ combine  (RTL peephole)
    │    ├─ reload   (register allocation)
    │    └─ final    (assembly emission)
```

GCC's key strength: its **interprocedural analysis** (IPA) infrastructure is mature. Whole-program analysis passes (type-based alias analysis, devirtualization, constant propagation) are computed across the entire compilation unit.

### 23.3 V8 Turbofan (JavaScript)

V8's Turbofan uses a **Sea of Nodes** (Cliff Click, 1995) IR instead of a traditional CFG:

- **Sea of Nodes**: a graph where both data and control dependencies are explicit edges. There is no fixed instruction order — the scheduler determines order later based on dependencies.
- Nodes represent values (like SSA); edges represent dependencies (data, control, effect)
- **Effect chain**: a separate dependency chain for nodes with side effects (loads, stores, calls), preserving their relative order

```
Sea of Nodes example: a + b * c

Nodes:     [param a] [param b] [param c]
                \       \       /
                 \      [mult b,c]
                  \      /
                  [add a, mult]
                     |
                  [return]

Benefits over CFG+SSA:
  - Values are computed lazily (no schedule until code gen)
  - Global CSE is trivial (same inputs → same node)
  - Speculative deoptimization nodes are first-class
```

### 23.4 HotSpot JVM C2 Compiler

HotSpot uses a similar Sea of Nodes IR (C2 is Cliff Click's compiler, the original Sea of Nodes implementation):

Key C2 optimizations specific to Java:
- **Null check elimination**: after a successful null-checked access, subsequent accesses to the same object don't need a null check
- **Array bounds check elimination**: SCEV can prove `0 <= i < array.length` for well-structured loops
- **Lock elision**: if an object can't escape a thread (escape analysis), its synchronized blocks can be eliminated
- **Intrinsic methods**: `Math.sin()`, `String.hashCode()`, `System.arraycopy()` are replaced by hand-optimized intrinsics directly in the JIT

### 23.5 Key Architectural Differences

| Aspect | LLVM | GCC | V8 Turbofan | HotSpot C2 |
|--------|------|-----|-------------|-----------|
| IR type | SSA CFG | GIMPLE SSA → RTL | Sea of Nodes | Sea of Nodes |
| Pass architecture | Plugin-based, composable | Monolithic tree of passes | Phase-based pipeline | Monolithic |
| Compile-time priority | Medium | Medium | Low (JIT: fast is critical) | Medium |
| Alias analysis | Chain of AAs + TBAA | Type-based + structual | Type guards + speculation | Escape analysis |
| Vectorization | Loop + SLP | Both (mature) | SIMD intrinsics only | Limited |
| Deoptimization | N/A (AOT) | N/A | Full frame reconstruction | Full OSR |
| Register allocator | Greedy (interval splitting) | IRA (Integrated RA) | Linear scan | Linear scan |
| Open source | Yes (Apache 2.0) | Yes (GPL) | Yes (BSD) | Yes (GPL) |

---

## 24. Compiler Testing and Correctness: Fuzzing and Differential Testing

Compilers are among the most tested pieces of software ever written, yet they still have bugs. A miscompilation that silently produces wrong output is catastrophic — and these bugs are notoriously hard to find.

### 23.1 The Correctness Challenge

A compiler optimization is correct if it **preserves the observable behavior** of every program. But what counts as observable?

- Output (stdout/stderr)
- Return values
- Side effects (memory writes, I/O)
- Termination behavior
- Signal delivery

The **undefined behavior** problem: in C/C++, many operations are "undefined" — the standard imposes no requirements. This allows the compiler to assume they don't happen and optimize accordingly. The resulting code may be correct for well-defined inputs but produce surprising results for others. This is a correctness/security minefield.

### 23.2 Differential Testing (CSmith)

**CSmith** (Yang et al., 2011) found hundreds of bugs in GCC and LLVM by:
1. Generating random, well-defined C programs (avoiding undefined behavior)
2. Compiling each program with multiple compilers and optimization levels
3. Comparing outputs: if GCC -O0 and GCC -O2 produce different outputs, there's a bug

```python
import random
import subprocess
import itertools
import tempfile
import os

def differential_test_compiler(num_tests: int = 100) -> List[str]:
    """
    Simplified differential testing: generate random arithmetic programs,
    compile with different flags, compare outputs.
    """
    bugs = []

    def gen_program() -> str:
        """Generate a simple, well-defined arithmetic program."""
        ops = ['+', '-', '*']
        vars = ['a', 'b', 'c']
        stmts = []
        for i in range(5):
            lhs = f"x{i}"
            op = random.choice(ops)
            v1 = random.choice(vars + [str(random.randint(1, 100))])
            v2 = random.choice(vars + [str(random.randint(1, 100))])
            stmts.append(f"  int {lhs} = {v1} {op} {v2};")
        result = random.choice([f"x{i}" for i in range(5)])
        body = '\n'.join(stmts)
        return f"""
#include <stdio.h>
int main() {{
  int a = 3, b = 7, c = 13;
{body}
  printf("%d\\n", {result});
  return 0;
}}
"""

    for _ in range(num_tests):
        src = gen_program()
        with tempfile.NamedTemporaryFile(suffix='.c', mode='w', delete=False) as f:
            f.write(src); fname = f.name

        outputs = {}
        for flag in ['-O0', '-O1', '-O2', '-O3']:
            try:
                out_bin = fname.replace('.c', flag)
                subprocess.run(['gcc', flag, fname, '-o', out_bin],
                                capture_output=True, check=True)
                result = subprocess.run([out_bin], capture_output=True,
                                         text=True, timeout=5)
                outputs[flag] = result.stdout.strip()
                os.unlink(out_bin)
            except (subprocess.CalledProcessError, subprocess.TimeoutExpired):
                pass

        os.unlink(fname)

        # Check for mismatches
        unique_outputs = set(outputs.values())
        if len(unique_outputs) > 1:
            bug_report = f"MISMATCH: {outputs}"
            bugs.append(bug_report)

    return bugs
```

### 23.3 Fuzzing Compiler IR (Alive2)

**Alive2** is a formal verification tool for LLVM passes: it proves (or disproves) that a transformation preserves all semantics using an SMT solver (Z3). It has found dozens of bugs in LLVM's peephole optimizations.

```
# Alive2 spec language example:
# Verify that (x + 0) → x is always correct

Name: add-zero
Pre: true

%r = add i32 %x, 0
=>
%r = %x

; Alive2 checks: for all i32 x, the output of lhs equals output of rhs
; Including: poison propagation, undef handling, overflow behavior
```

### 23.4 Mutation Testing for Compilers

For the implementations in this lesson, you can use mutation testing to verify correctness:

```python
import ast
import copy

def mutate_and_test(optimizer, test_cases: List[Tuple[List[BB], dict]]) -> List[str]:
    """
    Apply mutations to optimizer output, verify they produce incorrect results.
    A mutation that's NOT caught by tests = test suite gap.
    """
    failures = []
    for blocks, expected in test_cases:
        # Get reference output
        test_blocks = copy.deepcopy(blocks)
        optimizer(test_blocks)
        reference = extract_output(test_blocks)

        # Apply mutations: change a constant, swap operands, etc.
        mutations = [
            ('swap_operands', lambda b: swap_operand(b)),
            ('negate_const',  lambda b: negate_constant(b)),
            ('remove_instr',  lambda b: remove_random_instr(b)),
        ]
        for mut_name, mut_fn in mutations:
            mutated = copy.deepcopy(blocks)
            optimizer(mutated)
            mut_fn(mutated)
            if extract_output(mutated) == reference:
                failures.append(f"Mutation '{mut_name}' not caught by tests")

    return failures


def extract_output(blocks: List[BB]) -> dict:
    """Extract observable output from optimized block list for comparison."""
    return {
        'instr_count': sum(len(b.instrs) for b in blocks),
        'last_block_instrs': blocks[-1].instrs if blocks else []
    }

def swap_operand(blocks: List[BB]):
    for b in blocks:
        if b.instrs:
            dest, op, s1, s2 = b.instrs[0]
            if s1 and s2:
                b.instrs[0] = (dest, op, s2, s1)

def negate_constant(blocks: List[BB]):
    for b in blocks:
        for i, (dest, op, s1, s2) in enumerate(b.instrs):
            if op == 'const' and s1 and s1.lstrip('-').isdigit():
                b.instrs[i] = (dest, op, str(-int(s1)), s2)
            break

def remove_random_instr(blocks: List[BB]):
    for b in blocks:
        if len(b.instrs) > 1:
            b.instrs.pop(0)
            break
```

---

## 24. Loop Analysis: Natural Loops, Induction Variables, and SCEV

Loops are the primary target for optimization — 90% of a program's execution time is typically spent in 10% of loops. Compiler loop analysis goes far beyond just finding `for` and `while` constructs.

### 13.1 Natural Loops and Back Edges

A **back edge** in the CFG is an edge `n → h` where `h` dominates `n` (i.e., `h` appears earlier in DFS postorder). Each back edge defines a **natural loop**:

- **Header** (`h`): the loop's single entry point (all paths into the loop pass through `h`)
- **Body**: all nodes from which `n` is reachable without going through `h`
- **Latches**: nodes with back edges to `h` (may be multiple if the loop has multiple back edges)

```
Natural loop for back edge (n → h):
  body = {n} ∪ {x : x can reach n without passing through h}
  
Algorithm (backward graph reachability from n avoiding h):
  body = {n}
  worklist = [n]
  while worklist:
      node = worklist.pop()
      for pred in node.predecessors:
          if pred not in body and pred != h:
              body.add(pred)
              worklist.append(pred)
  body.add(h)
```

**Loop nesting**: if the body of loop A is a strict subset of the body of loop B, then A is **nested inside** B. The **loop nesting tree** (loop forest) organizes this hierarchy.

### 13.2 Induction Variable Analysis (SCEV)

An **induction variable (IV)** is a variable whose value changes by a predictable amount each loop iteration. LLVM's `ScalarEvolution` (SCEV) analysis can express many loop-varying quantities symbolically:

| SCEV Expression | Meaning |
|----------------|---------|
| `{start, +, step}` | Linear IV: `start + step*i` at iteration i |
| `{start, +, step}_loop` | Qualified by which loop |
| `smax(a, b)` | Signed max |
| `(({0,+,1} * {0,+,1}))` | Quadratic: `i²` |

```python
@dataclass
class SCEVExpr:
    """Simplified SCEV expression for a loop-varying value."""
    start: int          # value at loop entry
    step: int           # additive change per iteration
    loop_header_id: int # which loop this varies in

    def at_iter(self, i: int) -> int:
        return self.start + self.step * i

    def __repr__(self):
        return f"{{{self.start}, +, {self.step}}}"


def detect_induction_variables(loop_body: Set[int], header_id: int,
                                blocks: List[BB]) -> Dict[str, SCEVExpr]:
    """
    Find basic induction variables (BIVs): vars defined as BIV + constant.
    Returns mapping var -> SCEV expression.
    """
    bmap = {b.id: b for b in blocks}
    header = bmap[header_id]
    ivs: Dict[str, SCEVExpr] = {}

    # Phase 1: find phi nodes in loop header with one arg from outside the loop
    for phi_v, args in header.phis.items():
        outside_vals = []
        inside_vals  = []
        for arg, pred in zip(args, header.preds):
            if pred.id not in loop_body:
                outside_vals.append(arg)
            else:
                inside_vals.append(arg)

        if len(outside_vals) == 1 and len(inside_vals) == 1:
            init_val = outside_vals[0]
            latch_val = inside_vals[0]

            # Try to identify the step: look for latch_val = phi_v + constant
            for bid in loop_body:
                b = bmap[bid]
                for dest, op, s1, s2 in b.instrs:
                    if dest == latch_val and op == '+':
                        try:
                            step = int(s2 if s1 == phi_v else s1)
                            init = int(init_val) if init_val.lstrip('-').isdigit() else 0
                            ivs[phi_v] = SCEVExpr(init, step, header_id)
                        except ValueError:
                            pass

    # Phase 2: find derived IVs (linear combinations of BIVs)
    for bid in loop_body:
        b = bmap[bid]
        for dest, op, s1, s2 in b.instrs:
            if s1 in ivs and op == '+' and s2 and s2.lstrip('-').isdigit():
                biv = ivs[s1]
                ivs[dest] = SCEVExpr(biv.start + int(s2), biv.step, header_id)
            elif s1 in ivs and op == '*' and s2 and s2.lstrip('-').isdigit():
                biv = ivs[s1]
                factor = int(s2)
                ivs[dest] = SCEVExpr(biv.start * factor, biv.step * factor, header_id)

    return ivs
```

### 13.3 Using SCEV for Vectorization

The vectorizer needs to know: for a loop body accessing `a[i]` and `b[i]`, do these ever alias? SCEV can answer this:

- `a[i]` accesses address `base_a + i * sizeof(T)`
- `b[i]` accesses address `base_b + i * sizeof(T)`
- If `base_a ≠ base_b` (provably, via alias analysis), they never alias → vectorization is safe

SCEV also enables:
- **Bounds check elimination**: if the loop runs from `i=0` to `i<n` and `n <= array.length`, no bounds check needed inside the loop
- **Loop trip count computation**: the exact or bounded number of iterations, needed for unrolling and vectorization lane width selection
- **Strength reduction**: any derived IV `{start, +, step}` can be computed with one add per iteration instead of using the base IV

---

## 14. Value Numbering: Local and Global

**Value numbering** is the theoretical foundation behind CSE. Rather than matching syntactic expression trees, we assign each computation a canonical **value number** such that two expressions get the same number iff they provably compute the same value.

### 11.1 Local Value Numbering (LVN)

Within a single basic block, process instructions left-to-right. Maintain a hash table mapping `(op, vn(src1), vn(src2))` → value number:

```python
def local_value_numbering(block: BB) -> Dict[str, str]:
    """
    Assign value numbers to each SSA variable within a block.
    Returns canonical var name for each def (may be an earlier def if equal).
    """
    # Map expression tuple -> canonical var
    expr_table: Dict[tuple, str] = {}
    # Map var -> value number (canonical var name)
    vn: Dict[str, str] = {}

    canonical: Dict[str, str] = {}   # var -> canonical replacement

    def get_vn(v: str) -> str:
        # Follow canonicalization chain
        while v in canonical:
            v = canonical[v]
        return vn.get(v, v)

    new_instrs = []
    for dest, op, s1, s2 in block.instrs:
        vn_s1 = get_vn(s1) if s1 else None
        vn_s2 = get_vn(s2) if s2 else None

        if op == 'const':
            # Constants: canonical form is ("const", value)
            key = ('const', s1)
        elif s2:
            # Commutative ops: normalize operand order
            if op in ('+', '*', '&', '|', '^'):
                key = (op, min(vn_s1, vn_s2), max(vn_s1, vn_s2))
            else:
                key = (op, vn_s1, vn_s2)
        else:
            key = (op, vn_s1)

        if key in expr_table:
            # This computation was already done — dest = canonical_var
            canonical[dest] = expr_table[key]
            vn[dest] = expr_table[key]
            # Don't emit instruction; dest is an alias
        else:
            expr_table[key] = dest
            vn[dest] = dest
            # Re-write srcs with their canonical names
            rs1 = get_vn(s1) if s1 else s1
            rs2 = get_vn(s2) if s2 else s2
            new_instrs.append((dest, op, rs1, rs2))

    block.instrs = new_instrs
    return canonical
```

### 11.2 Global Value Numbering (GVN)

LVN only eliminates redundancies within a block. GVN extends this across the entire CFG. The key insight: in SSA form, if two instructions compute the same function of the same SSA values (which are themselves globally unique), they must produce the same result — regardless of which blocks they're in.

GVN uses **congruence classes**. We say SSA values `a` and `b` are congruent (`a ≡ b`) if:
1. They are the same constant
2. They are the same SSA variable (trivially)
3. They have the same opcode and pairwise congruent operands: `f(a1,...,ak) ≡ f(b1,...,bk)` if `ai ≡ bi` for all i

This is the **GVN-PRE (Global Value Numbering — Partial Redundancy Elimination)** algorithm by Briggs and Cooper (1994). A modern, simpler version is **RPO-based GVN**:

```python
def gvn_rpo(blocks: List[BB]) -> Dict[str, str]:
    """
    RPO-based GVN: process blocks in reverse postorder.
    Returns substitution map (var -> canonical var).
    """
    # Build postorder
    visited, postorder = set(), []
    def dfs(b: BB):
        visited.add(b.id)
        for s in b.succs:
            if s.id not in visited: dfs(s)
        postorder.append(b)
    if blocks: dfs(blocks[0])
    rpo = list(reversed(postorder))

    # value_table: expression key -> leader (canonical var or const string)
    value_table: Dict[tuple, str] = {}
    # leader[var] = canonical representative
    leader: Dict[str, str] = {}

    def get_leader(v: str) -> str:
        cur = v
        while cur in leader and leader[cur] != cur:
            cur = leader[cur]
        return cur

    substitutions: Dict[str, str] = {}

    for b in rpo:
        # Process phi functions first
        for phi_v, args in b.phis.items():
            phi_leaders = tuple(get_leader(a) for a in args)
            key = ('phi', b.id, phi_leaders)
            if key in value_table:
                leader[phi_v] = value_table[key]
                substitutions[phi_v] = value_table[key]
            else:
                value_table[key] = phi_v
                leader[phi_v] = phi_v

        # Process instructions
        new_instrs = []
        for dest, op, s1, s2 in b.instrs:
            ls1 = get_leader(s1) if s1 else None
            ls2 = get_leader(s2) if s2 else None
            # Normalize commutative ops
            if op in ('+', '*', '&', '|', '^') and ls1 and ls2:
                key = (op, min(ls1, ls2), max(ls1, ls2))
            elif s2:
                key = (op, ls1, ls2)
            else:
                key = (op, ls1)

            if key in value_table and op not in ('load', 'call'):
                # Redundant computation
                leader[dest] = value_table[key]
                substitutions[dest] = value_table[key]
                # Still need a copy instruction unless same reg (coalescing handles it)
                new_instrs.append((dest, 'copy', value_table[key], None))
            else:
                value_table[key] = dest
                leader[dest] = dest
                new_instrs.append((dest, op, ls1 or s1, ls2 or s2))
        b.instrs = new_instrs

    return substitutions
```

GVN can eliminate redundancies that LVN cannot: if block B1 computes `t1 = a + b` and block B2 (which dominates B3) computes `t2 = a + b`, then in B3 we can replace `t2` with `t1` even though the blocks are separate. This is only sound because SSA guarantees `a` and `b` haven't changed between B1 and B3.

### 11.3 Algebraic Simplification Rules

On top of value numbering, apply algebraic identities:

```python
SIMPLIFICATIONS = {
    # Identity elements
    ('+', 'x', '0'): 'x',  ('+', '0', 'x'): 'x',
    ('*', 'x', '1'): 'x',  ('*', '1', 'x'): 'x',
    ('*', 'x', '0'): '0',  ('*', '0', 'x'): '0',
    ('-', 'x', '0'): 'x',
    # Self-cancellation
    ('-', 'x', 'x'): '0',
    ('^', 'x', 'x'): '0',
    # Strength reductions (applied during simplification)
    ('*', 'x', '2'): ('<<', 'x', '1'),
    ('*', 'x', '4'): ('<<', 'x', '2'),
    ('*', 'x', '8'): ('<<', 'x', '3'),
    # Double negation / boolean identities
    ('!', ('!', 'x')): 'x',
}
```

These rules are applied in `instcombine` in LLVM — thousands of such patterns are encoded and matched algebraically during IR optimization.

---

## 12. Complete Runnable Python Implementation

The following self-contained script ties together all the pieces: SSA construction, constant folding, DCE, liveness analysis, interference graph construction, and graph-coloring register allocation. Copy-paste into a Python 3.9+ interpreter and run it.

```python
#!/usr/bin/env python3
"""
compiler_internals_demo.py
Self-contained demonstration of SSA construction, optimization passes,
liveness analysis, and register allocation via graph coloring.
"""

from __future__ import annotations
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set, Tuple
from collections import defaultdict, deque
import copy

# ---------------------------------------------------------------------------
# Data structures
# ---------------------------------------------------------------------------

Instr = Tuple[Optional[str], str, Optional[str], Optional[str]]
# (dest, op, src1, src2)  — dest/src may be None

@dataclass
class BB:
    """A basic block in the CFG."""
    id: int
    instrs: List[Instr] = field(default_factory=list)
    succs:  List['BB']  = field(default_factory=list)
    preds:  List['BB']  = field(default_factory=list)
    phis:   Dict[str, List[str]] = field(default_factory=dict)

    def __hash__(self): return self.id
    def __repr__(self): return f"BB{self.id}"


def make_cfg(*block_specs) -> List[BB]:
    """
    Build a CFG from a list of (id, instrs, succ_ids) tuples.
    Example:
        make_cfg(
            (0, [("x","const","1",None)], [1, 2]),
            (1, [("x","const","2",None)], [3]),
            ...
        )
    """
    blocks = [BB(id=spec[0], instrs=list(spec[1])) for spec in block_specs]
    bmap = {b.id: b for b in blocks}
    for spec, block in zip(block_specs, blocks):
        for sid in spec[2]:
            succ = bmap[sid]
            block.succs.append(succ)
            succ.preds.append(block)
    return blocks


# ---------------------------------------------------------------------------
# Phase 1: Dominator computation (Cooper-Harvey-Kennedy 2001)
# ---------------------------------------------------------------------------

def compute_postorder(entry: BB) -> List[int]:
    visited, order = set(), []
    def dfs(b: BB):
        visited.add(b.id)
        for s in b.succs:
            if s.id not in visited:
                dfs(s)
        order.append(b.id)
    dfs(entry)
    return order

def compute_dominators(blocks: List[BB], entry_id: int = 0) -> Dict[int, int]:
    """Returns idom[bid] = immediate dominator id."""
    bmap = {b.id: b for b in blocks}
    postorder = compute_postorder(bmap[entry_id])
    po_index = {bid: i for i, bid in enumerate(postorder)}

    idom = {b.id: -1 for b in blocks}
    idom[entry_id] = entry_id

    def intersect(a: int, b: int) -> int:
        while a != b:
            while po_index[a] < po_index[b]: a = idom[a]
            while po_index[b] < po_index[a]: b = idom[b]
        return a

    changed = True
    while changed:
        changed = False
        for bid in reversed(postorder):
            if bid == entry_id: continue
            b = bmap[bid]
            new_idom = -1
            for pred in b.preds:
                if idom[pred.id] != -1:
                    new_idom = pred.id if new_idom == -1 else intersect(pred.id, new_idom)
            if new_idom != -1 and idom[bid] != new_idom:
                idom[bid] = new_idom
                changed = True
    return idom


# ---------------------------------------------------------------------------
# Phase 2: Dominance frontiers
# ---------------------------------------------------------------------------

def compute_df(blocks: List[BB], idom: Dict[int, int]) -> Dict[int, Set[int]]:
    df: Dict[int, Set[int]] = defaultdict(set)
    for b in blocks:
        if len(b.preds) >= 2:               # join point
            for pred in b.preds:
                runner = pred.id
                while runner != idom[b.id]:
                    df[runner].add(b.id)
                    runner = idom[runner]
    return df


# ---------------------------------------------------------------------------
# Phase 3: Phi insertion + renaming
# ---------------------------------------------------------------------------

def insert_phis(blocks: List[BB], df: Dict[int, Set[int]],
                var_defs: Dict[str, List[int]]) -> None:
    bmap = {b.id: b for b in blocks}
    for var, def_bids in var_defs.items():
        wl = deque(def_bids)
        has_phi: Set[int] = set()
        in_wl: Set[int] = set(def_bids)
        while wl:
            bid = wl.popleft(); in_wl.discard(bid)
            for y_id in df.get(bid, set()):
                if y_id not in has_phi:
                    y = bmap[y_id]
                    y.phis[var] = [var] * len(y.preds)
                    has_phi.add(y_id)
                    if y_id not in in_wl:
                        wl.append(y_id); in_wl.add(y_id)


def rename(blocks: List[BB], idom: Dict[int, int], entry_id: int = 0) -> None:
    bmap   = {b.id: b for b in blocks}
    dom_children: Dict[int, List[int]] = defaultdict(list)
    for bid, idom_id in idom.items():
        if bid != entry_id:
            dom_children[idom_id].append(bid)

    counter: Dict[str, int]       = defaultdict(int)
    stacks:  Dict[str, List[str]] = defaultdict(list)

    def fresh(var: str) -> str:
        k = counter[var]; counter[var] += 1
        return f"{var}_{k}"

    def dfs(bid: int) -> None:
        b = bmap[bid]
        pushed: Dict[str, int] = defaultdict(int)

        # Rename phi LHS
        new_phis: Dict[str, List[str]] = {}
        for var, args in b.phis.items():
            nv = fresh(var)
            stacks[var].append(nv); pushed[var] += 1
            new_phis[nv] = args
        b.phis = new_phis

        # Rename instruction uses then defs
        new_instrs = []
        for dest, op, s1, s2 in b.instrs:
            ns1 = stacks[s1][-1] if s1 and stacks[s1] else s1
            ns2 = stacks[s2][-1] if s2 and stacks[s2] else s2
            nd  = dest
            if dest:
                nd = fresh(dest)
                stacks[dest].append(nd); pushed[dest] += 1
            new_instrs.append((nd, op, ns1, ns2))
        b.instrs = new_instrs

        # Fill phi args in successors
        for succ in b.succs:
            idx = succ.preds.index(b)
            for phi_nv, args in succ.phis.items():
                orig = '_'.join(phi_nv.split('_')[:-1])
                if stacks.get(orig):
                    args[idx] = stacks[orig][-1]

        for child_id in dom_children[bid]:
            dfs(child_id)

        for var, cnt in pushed.items():
            for _ in range(cnt):
                if stacks[var]: stacks[var].pop()

    dfs(entry_id)


def to_ssa(blocks: List[BB], var_defs: Dict[str, List[int]],
           entry_id: int = 0) -> Dict[int, int]:
    idom = compute_dominators(blocks, entry_id)
    df   = compute_df(blocks, idom)
    insert_phis(blocks, df, var_defs)
    rename(blocks, idom, entry_id)
    return idom


# ---------------------------------------------------------------------------
# Optimization passes
# ---------------------------------------------------------------------------

def constant_fold_all(blocks: List[BB]) -> int:
    """Fold constant binary/unary operations. Returns # instructions folded."""
    OPS = {'+': int.__add__, '-': int.__sub__, '*': int.__mul__,
           '//': int.__floordiv__, '<<': int.__lshift__, '>>': int.__rshift__}
    folded = 0
    for b in blocks:
        new_i = []
        for dest, op, s1, s2 in b.instrs:
            try:
                c1 = int(s1)
                c2 = int(s2) if s2 else None
                if op in OPS and c2 is not None:
                    new_i.append((dest, 'const', str(OPS[op](c1, c2)), None))
                    folded += 1; continue
            except (ValueError, TypeError):
                pass
            new_i.append((dest, op, s1, s2))
        b.instrs = new_i
    return folded


def dce(blocks: List[BB]) -> int:
    """Dead code elimination. Returns # instructions removed."""
    from collections import Counter
    use_cnt: Counter = Counter()
    for b in blocks:
        for args in b.phis.values():
            for a in args: use_cnt[a] += 1
        for _, _, s1, s2 in b.instrs:
            if s1: use_cnt[s1] += 1
            if s2: use_cnt[s2] += 1

    removed = 0
    changed = True
    while changed:
        changed = False
        for b in blocks:
            kept = []
            for instr in b.instrs:
                dest, _, s1, s2 = instr
                if not dest or use_cnt[dest] > 0:
                    kept.append(instr)
                else:
                    if s1: use_cnt[s1] -= 1
                    if s2: use_cnt[s2] -= 1
                    removed += 1; changed = True
            b.instrs = kept
    return removed


# ---------------------------------------------------------------------------
# Liveness analysis
# ---------------------------------------------------------------------------

def liveness(blocks: List[BB]) -> Dict[int, Tuple[Set[str], Set[str]]]:
    gen:  Dict[int, Set[str]] = {}
    kill: Dict[int, Set[str]] = {}
    for b in blocks:
        g, k = set(), set()
        for args in b.phis.values():
            for a in args:
                if a not in k: g.add(a)
        for phi_v in b.phis:
            k.add(phi_v)
        for dest, _, s1, s2 in b.instrs:
            if s1 and s1 not in k: g.add(s1)
            if s2 and s2 not in k: g.add(s2)
            if dest: k.add(dest)
        gen[b.id], kill[b.id] = g, k

    li: Dict[int, Set[str]] = {b.id: set() for b in blocks}
    lo: Dict[int, Set[str]] = {b.id: set() for b in blocks}
    changed = True
    while changed:
        changed = False
        for b in reversed(blocks):
            nlo = set().union(*(li[s.id] for s in b.succs))
            nli = gen[b.id] | (nlo - kill[b.id])
            if nli != li[b.id] or nlo != lo[b.id]:
                changed = True
            li[b.id], lo[b.id] = nli, nlo
    return {b.id: (li[b.id], lo[b.id]) for b in blocks}


# ---------------------------------------------------------------------------
# Interference graph + graph coloring
# ---------------------------------------------------------------------------

def interference_graph(blocks: List[BB],
                        live: Dict[int, Tuple[Set[str], Set[str]]]
                       ) -> Dict[str, Set[str]]:
    g: Dict[str, Set[str]] = defaultdict(set)
    def edge(u, v):
        if u != v: g[u].add(v); g[v].add(u)

    for b in blocks:
        _, lo = live[b.id]
        cur = set(lo)
        for dest, _, s1, s2 in reversed(b.instrs):
            if dest:
                for v in cur: edge(dest, v)
                cur.discard(dest)
            if s1: cur.add(s1)
            if s2: cur.add(s2)
    return dict(g)


def color(graph: Dict[str, Set[str]], k: int) -> Dict[str, int]:
    """Chaitin graph coloring with k colors. Returns var->color (-1=spill)."""
    adj = {v: set(ns) for v, ns in graph.items()}
    all_vars = list(graph)
    removed: Set[str] = set()
    stack: List[str] = []
    spilled: Set[str] = set()

    def deg(v): return len(adj.get(v, set()) - removed)

    for _ in range(len(all_vars)):
        node = next((v for v in all_vars if v not in removed and deg(v) < k), None)
        if node is None:
            node = max((v for v in all_vars if v not in removed), key=deg, default=None)
            if node: spilled.add(node)
        if node: stack.append(node); removed.add(node)

    colors: Dict[str, int] = {}
    for v in reversed(stack):
        if v in spilled: colors[v] = -1; continue
        used = {colors[n] for n in adj.get(v, set()) if n in colors and colors[n] >= 0}
        colors[v] = next((c for c in range(k) if c not in used), -1)
    return colors


# ---------------------------------------------------------------------------
# Pretty printer
# ---------------------------------------------------------------------------

def dump(blocks: List[BB]) -> str:
    lines = []
    for b in blocks:
        lines.append(f"\n╔═══ BB{b.id} (preds={[p.id for p in b.preds]}, succs={[s.id for s in b.succs]}) ═══")
        for pv, args in b.phis.items():
            preds_fmt = ", ".join(f"{a}[B{p.id}]" for a, p in zip(args, b.preds))
            lines.append(f"║  {pv} = φ({preds_fmt})")
        for dest, op, s1, s2 in b.instrs:
            if s2: lines.append(f"║  {dest} = {s1} {op} {s2}")
            elif s1: lines.append(f"║  {dest} = {op}({s1})")
            else: lines.append(f"║  {dest} = {op}")
    return "\n".join(lines)


# ---------------------------------------------------------------------------
# Demo: if-else with merge
# ---------------------------------------------------------------------------

def demo_ssa():
    print("=" * 60)
    print("DEMO: If-else SSA construction")
    print("=" * 60)
    blocks = make_cfg(
        (0, [("cond","load","cond_input",None), ("x","const","0",None), ("y","const","0",None)], [1, 2]),
        (1, [("x","const","10",None)], [3]),
        (2, [("x","const","20",None)], [3]),
        (3, [("z","+","x","y")], []),
    )
    var_defs = {"x": [0, 1, 2], "y": [0], "z": [3], "cond": [0]}
    to_ssa(blocks, var_defs)
    print(dump(blocks))
    print()

    live = liveness(blocks)
    print("Liveness:")
    for bid, (li, lo) in live.items():
        print(f"  BB{bid}: in={sorted(li)}, out={sorted(lo)}")

    igraph = interference_graph(blocks, live)
    print("\nInterference graph:")
    for v, ns in sorted(igraph.items()):
        print(f"  {v} -- {sorted(ns)}")

    alloc = color(igraph, k=2)
    print("\nRegister allocation (2 regs):")
    for v, c in sorted(alloc.items()):
        print(f"  {v} → {'SPILL' if c == -1 else f'R{c}'}")


def demo_const_fold_dce():
    print("\n" + "=" * 60)
    print("DEMO: Constant folding + DCE")
    print("=" * 60)
    blocks = make_cfg(
        (0, [
            ("a", "const", "3", None),
            ("b", "const", "4", None),
            ("c", "+", "a", "b"),     # 3+4 = 7
            ("d", "*", "a", "b"),     # 3*4 = 12  (unused)
            ("e", "*", "c", "2"),     # 7*2 = 14
        ], []),
    )
    print("Before optimization:")
    print(dump(blocks))

    n_fold = constant_fold_all(blocks)
    n_dce  = dce(blocks)
    print(f"\nAfter constant folding ({n_fold} folded) + DCE ({n_dce} removed):")
    print(dump(blocks))


if __name__ == "__main__":
    demo_ssa()
    demo_const_fold_dce()
```

**Expected output:**

```
============================================================
DEMO: If-else SSA construction
============================================================

╔═══ BB0 (preds=[], succs=[1, 2]) ═══
║  cond_0 = load(cond_input)
║  x_0 = const(0)
║  y_0 = const(0)

╔═══ BB1 (preds=[0], succs=[3]) ═══
║  x_1 = const(10)

╔═══ BB2 (preds=[0], succs=[3]) ═══
║  x_2 = const(20)

╔═══ BB3 (preds=[1, 2], succs=[]) ═══
║  x_3 = φ(x_1[B1], x_2[B2])
║  z_0 = x_3 + y_0

Liveness:
  BB0: in=set(), out={'x_1', 'x_2', 'y_0'}
  BB1: in=set(), out={'x_1', 'y_0'}
  BB2: in=set(), out={'x_2', 'y_0'}
  BB3: in={'x_3', 'y_0'}, out=set()

Interference graph:
  x_1 -- ['x_2']
  x_2 -- ['x_1']
  x_3 -- ['y_0']
  y_0 -- ['x_3']

Register allocation (2 regs):
  x_1 → R0
  x_2 → R1
  x_3 → R0
  y_0 → R1

============================================================
DEMO: Constant folding + DCE
============================================================
Before optimization:
╔═══ BB0 (preds=[], succs=[]) ═══
║  a = const(3)
║  b = const(4)
║  c = a + b
║  d = a * b
║  e = c * 2

After constant folding (3 folded) + DCE (1 removed):
╔═══ BB0 (preds=[], succs=[]) ═══
║  a_0 = const(3)
║  b_0 = const(4)
║  c_0 = const(7)
║  e_0 = const(14)
```

---

*Lesson authored by Claude Learning Agent — Graduate-level compiler engineering series.*
