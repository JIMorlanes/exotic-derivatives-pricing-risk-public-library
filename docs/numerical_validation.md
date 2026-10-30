# Numerical validation: four representative cases

This report combines current Linux CLI measurements with original private-project
validation records. Source and test names refer to the private implementation;
this public repository contains the Linux CLI executable, documentation and
diagrams. Source files and tests remain private.

Sections B and C use Linux Codespaces CLI output supplied on **2026-10-30**.
Sections A and D retain the original macOS Release-test results recorded on
**2026-10-27**. The complete six-case Linux comparison is in the README.
This report covers only four representative cases. The additional parameter
grids and repeated-seed experiments have been removed; existing core unit tests
are retained. No pricing formulas or model features were changed.

Prices are per unit of underlying. `carry` is the dividend yield `q`, so the
risk-neutral GBM drift is `r - q`. Unless a case overrides them, parameters are:

| Parameter | Value |
| --- | --- |
| Spot / strike | 100 / 100 |
| Maturity | 1 year |
| Volatility | 0.20 |
| Risk-free rate / dividend yield | 0.05 / 0.02 |

Sections A and D use full-precision private test records. Sections B and C use
the CLI's nine-decimal output; their derived comparisons use those reported
values. Analytical reference error, Monte Carlo standard error and a comparison
between two estimates are distinguished explicitly.

## A. European options

**One market, call and put, analytical pricing.** The references independently
evaluate the dividend-adjusted Black–Scholes formulas in
[the textbook, section 9.2](https://book.derivative-securities.org/Chapter_BlackScholes.html).
These are formula-derived values for our parameters, not published numerical
examples. They were evaluated with an 80-digit Python `Decimal` context and
rounded to `double`; the production pricer was not used to generate them.

With $d_1=[\log(S/K)+(r-q+\sigma^2/2)T]/(\sigma\sqrt T)$ and
$d_2=d_1-\sigma\sqrt T$, the reference formulas are

$$
C=Se^{-qT}\Phi(d_1)-Ke^{-rT}\Phi(d_2),\qquad
P=Ke^{-rT}\Phi(-d_2)-Se^{-qT}\Phi(-d_1).
$$

| Option | Independent reference | Library analytical price | Absolute error |
| --- | --- | --- | --- |
| Call | 9.227005508154047 | 9.2270055081540363 | 1.066e-14 |
| Put | 6.3300806275499184 | 6.3300806275499246 | 6.217e-15 |

Acceptance: absolute error **< 1e-12**, preserving the existing analytical test's
tolerance. No Monte Carlo uncertainty applies to this case.

Test: `european_calls_and_puts_use_existing_pricer`.

## B. Asian options

**One arithmetic Asian call: plain MC versus geometric control variate.** Use
the common market, four equally spaced fixings at **0.25, 0.50, 0.75, 1.00**,
**100,000 paths**, and **seed 42** for both methods. Time-zero spot is excluded
from the average; payment is at maturity. Both methods use the same simulated
paths, so their estimates are correlated.

The control uses the geometric payoff's analytical expectation for the same
fixings. Sources: [discrete geometric Asian equations 11.5–11.7](https://book.derivative-securities.org/Chapter_Asians.html)
and [geometric control variates, section 13.3](https://book.derivative-securities.org/Chapter_MonteCarlo.html).
This is a comparison at our chosen parameters, not a quoted textbook price.

| Method | Estimate | Standard error | Nominal 95% interval |
| --- | --- | --- | --- |
| Plain MC | 6.227787630 | 0.028642284 | [6.171648753, 6.283926507] |
| Control variate | 6.200066455 | 0.000816064 | [6.198466981, 6.201665930] |

The absolute **difference between estimates** is **0.027721175**.
The reported SE ratio is **35.10**, and the squared SE ratio is
**1231.88**. These are estimated uncertainty reductions for this run,
not measured runtime gains or empirical variance across independent seeds.

The original private integration check requires the price difference to be at most
`4 * (SE_plain + SE_control)`, here **0.117833392**, and requires
the control-variate SE to be smaller. The SE sum is a conservative scale for
the correlated difference; it is not an independently estimated difference SE.
The reported Linux values satisfy both inequalities; this comparison is not a
rerun of the private test executable.

There is no analytical arithmetic-Asian price in this comparison, so a true
pricing error is **not available**. Plain MC uses estimate ± 1.96 SE; fitted CV
uses the existing regression SE and a Student-t interval with 99,998 residual
degrees of freedom. CV intervals are approximate for nonlinear payoffs and an
in-sample fitted coefficient. One seeded comparison does not establish bias,
convergence or confidence-interval coverage.

Original private regression test: `asian_mc_and_control_variate_integration`.
Its existing put, geometric-pricing and reproducibility assertions are preserved.

## C. Barrier options

**One continuously monitored, zero-rebate down-and-out call.** Match the
[published numerical example following equation 12.9](https://book.derivative-securities.org/Chapter_OtherExotics.html)
exactly: **S=100, K=120, barrier=80, T=1, sigma=0.20, r=0.05, q=0**.
Use **100,000 paths**, **52 equally spaced bridge intervals**, and **seed 42**.
The published analytical price is **3.2407203551989525**.

The library evolves exact GBM endpoints and weights the terminal payoff by
conditional Brownian-bridge survival probabilities. The 52 intervals specify
the simulation grid; the reference contract remains continuously monitored.

| Published reference | MC estimate | Absolute error | Standard error | Nominal 95% interval |
| --- | --- | --- | --- | --- |
| 3.2407203551989525 | 3.206151383 | 0.034568972 | 0.027315764 | [3.152612485, 3.259690280] |

The error is **1.265532 SE**. The existing regression criterion is
absolute error ≤ **4 SE = 0.109263056**, which the reported Linux values satisfy. This is a
normal-approximation sampling diagnostic, not a deterministic price tolerance
or a claim of exact 95% coverage. No new barrier formula was added.

Original private regression test: `down_and_out_matches_analytical_reference`.

## D. Numerical risk

**One European call at the common market.** Reuse the existing deterministic
Black–Scholes repricing report from `RiskEngine`, with central-difference bumps
in factor order **Spot, Volatility, Rate, Carry, Maturity**:

```
(0.01, 0.001, 0.001, 0.001, 1/365)
```

The existing test uses 14 workers and seed 12345. The analytical callback is
deterministic, so the seed has no numerical effect in this comparison.

Delta, Gamma and Vega references follow
[the textbook, section 9.4](https://book.derivative-securities.org/Chapter_BlackScholes.html).
The mixed derivatives are independently derived by differentiating those
formulas, not published numerical examples or calls to `BlackScholesGreeks`:

$$
V_{S\sigma}=-e^{-qT}\phi(d_1)d_2/\sigma,\qquad
V_{Sr}=e^{-qT}\phi(d_1)\sqrt T/\sigma,\qquad
V_{\sigma r}=-\mathrm{Vega}\,d_1\sqrt T/\sigma.
$$

Vega and rate derivatives use absolute parameter units: a volatility change of
1.0 or a rate change of 1.0, rather than one percentage point or one basis point.

| Derivative | Independent analytical reference | RiskEngine | Absolute error | Allowed absolute error |
| --- | --- | --- | --- | --- |
| Delta | 0.586851146135 | 0.586851139029 | 7.106e-09 | 1.0e-06 |
| Gamma | 0.018950578755 | 0.018950578671 | 8.427e-11 | 1.0e-07 |
| Vega | 37.901157510017 | 37.901145295514 | 1.221e-05 | 5.0e-05 |
| Spot–Volatility | -0.094752893775 | -0.094759851565 | 6.958e-06 | 1.0e-05 |
| Spot–Rate | 1.895057875501 | 1.895050402911 | 7.473e-06 | 1.0e-05 |
| Volatility–Rate | -47.376446887522 | -47.376556468137 | 1.096e-04 | 5.0e-04 |

The fixed derivative-accuracy targets above allow central-difference truncation
and floating-point cancellation at the stated bumps. All six comparisons pass.
They are deterministic checks, not Monte Carlo confidence bounds or certification
of all 25 Black–Scholes Hessian entries. Existing linear/quadratic Hessian,
input-validation and analytical call/put Greek tests remain intact.

Test: `test_risk_engine.cpp`, using the existing
call report plus independent test-only analytical derivatives.

## Original macOS validation and command examples

Recorded toolchain: macOS arm64, **Apple Clang 21.0.0**, **CMake 4.2.1**, C++23,
Release (`-O3 -DNDEBUG`), Boost headers and OS threads. Python 3.12.15 runs the
existing CLI integration tests. There is no Eigen3 dependency. Standard-library
normal generation can differ across platforms; bit-for-bit cross-toolchain MC
results are not promised.

**Original macOS validation: build succeeded; all 10 registered CTest tests passed, zero
failures (6.19 seconds for this suite run).** Test runtime is not a pricing
performance or speedup benchmark. No extra statistical CTest targets remain.

The macOS build and test executables used for sections A and D remain private.
The commands below run corresponding CLI cases using the supplied Linux
executable in the Codespace terminal. Sections B and C report measurements from
this Linux CLI; they do not regenerate the original full-precision test records
or independent reference calculations.

```sh
# European analytical call
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 --volatility 0.20 --rate 0.05 --carry 0.02

# Arithmetic Asian: plain MC, then geometric control variate
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 --volatility 0.20 --rate 0.05 --carry 0.02 --product asian --average arithmetic --steps 4 --method mc --paths 100000 --seed 42
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 --volatility 0.20 --rate 0.05 --carry 0.02 --product asian --average arithmetic --steps 4 --method cv --paths 100000 --seed 42

# Continuous down-and-out barrier
pricing_cli price --type call --spot 100 --strike 120 --maturity 1 --volatility 0.20 --rate 0.05 --carry 0 --product barrier --barrier-type down-out --barrier 80 --steps 52 --method mc --paths 100000 --seed 42

# Price, Greeks and full five-factor Hessian
pricing_cli risk --type call --spot 100 --strike 100 --maturity 1 --volatility 0.20 --rate 0.05 --carry 0.02 --threads 2
```

These four cases provide focused evidence at the stated parameters. The Linux
CLI comparisons in B and C supplement the original private validation; they do
not establish validity over every contract or market configuration.

See [README](../README.md) and [architecture documentation](architecture.md).
