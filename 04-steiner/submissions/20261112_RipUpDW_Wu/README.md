# Rip-up-and-reroute with exact Dreyfus-Wagner trees for Steiner tree packing

Submitter: Ken Wu (Independent researcher), 2026-11-12.

Code: https://github.com/KenWuqianghao/qoblib-solvers (`solvers/stp_route/stp_route.cpp`, Apache-2.0). The
repository also holds the validation scripts and the reproduction steps.

## Results

Five instances. The site lists each of them as open, with no best-known value. Each value is lower than the cost
of the generator solution `04-steiner/instances/<instance>/sol.txt` and lower than every value that an open pull
request claims for the instance (#62 and #88, read on 2026-11-12 with upstream `main` at `5538450`). No value is
proven optimal. The lower bound in the table is the sum of the exact Steiner tree costs of the single nets, which
the program prints; it is not in the CSV files, because it is too weak to be useful as an `Optimality Bound`.

| Instance | Generator `sol.txt` | PR #62 | PR #88 | This submission | Lower bound | Run |
| :------- | ------------------: | -----: | -----: | --------------: | ----------: | :-: |
| stp_s080_l2_t5_h0_rs24098 | 2232 | 1904 | 1903 | **1902** | 1860 | P3 |
| stp_s090_l5_t6_h0_rs37235 | 2158 | 1623 | 1619 | **1618** | 1592 | P1 |
| stp_s100_l2_t3_h3_rs24098 | 2277 | 2252 | 2251 | **2250** | 2226 | P3 |
| stp_s100_l4_t4_h3_rs24098 | 2095 | 2053 | 2047 | **2046** | 2018 | P1 |
| stp_s100_l5_t3_h1_rs24098 | 1604 | 1600 | 1599 | **1597** | 1583 | P1 |

The same program was run on more instances. Its other results are equal to or higher than a value in an open
pull request, so they are not part of this submission.

## Method

The method is classical and uses no MIP model. No AI system is part of the optimization method. Each instance is
a layered grid graph with several nets. Each net needs a tree that connects its terminals, no two trees can share
a node, and the objective is the total number of edges. The program `stp_route` does these steps:

1. Lower bound: the sum over the nets of the exact Steiner tree of each net (Dreyfus-Wagner dynamic program),
   where the other nets block only their terminals.
2. First packing by negotiated congestion (as in the PathFinder router): each net gets an exact node-weighted
   Dreyfus-Wagner tree, and a node that several nets use becomes more expensive in each round, until no node is
   shared. With `--init`, the program starts from an existing packing instead.
3. Rip-up-and-reroute: each net is rerouted exactly with all other nets fixed. Then groups of 2 to 4 nets (a net
   and the nets that block it) are removed and rerouted in several orders. A move is accepted if the total cost
   does not increase.
4. Iterated local search: after a pass with no improvement, 3 to 5 random nets are rerouted with random node
   costs. After 6 such passes, the search restarts from the best packing found. The best packing is written at
   the time limit.

## Runs

| Run | Command |
| :-- | :------ |
| P1 | `stp_route <instance dir> <out.sol> --time 120 --seed 1`, from scratch |
| P2 | `stp_route <instance dir> <out.sol> --time 240 --seed 2 --init <packing from P1>` |
| P3 | `stp_route <instance dir> <out.sol> --time 300 --seed 3 --init <best packing from P1 or P2>` |

One run per stage. On the two P3 instances the chain gave 1904, 1903, 1902 (stp_s080_l2_t5_h0_rs24098) and 2252,
2252, 2250 (stp_s100_l2_t3_h3_rs24098). For these two rows the runtime in the CSV file covers the P3 run only,
and the Remarks column says so. A run is time-limited and randomized: the result depends on the seed and on the
number of passes that fit into the time limit, so a rerun on other hardware can end at a different value.

## Verification

Each solution file was checked on 2026-11-12 with the official checker `04-steiner/check` (`check_steiner`, exit
code 0, same cost as the CSV file) and with an independent Python implementation of the checker
(`harness/validate.py` in the code repository). The Python check is stricter: the edges of each net must form one
tree, with no duplicate edge and no missing net. `uv run --project misc qoblib-check-submission
04-steiner/submissions/20261112_RipUpDW_Wu` passes for all five instances.

## Hardware and software

One shared cloud VM: Intel(R) Xeon(R) Processor @ 2.10GHz, 4 vCPUs, 15 GB RAM, no GPU. Each run is one
single-threaded process at `nice 10`. Up to three solver processes ran at the same time, sometimes with other
jobs on the machine, so the runtimes are not benchmark timings. `stp_route` is C++17 with no dependencies,
compiled with `g++ -std=c++17 -O2` (g++ 13). The runs are from 2026-09-28.

## AI disclosure

`generated`: the solver code and this write-up are predominantly AI-generated.

- Tool: Claude Code, the coding agent of Anthropic, between 2026-09 and 2026-11.
- Parts that the tool produced: the solver source `stp_route.cpp`, the validation and staging scripts, the text
  of the summary CSV files, this README and the pull request text. The tool also started the runs and recorded
  their results.
- Model versions: the session log of 2026-09 records the models `claude-opus-5-5`, `claude-fable-5-1` and
  `claude-opus-5`; it does not record which of them wrote each file of the solver.
  This README and the pull request text were written on 2026-11-12 with model `claude-fable-5-2`.
- The submitter directed the work and is responsible for the submission.
- No AI system is part of the optimization method. The method is the classical search described above.
- Every solution passes the official checker.
