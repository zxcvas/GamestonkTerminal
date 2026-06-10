# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GamestonkTerminal is a Python-based CLI investment research terminal — a Bloomberg Terminal alternative for retail investors. It aggregates data from many financial APIs (Alpha Vantage, Yahoo Finance, Financial Modeling Prep, Reddit, Twitter, FRED, etc.) into an interactive menu-driven shell.

## Commands

### Installation

```bash
# Recommended: Poetry
poetry install                       # base install
poetry install -E prediction         # include ML/prediction features (PyTorch, TensorFlow, Prophet)

# Alternative: pip
pip install -r requirements.txt

# Conda
conda env create -f conda-environment.yaml
```

### Running the app

```bash
python terminal.py
```

### Tests

```bash
# Run full test suite (set MPLBACKEND=Agg for headless environments)
MPLBACKEND=Agg pytest tests/

# Run a single test file
MPLBACKEND=Agg pytest tests/test_discovery_fidelity_api.py

# Run a single test
MPLBACKEND=Agg pytest tests/test_discovery_fidelity_api.py::TestDiscoveryFidelityApi::test_orders
```

### Linting

All of these must pass before CI merges:

```bash
black --check .                                                         # formatting
flake8 . --count --ignore=E203,W503 --max-line-length=122              # style
mypy --ignore-missing-imports .                                         # types
pylint **/*.py                                                          # code quality
codespell --ignore-words-list="mape","ba" --quiet-level=2 --skip=".git"
bandit -r .                                                             # security
safety check                                                            # dependency vulns
```

Auto-fix formatting with `black .`.

## Architecture

### Menu-driven REPL

The app is a nested REPL. `terminal.py:main()` is the top-level loop; each sub-domain has a `*_menu()` function that runs its own input loop. Menus return `False` to go back one level or `True` to quit the whole app.

**Top-level commands** (from `terminal.py`): `load`, `view`, `candle`, `export`, `disc`, `fa`, `ta`, `dd`, `ba`, `ca`, `op`, `pred`, `fred`, `mill`, `res`.

### State threading

The global state — `s_ticker`, `df_stock`, `s_start`, `s_interval` — is loaded at the main level via `main_helper.load()` and threaded down into every sub-menu call. Menus do not fetch the ticker themselves; they receive it as arguments.

### Module layout

Each analysis domain lives in its own package under `gamestonk_terminal/`:

| Package | Purpose |
|---|---|
| `discovery/` | Stock screeners (Finviz, ARK, Fidelity, etc.) |
| `due_diligence/` | News, analyst ratings, SEC filings, insider trading |
| `fundamental_analysis/` | Income/balance/cash statements, earnings |
| `technical_analysis/` | TA indicators via pandas-ta |
| `behavioural_analysis/` | Reddit/Twitter/StockTwits sentiment |
| `comparison_analysis/` | Cross-ticker correlation/comparison |
| `options/` | Options chains and Greeks |
| `prediction_techniques/` | ML forecasting (ARIMA, Prophet, RNN, LSTM) |
| `fred/` | Federal Reserve economic data |
| `papermill/` | Jupyter notebook report generation |

Each package typically has:
- `*_menu.py` — the interactive menu loop
- `*_api.py` (one per data source) — API calls + argparse commands + print/plot output

### Command function pattern

Every user-facing command follows this exact pattern:

```python
def some_command(l_args):
    parser = argparse.ArgumentParser(add_help=False, prog="cmd", description="...")
    parser.add_argument("-n", "--num", type=check_positive, default=10, dest="n_num")
    ns_parser = parse_known_args_and_warn(parser, l_args)
    if not ns_parser:
        return
    # fetch data, build DataFrame, print table or show plot
```

`parse_known_args_and_warn` (from `helper_funcs.py`) prints a warning and returns `None` on parse errors — always guard with `if not ns_parser: return`.

### Configuration

**API keys** (`config_terminal.py`): All keys fall back to env vars prefixed `GT_`:
`GT_API_KEY_ALPHAVANTAGE`, `GT_API_KEY_FINANCIALMODELINGPREP`, `GT_API_KEY_QUANDL`, `GT_API_REDDIT_*`, `GT_API_POLYGON_KEY`, `GT_API_TWITTER_*`, `GT_FRED_API_KEY`.

**Feature flags** (`feature_flags.py`): Runtime behaviour via env vars prefixed `GTFF_`:
- `GTFF_USE_PROMPT_TOOLKIT` — enables tab-completion shell (default `False`; set `false` in CI)
- `GTFF_USE_ION` — enables `plt.ion()` interactive mode (set `false` in CI)
- `GTFF_USE_COLOR` — colorama terminal colours
- `GTFF_ENABLE_PREDICT` — gates ML prediction menu (requires `-E prediction` install)
- `GTFF_USE_PLOT_AUTOSCALING` — auto-scale plots to monitor size

### Naming conventions

The codebase uses Hungarian-style prefixes on variable names:
- `s_` — string (`s_ticker`, `s_start`)
- `df_` — pandas DataFrame (`df_stock`)
- `l_` — list (`l_args`)
- `n_` — numeric int/float (`n_length`)
- `b_` — boolean (`b_is_market_open`)
- `ns_` — argparse Namespace (`ns_parser`)
- `d_` — dict

### Visualization

Plots use matplotlib (`mplfinance` for candle charts). Always call `plot_autoscale()` from `helper_funcs.py` for `figsize`. Check `gtff.USE_ION` before calling `plt.ion()`. Use `MPLBACKEND=Agg` in headless/test environments to suppress display.

### Prediction module

The `prediction_techniques/` menu is only available when `GTFF_ENABLE_PREDICT=True` and the package was installed with `poetry install -E prediction`. Adding new models there requires adding them to `pred_menu.py` choices and wiring the import.
