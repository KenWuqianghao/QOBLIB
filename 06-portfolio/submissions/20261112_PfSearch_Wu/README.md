# Exact MIP, local search and per-period block coordinate descent for multi-period portfolio optimization

Submitter: Ken Wu (Independent researcher), 2026-11-12.

Code: https://github.com/KenWuqianghao/qoblib-solvers (Apache-2.0) holds `solvers/pf_ls/pf_ls.py`, which
produced the MIP and ILS rows below, the validation scripts and the reproduction steps. The program of the BCD
rows, `pf_bcd.py`, is not in that repository on 2026-11-12. This README is the reference for those rows; their
`Reference` column says so.

## Results

Fifteen sub-instances (a200 with budget 50, a400 with budget 100). The site lists each of them as open, with no
best-known value. Each value is lower than every value that an open pull request claims for the key. The open
pull requests of this class are #44, #64, #87 and #92 (read on 2026-11-12 with upstream `main` at `5538450`);
only #64 and #87 claim values for these keys. The objective is minimized.

| Key | PR #64 | PR #87 | This submission | Gain over the best claim | Type | Run | CPU s |
| :-- | -----: | -----: | --------------: | -----------------------: | :--- | :-- | ----: |
| a200_t10_orig_b050_l1e-03 | -792414 | -839888 | **-841017** | 1129 (0.13 %) | heuristic | BCD, warm | 348.4 |
| a200_t10_s00_b050_l1e-03 | -733940 | -827881 | **-833925** | 6044 (0.73 %) | heuristic | BCD, warm | 349.8 |
| a200_t10_s01_b050_l1e-03 | -671077 | -717799 | **-727288** | 9489 (1.32 %) | heuristic | BCD, warm | 347.8 |
| a200_t10_s01_b050_l5e-04 | -743266 | -779455 | **-783015** | 3560 (0.46 %) | heuristic | BCD, warm | 352.3 |
| a200_t10_s02_b050_l1e-03 | -651157 | -731041 | **-733780** | 2739 (0.37 %) | heuristic | BCD, warm | 348.7 |
| a200_t15_orig_b050_l1e-03 | -1251396 | -1328900 | **-1335103** | 6203 (0.47 %) | heuristic | BCD, warm | 349.5 |
| a200_t15_orig_b050_l1e-04 | -1615467 | -1615873 | **-1615929** | 56 (0.003 %) | heuristic | BCD | 351.5 |
| a200_t15_orig_b050_l5e-04 | -1372185 | -1397499 | **-1398414** | 915 (0.07 %) | heuristic | BCD, warm | 353.1 |
| a200_t15_s00_b050_l1e-05 | -2177885 | -2177886 | **-2177889** | 3 (0.0001 %) | heuristic | ILS | 59.7 |
| a200_t15_s01_b050_l1e-03 | -989382 | -1089152 | **-1094484** | 5332 (0.49 %) | heuristic | BCD, warm | 350.0 |
| a200_t15_s01_b050_l1e-05 | -2011462 | -2011450 | **-2011505** | 43 (0.002 %) | heuristic | ILS | 59.9 |
| a200_t15_s01_b050_l5e-04 | -1107155 | -1188800 | **-1191036** | 2236 (0.19 %) | heuristic | BCD | 353.0 |
| a200_t15_s02_b050_l0 | -2200644 | -2200673 | **-2200702** | 29 (0.001 %) | proven optimum | MIP | 1.5 |
| a400_t10_orig_b100_l0 | -2695274 | -2695274 | **-2695464** | 190 (0.007 %) | proven optimum | MIP | 2.1 |
| a400_t15_s02_b100_l0 | -4506693 | -4506693 | **-4506741** | 48 (0.001 %) | proven optimum | MIP | 3.2 |

The three lambda = 0 rows are proven optima and have `Optimality Bound` = `Best Objective Value`. The other
twelve rows are heuristic results with `Optimality Bound` = N/A. The same programs were run on more keys. Their
other results are equal to or higher than a value in an open pull request, so they are not part of this
submission.

## Method

All methods are classical. No AI system is part of the optimization method. All programs use the model of the
checker (`bqp_u3_c10.zpl`): integer unit counts 0 to 3 for each asset, direction and period, with the integer
coefficients and the rounding of the checker, so each step is scored exactly.

**MIP (lambda = 0), `pf_ls.py`.** For lambda = 0 the objective is linear. The program solves the count MIP with
SCIP (through OR-Tools). It reports a proven optimum only if CP-SAT also proves the same value optimal.

**ILS (lambda > 0), `pf_ls.py`.** Iterated local search from the empty portfolio with exact integer deltas. The
moves are single-variable block moves over a range of periods, all two-variable moves (hedges and swaps), and
random block kicks.

**BCD (lambda > 0), `pf_bcd.py`.** The periods interact only through the rebalancing cost between consecutive
periods. The program does a block coordinate descent over the periods:

1. Start from the best of the start solutions (the empty portfolio and, for the rows marked "warm", our own
   earlier solution for the same key).
2. For each period in random order, with both neighbour periods fixed: an exact integer iterated local search on
   that period for 4 seconds. The moves are +1 and -1 on one unit count and all swaps between two unit counts,
   with exact deltas that include the rebalancing cost to both neighbours. A kick is 6 random swaps or adds. Up
   to 12 distinct local optima of each visit go into the pool of that period (at most 40 entries for each
   period).
3. After each sweep over all periods, an exact dynamic program along the chain of periods selects one pool entry
   for each period, with the exact rebalancing cost between consecutive entries.
4. When the time limit is near, a multi-period iterated local search (the ILS above) polishes the result for
   15 seconds.

A dynamic program along the chain of periods over per-period candidate sets is also the method that the READMEs
of the open pull requests #87 and #92 describe; those READMEs led to this decomposition. For a comparison of cost
terms (risk and rebalancing cost), one solution file of PR #87 (`a200_t10_orig`, lambda 1e-3) was read. No
solution of another submission was used as a start solution or as a pool entry.

## Runs

| Run | Command | Date |
| :-- | :------ | :--- |
| MIP, ILS | `pf_ls.py --base <base> --lambdas 0,0.000001,0.00001 --time 60` (60 s for each lambda, seed 1) | 2026-09-28 |
| BCD | `pf_bcd.py --base <base> --lam <lambda> --init zero --solver ls --time 360 --period-time 4 --polish 15 --seed 1` | 2026-10-20 |
| BCD, warm | the BCD command with `--init zero <our earlier solution for the key>` | 2026-10-20 |

One run for each row. A warm BCD run started from our earlier solution for the same key, which came from the
factor-space MIQP program `pf_miqp.py` of the code repository (convex MIQP in the eigenbasis of each period's
covariance matrix, solved with SCIP through PySCIPOpt, then an ILS polish). The run for
`a200_t10_orig_b050_l1e-03` also started from the results of two earlier pilot runs of `pf_bcd.py` on that key
(300 s with `--solver miqp`, which solves each period as a MIQP with SCIP, and 600 s with `--solver ls`). For
the warm rows, the runtime in the CSV file covers the last run only; the Remarks column
gives the best start value. The heuristic runs are time-limited and randomized, so a rerun can end at a
different value. The proven optima reproduce exactly.

## Verification

Each solution file was checked on 2026-11-12 with the official checker `06-portfolio/check`
(`check_portfolio`, exit code 0, same objective as the CSV file and as the `objective` line of the solution
file) and with an independent Python implementation of the checker (`harness/pf_model.py` in the code
repository). `uv run --project misc qoblib-check-submission 06-portfolio/submissions/20261112_PfSearch_Wu`
passes for all fifteen keys.

File names: each instance directory and its files use the canonical instance key (for example
`a200_t10_s01_b050_l1e-03`: no `po_` prefix, exponent with two digits), in a `po_<base>/` folder.
`misc/ci/update_bkv.py` looks for `<canonical key>_solution.*` when it credits a best-known value.

## Hardware and software

- MIP and ILS rows: one shared cloud VM, Intel(R) Xeon(R) Processor @ 2.10GHz, 4 vCPUs, 15 GB RAM, no GPU.
  Python 3.11, NumPy 2.4.6, OR-Tools 9.15.6755 (SCIP and CP-SAT).
- BCD rows: AMD Custom APU 0932 (Steam Deck), 4 cores and 8 threads, 14 GB RAM, no GPU. Python 3.13, NumPy 2.5.4.

Each run is one single-threaded process at `nice 10` on a machine that other jobs used at the same time. The
runtimes are CPU seconds of the process and are not benchmark timings.

## AI disclosure

`generated`: the solver code and this write-up are predominantly AI-generated.

- Tool: Claude Code, the coding agent of Anthropic, between 2026-09 and 2026-11.
- Parts that the tool produced: the solver sources (`pf_ls.py`, `pf_miqp.py`, `pf_bcd.py`), the validation and
  staging scripts, the text of the summary CSV files, this README and the pull request text. The tool also
  started the runs and recorded their results.
- Model versions: the session log of 2026-09 (`pf_ls.py`, `pf_miqp.py`) records the models
  `claude-opus-5-5`, `claude-fable-5-1` and `claude-opus-5`; it does not record which of them wrote each file.
  The session of 2026-10-20 (`pf_bcd.py`) used `claude-opus-5-5`. This README and the pull request text were written on 2026-11-12 with model
  `claude-fable-5-2`.
- The submitter directed the work and is responsible for the submission.
- No AI system is part of the optimization method. The methods are the classical methods described above.
- Every solution passes the official checker.
