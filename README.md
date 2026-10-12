# Excavated slope stability in soft Bangkok clay

This project predicts two things for unsupported excavated slopes in soft Bangkok clay: the critical excavation height (Hcr) and the factor of safety (FS). It starts from a database of staged finite element (FEM) excavation analyses and builds two independent prediction routes on top of it. One route uses compact engineering equations. The other uses machine-learning surrogates. A browser-based GUI puts the results side by side.

You can try the GUI online at https://saharat012.github.io/slope-Prediction-GUI/.

## What it covers

The FEM database spans three representative soil profiles from the Chao Phraya River basin, slope angles from 20 to 80 degrees, excavation depths from 1 to 10 m, three groundwater conditions (water table at ground level, dry, and rapid drawdown), and a fixed 10 kPa surcharge behind the crest.

The predictions are for preliminary screening inside that range. They do not replace a site-specific FEM analysis, and a new soil profile, a different surcharge, or a depth beyond 10 m is outside what the models have seen. The results here come from grouped cross-validation on one FEM database, so treat them as development evidence and not as independent-site validation.

## How the work flows

1. FEM database. Each excavation is simulated in stages, removing 1 m at a time, with a safety calculation after every stage. The database has 63 excavation curves. For 47 of them the FEM brackets Hcr between the last stable depth and the first failed depth. The other 16 never fail within 10 m, so they only give a lower bound. The 372 FS values come from stable stages. The PLAXIS models are not part of this repository, so the FEM runs cannot be repeated from here.

2. Equations. Method A estimates Hcr with a Gaussian likelihood that handles both the intervals and the lower bounds, without pretending the failure depth was observed exactly. Softplus-7 estimates FS from the stable stages and returns values above 1 by construction. Both are calibrated from the FEM data alone.

3. Machine learning. XGBoost and xDeepFM are trained straight from the same FEM data on the same compact inputs as the equations. They never see the equations' outputs, and the equations never see theirs.

4. Shared validation. All routes are scored with the same repeated grouped cross-validation: 10 repeats of five folds, grouped by soil profile and slope angle, so related excavation stages never straddle a training and validation split. Hcr is judged mainly by interval coverage, distance to the interval, and violations of the lower bound. FS is judged by R², RMSE, MAE, MAPE, and the share of predictions within 10% of FEM.

5. GUI. The equations are evaluated directly in the browser. The XGBoost and xDeepFM panels show stored predictions and interpolate only inside the FEM grid.

## Using the GUI

Open the online version, or open the HTML file in the `gui` folder in any modern browser. No server or installation is needed. The page loads a web font when you are online and falls back to a system font otherwise.

Choose a soil group (G1 is the strongest profile, G3 the weakest), a water condition, a slope angle, and an excavation depth. The Calculator tab shows Hcr and FS from the equations, XGBoost, and xDeepFM, with a plot of FS against depth. The FEM accuracy tab shows how each route performed in cross-validation. The Method definitions tab lists the equations and fitting notes.

A few things to keep in mind when reading the output:

- An Hcr shown as "> 10 m" means the excavation stayed stable through the deepest FEM stage. It is not a predicted failure height.
- FS appears as N/A when the depth is outside the stable range the FEM resolved.
- When the three routes disagree, check the case against FEM. When they agree, that is still not independent confirmation of safety.
- FS values near 1 deserve the most caution. The equation tends to overestimate FS there.

## Predicting FS from Python

The refined xDeepFM model for FS can be run without the GUI. Set up an environment first:

```sh
python -m venv .venv
source .venv/bin/activate        # on Windows: .venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Prepare a CSV with one row per case and these columns: `G` (1, 2, or 3), `Water` (`WET`, `DRY`, or `RAPID`), `theta` (slope angle in degrees), and `H` (excavation depth in metres). Then run:

```sh
python code/outputs/reconciled/predict_refined_fs.py input.csv output.csv
```

The output keeps your columns and adds `refined`, the median of three ensemble members, along with each member's prediction and `member_spread`. The spread measures disagreement between members. It is not a calibrated uncertainty interval. No clipping is applied, so check that your inputs stay inside the FEM range listed above.

## Reproducing the results

You only need this section to retrain or recompute. The scripts live in `code/outputs/reconciled` and run in this order:

```sh
python code/outputs/reconciled/reconcile.py                 # calibrate the equations from FEM
python code/outputs/reconciled/train_shared.py              # train XGBoost and xDeepFM, run shared validation
python code/outputs/reconciled/refine_xdeepfm_fs.py         # refine the xDeepFM model for FS
python code/outputs/reconciled/summarize_fs_refinement.py   # summarize the refinement
python code/outputs/reconciled/publication_figures.py       # regenerate figures
```

Two cautions before you start. The scripts write into the same output folders that hold the saved results, so work on a copy of the repository if you want to keep the originals. Retraining also does not update the GUI. New predictions have to be checked and exported into the GUI before it reflects them.

Training uses CPU only and takes noticeably longer than loading the saved models. Each run records its protocol and fold assignments next to its results, which makes it possible to compare a new run against the saved one.

## Repository layout

- `data`: the FEM results and the coefficient tables
- `code`: the calibration, training, validation, and prediction scripts, plus the saved models and results
- `gui`: the browser interface and screenshots of its tabs
- `manuscript`: the accompanying paper
