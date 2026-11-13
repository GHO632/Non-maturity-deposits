# Non-Maturity Deposits: Deposit Volume Forecasting

This project studies how the volume of German **Non-Maturity Deposits (NMD)** can be forecast using different machine learning approaches. The focus is on out-of-sample forecasting, comparing model performance across economic covariates, and assessing forecast stability under different time windows and interest-rate regimes.

> Note: The results depend on the dataset, model configuration, and evaluation setup. This project is for research and analytical purposes and is not a recommendation to buy, sell, or manage financial assets.

## Contents

- [Project objective](#project-objective)
- [Methodology](#methodology)
- [Repository structure](#repository-structure)
- [Requirements and installation](#requirements-and-installation)
- [Quick start](#quick-start)
- [Benchmark options](#benchmark-options)
- [Generate a forecast plot](#generate-a-forecast-plot)
- [Data](#data)
- [Results and interpretation](#results-and-interpretation)
- [Reproducibility and limitations](#reproducibility-and-limitations)
- [License and sources](#license-and-sources)

## Project objective

The repository implements a Python-based forecasting pipeline to:

1. forecast deposit volume over a fixed horizon.
2. compare classical and neural forecasting models against naive baselines.
3. evaluate the impact of different macro-financial covariates.
4. analyze single-point and rolling-origin results via tables, plots, and exported CSV files.

## Methodology

### Models

| Model | Description |
| --- | --- |
| `naive_rw` | Random-walk baseline: extends the most recent observed change in volume. |
| `naive_hold` | Naive-hold baseline: keeps the latest volume level constant. |
| `linreg` | Linear sequence model using ridge regularization during tuning. |
| `ffn` | Feedforward neural network trained on historical windows. |
| `xgboost` | Gradient-boosted trees on sequence features. |
| `tft` | Temporal Fusion Transformer for time-series forecasting. |

### Covariates

The shared pipeline (`common.py`) uses the following variables by default:

- `€STR`
- `Einlagezinssatz`
- `GPRC_DEU`
- `MoM Inflation`
- `DAX`
- `10Y Bond`

For sequence models, lagged covariates can be activated for 1, 5, and 20 trading days.

### Forecasting and evaluation

The main pipeline models the first difference of deposit volume (`Diff_Volume`) and reconstructs the volume level from the predicted differences. The default training end is **2023-11-30**, the default forecast horizon is **100 observations**, and the default sequence memory is **100 observations**.

Evaluation is based on MAE, RMSE, and R² for both differences and volume. A rolling-origin evaluation repeats the forecast over several training endpoints. Results can also be annotated by interest-rate regime, and model comparisons can include a Diebold-Mariano test.

## Repository structure

```text
.
├── common.py                 # Shared data, model, and evaluation pipeline
├── covariate_benchmark.py    # CLI for model/covariate benchmarking
├── benchmark_viz.py          # Benchmark and rolling-origin visualizations
├── forecast_plot.py          # CLI for forecast plotting
├── treesearch_plot.py       # Shared plotting utilities
├── linreg.py                # Linear regression entry point
├── ffn.py                   # FFN entry point
├── xgboost1.py              # XGBoost entry point
├── TFT.py                   # Standalone TFT example script
├── Data all variables.xlsx  # Main input dataset for the pipeline
├── Preprocessing/           # Data preparation scripts and derived files
├── Results/                 # Saved results and visualizations
```

## Requirements and installation

Python 3.10 or newer is recommended. The project uses libraries such as pandas, NumPy, scikit-learn, PyTorch/Lightning, `pytorch-forecasting`, XGBoost, and Matplotlib. Excel files require `openpyxl`. Some preprocessing scripts also depend on SciPy or yfinance.

Create and activate a virtual environment:

```bash
python -m venv .venv
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Windows (PowerShell):

```powershell
.venv\Scripts\Activate.ps1
```

Install the required dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install numpy pandas scikit-learn openpyxl xgboost \
  torch lightning pytorch-forecasting matplotlib seaborn tqdm \
  scipy joblib threadpoolctl yfinance
```

The exact package versions are not pinned in this repository. For reproducible runs, it is recommended to document or lock the environment versions used locally.

## Quick start

Run commands from the project root so relative file paths resolve correctly.

### Benchmark across predefined covariate sets

```bash
python covariate_benchmark.py \
  --modelle naive_rw naive_hold linreg xgboost \
  --kovariaten-modus presets \
  --output Results/covariate_benchmark_results.csv \
  --viz all
```

This benchmark writes results to CSV and generates visualizations when `--viz all` is enabled.

### Rolling-origin evaluation

```bash
python covariate_benchmark.py \
  --modelle naive_rw naive_hold linreg xgboost \
  --kovariaten-modus presets \
  --rolling \
  --rolling-freq QS \
  --rolling-output Results/covariate_benchmark_rolling.csv
```

Rolling-origin evaluations can take substantially longer, especially with larger model sets and more covariate combinations.

### CLI help

```bash
python covariate_benchmark.py --help
python forecast_plot.py --help
```

## Benchmark options

`covariate_benchmark.py` supports several command-line options, including:

| Option | Meaning |
| --- | --- |
| `--modelle ...` | Select models from `naive_rw`, `naive_hold`, `linreg`, `ffn`, `xgboost`, `tft`. |
| `--kovariaten-modus presets` | Use predefined covariate sets. |
| `--kovariaten-modus singletons` | Evaluate each covariate individually. |
| `--kovariaten-modus pairs` | Evaluate all pairwise covariate combinations. |
| `--kovariaten-modus all` | Combine presets, singletons, and pairs. |
| `--kovariaten-modus full` | Evaluate all non-empty subsets of the covariates; computationally heavy. |
| `--rolling` | Run a rolling-origin evaluation. |
| `--rolling-freq MS` | Set the frequency for automatic origins, e.g. `MS` or `QS`. |
| `--rolling-origins DATUM ...` | Provide explicit training endpoints. |
| `--lags` | Add lagged covariates for sequence models. |
| `--tune` | Enable grid-search tuning for sequence models. |
| `--regime` | Attach a regime label to each rolling origin based on the target variable, defaulting to `€STR`. |
| `--dm-test` | Run a pairwise Diebold-Mariano test in rolling evaluation. |
| `--viz all` | Generate heatmap, boxplot, scatter plot, and Top-5 table. |

Modes such as `all` and especially `full`, when combined with rolling evaluation and tuning, can multiply the number of model runs substantially. It is good practice to start with a small subset before running a full sweep.

## Generate a forecast plot

Example for an XGBoost forecast using the preset `nur_makro`:

```bash
python forecast_plot.py \
  --modell xgboost \
  --label nur_makro \
  --simulationen 5 \
  --output Results/forecast_xgboost_nur_makro.png
```

Additional examples:

```bash
# Linear regression with all standard covariates
python forecast_plot.py --modell linreg --label alle

# XGBoost with lagged features and tuning enabled
python forecast_plot.py --modell xgboost --label nur_zinsen --lags --tune

# TFT with an explicit training end
python forecast_plot.py --modell tft --label alle --training-end 2023-11-30
```

Use `--show` to display the plot instead of saving it. `--kovariaten` allows an explicit list of variables to override a preset label.

## Data

The main pipeline expects `Data all variables.xlsx` in the project root. The file should contain the columns used by `common.py`, including:

```text
Datum, Einlagevolumen, €STR, Einlagezinssatz, GPRC_DEU,
MoM Inflation, DAX, 10Y Bond
```

`Datum` must be interpretable as a date, and the time series is sorted chronologically before evaluation. The provided Excel file is intended for use with the main forecasting pipeline. Scripts in `Preprocessing/` may require additional raw files or source data that are not necessarily included in the repository. Check the path definitions in each script before running them.

> Confidentiality and data rights: before publishing or sharing the project, ensure that any underlying data licenses and usage rights are respected.

## Results and interpretation

The `Results/` folder contains example forecast charts and benchmark exports, including:

- [Model comparison](Results/Comparison.jpeg)
- [MAE by covariate and time period](Results/MAE%20per%20covariate%20per%20time.jpeg)
- [Performance vs. stability](Results/Performance%20vs.%20Stability.jpeg)
- [Best covariate set by model](Results/best%20set%20per%20model.jpeg)
- [Rolling-origin summary](Results/covariate_benchmark_rolling_summary.png)

A single test window is not sufficient to conclude broadly that one model is superior. Prefer rolling-origin evaluations and treat benchmark rankings as exploratory when working across many model and covariate combinations.

## Reproducibility and limitations

- Core settings such as target variable, training end, forecast horizon, covariates, and random seed are defined in `common.py`.
- Results depend on package versions and hardware setup, especially for PyTorch and GPU usage.
- Tuning, TFT training, and full rolling benchmarks can be computationally expensive.
- The TFT pipeline uses the last known value of external covariates in the forecast window and computes calendar features from future dates. This is an analytical assumption and does not represent true future economic information.
- `TFT.py` is a standalone example script with its own data configuration and is not identical to the main shared pipeline in `common.py`.
- Before scientific or production use, the data preparation, time splits, modeling choices, and metric definitions should be reviewed in context.

## License and sources

No explicit license file is currently present in the repository. If the project is to be used publicly or redistributed, add a suitable `LICENSE` file and confirm the rights for code and data reuse.

External datasets, indexes, and software packages should be documented with their source, version, and licensing terms. For financial and macroeconomic data, specific usage conditions may also apply.
