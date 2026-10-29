# C++23 Exotic Derivatives Pricing & Risk Library — Public CLI Demo

Binary-only demonstration of a private C++23 library for pricing European, Asian
and continuously monitored barrier options under geometric Brownian motion (GBM).
Combines analytical valuation, Monte Carlo variance reduction, parallel five-factor risk and
selectable implied-volatility solvers.

The financial focus is valuation with explicit sampling uncertainty and first-
and second-order sensitivities. Analytical prices, parity relationships and
seeded experiments provide checks on the numerical methods.

This public repository includes the compiled Linux x86-64 CLI, architecture,
numerical validation and UML diagrams. The C++ source, headers and test code
remain private.

## Run in GitHub Codespaces

1. [Open this repository in GitHub Codespaces](https://codespaces.new/JIMorlanes/exotic-derivatives-pricing-risk-public-library?quickstart=1),
   or select **Code → Codespaces → Create codespace on main** on GitHub.
2. Wait for the container setup to finish.
3. Run the commands below in the Codespace terminal, starting with:

```sh
pricing_cli --help
```

The container uses the supplied `bin/pricing_cli` executable and adds `bin/` to
`PATH`. It installs the Linux runtime libraries; it does not compile the project.
A GitHub account is required, and Codespaces usage is subject to your account's
allowance. If you created a Codespace before this configuration was added,
rebuild its container.

## CLI demonstration

Arithmetic Asian call with a geometric Asian control variate: four quarterly
fixings, 100,000 paths and seed 42.

```sh
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 \
  --volatility 0.20 --rate 0.05 --carry 0.02 --product asian --average arithmetic --steps 4 \
  --method cv --paths 100000 --seed 42
```

Actual output from the Release executable:

```text
============================================================================================
PRICING REPORT | Arithmetic Asian Call | Model: GBM
============================================================================================
Method: Control-variate Monte Carlo

OPTION PRICE               :          6.199636815
-------------------------------------------------

MONTE CARLO ESTIMATE
Standard error             :          0.000808325
95% confidence interval    : [6.198052508, 6.201221122]
Paths                      :               100000
```

## Pricing and risk capabilities

All products support calls and puts.

| Product | Analytical pricing | Monte Carlo methods |
|---|---|---|
| European | Black–Scholes | Plain MC; antithetic variates |
| Geometric Asian | Discrete geometric-average formula | Plain MC |
| Arithmetic Asian | — | Plain MC; geometric Asian control variate |
| Up/down, in/out barriers | — | Conditional Brownian-bridge survival weighting |

- **Continuous barriers:** zero-rebate valuation accounts for crossings between
  simulated endpoints through conditional survival weights.
- **Risk:** Boost.Asio parallel repricing produces a five-factor gradient and
  full 5×5 Hessian in spot, volatility, rate, carry and maturity. Reports include
  Delta, Gamma, Vega, Rho, carry sensitivity and annual Theta.
- **Monte Carlo uncertainty:** option-price standard error, nominal 95%
  confidence interval and path count; common random numbers for bumped risk.
- **Implied volatility:** European Black–Scholes inversion using
  `--solver toms748|bisection|newton`; TOMS 748 is the default.
- **CLI:** `price`, `risk`, `iv` and `--help`, with input validation and explicit
  errors for unsupported combinations.

## Measured numerical results

Recorded on 2026-10-27; calls, 100,000 paths, seed 42. Base market:
S=K=100, T=1 year, volatility=20%, r=5%, q=2%; Asian fixings at
0.25, 0.50, 0.75 and 1.00 years. The barrier reference case instead uses K=120,
q=0, barrier=80 and 52 bridge intervals.

| Case | Analytical/reference | MC estimate | Standard error |
| --- | --- | --- | --- |
| European: plain MC | 9.227005508 | 9.288828450 | 0.044039111 |
| European: antithetic MC | 9.227005508 | 9.229877274 | 0.032511041 |
| Geometric Asian: MC | 6.016110665 | 6.026632045 | 0.027985661 |
| Arithmetic Asian: MC | — | 6.210408505 | 0.028662856 |
| Arithmetic Asian: control variate | — | 6.199636815 | 0.000808325 |
| Down-and-out barrier: MC | 3.240720355 | 3.206613946 | 0.027317496 |

For the arithmetic Asian call, the control variate gave **35.46× lower
reported standard error** than plain MC at the same path count. This measures
sampling uncertainty, not runtime speedup. No analytical arithmetic-Asian price
is claimed.

The [numerical validation report](docs/numerical_validation.md) focuses on four
representative comparisons: analytical European calls/puts, arithmetic Asian
MC versus control variate, a published continuous-barrier benchmark, and
analytical Greeks with selected mixed Hessian entries. Parameters, errors,
uncertainty and reproduction commands accompany the results.
**Original private validation: all 10 registered CTest tests passed locally.**

## C++23 engineering design

The following describes the private implementation behind the executables.

```text
CLI → PricingRequest → GBMPricingEngine → numerical pricers → PricingResult
```

- **Dynamic polymorphism at the API boundary:** the virtual `PricingEngine`
  contract lets API callers depend on a common engine interface.
  `GBMPricingEngine` validates and dispatches supported product/method combinations.
- **Type-safe runtime product selection:** `std::variant` stores product/model
  alternatives; `std::visit` selects the product at runtime and `if constexpr`
  chooses its compiled branch.
- **Static polymorphism in numerical components:** the model, RNG and statistics
  policies of `MonteCarloPricer<Model, RNG, Statistics>` are bound at compile time,
  exposing concrete types for compiler specialization and inlining.
  `PathGenerator<Model, RNG>` follows the same model/RNG policy design; references
  are non-owning.
- Products own their fixing/monitoring schedules. Market inputs and pricing
  results remain separate value types.
- `RiskEngine::calculate` accepts a pricing callable and owns a Boost.Asio worker
  pool. The CLI retains the base pricing statistics and constructs local RNGs
  for bumped calculations using a common seed.

UML diagrams (PNG):

1. [Pricing architecture](./docs/uml/01_pricing_architecture.png)
2. [Pricing, simulation and policies](./docs/uml/02_pricing_simulation_policies.png)
3. [Monte Carlo execution](./docs/uml/03_monte_carlo_execution.png)
4. [Implied volatility and solvers](./docs/uml/04_implied_volatility_solvers.png)
5. [Finite-difference risk and Hessian](./docs/uml/05_risk_numerics_detail.png)

## CLI command examples

Run these commands in the Codespace terminal.

```sh
# Analytical European call
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 \
  --volatility 0.20 --rate 0.05 --carry 0.02

# Price, Greeks and full Hessian
pricing_cli risk --type call --spot 100 --strike 100 --maturity 1 \
  --volatility 0.20 --rate 0.05 --carry 0.02 --threads 4

# Select any of the three IV solvers; --method is reserved for pricing
pricing_cli iv --type call --spot 100 --strike 100 --maturity 1 \
  --rate 0.05 --carry 0.02 --market-price 9.227005508154036 --solver newton

pricing_cli --help
```

Rates and volatility are decimals; maturity and observation times are years.
MC requires `--paths` and `--seed`. Asian/barrier grids use exactly one of
`--steps N` or `--times t1,t2,...`. Detailed examples are in the validation report.
Fitted Asian CV needs at least three paths. A single-fixing CV uses an exact
control identity and accepts two paths; all requested paths are still simulated.

## Numerical methods and limitations

GBM paths use the exact lognormal transition without Euler discretization,
avoiding the associated time-discretization bias. Monte Carlo pricing uses a
streaming design to limit memory use: Welford's algorithm updates mean and sample
variance in one pass with centered updates that reduce cancellation, while
control-variate covariance is accumulated online. Asian and barrier pricing
retain running payoff aggregates instead of complete paths or payoff samples.
For a fixed observation schedule, working memory per pricing call does not grow
with the number of simulated paths.

Continuous zero-rebate barriers use conditional Brownian-bridge survival weights
to account for crossings between simulated endpoints. To reduce sampling
uncertainty, European antithetic MC pairs opposite normal draws, while arithmetic
Asian control variates use the correlated geometric payoff with its analytical
mean. Antithetic errors are estimated from pair averages.

Risk uses central finite differences so the same pricing callable supplies
first- and second-order sensitivities. Boost.Asio distributes the separate bumped
repricings across worker threads to compute the five-factor gradient and full
symmetric Hessian. For Monte Carlo, common seeds help reduce sampling noise in
the price differences; Theta is `-dPrice/dMaturity`.

Scope is constant-parameter GBM, European/Asian payoffs and continuous zero-rebate
barriers. Heston pricing, zero-volatility/expiry limiting formulas and general
rebates are unsupported. Spot, strike, volatility and maturity must be positive;
finite negative rates/carry are accepted.

Plain/antithetic MC intervals use a normal approximation. Fitted CV uses an
approximate regression interval with residual degrees of freedom and a Student-t
multiplier; three samples is a minimum, not a reliable-inference recommendation.
Greek/Hessian uncertainty is not estimated, and nonsmooth payoffs remain
sensitive to bumps and sample size.
Seeded streams are reproducible within the same standard-library environment.
TOMS 748 and Bisection are bracketed; Newton uses analytical Vega and may fail
when an iterate leaves the admissible interval or Vega is too small. There is
no automatic fallback. `--max-evaluations` counts function evaluations for
TOMS 748 and iterations for Bisection/Newton.

## Detailed documentation

- [Numerical validation and reproducible experiments](docs/numerical_validation.md)
- [Architecture, four UML class views and execution sequences](docs/architecture.md)
