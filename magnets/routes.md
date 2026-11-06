# The instruments: Ouro routes for magnets

Every measurement in this skill comes from a route run on Ouro, so it has an
action id with inputs and outputs anyone can open. Call routes through the
Ouro MCP server or the `ouro` Python package, by the **id** below:

- `get_asset(id=<route id>, detail="full")` returns the input schema, price
  and execution mode.
- `execute_route(route_id=<id>, input_assets={...}, body={...}, wait=false, notify="failure")`
  starts a run. Pass structures as `input_assets` (a file id on Ouro), never
  as hand-built file objects.
- `get_action(id=..., include_response=true)` returns the result;
  `get_action(id=..., wait=true)` waits for it.
- `search_assets(asset_type="route", query=...)` finds a route not listed.

Send only the parameters you mean to change. Ids and prices were checked on
2026-11-06; `get_asset` shows the current price.

Price tiers: **free**, or **$/s** billed per second of compute (held up
front at the run's maximum and settled on actual runtime). "DFT" means an ABACUS run that
takes minutes to hours.

## Structures in

| Route | id | What | Price |
|---|---|---|---|
| `mmoderwell/check-whether-a-crystal-structure-is-already-known` | `3dced825-cfea-4711-89a9-1d7998242001` | matches a CIF against Materials Project, Alexandria, GNoME and COD; returns known / new polymorph / new composition plus `experimentally_known` | free |
| `mmoderwell/search-materials-project-materials` | `c32019e9-92d8-4c95-ac3a-2e3dd029eb40` | MP search by formula, chemistry or symmetry | free |
| `mmoderwell/get-the-cif-of-a-materials-project-alexandria-gnome-or-cod-material` | `e4812d4a-caae-4c11-a1a1-ff92af687d8d` | MP / Alexandria / GNoME / COD id → CIF with a license header | free |
| `mmoderwell/fetch-a-material-cif-from-materials-project` | `11224ef4-70e9-45f3-9f98-dd18ab8e0d4b` | MP id → CIF file | free |

## Generate and build

| Route | id | What | Time | Price |
|---|---|---|---|---|
| `mmoderwell/explore-a-chemical-system-with-ggen` | `25945f7c-d83a-45ba-8c40-30b69d9a3df7` | generate + relax across a system's stoichiometries, rank by hull | 15 to 60 min, async | $0.0012/s |
| `mmoderwell/scout-candidate-elements-with-ggen` | `4623c347-ec8b-4516-81eb-c989cbf5c940` | screen elements X for a template such as Fe-Co-{X} | minutes, async | $0.0012/s |
| `mmoderwell/generate-a-crystal-structure-using-ggen` | `454b4729-839e-494a-9ea5-ae3eab07570b` | one structure for an exact formula | minutes | $0.0012/s |
| `mmoderwell/generate-crystal-structures-with-mattergen-and-magnetic-density-conditioning` | `1126f5a0-bc42-4a76-a51d-ca9a903fcc06` | MatterGen samples biased toward a target magnetic density | minutes | free |
| `mmoderwell/generate-crystal-structures-with-mattergen-magnetic-density-and-hhi-score-conditioning` | `f156d895-242d-4fa5-b65b-fa8f08deb755` | the same, steered toward a safer supply chain too | minutes | free |
| `mmoderwell/generate-point-defect-candidates` | `a8420297-143c-4cd3-8cc0-9a23db6b9efd` | vacancies, substitutions, interstitials from a host | async | free |
| `mmoderwell/create-an-interstitially-doped-structure` | `be1c9062-9c73-4c56-a91f-6fbbbcad6d4f` | an interstitial dopant at a concentration | | free |
| `hermes/expand-a-crystal-structure-into-a-supercell` | `0bd98417-bc11-43e6-a67e-f301d57035cc` | supercell CIF | | free |

GGen explore body: `system` ("Fe-Co-B"), `max_atoms`, `num_trials`,
`e_above_hull`, `crystal_systems`, `min_fraction` / `max_fraction`,
`require_all_elements`, `include_phase_diagram`. Outputs: candidate CIFs, a
report, a phase diagram and a summary. Generated cells are relaxed and
relaxation changes their symmetry, so read the space group from the result.
`crystal_systems` steers generation only; no symmetry filter predicts
anisotropy, the MAE run does. At the default 15 trials GGen misses known
ground states about half the time for 12 to 16-atom cells; about 75 trials
reaches 95%. Alloy cells (ordering-variant landscapes) never converge by
random restart.

For exact edits (one site, a chosen ordering, a strain) build the CIF
yourself with pymatgen, upload it with `create_file`, and relax it before
measuring anything. When enumerating supercells, use upper-triangular HNF
matrices with `make_supercell` (it takes rows as the new lattice vectors);
the lower-triangular form repeats lattices. pymatgen reorders sites on CIF
read, so label Wyckoff sites geometrically, never by index across files.

## Relax

| Route | id | What | Price |
|---|---|---|---|
| `mmoderwell/relax-a-structure-and-return-a-magcif-with-ground-state-moments` | `b9593f22-3520-4439-ac88-f0931de9fd19` | Prophet relax (positions and cell), then a collinear order screen; outputs a P1 magCIF with the lowest ordering's moments. Body: `fmax` 0.05, `max_steps` 300, `relax_cell` true | free |
| `mmoderwell/relax-a-crystal-structure` | `d040d3b6-faad-40cf-9d7c-999a5c769ed8` | ML-potential relax with Orb, MACE or CHGNet, async | free |
| `mmoderwell/structure-relaxation-via-nequip-oam-xl` | `cac67dd7-4ca4-40bf-aa8a-51c51692681f` | NequIP-OAM-XL relax | $0.0009/s |
| `mmoderwell/dft-structure-relaxation` | `d180f520-fc88-48b7-9eca-3d123f5cf680` | DFT (ABACUS) variable-cell relax; `basis_size` DZP by default, pass `"TZDP"` for headline cells | $0.0006/s |

DFT relax notes: 16 to 110 minutes per 8 to 18-atom cell at kspacing 0.2;
$0.6 to 0.9 per 18-atom cell at kspacing 0.3. It may stop a few steps in with
the stress converged and the max force just above 0.04 eV/Å and label the
output "not converged"; that geometry is fine for the anisotropy route with
`allow_unrelaxed: true`. A second call with the same input resumes from the
last geometry. On Mn cells, relax without `initial_magmoms` and compare the
round-1 and final energies in the log: a seeded restart has switched
magnetic state three times out of three.

## Prophet: fast magnetic screening

Prophet is Ouro's ML potential with a spin-aware head. Its Heisenberg-form
exchange layer reads the couplings straight from the model. All routes are
free and synchronous and take one CIF as input `file`. Use them to screen
order, Js and Tc on every plausible candidate, then spend DFT on the ones
that survive.

| Route | id | What | Notes |
|---|---|---|---|
| relax + magCIF (above) | `b9593f22-3520-4439-ac88-f0931de9fd19` | relax, then the order screen at the relaxed geometry | seeds the Curie route and DFT |
| `mmoderwell/screen-collinear-magnetic-order-and-saturation-magnetization` | `a0f8fd1c-faab-4b08-aff1-6c2e79d4670e` | ranks FM, two-sublattice AFM and ferrimagnetic seeds by energy; local moments, Ms, net moment | a short collinear menu, not a magnetic space-group search. An AFM winner means Js ≈ 0 |
| `mmoderwell/estimate-the-curie-temperature-from-prophet-spin-exchange` | `fc0b4cd4-baaf-480d-8846-447ff172063f` | couplings in a supercell → mean-field Tc and Monte Carlo Tc | feed it the magCIF; quote the Monte Carlo value; `mean_field_ordering_overlap` near 0 means the couplings prefer another order (read Tc as unknown); sites under 0.5 μB are dropped |
| `mmoderwell/predict-spin-dependent-energy-forces-and-stress` | `9e09a76f-05aa-4c88-9e59-1f2cb1fdde08` | energy, forces, stress for a chosen magnetic state | pass `magnetic_moments` (μB per site in CIF order) to compare orderings you choose |
| `mmoderwell/score-a-rare-earth-free-permanent-magnet-candidate` | `0c0b4692-5d3e-4369-be15-a9174511897c` | Orb relax → Prophet order + Curie → hull → supply risk, one score out of 100 | the leaderboard's grader; free, async; stores the submitted cell in the shared crystal DB |

The ordering route (DFT, below) rejects the Prophet magCIF as
"non-collinear" because of its moment vector loop; strip the moments with
pymatgen and upload a plain copy for it. The anisotropy and relax routes
accept the magCIF as is.

## Measure

| Property | Route | id | Price | Notes |
|---|---|---|---|---|
| Hull distance | `mmoderwell/calculate-energy-above-the-convex-hull` | `75fe7f4b-22f5-4883-aa1d-378fd42da47c` | free, async | input `structure`, relaxed (max force ≤ 0.05 eV/Å) unless `allow_unrelaxed: true`. `energy_surface` Orb v3 by default; the PBE option scores your cell with Orb against published PBE references, so the two surfaces differ by up to 20 meV/atom. Quote both. `include_phase_diagram: true` returns the neighbors as a file |
| Dynamic stability | `mmoderwell/check-phonon-stability` | `69fc4538-ff58-42fd-a881-9ede1801a6f3` | $0.0009/s | 50 to 130 s for 9 to 18 atoms. Reproduces the PBE result on DFT-relaxed Fe2P. It flagged a made compound with site disorder unstable while its ordered relatives passed, so in a disordered family an imaginary mode is a flag to check, not a rejection |
| Phonon bands | `mmoderwell/calculate-phonon-dispersion-and-band-structure` | `c4d7aa14-f5a7-496e-93a5-542e09482eb5` | free, async | |
| Raw-material cost | `hermes/estimate-raw-material-cost-per-kg` | `9028cd75-e8ee-4f0a-bf41-c360f64cb8ad` | free | Nd2Fe14B 15.69 $/kg on this route (2019 prices); Fe2(P,Si) 0.88 |
| Supply risk | `will/score-a-composition-or-cif` | `5e31766b-278b-42e5-9cad-b7351db4cd2d` | free | HHI and cost per kg |
| Moments, Ms (ML) | `hermes/estimate-magnetic-moments-and-ms-from-a-cif` | `d1fdf6d1-2b35-47af-956f-1b83c2fca036` | free | CHGNet, unsigned moments, assumes collinear FM: antiferromagnetic Mn compounds look high |
| Moments, Ms (ligand-bridged) | `mmoderwell/magnetic-moment-ms-for-ligand-bridged-magnets-oxides-nitrides-mxm` | `091f9f57-f11d-40d0-9fe4-590c05a756e7` | free | oxides, nitrides, M–X–M exchange |
| Curie (ML, CHGNet + CatBoost) | `hermes/predict-curie-temperature-from-a-cif` | `daf42af4-a3e4-4f9e-af65-6ecaafc26334` | free | recall of its training table for known compounds; see calibration.md |
| Ground state (DFT) | `mmoderwell/ground-state-scf` | `e4ed9888-0c34-4a9d-a227-02fa9d7521e0` | $0.0003/s | energy, magnetization, site moments; seed orderings with `initial_magmoms` |
| Moments, Ms (DFT) | `mmoderwell/magnetic-moments` | `0a23817e-af47-485a-9c56-5f2df0178b80` | $0.0003/s | total and site moments, Ms; records whatever FM state it lands in |
| Magnetic order (DFT) | `mmoderwell/magnetic-ordering-fm-vs-afm` | `67393536-da30-4983-9c21-37c9fabbb296` | $0.0012/s | one SCF per ordering. With three magnetic species and `ferrimagnetic_by_species` it enumerated 40 configurations (~$7); use the default strategies, or seed two or three orderings on the ground-state route for ~$0.10 |
| Exchange couplings (DFT) | `mmoderwell/exchange-couplings-jij` | `def3b73b-b790-4716-bb37-60b565342d48` | $0.0003/s | |
| Curie (DFT) | `mmoderwell/curie-temperature-tc` | `29fd80eb-7cf5-4895-a349-4e9864407567` | $0.0003/s | Heisenberg Monte Carlo on DFT couplings |
| Anisotropy (DFT) | `mmoderwell/magnetic-anisotropy-energy` | `331d9faf-4b44-4679-a958-5a9a816036e1` | $0.0006/s | see below |
| Density of states (DFT) | `mmoderwell/density-of-states` | `3fa25076-f993-4a57-90dd-799f4d2b6e1b` | $0.0003/s | |
| Structure facts | `hermes/extract-crystal-structure-metadata` | `056f8ca8-8e0b-4a51-b365-f4b59395b2aa` | free | formula, space group, lattice |

## The anisotropy route

`mmoderwell/magnetic-anisotropy-energy` runs one collinear SCF, then builds
the spin-orbit operator from two fixed-density steps and sums the
second-order band energy on its own mesh (method `perturbation`, the
default); `force_theorem` diagonalizes with spin-orbit on the same
Hamiltonian; `total_energy` runs self-consistent spin-orbit per direction.
The three agree within a few percent on FePt and YFe11Ti. The route is
calibrated on Fe2P, L1₀ FePt, L1₀ FeNi and α''-Fe16N2 and misreads several
families; see [calibration.md](calibration.md).

- **Settings.** `kspacing` 0.2 (SCF), `sum_kspacing` 0.1, Gaussian 0.1 eV
  on the band sum. Leaders re-run at 0.15 / 0.08 moved ≤ 0.04 MJ/m³.
  `basis_size` DZP by default; `"TZDP"` costs 3.5 to 4× and is the basis to
  quote from.
- **Output.** `K1` is E(hard) − E(easy). `band_energies` holds every
  direction; `direction_frame.directions_cartesian` says which axis each
  label is. Compute the second gap yourself.
- **Input.** Rejects DFT-unrelaxed input by default; pass
  `allow_unrelaxed: true` and record the gate's force and stress from the
  response. The SCF is cached by input, so a rerun with other sum settings
  costs minutes.
- **Cost.** 2 to 8 min and $0.10 to 0.30 for a cell up to about 20 atoms at
  the default settings. A 74-atom cell did not finish in the 3-hour worker
  limit. Keep DFT cells under about 30 atoms, and submit reduced cells: a
  hand-skewed cell got 1836 k-points where the reduced one got 405.
- **SCF trouble.** Magnetic cells default to `mixing_beta` 0.2, `scf_nmax`
  200, with one warm-started rescue at half the mixing. If it still fails,
  halve again and seed the moments. Some cells (Mn3Co4Sb) never converge
  without constrained moments.
