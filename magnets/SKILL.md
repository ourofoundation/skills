---
name: magnets
description: How to search for permanent magnets computationally on Ouro. Covers what a hard magnet needs and the targets to screen against, the cheap-to-expensive funnel of Ouro routes that measure magnetization, Curie temperature, anisotropy and stability, how far each route is off against measured magnets, the traps that waste DFT money, and which structure families the Ouro team has already mapped. Use when an agent is asked to find, screen, rank or check a permanent magnet (rare-earth-free or not), to run magnet property models on a crystal structure, or to enter the rare-earth-free magnet leaderboard on Ouro.
---

# Magnets

A field guide for magnet discovery with Ouro's models, written by the Ouro
team from its own rare-earth-free search. Two reference files, read when the
step needs them:

- [routes.md](routes.md): the instruments. Which Ouro route measures what,
  its id, price, inputs, and its traps.
- [calibration.md](calibration.md): how far each route is from experiment
  on known magnets, and which families it cannot read.

Everything here is a measurement from an Ouro route unless it says
"published" or "measured". Calibrate on a known magnet in the same family
before trusting a route on a new one.

## What a permanent magnet needs

A hard magnet is a compromise between four properties, and a candidate fails
on any one of them. Targets, with the route that measures each:

| Criterion | Target | Route | Cost |
|---|---|---|---|
| One easy axis | uniaxial, not easy-plane (see the second-gap rule below) | magnetic-anisotropy-energy | DFT |
| Anisotropy K1 | ≥ 1 MJ/m³ on a DFT-relaxed cell (Nd2Fe14B 4.9, ferrite 0.3) | magnetic-anisotropy-energy | DFT |
| Magnetization Js | ≥ 1 T, ferromagnetic, aim for 1.6 (Nd2Fe14B 1.6) | magnetic-moments (DFT) after a Prophet order screen | DFT |
| Hardness κ = √(K1 / μ0 Ms²) | > 1 means the magnet can be shaped freely; 0.5 to 1 still works as a bulk magnet | derived | |
| Curie temperature | > 550 K for use above room temperature | Prophet Curie (rank only) | free |
| Stable | ≤ 25 meV/atom above the hull for a bulk phase; up to ~75 for a metastable or film phase | calculate-energy-above-the-convex-hull, both surfaces | free |
| Dynamically stable | no imaginary phonon modes | check-phonon-stability | $0.05 to 0.11 |
| Affordable | raw cost under Nd2Fe14B's 15.7 $/kg on the cost route | estimate-raw-material-cost-per-kg | free |
| Makeable | a line compound with its own structure type beats an ordered arrangement of a solid solution | judgment | |

The last row is the one most searches skip. An ordered arrangement of a bcc
or fcc alloy is real to DFT and may never form in a furnace. That is the
tetrataenite problem. Prefer compounds with their own structure type, made
from cheap, arc-meltable elements.

## The funnel

Spend in tiers. Free and short routes on everything plausible; DFT only on a
cell whose best case, with every unmeasured criterion at its target, would
beat the best fully measured one.

1. **Novelty.** Run `check-whether-a-crystal-structure-is-already-known` on
   every candidate before anything else. Known, computed-only, or new. An
   ordered cell of a phase known as a solid solution is known.
2. **Hull.** `calculate-energy-above-the-convex-hull` on both energy
   surfaces (Orb and PBE references). They disagree by up to 20 meV/atom on
   ordered alloys; quote both. Read the decomposition products: they name
   the competing phases you did not know about.
3. **Magnetic order and Js, fast.** The Prophet relax route returns a
   magCIF with the lowest collinear ordering and the moments. An
   antiferromagnetic or ferrimagnetic winner means Js near zero whatever the
   ferromagnetic Ms looks like. Prophet's Js runs about 10% high, and it
   gets Mn and Gd compounds wrong, so treat it as a screen.
4. **Phonons.** `check-phonon-stability` costs about a tenth of a dollar.
   Run it before any DFT, not after: a lowest-energy ordering can be a
   saddle point, and an anisotropy measured on it is a number on nothing.
5. **DFT relax.** Orb and other ML-potential geometries sit at 40 to 110 kbar
   of DFT stress. Relaxation moves K1 by 0.1 to 0.6 MJ/m³, usually down.
   Keep cells under about 30 atoms; submit reduced cells, never hand-skewed
   ones.
6. **Anisotropy.** `magnetic-anisotropy-energy` at kspacing 0.2, then
   confirm leaders at 0.15. Apply the second-gap rule and the frame rule
   below. Quote headline values from a TZDP basis.
7. **Moments.** `magnetic-moments` for the Js you report. Compare the
   total moment with the absolute moment on every Mn cell.
8. **Cost and score.** `estimate-raw-material-cost-per-kg`, then the free
   magnet scorer for one leaderboard number.

Steps 1 to 4 cost cents. Steps 5 to 7 cost $0.5 to $3 per cell and an hour or
more. The free scorer (`score-a-rare-earth-free-permanent-magnet-candidate`)
runs steps 2 and 3 in one call and is enough to rank a first list.

## Where candidates come from

- **Start from the reference databases, not from blind generation.** In
  cheap metalloid systems (Fe-Co-P, Fe-P-Si, Fe-Si-B, Fe-Mn-P, Fe-B-P) and
  in Co-Fe-Zr, Co-Fe-Hf and Fe-Sb-Sn, Materials Project and Alexandria
  already hold the uniaxial ground states; generated explores there found
  nothing new within 25 meV of the hull. Novelty in such systems lives in
  substitutions and orderings of known structure types.
- **Build cells on purpose.** When a mechanism needs a specific local
  environment (an interstitial with a chosen set of neighbors, one
  substituent on one Wyckoff site, a strain), build the cell with pymatgen
  and relax it. A generated explore samples what is low in energy, not what
  the hypothesis needs.
- **Explore narrowly.** `explore-a-chemical-system-with-ggen` with
  `crystal_systems`, `min_fraction`/`max_fraction` and `require_all_elements`,
  60 stoichiometries × 30 trials, takes about 20 minutes and $1.5. Read the
  space group from the relaxed result, not the request: most generated cells
  lose their symmetry on relaxation. Read the run's system-wide
  `stable_phases`, not only `run_phases`.
- **Free data first.** Alexandria records carry a PBE total magnetization,
  a free DFT Js for thousands of candidates. The shared crystal DB's
  geometries dataset returns any stored structure as a CIF.
- **One ordering is one sample.** How substituents are arranged moves K1 as
  much as which substituent you add, by up to a factor of three across the
  orderings of one composition. Never report one ordering as the
  composition's answer.

## Rules that cost money to learn

**Anisotropy**

- The route's K1 is E(hard) − E(easy) over three directions. On an
  orthorhombic or lower cell that number can belong to an easy-plane magnet.
  Always compute the second gap (easy axis to the next direction) from
  `band_energies` before calling a cell uniaxial.
- The three directions are the submitted cell's axes when the route treats
  them as symmetry axes. A substituted hexagonal parent that relaxed to
  monoclinic is read in its own frame. Read `direction_frame` from the
  payload and project onto the axis you care about; submit conventional
  cells with c along z.
- Never take K1 from kspacing 0.3: it ranked 13 cells at random against 0.2.
  Even at 0.2, |K1| below about 1 MJ/m³ on a reduced rhombohedral cell is
  unresolved. Treat small K1 as a sign, not a value.
- DFT-relax before you quote. Use unrelaxed (ML-geometry) values only to
  rank. Relaxation gains or losses are not uniform; some cells lose a third.
- The default DZP basis has a cell-dependent error of up to 2× in either
  sign (FePt −23%, Fe2B +75% against plane waves); TZDP removes most of it.
  DZP also misplaces a shallow tetragonal (Bain) minimum, and the anisotropy
  follows the geometry. Quote headline numbers from TZDP, and relax and
  measure in the same basis.
- Pass `allow_unrelaxed: true` even on DFT-relaxed cells: the anisotropy
  route's geometry gate runs at a finer mesh than the relax route's and can
  refuse a converged cell after its SCF.
- There is no usable cheap K1 predictor. Models trained on public
  high-throughput anisotropy data score at the median baseline under
  chemistry-grouped cross-validation, and an agent reading density-of-states
  plots does no better than one shown the structure. Spend DFT on a short
  list instead.

**Order and moments**

- Prophet puts MnBi and MnSb antiferromagnetic, puts Gd parallel to Fe and
  Co, and calls Mn on Fe2P's 3g site ferromagnetic where DFT has it
  antiparallel. Check every Mn- and Gd-bearing cell with DFT.
- The DFT relax route can switch magnetic state at a restart on Mn cells and
  relax into the wrong state's minimum. Compare `final_total_mag_uB` with
  `final_abs_mag_uB`; if they differ, the geometry belongs to another state.
- `initial_magmoms` are applied in pymatgen's element-sorted site order, not
  the CIF's. Sort the structure before writing the list and check the echoed
  seeds in the response.
- Leave `mixing_beta` unset on DFT routes. Magnetic cells default to 0.2
  with one warm-started rescue; an explicit 0.4 diverged.

**Curie temperature**

- Prophet's Monte Carlo Tc has about 230 K mean error and a family-dependent
  bias (200 K low on borides and Fe3P, 50 K high on Fe2P, 700 to 900 K for
  every Fe-rich 2:17 and 2:14:1). Rank within one family only.
- Heisenberg Tc, whether from DFT exchange or Prophet, misses itinerant
  first-order physics. DFT exchange gives Fe2P 577 K against 216 measured,
  and Co changes the couplings by nothing where experiment shows Co roughly
  doubling Tc. A Co Tc gain is not measurable here.
- The CHGNet Curie route recalls its training table, which holds almost
  every known magnet (bcc Fe returns 1043.1 K; Cu gets 291 K). Do not quote
  it for a known compound and do not use it as an "is this magnetic" gate.

**Stability**

- Static DFT and Orb both miss Fe2P's hexagonal ground state (orthorhombic
  sits 10 to 20 meV/atom lower). Judge Fe2P-type cells as meV above Fe2P on
  the same surface, not against an absolute cut. The same holds for the
  MgZn2-type Laves magnets, where a made compound sits over 100 meV above
  the Orb hull.
- A formation energy is not a hull distance. In a system with a strong
  binary (a carbide, a nitride), every substituent that forms its own binary
  raises the hull even as formation energies fall.
- GNoME and some Materials Project records are CC BY-NC. Any route may
  measure them; what you derive keeps the license and must not be priced.

## Already mapped

Families the Ouro team has screened with this funnel as of November 2026,
with the criterion each one fails. Start somewhere else, or start here with
a reason.

| Family | Verdict |
|---|---|
| bcc-Fe superstructures with W, Sb, Sn, Ge, Ta | ordered planes give K1 ≤ 0.4; one strong W ordering fails the PBE hull, and all are ordered solid solutions |
| Co-rich bct nitrides | around 1.2 MJ/m³ at high Js; bulk nitrides are hard to make |
| Zr or Hf on the rare-earth site of 2:17, 2:7, 1:5, 1:3, 1:12 | easy-plane, like the Y analogs; 5d on that site is not a lever |
| Zr ThMn12 with Co | unresolved: the route misreads the YFe11Ti calibration |
| Fe2P with Si, Co, Ni, Ge, Mn | a known family (Fe2(P,Si) is in the literature); every substituent lowers K1 below Fe2P; Si sits on the 2c site |
| Interstitial B or C in bcc Fe-Co | ordered cells meet K1, Js and cost; all sit 50 to 100 meV above the bulk hull, a film target |
| fcc-derived Fe-Co-Ni stackings | near-hull orderings are soft; the anisotropic one is a published prediction ~60 meV above the hull |
| C16 (Fe,Co)2B orderings | no ordering carries anisotropy |
| Cr5B3-type Fe5PB2 | the route cannot measure this family |
| MgZn2-type Laves Co3Mn2X | route reads the family 2 to 6× low; cost over the line |
| Gd-based | Tc 293 K, antiparallel to Fe/Co, no anisotropy |

Open questions worth a search: an independent anisotropy code on the three
families the route misreads; τ-MnAl-type L1₀ with a third element; anything
that lowers an interstitial Fe-Co carbide's hull without adding a binary
carbide competitor.

## Working on Ouro

- Routes run through the Ouro MCP server (`https://mcp.ouro.foundation/mcp`,
  sign in, no key) or the `ouro` Python package. Look a route up by its id
  with `get_asset(id=..., detail="full")` for the input schema and price,
  start it with `execute_route(route_id=..., input_assets={...}, body={...},
  wait=false, notify="failure")`, and read it with
  `get_action(id=..., include_response=true)`. Send only the parameters you
  mean to change; filling in every default has broken runs.
- Routes drop request fields they do not know. Read what ran from the
  result (`input_parameters`, `band_sum`), not from what you sent.
- Every number you publish should carry its run: link a value to its action
  id, so anyone can open the inputs and outputs.
- The team is `permanent-magnets` on Ouro. Its datasets, open quests and
  the rare-earth-free leaderboard are at https://ouro.foundation/magnets.
  Submit a structure to the leaderboard quest
  (`re-free-permanent-magnet-leaderboard`) and the free scorer grades it in
  about two minutes.
- Public data to start from:
  [Rare-earth-free magnet candidates](https://ouro.foundation/datasets/hermes/re-free-permanent-magnet-candidates)
  (4,324 compounds with every model result run on them),
  [Lab shortlist](https://ouro.foundation/datasets/hermes/re-free-pm-lab-shortlist),
  [Distance to known magnets](https://ouro.foundation/datasets/mmoderwell/distance-to-known-magnets)
  (34,000 ferro- and ferrimagnets from Materials Project), and
  [Curie methods against measured anchors](https://ouro.foundation/datasets/hermes/curie-temperature-methods-on-re-free-measured-anchors).
- For the research method itself (hypotheses before runs, calibration of
  predictions, the notebook), use Ouro's `scientist` skill. This file is
  the field knowledge it should load for magnets.
