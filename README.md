# cSAGE Flux Calculations
Code and supplemental data for manuscript Conjugation-based Serine Recombinase-Assisted Genome Engineering (cSAGE) Enables Bioproduction in Non-model Bacteria. This repo contains all code required to recreate the metabolic flux map in Figure 5.

`cSAGE_flux_calculations.ipynb` performs flux balance analysis (FBA) on the `rpal.xml`
genome-scale metabolic model using [COBRApy](https://opencobra.github.io/cobrapy/). It loads
the model, sets exchange/uptake bounds for a specific growth condition (coumarate as the sole
carbon source, "Janusian" conditions), solves for the optimal flux distribution, and shows how
to inspect the resulting fluxes for individual reactions and metabolites.

The model (`rpal.xml`) and its reaction annotations (`modelDetails.xlsx`) can be sourced from
doi: [10.1186/s12859-019-2844-z](https://doi.org/10.1186/s12859-019-2844-z), as noted in the
notebook. Both files are expected to sit in the same directory as the notebook.

## Software dependencies and operating system

| Dependency  | Purpose                                            |
|-------------|-----------------------------------------------------|
| Python 3.12 | Language runtime                                     |
| cobra       | Reads the SBML model and runs FBA                    |
| pandas      | Reads `modelDetails.xlsx`                             |
| openpyxl    | Excel engine used by pandas to read `.xlsx` files    |
| jupyterlab / ipykernel | Run the `.ipynb` notebook                  |

`cobra` pulls in its own dependencies automatically (numpy, scipy, `python-libsbml`, `optlang`,
and the bundled GLPK solver via `swiglpk`) — no separate solver installation is required.

This notebook was developed and tested on **Windows 11** (build 10.0.26200). COBRApy and its
dependencies are pure-Python/cross-platform packages that also run on macOS and Linux, but only
the Windows setup described here has been verified for this analysis.

## Versions tested

- Python 3.12.11
- cobra 0.29.1
- pandas 2.2.3
- openpyxl 3.1.5
- python-libsbml 5.20.5
- jupyterlab 4.4.3

An `environment.yml` capturing these versions is included in this repository.

## Installation guide

1. Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html) or Anaconda if you don't
   already have a conda installation.
2. From the repository root, create the environment:
   ```
   conda env create -f environment.yml
   conda activate cSAGE-flux
   ```
   Alternatively, install the packages directly with pip into an existing Python 3.12
   environment:
   ```
   pip install cobra==0.29.1 pandas==2.2.3 openpyxl==3.1.5 jupyterlab ipykernel
   ```
3. Launch JupyterLab and open `cSAGE_flux_calculations.ipynb`:
   ```
   jupyter lab
   ```

**Typical install time:** on a normal desktop/laptop with a broadband internet connection,
creating the environment and downloading all packages takes roughly **5–10 minutes** (conda's
dependency solve step is usually the slowest part; the pip-only route is faster, typically
2–5 minutes).

## Demo

**Running it:**
1. Make sure `rpal.xml` and `modelDetails.xlsx` are in the same folder as the notebook (they
   are, by default, in this repository).
2. Open `cSAGE_flux_calculations.ipynb` in JupyterLab/Jupyter Notebook and select the
   `cSAGE-flux` kernel.
3. Run all cells top to bottom ("Run All").

**Expected output:**
- The model loads with 1253 reactions, 1108 metabolites, and 1126 genes. While loading, cobra
  prints a long series of `UserWarning`-style messages to the console (e.g. *"SBML package
  'layout' not supported by cobrapy"*, *"Use of the species charge attribute is discouraged"*).
  These are expected/harmless — they reflect optional SBML annotations that cobra doesn't parse,
  not errors, and do not affect the FBA result.
- `model.optimize()` returns an `optimal` solution. With the bounds set in the notebook
  (acetate uptake blocked, CO2 exchange open, coumarate uptake fixed at 0.29 mmol/gDW/hr), the
  objective (growth rate) evaluates to approximately **0.072 1/hr**.
- `model.metabolites.C00074.summary()` prints a table of the producing/consuming reactions for
  that metabolite at the optimal solution.
- `model.reactions.R00658.flux` returns the flux through that reaction, approximately
  **-0.075** (mmol/gDW/hr; sign indicates directionality).

**Expected runtime:** on a normal computer, reading and parsing `rpal.xml` takes about 2
seconds, and solving the FBA problem takes well under a second. Running the full notebook
top-to-bottom, including printing the SBML parsing warnings, typically completes in **under 15
seconds**.

## Instructions for use

- **Loading the model:** the model is read with `cobra.io.read_sbml_model`, as shown in the
  first code cell. Do not edit the existing notebook cells — copy cells into new ones, or work
  in a separate notebook, if you want to run a different analysis on the model.
- **Changing uptake/exchange bounds:** reaction bounds are set as
  `model.reactions.<reaction_id>.bounds = (lower, upper)`, e.g.
  `model.reactions.XR242.bounds = (0.29, 0.29)` fixes coumarate uptake at a specific rate.
  Reaction IDs and descriptions can be looked up in `mdetails` (loaded from
  `modelDetails.xlsx`) or via `model.reactions.get_by_id(...)`.
- **Simulating a gene/reaction knockout:** set the bounds of the reaction to be knocked out to
  `(0, 0)`, then re-run `model.optimize()` — see `model.reactions.XR57.bounds = (0, 0)` in this
  notebook for an example (used there to block acetate uptake, not as a knockout, but the
  mechanism is the same).
- **Finding the maximal growth rate for a given carbon source:** incrementally increase the
  import flux (upper bound) of the exchange reaction for that carbon source and re-solve at
  each step, until the objective (growth rate) plateaus. Worked examples of this procedure are
  in the notebooks under `Figure Code/`.
- **Inspecting results:** `model.metabolites.<id>.summary()` shows the producing/consuming
  reactions for a metabolite at the current solution; `model.reactions.<id>.flux` returns the
  flux through a specific reaction after `model.optimize()` has been run.

  ## Code description

The genome-scale metabolic reconstruction of R. palustris CGA009 is parsed from SBML (“rpal.xml”) into a constraint-based model “M”, consisting of a stoichiometric matrix “S”, per-reaction flux bounds “(lb, ub)”, and a biomass objective vector `c`; a companion spreadsheet (`modelDetails.xlsx`) is loaded only as a human-readable reaction-ID lookup table. To simulate growth on a single defined carbon source, the bounds of each relevant exchange reaction are set directly. Carbon sources other than the one being tested are blocked (“bounds = (0, 0)”). The source under study is fixed at its empirically observed uptake rate, and byproduct-export reactions are left open, after which flux balance analysis is performed by solving the linear program “maximize c^T v subject to S v = 0, lb ≤ v ≤ ub” with an LP solver (GLPK via optlang), yielding an optimal flux vector “v*” and objective value “z*” (the predicted growth rate). The resulting solution can then be queried per metabolite (summing production/consumption across all reactions involving it, to see which reactions carry flux to or from it) or per reaction (reading its flux from “v*”), without needing to re-solve. The same bound-setting/solve pattern generalizes to two related analyses used elsewhere in this study: a gene/reaction knockout is simulated by fixing the target reaction's bounds to “(0, 0)” and re-solving to compare mutant vs. wild-type growth rate, and the maximal growth rate supported by a given carbon source is found by repeatedly increasing that source's upper uptake bound and re-solving until the objective value plateaus.
