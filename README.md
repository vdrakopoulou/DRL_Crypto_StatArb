# Replication Package for Cointegration-Based DRL Cryptocurrency Statistical Arbitrage

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20624264.svg)](https://doi.org/10.5281/zenodo.20624264)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![Replication](https://img.shields.io/badge/replication-package-brightgreen)
![DRL](https://img.shields.io/badge/DRL-DQN%20%7C%20PPO%20%7C%20A2C-purple)

This repository contains the replication package for the study:

> **Asset-Universe Design and Action-Space Granularity in Cointegration-Based Deep Reinforcement Learning for Cryptocurrency Statistical Arbitrage**

The package contains code, configuration files, saved model-summary outputs, manuscript-ready tables, figures, validation logs, and documentation required to reproduce the reported manuscript tables and figures.

## Creator

**Veliota Drakopoulou**  
Higher Colleges of Technology, United Arab Emirates  
Embry-Riddle Aeronautical University, United States  
ORCID: `0000-0002-1670-8033`  
Corresponding author: Veliota Drakopoulou  
E-mail: `vdrakopoulou@gmail.com`

## Citation

Drakopoulou, V. (2026). *Replication package for cointegration-based DRL cryptocurrency statistical arbitrage* (Version 1.0.0) [Software and data]. Zenodo. https://doi.org/10.5281/zenodo.20624264

## What this package reproduces

The study evaluates a cointegration-based deep reinforcement learning framework for cryptocurrency statistical arbitrage using:

- four predefined asset universes;
- three DRL agents: DQN, PPO, and A2C;
- two action-space specifications: binary and granular;
- a non-DRL COIN benchmark;
- seed-42 main results and robustness tests across seeds 42, 101, and 202.

The empirical design uses 30-minute USDT-denominated Binance spot-market OHLCV data. The training period is `2023-01-01` to `2024-12-10`, followed by a separation window from `2024-12-11` to `2024-12-17`, and an out-of-sample testing period from `2024-12-18` to `2025-05-17`.

## One-command replication

```bash
python replicate.py doctor
python replicate.py run --show-summary
```

## Useful commands

```bash
python replicate.py about
python replicate.py doctor
python replicate.py tables
python replicate.py figures
python replicate.py validate
python replicate.py summary
python replicate.py journal-map
python replicate.py cite
python replicate.py metadata
python replicate.py hashes
python replicate.py urls --start 2023-01 --end 2025-05 --dry-run
python replicate.py zip
```

## Recommended journal table placement

The package retains some internal filenames for reproducibility, but the journal manuscript can use this leaner numbering:

| Journal item | Content | Package source file |
|---|---|---|
| Table 1 | Asset universes used in the empirical design | `results/tables/Table1_asset_universes.csv` |
| Table 2 | Main model results by asset universe | `results/tables/Table3_main_model_results_seed42.csv` |
| Table 3 | Binary versus granular action-space comparison | `results/tables/Table4_binary_vs_granular_action_space.csv` |
| Table 4 | Robustness statistics across random seeds | `results/tables/Table5_updated_robustness_stats.csv` |
| Supplement | Best-strategy summary by universe | `results/tables/Table2_best_strategy_by_universe_seed42.csv` |
| Supplement | Seed completion check | `results/tables/Appendix_Table_A1_seed_completion_check.csv` |
| Supplement | Universe label crosswalk | `results/tables/Appendix_Table_A2_universe_label_crosswalk.csv` |

## Raw data note

Raw Binance 30-minute OHLCV files are not redistributed in this archive. The package provides configuration files and a downloader scaffold that can be used to reconstruct the required Binance Public Data file list.

To preview URLs without downloading:

```bash
python replicate.py urls --start 2023-01 --end 2025-05 --dry-run
```

## Reproduction scope

This package reproduces manuscript tables and figures from saved model-summary outputs. It is not intended to redistribute raw Binance market data or trained DRL checkpoint files.
