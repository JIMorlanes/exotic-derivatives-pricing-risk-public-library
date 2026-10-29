# Numerical validation: four representative cases

This report preserves the original private-project results. Source and test
names refer to the private implementation; this public folder contains documentation and diagrams, not the executable,
tests or source files.

Recorded **2026-10-27** from the actual C++23 Release test executables.
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

Results below come from full-precision test output and are rounded only for
presentation. Analytical reference error, Monte Carlo standard error and a
comparison between two estimates are distinguished explicitly.

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
| Plain MC | 6.210408504793 | 0.028662855854 | [6.154229307318, 6.266587702267] |
| Control variate | 6.199636814876 | 0.000808324964 | [6.198052507882, 6.201221121869] |

The absolute **difference between estimates** is **0.010771689917**.
The reported SE ratio is **35.46**, and the squared SE ratio is
**1257.38**. These are estimated uncertainty reductions for this run,
not measured runtime gains or empirical variance across independent seeds.

The retained integration check requires the price difference to be at most
`4 * (SE_plain + SE_control)`, here **0.117884723274**, and requires
the control-variate SE to be smaller. The SE sum is a conservative scale for
the correlated difference; it is not an independently estimated difference SE.

There is no analytical arithmetic-Asian price in this comparison, so a true
pricing error is **not available**. Plain MC uses estimate ± 1.96 SE; fitted CV
uses the existing regression SE and a Student-t interval with 99,998 residual
degrees of freedom. CV intervals are approximate for nonlinear payoffs and an
in-sample fitted coefficient. One seeded comparison does not establish bias,
convergence or confidence-interval coverage.

Test: `asian_mc_and_control_variate_integration`.
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
| 3.2407203551989525 | 3.206613945679 | 0.034106409519 | 0.027317495978 | [3.153071653563, 3.260156237796] |

The error is **1.248519 SE**. The existing regression criterion is
absolute error ≤ **4 SE = 0.109269983911**, which passes. This is a
normal-approximation sampling diagnostic, not a deterministic price tolerance
or a claim of exact 95% coverage. No new barrier formula was added.

Test: `down_and_out_matches_analytical_reference`.

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

## Original validation and command examples

Recorded toolchain: macOS arm64, **Apple Clang 21.0.0**, **CMake 4.2.1**, C++23,
Release (`-O3 -DNDEBUG`), Boost headers and OS threads. Python 3.12.15 runs the
existing CLI integration tests. There is no Eigen3 dependency. Standard-library
normal generation can differ across platforms; bit-for-bit cross-toolchain MC
results are not promised.

**Observed: build succeeded; all 10 registered CTest tests passed, zero
failures (6.19 seconds for this suite run).** Test runtime is not a pricing
performance or speedup benchmark. No extra statistical CTest targets remain.

The build and test executables used for the recorded results remain private.
The following commands document the corresponding CLI cases for the planned
browser interface backed by the private Linux executable. CLI output uses nine
decimal places; the original
full-precision test records and independent reference calculations are not
regenerated by these commands. Monte Carlo streams may differ across platforms.

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

These four recorded cases provide focused regression evidence for the stated
parameters; they do not establish validity over every contract or market
configuration.

See [README](../README.md) and [architecture documentation](architecture.md).
