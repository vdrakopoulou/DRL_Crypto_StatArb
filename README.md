#!/usr/bin/env python3
"""
GitHub replication launcher for:
Replication Package for Cointegration-Based DRL Cryptocurrency Statistical Arbitrage

Citation:
Drakopoulou, V. (2026). Replication package for cointegration-based DRL cryptocurrency
statistical arbitrage (Version 1.0.0) [Software and data]. Zenodo.
https://doi.org/10.5281/zenodo.20624264

This file is intentionally dependency-light. It wraps the original reproduction scripts in
code/ and adds GitHub-friendly metadata, diagnostics, summaries, citation output,
checksums, and release-archive helpers.
"""

from __future__ import annotations

import argparse
import csv
import hashlib
import os
import platform
import shutil
import subprocess
import sys
import textwrap
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Iterable, Sequence

__version__ = "1.0.0"

ROOT = Path(__file__).resolve().parent
CODE_DIR = ROOT / "code"
CONFIG_DIR = ROOT / "config"
DATA_DIR = ROOT / "data"
RESULTS_DIR = ROOT / "results"
TABLES_DIR = RESULTS_DIR / "tables"
FIGURES_DIR = RESULTS_DIR / "figures"
LOGS_DIR = RESULTS_DIR / "logs"


@dataclass(frozen=True)
class PackageMeta:
    title: str = "Replication Package for Cointegration-Based DRL Cryptocurrency Statistical Arbitrage"
    study: str = "Asset-Universe Design and Action-Space Granularity in Cointegration-Based Deep Reinforcement Learning for Cryptocurrency Statistical Arbitrage"
    creator: str = "Veliota Drakopoulou"
    creator_indexed: str = "Drakopoulou, Veliota"
    affiliation_a: str = "Higher Colleges of Technology, United Arab Emirates"
    affiliation_b: str = "Embry-Riddle Aeronautical University, United States"
    orcid: str = "0000-0002-1670-8033"
    email: str = "vdrakopoulou@gmail.com"
    version: str = "1.0.0"
    year: str = "2026"
    doi: str = "10.5281/zenodo.20624264"
    doi_url: str = "https://doi.org/10.5281/zenodo.20624264"
    python_min: str = "3.10"

    @property
    def apa(self) -> str:
        return (
            "Drakopoulou, V. (2026). Replication package for cointegration-based DRL "
            "cryptocurrency statistical arbitrage (Version 1.0.0) [Software and data]. "
            f"Zenodo. {self.doi_url}"
        )


META = PackageMeta()


REQUIRED_FILES = [
    "README.md",
    "CITATION.md",
    "requirements.txt",
    "environment.yml",
    "run_all.sh",
    "run_all.bat",
    "config/study_config.yaml",
    "config/universe_crosswalk.csv",
    "code/00_validate_inputs.py",
    "code/01_download_binance_klines.py",
    "code/02_recreate_tables.py",
    "code/03_make_figures.py",
    "code/04_validate_results.py",
    "data/source_outputs/modeling_comparison_summary.csv",
    "data/source_outputs/robustness_stats_by_model.csv",
    "data/source_outputs/robustness_seed_check.csv",
]

EXPECTED_TABLES = [
    "Table1_asset_universes.csv",
    "Table2_best_strategy_by_universe_seed42.csv",
    "Table3_main_model_results_seed42.csv",
    "Table4_binary_vs_granular_action_space.csv",
    "Table5_updated_robustness_stats.csv",
    "Appendix_Table_A1_seed_completion_check.csv",
    "Appendix_Table_A2_universe_label_crosswalk.csv",
]

EXPECTED_FIGURES = [
    "figure3_best_total_return_by_universe.png",
    "figure4_return_drawdown_best_strategies.png",
    "figure5_granular_minus_binary_return_delta.png",
    "figure6_robustness_return_ranges.png",
]

KEY_RESULTS = [
    ("Baseline 14, DQN binary, seed-42 return", "30.24%"),
    ("Payment-Coin 6, DQN binary, seed-42 return", "32.26%"),
    ("Broad 20, COIN benchmark, seed-42 return", "3.25%"),
    ("Layer-1 Platform 9, PPO granular, seed-42 return", "-7.83%"),
    ("Payment-Coin 6, DQN binary, robustness mean return", "16.50%"),
]

JOURNAL_TABLE_MAP = [
    ("Table 1", "Asset universes used in the empirical design", "results/tables/Table1_asset_universes.csv"),
    ("Table 2", "Main model results by asset universe", "results/tables/Table3_main_model_results_seed42.csv"),
    ("Table 3", "Binary versus granular action-space comparison", "results/tables/Table4_binary_vs_granular_action_space.csv"),
    ("Table 4", "Robustness statistics across random seeds", "results/tables/Table5_updated_robustness_stats.csv"),
    ("Supplement", "Best-strategy summary by universe", "results/tables/Table2_best_strategy_by_universe_seed42.csv"),
    ("Supplement", "Seed completion check", "results/tables/Appendix_Table_A1_seed_completion_check.csv"),
    ("Supplement", "Universe label crosswalk", "results/tables/Appendix_Table_A2_universe_label_crosswalk.csv"),
]

WORKFLOW = [
    ("validate-inputs", "00_validate_inputs.py", "Check configuration and saved source outputs"),
    ("recreate-tables", "02_recreate_tables.py", "Rebuild manuscript and appendix CSV tables"),
    ("make-figures", "03_make_figures.py", "Rebuild manuscript PNG figures"),
    ("validate-results", "04_validate_results.py", "Validate key values and expected outputs"),
]


class Colors:
    def __init__(self, enabled: bool = True) -> None:
        self.enabled = enabled and sys.stdout.isatty() and os.environ.get("NO_COLOR") is None

    def code(self, code: str, text: str) -> str:
        return f"\033[{code}m{text}\033[0m" if self.enabled else text

    def bold(self, text: str) -> str:
        return self.code("1", text)

    def dim(self, text: str) -> str:
        return self.code("2", text)

    def red(self, text: str) -> str:
        return self.code("31", text)

    def green(self, text: str) -> str:
        return self.code("32", text)

    def yellow(self, text: str) -> str:
        return self.code("33", text)

    def blue(self, text: str) -> str:
        return self.code("34", text)

    def magenta(self, text: str) -> str:
        return self.code("35", text)

    def cyan(self, text: str) -> str:
        return self.code("36", text)


C = Colors()


class ReplicationError(RuntimeError):
    """Raised when a wrapped replication command fails."""


def rel(path: Path) -> str:
    try:
        return path.relative_to(ROOT).as_posix()
    except ValueError:
        return path.as_posix()


def hr(char: str = "=") -> str:
    return char * 88


def banner(title: str, subtitle: str | None = None) -> None:
    print(C.cyan(hr("=")))
    print(C.bold(title.center(88)))
    if subtitle:
        print(C.dim(subtitle.center(88)))
    print(C.cyan(hr("=")))


def panel(title: str, body: str) -> None:
    width = 88
    print(C.magenta("+" + "-" * (width - 2) + "+"))
    print(C.magenta("|") + C.bold(title.center(width - 2)) + C.magenta("|"))
    print(C.magenta("+" + "-" * (width - 2) + "+"))
    for line in textwrap.wrap(body, width=width - 6) or [""]:
        print(C.magenta("|  ") + line.ljust(width - 6) + C.magenta("  |"))
    print(C.magenta("+" + "-" * (width - 2) + "+"))


def section(title: str) -> None:
    print()
    print(C.blue("-- " + title))


def ok(msg: str) -> None:
    print(C.green("[OK] ") + msg)


def warn(msg: str) -> None:
    print(C.yellow("[WARN] ") + msg)


def fail(msg: str) -> None:
    print(C.red("[FAIL] ") + msg)


def info(msg: str) -> None:
    print(C.dim("[INFO] ") + msg)


def human_size(num: int) -> str:
    units = ["B", "KB", "MB", "GB"]
    size = float(num)
    for unit in units:
        if size < 1024 or unit == units[-1]:
            return f"{size:.1f} {unit}" if unit != "B" else f"{int(size)} {unit}"
        size /= 1024
    return f"{num} B"


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def read_csv(path: Path) -> list[dict[str, str]]:
    with path.open(newline="", encoding="utf-8-sig") as fh:
        return list(csv.DictReader(fh))


def run(command: Sequence[str], check: bool = True) -> subprocess.CompletedProcess[str]:
    printable = " ".join(command)
    info("Running: " + printable)
    start = time.perf_counter()
    proc = subprocess.run(list(command), cwd=str(ROOT), text=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    elapsed = time.perf_counter() - start
    output = proc.stdout.strip()
    if output:
        print(textwrap.indent(output, "  "))
    if proc.returncode == 0:
        ok(f"Completed in {elapsed:.2f}s: {printable}")
    else:
        fail(f"Exit code {proc.returncode}: {printable}")
        if check:
            raise ReplicationError(printable)
    return proc


def run_script(script_name: str) -> None:
    script_path = CODE_DIR / script_name
    if not script_path.exists():
        raise ReplicationError(f"Missing {rel(script_path)}")
    run([sys.executable, str(script_path)])


def list_files(folder: Path, names: Iterable[str], quiet: bool = False) -> int:
    missing = 0
    for name in names:
        path = folder / name
        if path.exists():
            ok(f"{rel(path)} ({human_size(path.stat().st_size)})")
        else:
            missing += 1
            (warn if quiet else fail)(f"Missing {rel(path)}")
    return missing


def command_about(args: argparse.Namespace) -> int:
    banner("COINTEGRATION-BASED DRL CRYPTO STATARB", "GitHub replication command center")
    panel(META.title, f"This package supports the study: {META.study}")
    section("Creator")
    print(f"Name:                 {META.creator}")
    print(f"Indexed creator:      {META.creator_indexed}")
    print(f"Affiliation a:        {META.affiliation_a}")
    print(f"Affiliation b:        {META.affiliation_b}")
    print(f"ORCID:                {META.orcid}")
    print(f"Corresponding author: {META.creator}")
    print(f"E-mail:               {META.email}")
    section("Archive")
    print(f"Version:              {META.version}")
    print(f"DOI:                  {META.doi}")
    print(f"DOI URL:              {META.doi_url}")
    section("Empirical design")
    print("Market data:          30-minute USDT-denominated Binance spot OHLCV")
    print("Training period:      2023-01-01 to 2024-12-10")
    print("Separation window:    2024-12-11 to 2024-12-17")
    print("Testing period:       2024-12-18 to 2025-05-17")
    print("Agents:               DQN, PPO, A2C")
    print("Benchmark:            COIN")
    print("Action spaces:        binary and granular")
    print("Universes:            Baseline 14, Broad 20, Payment-Coin 6, Layer-1 Platform 9")
    return 0


def command_doctor(args: argparse.Namespace) -> int:
    banner("REPLICATION DOCTOR", "Repository, dependency, and file checks")
    section("System")
    print(f"Python:   {platform.python_version()} ({sys.executable})")
    print(f"Platform: {platform.platform()}")
    print(f"Root:     {ROOT}")
    if sys.version_info < (3, 10):
        warn(f"Python {META.python_min}+ is recommended.")
    else:
        ok(f"Python version satisfies {META.python_min}+")

    section("Required files")
    missing = 0
    for item in REQUIRED_FILES:
        path = ROOT / item
        if path.exists():
            ok(item)
        else:
            fail(item)
            missing += 1

    section("Dependencies")
    try:
        import matplotlib  # type: ignore
        ok(f"matplotlib {matplotlib.__version__}")
    except Exception as exc:
        warn(f"matplotlib unavailable: {exc}")
        warn("Install dependencies with: pip install -r requirements.txt")

    section("Output folders")
    for folder in [TABLES_DIR, FIGURES_DIR, LOGS_DIR]:
        folder.mkdir(parents=True, exist_ok=True)
        ok(rel(folder))

    if not args.skip_scripts:
        section("Input validation script")
        run_script("00_validate_inputs.py")

    if missing:
        fail(f"Doctor found {missing} missing required file(s).")
        return 1
    ok("Repository health check passed.")
    return 0


def command_run(args: argparse.Namespace) -> int:
    banner("ONE-COMMAND REPLICATION", "validate -> tables -> figures -> validate-results")
    for name, script_name, description in WORKFLOW:
        section(f"{name}: {description}")
        if args.skip_figures and script_name == "03_make_figures.py":
            warn("Skipping figure generation because --skip-figures was used.")
            continue
        run_script(script_name)
    ok("Full replication workflow completed.")
    if args.show_summary:
        command_summary(argparse.Namespace())
    return 0


def command_tables(args: argparse.Namespace) -> int:
    banner("RECREATE TABLES", "Rebuild manuscript and supplementary CSV tables")
    run_script("02_recreate_tables.py")
    list_files(TABLES_DIR, EXPECTED_TABLES)
    return 0


def command_figures(args: argparse.Namespace) -> int:
    banner("RECREATE FIGURES", "Rebuild manuscript PNG figures")
    run_script("03_make_figures.py")
    list_files(FIGURES_DIR, EXPECTED_FIGURES)
    return 0


def command_validate(args: argparse.Namespace) -> int:
    banner("VALIDATE RESULTS", "Check reported values and expected outputs")
    run_script("04_validate_results.py")
    return 0


def command_summary(args: argparse.Namespace) -> int:
    banner("REPLICATION SUMMARY", "Key design choices, outputs, and reported values")
    section("Citation")
    print(META.apa)

    section("Asset universes")
    table1 = TABLES_DIR / "Table1_asset_universes.csv"
    if not table1.exists():
        table1 = DATA_DIR / "final_tables" / "Table1_asset_universes.csv"
    if table1.exists():
        for row in read_csv(table1):
            print(f"  {row.get('Universe label', '')}: {row.get('No. of assets', '')} assets")
    else:
        warn("Table1_asset_universes.csv not found. Run: python replicate.py tables")

    section("Key reported values")
    for label, value in KEY_RESULTS:
        print(f"  {label}: {C.bold(value)}")

    section("Tables")
    list_files(TABLES_DIR, EXPECTED_TABLES, quiet=True)
    section("Figures")
    list_files(FIGURES_DIR, EXPECTED_FIGURES, quiet=True)
    return 0


def command_journal_map(args: argparse.Namespace) -> int:
    banner("JOURNAL TABLE PLACEMENT MAP", "Recommended Q1-style table numbering")
    print("Use the journal numbering below when preparing the manuscript. Some package filenames")
    print("retain earlier internal numbering for traceability.")
    print()
    for label, title, path in JOURNAL_TABLE_MAP:
        full = ROOT / path
        status = "available" if full.exists() else "missing"
        print(f"{label:12} | {title:58} | {path} [{status}]")
    return 0


def command_cite(args: argparse.Namespace) -> int:
    banner("CITATION", "APA citation, DOI, and BibTeX")
    section("APA")
    print(META.apa)
    section("DOI")
    print(META.doi_url)
    section("BibTeX")
    print(textwrap.dedent(f"""
    @misc{{drakopoulou2026drlcryptoarb,
      author       = {{Drakopoulou, Veliota}},
      title        = {{{META.title}}},
      year         = {{2026}},
      version      = {{{META.version}}},
      publisher    = {{Zenodo}},
      doi          = {{{META.doi}}},
      url          = {{{META.doi_url}}}
    }}
    """).strip())
    return 0


def command_metadata(args: argparse.Namespace) -> int:
    banner("ZENODO / GITHUB METADATA", "Creator, affiliations, ORCID, and archive metadata")
    rows = [
        ("Title", META.title),
        ("Related study", META.study),
        ("Creator", META.creator_indexed),
        ("Affiliation a", META.affiliation_a),
        ("Affiliation b", META.affiliation_b),
        ("ORCID", META.orcid),
        ("Corresponding author", META.creator),
        ("E-mail", META.email),
        ("Version", META.version),
        ("DOI", META.doi),
        ("DOI URL", META.doi_url),
        ("Resource type", "Software and data"),
        ("License note", "See LICENSE_DATA_NOTE.md"),
    ]
    for key, val in rows:
        print(f"{key:22}: {val}")
    return 0


def command_urls(args: argparse.Namespace) -> int:
    banner("BINANCE PUBLIC DATA URL BUILDER", "Dry-run or download OHLCV zip files")
    command = [sys.executable, str(CODE_DIR / "01_download_binance_klines.py"), "--all"]
    if args.start:
        command.extend(["--start", args.start])
    if args.end:
        command.extend(["--end", args.end])
    if args.dry_run:
        command.append("--dry-run")
    run(command)
    return 0


def command_hashes(args: argparse.Namespace) -> int:
    banner("SHA-256 CHECKSUMS", "Integrity hashes for generated tables and figures")
    targets = [TABLES_DIR / name for name in EXPECTED_TABLES] + [FIGURES_DIR / name for name in EXPECTED_FIGURES]
    existing = [p for p in targets if p.exists()]
    if not existing:
        warn("No generated outputs were found. Run: python replicate.py run")
        return 1
    LOGS_DIR.mkdir(parents=True, exist_ok=True)
    out = LOGS_DIR / "checksums_sha256_replicate_py.txt"
    with out.open("w", encoding="utf-8") as fh:
        for path in existing:
            line = f"{sha256(path)}  {rel(path)}"
            print(line)
            fh.write(line + "\n")
    ok(f"Wrote {rel(out)}")
    return 0


def command_manifest(args: argparse.Namespace) -> int:
    banner("PACKAGE MANIFEST", "Top-level contents for GitHub and Zenodo")
    manifest = ROOT / "PACKAGE_MANIFEST.csv"
    if manifest.exists():
        rows = read_csv(manifest)
        for row in rows[: args.limit]:
            joined = " | ".join(str(v) for v in row.values())
            print(joined)
        if len(rows) > args.limit:
            info(f"Showing {args.limit} of {len(rows)} manifest rows. Use --limit to show more.")
    else:
        warn("PACKAGE_MANIFEST.csv not found. Printing compact tree instead.")
        return command_tree(argparse.Namespace(depth=2))
    return 0


def command_tree(args: argparse.Namespace) -> int:
    banner("REPOSITORY TREE", "Compact package layout")
    max_depth = max(args.depth, 1)
    ignored = {".git", "__pycache__", ".ipynb_checkpoints"}

    def walk(path: Path, prefix: str = "", depth: int = 0) -> None:
        if depth > max_depth:
            return
        entries = sorted([p for p in path.iterdir() if p.name not in ignored], key=lambda p: (p.is_file(), p.name.lower()))
        for idx, entry in enumerate(entries):
            connector = "`-- " if idx == len(entries) - 1 else "|-- "
            print(prefix + connector + entry.name + ("/" if entry.is_dir() else ""))
            if entry.is_dir():
                walk(entry, prefix + ("    " if idx == len(entries) - 1 else "|   "), depth + 1)

    print(ROOT.name + "/")
    walk(ROOT, depth=1)
    return 0


def command_clean(args: argparse.Namespace) -> int:
    banner("CLEAN GENERATED OUTPUTS", "Remove reproduced outputs under results/")
    if not args.yes:
        warn("This will remove generated tables, figures, and replicate.py checksum logs.")
        warn("Run again with --yes to confirm.")
        return 0
    removed = 0
    for folder, names in [(TABLES_DIR, EXPECTED_TABLES), (FIGURES_DIR, EXPECTED_FIGURES)]:
        for name in names:
            path = folder / name
            if path.exists():
                path.unlink()
                removed += 1
                ok(f"Removed {rel(path)}")
    log = LOGS_DIR / "checksums_sha256_replicate_py.txt"
    if log.exists():
        log.unlink()
        removed += 1
        ok(f"Removed {rel(log)}")
    ok(f"Cleaned {removed} file(s).")
    return 0


def command_zip(args: argparse.Namespace) -> int:
    banner("CREATE RELEASE ZIP", "Archive package for GitHub Releases or Zenodo")
    output = Path(args.output).resolve() if args.output else ROOT.parent / f"{ROOT.name}_v{META.version}_github.zip"
    if output.exists():
        output.unlink()
    archive = shutil.make_archive(str(output.with_suffix("")), "zip", ROOT.parent, ROOT.name)
    ok(f"Created archive: {archive}")
    return 0


def build_parser() -> argparse.ArgumentParser:
    epilog = """
Examples:
  python replicate.py about
  python replicate.py doctor
  python replicate.py run
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
"""
    parser = argparse.ArgumentParser(
        prog="replicate.py",
        description="Fancy GitHub-ready replication CLI for cointegration-based DRL cryptocurrency statistical arbitrage.",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=textwrap.dedent(epilog).strip(),
    )
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    parser.add_argument("--no-color", action="store_true", help="Disable colored terminal output.")
    sub = parser.add_subparsers(dest="command", required=True)

    sub.add_parser("about", help="Show study, author, DOI, and design metadata.").set_defaults(func=command_about)

    p_doctor = sub.add_parser("doctor", help="Check repository health and dependencies.")
    p_doctor.add_argument("--skip-scripts", action="store_true", help="Skip code/00_validate_inputs.py.")
    p_doctor.set_defaults(func=command_doctor)

    p_run = sub.add_parser("run", help="Run the full replication workflow.")
    p_run.add_argument("--skip-figures", action="store_true", help="Skip figure generation.")
    p_run.add_argument("--show-summary", action="store_true", help="Print summary after workflow.")
    p_run.set_defaults(func=command_run)

    sub.add_parser("tables", help="Recreate tables.").set_defaults(func=command_tables)
    sub.add_parser("figures", help="Recreate figures.").set_defaults(func=command_figures)
    sub.add_parser("validate", help="Validate outputs.").set_defaults(func=command_validate)
    sub.add_parser("summary", help="Print key results and outputs.").set_defaults(func=command_summary)
    sub.add_parser("journal-map", help="Show recommended journal table placement.").set_defaults(func=command_journal_map)
    sub.add_parser("cite", help="Print APA and BibTeX citation.").set_defaults(func=command_cite)
    sub.add_parser("metadata", help="Print Zenodo/GitHub metadata.").set_defaults(func=command_metadata)
    sub.add_parser("hashes", help="Write SHA-256 checksums for generated outputs.").set_defaults(func=command_hashes)

    p_manifest = sub.add_parser("manifest", help="Print package manifest preview.")
    p_manifest.add_argument("--limit", type=int, default=40, help="Number of rows to show. Default: 40.")
    p_manifest.set_defaults(func=command_manifest)

    p_tree = sub.add_parser("tree", help="Print compact repository tree.")
    p_tree.add_argument("--depth", type=int, default=2, help="Maximum depth. Default: 2.")
    p_tree.set_defaults(func=command_tree)

    p_urls = sub.add_parser("urls", help="Run Binance Public Data downloader scaffold.")
    p_urls.add_argument("--start", default="2023-01", help="Start month YYYY-MM. Default: 2023-01.")
    p_urls.add_argument("--end", default="2025-05", help="End month YYYY-MM. Default: 2025-05.")
    p_urls.add_argument("--dry-run", action="store_true", help="Print URLs without downloading.")
    p_urls.set_defaults(func=command_urls)

    p_clean = sub.add_parser("clean", help="Remove generated outputs.")
    p_clean.add_argument("--yes", action="store_true", help="Confirm deletion.")
    p_clean.set_defaults(func=command_clean)

    p_zip = sub.add_parser("zip", help="Create release zip.")
    p_zip.add_argument("--output", help="Optional output zip path.")
    p_zip.set_defaults(func=command_zip)
    return parser


def main(argv: Sequence[str] | None = None) -> int:
    global C
    parser = build_parser()
    args = parser.parse_args(argv)
    if args.no_color:
        C = Colors(enabled=False)
    try:
        return int(args.func(args))
    except ReplicationError as exc:
        fail(str(exc))
        return 1
    except KeyboardInterrupt:
        print()
        warn("Interrupted by user.")
        return 130


if __name__ == "__main__":
    raise SystemExit(main())
