# microstructure-lab

Simulation-based toolkit for market microstructure research: self-exciting order flow (Hawkes processes), optimal execution, leakage-free alpha validation, and a C++20 limit-order-book matching engine verified trade-by-trade against independent Python implementations.

> **Scope.** Everything here runs on **simulated data with known ground truth**. There is no market data and no live trading. The goal is to show that each method is implemented correctly and that its claims can be checked against a known answer.

## Results (reproduce with `python experiments/run_all.py`)

All numbers below come from `results/results.json`, produced by that script.

### 1. Hawkes process, maximum likelihood
Exponential kernel, true parameters (mu, alpha, beta) = (0.5, 0.6, 2.0), horizon T = 2000, about 2,500 events per path, 300 simulated paths. Exact simulation by Ogata thinning, O(n) likelihood via the recursive state, Wald intervals from the numerical Hessian.

| | mu | alpha (branching ratio) | beta |
|---|---|---|---|
| mean estimate | 0.4998 | 0.5992 | 2.0152 |
| RMSE | 0.0236 | 0.0211 | 0.1319 |
| 95% interval coverage | 97.0% | 96.7% | 94.0% |

Time-rescaling test (residuals should be iid Exp(1)): the true model passes at the 5% level in 96.7% of paths; a Poisson model with the same event rate is rejected in 100% of paths. With 300 paths the standard error of a coverage estimate is about 1.3 percentage points, so the coverage is consistent with the nominal 95%.

### 2. Optimal execution
Almgren-Chriss closed-form trajectory, verified against Monte Carlo (200,000 paths): analytic mean cost 5,118,893 vs. simulated 5,119,164 (standard error 372); analytic variance 2.7645e10 vs. simulated 2.7642e10.

Dynamic programming under Markov-switching temporary impact (two regimes, eta = 5e-6 and 1.5e-5; transition matrix [[0.85, 0.15], [0.30, 0.70]]). The policy observes the regime before each step. Improvement of the mean-variance objective over the best static Almgren-Chriss schedule (5 seeds x 100,000 paths, mean +- sd over seeds):

| risk aversion lambda | improvement |
|---|---|
| 2e-6 | 21.5% +- 0.3 |
| 2e-5 | 33.5% +- 0.5 |
| 2e-4 | 68.0% +- 0.3 |

The size of the gain depends strongly on the assumed regime contrast and on lambda. It is a property of this toy model, not a statement about real markets.

### 3. Alpha validation
1,000 simulated signals, 50 with a true effect (0.35 standard deviations per period), 250 observations each, 200 repetitions, level 5%:

| method | false discovery proportion | power |
|---|---|---|
| uncorrected | 48.6% | 99.98% |
| Bonferroni | 0.09% | 92.3% |
| Holm | 0.09% | 92.5% |
| Benjamini-Hochberg | 4.5% | 99.5% |

With **no** true signals, uncorrected testing declares about 49 of 1,000 signals significant, and every one of them is false.

Leakage: persistent features and 20-step overlapping labels, pure noise (true R^2 <= 0). Shuffled 5-fold CV reports R^2 = +0.746 (spurious). Purged 5-fold CV with embargo reports R^2 = -0.608, i.e. no skill, as it should (20 repetitions).

### 4. C++20 matching engine
Price-time priority, limit / market / cancel messages, flat price-level arrays with intrusive FIFO queues.

* **Correctness:** trade-by-trade identical to an independent Python implementation (dict + deque + bisect), which is itself checked against a brute-force oracle. Covered: crossing limits, sweeps over many levels, market orders on an empty book, cancels of live, dead and never-issued ids. A 150,000-message stream gives the same trade count and the same FNV-1a checksum in C++ and Python.
* **Throughput:** median **22.4 million messages/s** (range 19.9 to 26.6 million over 7 runs) on an Intel Xeon @ 2.1 GHz, `g++ -O2`, 10 million synthetic messages, single thread, in memory.

This throughput figure is for a synthetic stream without network, persistence, or risk checks. It is **not** comparable to production exchange systems, and it will differ on your hardware.

## Layout

```
src/mslab/hawkes.py          simulation, likelihood, MLE, time-rescaling test
src/mslab/execution.py       Almgren-Chriss, Monte Carlo check, DP with regime switching
src/mslab/alpha.py           multiple-testing rules, purged K-fold, leakage demo
src/mslab/orderbook_ref.py   brute-force oracle and reference order book
src/mslab/streams.py         random message streams for differential testing
cpp/engine.cpp               C++20 matching engine (run / gen / bench modes)
tests/                       38 tests (pytest)
experiments/run_all.py       reproduces every number and figure above
results/                     results.json and figures
```

## Run

```bash
pip install -r requirements.txt
pytest -q                           # builds the C++ engine and runs all tests
python experiments/run_all.py       # about 2-3 minutes, writes results/
```

Needs Python 3.10+ and a C++20 compiler (g++ 11+ or clang 14+).

## Known limitations

* Simulated data only. Nothing here says anything about real-market profitability.
* The DP recursion minimises E[cost] + lambda * E[Var(cost | regime path)]. It does not penalise the extra variance that comes from the regime path itself, so the policy is not an exact optimiser of E + lambda * Var. Both policies are always compared on the full functional by Monte Carlo. The static baseline's parameter is chosen on the same samples it is evaluated on, which slightly favours it, so the reported improvement is conservative.
* The Hawkes fit uses a single exponential kernel and a one-dimensional process. Real order flow has several event types and heavier-tailed kernels.
* The engine has no modify messages, no self-trade prevention, and a fixed tick range.

## License
MIT
