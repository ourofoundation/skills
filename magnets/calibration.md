# Calibration: the routes against measured magnets

Run a route on a known material in the same family before trusting it on a
new one. These are the calibrations we have; add your own and compare. K1 in
MJ/m³, Js in T, Tc in K. "Route" values are ours on Ouro; "published" are
PBE or all-electron literature values; "measured" are experiments.

## Anisotropy route (`magnetic-anisotropy-energy`)

Known magnets the route reads correctly:

| Material | Route | Published (PBE) | Measured | Verdict |
|---|---|---|---|---|
| Fe2P, P-62m | +2.38 uniaxial c (2.37 k-converged, 2.03 DFT-relaxed) | | ~2.3 at low T | trust on Fe2P-type cells |
| L1₀ FePt | 2.87 meV/f.u. (total energy, kspacing 0.3) | 2.7 to 3.0 | | within the published spread; plane waves give 2.18 at a denser mesh, so DZP reads it 23% low |
| L1₀ FeNi (tetrataenite) | +0.60 uniaxial c | 0.5 | 1.3 | matches PBE; PBE itself is low on this one |
| α''-Fe16N2 | +0.66 uniaxial c | 0.6 | ~1.0 | matches PBE |
| Fe3Sn, D0₁₉ | easy-plane | | easy-plane | direction right |
| Fe2B, C16 | −1.66 planar | −0.31 to −0.64 | −0.80 | direction right, magnitude 2×; TZDP gives −1.00, plane waves −0.98 |

Families the route does not read, one line each. Do not put a K1 from this
route on these cells in a measured column, as a value, a ratio or a ranking:

- **ThMn12 (YFe11Ti and relatives).** Easy-plane on the route in every
  setting we tried; experiment and published PBE are uniaxial. Unresolved.
- **Cr5B3-type (Fe5PB2 and relatives).** Reads near zero, and gets the sign
  wrong on a made uniaxial compound of the family. Unresolved; it needs a
  second code.
- **MgZn2-type Laves (Co3Mn2X).** Right sign and axis, magnitude 2 to 6×
  low, and the better basis moves it further off.
- **Co-rich bct Fe-Co-Ni metals.** Reads about 1.3 to 1.6× the published
  VASP value; part of that is the DZP basis. Take ratios within our own
  numbers, not against VASP, in this chemistry.

Mesh facts: the band sum is smeared (Gaussian 0.1 eV) on its own mesh.
Kspacing 0.3 versus 0.2 ranked 13 cells at random (Spearman 0.09) and
flipped the sign of 2 of 10 Fe-rich tetragonal cells. At 0.2 the in-plane
split on a Niggli-reduced rhombohedral cell is about 1 meV, so |K1| under
about 1 MJ/m³ is unresolved there.

Basis facts: against plane waves on the same cells, DZP reads FePt −23% and
Fe2B +75%; TZDP reads them −16% and +2%. The pseudopotential family moves
K by a few percent only. DZP misplaces a shallow tetragonal (Bain) minimum
but keeps the same shape as TZDP on interstitial and Fe2P-type cells.

The perturbation method leaves out off-site spin-orbit worth about 20% on
FePt; for 5d cells check with `force_theorem` or `total_energy`.

## Magnetization

| Route | Calibration | Verdict |
|---|---|---|
| Prophet order screen, Js | bcc Fe 2.77 T (DFT 2.30); Prophet / PBE median 1.11 over 984 ferromagnetic Alexandria overlaps; Fe5PB2 1.69 vs 1.43 DFT, 1.22 measured | runs ~10% high, more on borides; use for FM/AFM order, take Js from DFT |
| Prophet order screen, order | MnBi and MnSb come out AFM (both ferromagnets); γ'-Fe4N ferrimagnetic until the cell is relaxed; Mn on Fe2P 3g FM where DFT is antiparallel; Gd parallel to Fe and Co (really antiparallel) | check Mn and Gd cells with DFT |
| CHGNet moments (`estimate-magnetic-moments-and-ms-from-a-cif`) | within ~0.3 T of DFT on 19 Fe-rich cells, usually low; moments unsigned | AFM Mn compounds look high |
| DFT moments (`magnetic-moments`) | matched Prophet within 0.1 T on simple Fe-rich cells | records whatever FM state it lands in; Mn is seeded antiparallel, so Mn results depend on the seed |

## Curie temperature

| Route | Calibration | Verdict |
|---|---|---|
| Prophet Monte Carlo Tc | mean error ~230 K on 39 measured magnets; Fe5SiB2 553 vs 784; Fe3P 522 vs 716; Fe2P 265 vs 217; YFe11Ti 716 vs ~524; Fe5PB2 416 vs 615; every Fe-rich 2:17 and 2:14:1 at 715 to 905 K (Sm2Fe17 787 vs 389); fcc Fe a 761 K ferromagnet; within ~10% on Fe, Fe16N2, Co2MnSi, MnAl, Sm2Fe17N3, Gd2Fe14B | rank within one family only; read a Tc with `mean_field_ordering_overlap` near 0 as unknown |
| DFT exchange + Monte Carlo (`curie-temperature-tc`) | Fe2P 577 vs 216 measured; Co substitution changes the couplings by nothing where experiment shows Co doubling Tc | Heisenberg misses itinerant first-order physics; ratios within a family at best; Co's Tc gain is not measurable here |
| CHGNet + CatBoost (`predict-curie-temperature-from-a-cif`) | bcc Fe returns 1043.1 K; Cu 291 K, Al 150 K; held out by formula the error is 167 K, biased ~145 K low | recall of its training table (almost every known magnet); never a gate for "is this magnetic" |

The public comparison of all three on 39 magnets:
[Three Curie temperature estimates against 39 measured magnets](https://ouro.foundation/posts/mmoderwell/three-curie-temperature-estimates-against-39-measured-magnets-the-best-scorer-is-recalling-its-training-set),
dataset
[Curie temperature model comparison on 39 measured magnets](https://ouro.foundation/datasets/mmoderwell/curie-temperature-model-comparison-on-39-measured-magnets).

## Stability

| Route | Calibration | Verdict |
|---|---|---|
| Hull, Orb surface | Fe2P (a stable compound) 21 meV above; Fe3Sn 32; hexagonal Fe2P sits above orthorhombic Pnma Fe2P by 21 (PBE 11; only zero-point spin fluctuations restore the real phase); a made MgZn2-type Laves magnet over 100 above; α''-Fe16N2 18; L1₀ FeNi 0 | judge Fe2P-type and Laves cells against the made parent on the same surface; an absolute 25 meV cut is wrong for them |
| Hull, Orb vs PBE surface | Fe4W 23 vs 44; Fe2P 21 vs 11 | up to 20 meV/atom apart on ordered alloys; quote both |
| Phonons (NequIP-OAM-XL) | DFT-relaxed Fe2P stable, as published; a made compound with site disorder flagged unstable while its ordered relatives passed | reliable on ordered line compounds; in a family with site disorder an imaginary mode is a flag |

## What no route here can measure

- A cheap anisotropy. Public high-throughput K1 data gives no transferable
  signal (chemistry-grouped Spearman 0.12 to 0.20, sign accuracy at the base
  rate), a hand-built geometric index reached Spearman 0.6 on 20 references
  but is blind to sign, and density-of-states reading by an agent did no
  better than structure alone.
- The Tc gain from Co in an itinerant magnet (Heisenberg models see none).
- Anisotropy in the ThMn12, Cr5B3-type and MgZn2-type Laves families (see
  above).
- Site disorder. Every cell here is ordered; a made compound with mixed
  occupancy is approximated by one ordering.
