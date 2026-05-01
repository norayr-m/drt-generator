# Generator, Cell Simulator, Scanner — a sparse matrix-vector trio for bio digital twins

> **Humble disclaimer.** Amateur engineering project. We are not HPC professionals and make no competitive claims. The work is openly in progress; this paper is honest about what is solved, what is wired but not yet validated against biology, and where the framing is aspirational. Errors are likely.

GPLv3.

---

## Contents

- [What — three repos, one workflow](#what)
- [Why — advantages over standard tools](#why)
- [How — bio digital twins, end to end](#how)
- [The math, kept tight](#math)
- [What is not in the paper](#not)
- [References and acknowledgments](#refs)

---

## <a id="what"></a>What — three repos, one workflow

Three small browser demos, taken together, walk through one complete computational pattern: produce a signal, propagate it through a population of locally-connected receivers, then verify whether the result can be reconstructed back to the original. Every step is a sparse matrix-vector multiplication; the trio stays inside that single arithmetic.

- **[drt-generator](https://github.com/norayr-m/drt-generator)** produces the sparse projection matrices. A clock tick generates an integer index, a one-hot row-pick selects a row of a routing matrix, the row defines a sparse weight vector, an element-wise activation fires, a small dense propagation finishes the per-tick output. The seven-column pipeline is the visible per-tick mat-vec chain. The same arithmetic is drawable as a pipeline or as a deep graph; the demo shows both.
- **[drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator)** hosts the receivers. Each cell is a small node on a graph topology — a cellular automaton with local rules over its neighbours' edges. The signal produced by the generator arrives at each cell as a per-tick projection; the cell's local rules combine the projection with its own state to produce the next-tick state. Distributed local transitions; emergent global behaviour; the public tagline reads *DRT as biology*. Each cell, in this paper's language, is a *receptor*.
- **[drt-scanner](https://github.com/norayr-m/drt-scanner)** runs the transpose pass. It takes a target signal and projects it back through the transposes of the same matrices the generator used, in reverse order. Where the reverse pass agrees with the forward pass at every column, the chain is well-conditioned for inversion at that configuration; where it diverges, information has been lost — the forward map was not invertible at that state, or numerical conditioning broke down.

Read in order, the trio is *generator → cell simulator → scanner*. The generator emits, the cell field receives, the scanner verifies. End to end is forward-pass plus backward-pass through the same matrices.

That is one mathematical workflow expressed across three public repos. The next two sections are about why this shape matters and how it lands on a real applied target.

---

## <a id="why"></a>Why — advantages over standard tools

The trio is one solution among several to the same general question: *how do you simulate a population of locally-connected receivers responding to a propagating signal, verify the result, and do it on a laptop?* Three standard tools have answered that question for decades. Each has an honest cost.

### Versus molecular dynamics

Molecular dynamics simulates atoms-by-atoms under explicit physical force fields. It is the gold standard for biophysical accuracy and the most expensive tool by far. A microsecond of MD on a million-atom system runs for days on a GPU cluster. The cost is not a code optimisation issue; the cost is the price of explicit physics at full resolution.

The trio sits at a different level of abstraction. The generator's matrices are not physical force fields; they are coarse-grained projections, hand-designed or extracted from experimental data. The cell simulator's local rules are not Newtonian motion; they are local response models, often a few coefficients per cell. The result is many orders of magnitude cheaper — a million-cell simulation runs at interactive frame rates on a laptop — at the cost of explicit accuracy guarantees. The trio is right when coarse-grained signal propagation through a topologically-correct receiver field is the question; MD is right when atomistic accuracy is what matters. The two tools are not in competition; they answer different questions.

### Versus graph neural networks

GNNs are also sparse-mat-vec on a graph. The architectural similarity to the cell simulator is real, and we say so explicitly: the demo's graph view of the seven-column generator IS a small neural network, with weights that happen to be hand-set instead of learned. The difference is in where the weights come from.

A GNN ships its trained weights as opaque tensors. To reproduce the GNN's output you need the trained model file, you cannot inspect what each layer "does" without dedicated interpretability tooling, and you cannot easily swap in a different biological hypothesis without retraining. The trio's matrices are explicit, hand-set, and inspectable: the routing matrix of the generator is a literal CSV, the cell simulator's local rules fit on a slide, the scanner's transpose is a transpose. Same arithmetic; entirely different ergonomics for the model-builder.

The trio is right when you want to see the matrix, change one entry, and immediately watch the propagation downstream. A GNN is right when no closed-form mapping exists from inputs to outputs and the model has to be learned from data. Again — not in competition; complementary.

### Versus finite-element methods

FEM is the standard tool for spatially-distributed PDE problems. It is mature, robust, and has decades of optimised libraries. It also has a known scaling shape: each element carries a small dense matrix; the global system is sparse but the per-element work is dense. At biological scales — tissue volumes with hundreds of millions of cells — the per-element constant becomes the bottleneck, even though the global matrix is sparse.

The trio's per-cell work is, by construction, a sparse mat-vec at `O(nnz)` not a dense `O(d^2)`. There is no per-element dense block. The cost-per-cell is bounded by the cell's own degree in the graph, not by an embedded dimension. For a hex-grid or a contact graph with average degree under 10, that is a single-digit number of multiply-adds per cell per tick. The trio scales linearly to the cell count; FEM scales linearly to (cells × per-element-dense-cost).

The trio is right when a graph topology accurately captures the connectivity of the receiver field. FEM is right when the underlying PDE has continuous physics that needs proper element-wise integration. A tissue connectivity graph fits the trio better; a fluid-dynamics field fits FEM better.

### Versus classical numerical libraries

Sparse-mat-vec libraries are everywhere: BLAS, MKL, cuSPARSE, and dozens of academic packages. The trio does not compete with these libraries; it is *built on top of* the same kernel. The contribution is not a faster mat-vec. The contribution is the assembled workflow — generator + receptor field + scanner, three repos that snap together to walk a complete forward-and-backward propagation cycle on a small enough example that the whole thing fits in a browser tab. The mat-vec is canonical; the assembly is the part you can copy.

---

## <a id="how"></a>How — bio digital twins, end to end

A bio digital twin is a computational model of a specific biological system — a heart chamber, a hepatic lobule, a section of cortex — wired tightly enough that it can predict the system's response to a stimulus before the stimulus is applied to the real tissue. The applied target for the trio is the construction of bio digital twins at tissue scale: hundreds of thousands to a few million cells, each with a small local rule, all responding to a signal that propagates through the connectivity.

The end-to-end mapping from the trio onto a bio digital twin:

### The generator side: signals as sparse projections

Each "signal wave" the digital twin receives — a hormone bolus, an electrical pulse, a drug dose — projects onto the receiver field as a sparse, time-varying weight vector. Most cells in the field are not directly affected at a given tick; the projection is sparse by physics. The generator's seven-column pipeline is the visible per-tick example of how that projection is constructed: a clock tick selects a row of the routing matrix, the row's nonzero entries are the cells that *will* respond at this tick. The routing matrix itself is the digital twin's encoding of which cells receive which inputs.

For a real tissue model, the routing matrix is filled from experimental data — receptor density per cell, anatomical constraints on signal arrival, distance-from-source decay. The generator demo runs this on a small synthetic example for clarity; the same machinery accepts a routing matrix at any scale.

### The cell simulator side: receptors as graph CA

A receptor cell, in this trio, is exactly what the [drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator) repo calls it: a node on a graph topology with local rules over its neighbours' edges. *DRT as biology* is the public tagline. Each cell carries a small state vector, applies a per-tick rule over (its own previous state, the projection arriving from the generator, the states of its connected neighbours), and produces the next-tick state.

For a bio digital twin, the cell's local rule is the receptor's response model: a Hill function, a Michaelis-Menten saturation, a small ODE step. The graph's edges are the actual connectivity of the tissue: gap junctions in cardiac muscle, sinusoid topology in liver, synaptic connectivity in cortex. The cell simulator's existing demo shows the abstract case (cellular automata on graph topology, emergent global behaviour from distributed local transitions); the bio digital twin instantiates that abstraction with biological connectivity and biological rules.

The cost-per-cell-per-tick is the per-cell sparse mat-vec: the cell reads its own state, the projection from the generator, and its neighbours' states; combines them via the local rule; writes the next-tick state. For a graph with average degree `k`, the per-cell work is order `k`. For a tissue model with `N` cells, the total work is order `kN` per tick, which scales linearly. This is the property that makes laptop-scale digital twins achievable.

### The scanner side: reconstruction fidelity as inversion check

A good digital twin is not just one that matches average behaviour; it is one whose forward map is well-enough conditioned that an inverse can be applied. If the twin maps a stimulus at time `t` to a tissue state at time `t+1`, the question *what stimulus produced this tissue state?* should have a recoverable answer when the forward map is locally invertible. The scanner demonstrates that inversion check directly: it runs the same matrices in reverse via their transposes, projects a target tissue state back through the chain, and reports column-by-column whether the reverse pass agrees with the forward pass.

For the bio digital twin user, this is exactly the question of *can I attribute a measured tissue state to a known stimulus?* — the inverse-modelling question that medical imaging and biomedical inverse problems both ask. The scanner does not solve the full inverse problem; it does something more modest and more honest. It tells you *at which step in the forward chain the inversion fails, if it does*. A column that lights up green at one configuration but red at another is the digital twin telling you which regions of input space it is reliable for, and which regions it is not. That information is more useful than a single yes/no.

### End-to-end: a worked applied loop

A complete loop on a bio digital twin, expressed in the trio's three repos:

1. **Generator round.** A signal projection is constructed from the model's stimulus parameters. The seven-column generator produces, per tick, the sparse weight vector of which cells receive the input at that tick. For a tissue model, this is the propagation of a hormone bolus through the vascular topology, or the spread of an electrical pulse through gap junctions.
2. **Cell simulator update.** The receptor field receives the projection. Each cell applies its local rule, combining the projection with its current state and the states of its neighbours, to produce the next-tick state. After many ticks, the field has settled into a response — the predicted tissue state.
3. **Scanner pass.** The predicted tissue state is run back through the transposes of the same matrices. Column-by-column agreement is reported. Where the chain reconstructs back to the original stimulus, the model is locally invertible at that configuration; that region of input space is one where the digital twin can attribute observed tissue states to causes. Where the chain diverges, information has been lost during forward propagation; that region is one where the twin can predict but not retrodict.

That loop runs end-to-end on a laptop. The math is sparse mat-vec the whole way through. The applied target is digital twins at tissue scale.

---

## <a id="math"></a>The math, kept tight

The trio's mathematics fits in a few formulas. Each formula gets a one-sentence plain-language gloss in the surrounding text.

The generator's per-tick output:

```
y_t = M · σ( W · x_t )    with    x_t = sparse_select( S, p_t )    and    p_t = LUT( t )
```

> The clock tick `t` is sent through a lookup table to a pattern identifier `p_t`. A one-hot selection matrix `S` picks the corresponding row, producing a sparse weight vector `x_t`. An element-wise nonlinearity `σ` and a small dense propagation matrix `W` produce an intermediate vector. An output mixing matrix `M` produces the per-tick observable.

The cell simulator's per-cell, per-tick update on a graph with cell `i` and neighbour set `N(i)`:

```
s_{t+1, i} = R( s_{t, i} , y_t , { s_{t, j} : j ∈ N(i) } )
```

> Cell `i`'s state at tick `t+1` is a function `R` of (its own prior state, the per-tick projection arriving from the generator, the prior states of its graph neighbours). For each cell, the work is bounded by the cell's degree; for a graph with average degree `k` and `N` cells, the total work per tick is order `k · N`.

The scanner's inversion check, running in reverse:

```
x̂_t = S^T · σ^{-1}( W^T · M^T · y_t )
```

> Take the per-tick observable `y_t`. Apply the transpose of the output mixing, the inverse of the activation, the transpose of the dense propagation, and the transpose of the selection. The result `x̂_t` is the reverse pass's reconstruction of the original sparse weight vector. Agreement (`x̂_t ≈ x_t`) means the forward chain is locally invertible at this configuration; divergence flags information loss.

Three matrices, three transposes, two functions, one local rule. The whole trio fits in those four lines.

---

## <a id="not"></a>What is not in the paper

The honest list of what this paper deliberately does not claim:

- **Validation against a specific biological dataset.** The trio is a substrate, not a finished bio digital twin. The applied story above is the use-case framing; concrete validation runs against published tissue data are future work and would land in repositories of their own.
- **A norm-growth inequality.** Earlier prose around this family of demos asserted, for the aggregate of many partial views, that the aggregate had bigger norm than the original signal. That inequality has been retracted in the underlying v0.1 draft. Nothing in this paper depends on it. The trio's claims are mechanical (these matrices, these transposes, this composition) not analytical.
- **A general inversion theorem.** The scanner's column-by-column inversion check is a *diagnostic*, not a guarantee. It tells you at which step the inversion fails on a given configuration; it does not tell you that any specific input region is globally invertible. Bio digital twin users who need rigorous inverse-problem theory will need additional machinery (Tikhonov regularisation, posterior sampling, etc.) layered on top.
- **A speedup claim against MD or GNN or FEM.** The "Why" section above frames the trio's place against those tools by their cost shapes and use-case fits. It does not benchmark a specific case. Speedup numbers, when claimed, will live in repositories that ship reproducible benchmarks; this paper is the framing, not the timing.

---

## <a id="refs"></a>References and acknowledgments

- **[drt-generator](https://github.com/norayr-m/drt-generator)** — sparse matrix-vector multiplication, drawn out. Source repo with the live demo and the [presentation deck](https://norayr-m.github.io/drt-generator/presentation.html).
- **[drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator)** — cellular automata on graph topology. The receptor-field demo.
- **[drt-scanner](https://github.com/norayr-m/drt-scanner)** — the transpose pass, the inversion-fidelity diagnostic.
- **Holling, C. S. (1959).** *Some characteristics of simple types of predation and parasitism.* The general functional-form pattern referenced when discussing local response models in §How.
- **Distributed Reconstruction work, v0.1 in preparation by N. Matevosyan and A. Petrosyan.** The paper that frames the family of demos this trio belongs to, including the retraction noted in §What is not in the paper.

Visualization and code co-authored with Claude (Anthropic).

---

> **Humble disclaimer.** Amateur engineering project. We are not HPC professionals and make no competitive claims. Numbers speak; ego doesn't. Errors likely.

GPLv3. See [`LICENSE`](LICENSE).
