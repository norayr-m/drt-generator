# DRT Generator — the 7-Column Pipeline

> This is an amateur engineering project. We are not HPC professionals and make no competitive claims. Errors likely.

A browser demo that shows the same computation in two equivalent shapes side-by-side: as a rigid seven-column logic pipeline, and as a deep, fully-connected graph. The point is to make the equivalence visually obvious — that "discrete rule-based logic" and "deep network" can be the same object drawn two different ways.

**[Live demo →](https://norayr-m.github.io/drt-generator/)**

## What it does

There are seven columns. From left to right:

1. **Phase** — a rotating clock ticks out a phase value $0, 1, 2, \ldots$
2. **ID LUT** — a lookup table maps the phase to a pattern ID.
3. **Routing matrix** — the pattern ID picks one row.
4. **Weighted nodes** — that row extrudes a set of literal "dice" with weights.
5. **Activation** — non-linear functions (ReLU, sigmoid, tanh) applied to the weighted nodes.
6. **Hidden layers** — the activations propagate through a small feed-forward network.
7. **Wave assembly** — the network output is reassembled into the harmonic frequencies of the demo's audio.

Phase 1 of the demo shows the columns as discrete boxes connected by arrows — programmer's flowchart shape. Phase 2 then redraws the same computation as a 7-layer deep graph, where every box becomes a node, every arrow becomes a weighted edge, and the boundary between "rule-based" and "neural" disappears.

The demo also has a counter-clockwise phase extractor view (Phase Tick → Pattern → Row → Vector Weights running backwards), mute / unmute audio, and a pause control.

## Why it matters (DRT angle)

This is the **operational core** of the Distributed Reconstruction sketch and the one Norayr has named as needing the most detailed page. It illustrates two things at once:

1. **The encoder shape.** The seven columns are the structural skeleton of an "encoder" in the framework — phase to pattern, pattern to weights, weights to output. Each column is a small projection.
2. **The equivalence.** Showing that the rigid 7-column pipeline and the 7-layer deep graph compute the same thing makes the framework's point concrete: a "pipeline" and a "neural network" are different drawings of the same object. Once you see them side-by-side, the question of "is it logic or is it learning" stops being interesting; the architecture is the same either way.

The honest current claim from the v0.1 draft is conditional and weaker than earlier prose suggested: the aggregate of distributed observer completions exposes structural features not accessible from the original signal alone, under a bounded computational budget and four explicit hypotheses. This demo is a visual playground for the encoder side of that picture; the dual scanner repo is the playground for inspecting the output.

## How to run

Single HTML, open in a browser:

```
git clone https://github.com/norayr-m/drt-generator.git
open drt-generator/index.html
```

No build step. No server. No external dependencies. Audio is opt-in (UNMUTE AUDIO button).

## The trio — generator, cell simulator, scanner

This repo is one of three that snap together into one workflow:

1. **drt-generator** (this repo) — produces sparse projection matrices from a clock tick.
2. **[drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator)** — receives the projections as cellular automata on graph topology. Each cell is a *receptor* with local rules over its neighbours' edges; *DRT as biology* is the public tagline.
3. **[drt-scanner](https://github.com/norayr-m/drt-scanner)** — runs the transpose pass. Projects a target signal back through the transposes of the same matrices the generator used; reports column-by-column whether the reverse pass agrees with the forward pass — a per-configuration inversion-fidelity diagnostic.

End to end is *generator → cell simulator → scanner* — forward pass plus backward pass through the same matrices. Every step is sparse matrix-vector multiplication. The applied target is **bio digital twins** at tissue scale: the generator emits signal projections (a hormone bolus, an electrical pulse, a drug dose), the cell simulator's receptor field receives them and propagates state through tissue connectivity, the scanner verifies whether the forward map is invertible at that configuration.

Full write-up: **[Generator, Cell Simulator, Scanner — a sparse matrix-vector trio for bio digital twins](https://norayr-m.github.io/drt-generator/whitepaper.html)** ([markdown source](https://norayr-m.github.io/drt-generator/whitepaper.md)).

## Companion repository

- **DRT_Scanner** — the dual demo, where the signal flows the other way and you inspect reconstruction quality.

## References

- Distributed Reconstruction work — v0.1 in preparation, N. Matevosyan and A. Petrosyan.

Visualization co-authored with Claude (Anthropic).

## Author

Norayr Matevosyan

## License

GPLv3
