# Architecture and UML: Exotic Derivatives Pricing & Risk Library

Public documentation of the private implementation. Source and test references
identify the original project; those files are not distributed in this package.
The five PNG views are available in [uml/](uml/).

This document describes the implemented C++23 project. The source files, headers,
CMake targets, examples and tests were inspected directly. It documents the
current application, including reuse of the base `PricingResult` in risk reports.

Class views: [API and products](#21-request-domain-values-and-engine-contract),
[pricing and simulation](#22-pricing-simulation-and-policy-types),
[risk](#23-risk-engine-and-result-structures), and
[IV solvers](#24-implied-volatility-and-numerical-solvers).

## Reading the diagrams

- Component boxes represent modules, data values or functions, not additional
  implementation classes. The CLI is a translation unit containing free functions.
- In class diagrams, `*--` means owned value/subobject, `-->` means a non-owning
  association, `..>` means a use/creation dependency and `<|--` means inheritance.
  The `*` operation marker means abstract; `$` marks a static operation. These
  markers describe UML operations, not C++ pointer or ownership types.
- `<<template>>` marks a real class template. Its parameter list appears in a
  note; policy-binding labels identify the concrete arguments used by the
  dispatcher. These bindings are not inheritance relationships.
- Nodes marked `variant_alias` or `type_alias` represent existing C++ `using`
  declarations. They are not new classes. Variant alternatives are mutually
  exclusive; an object does not contain all alternatives simultaneously.
- Class identifiers are unqualified for readability; their source paths and
  namespaces are identified below. Operation parameter lists in diagrams are
  abbreviated. Exact C++ declarations remain private; diagram signatures are abbreviated.
- Sequence lifelines denote actual objects, free functions, lambdas or external
  library functions. The risk callback is one shared CLI lambda; its captured
  request and base result are read-only. The local-engine lifeline represents
  a separate pricer object per bumped callback invocation.

The API headers live in `include/qf/api/`, but their namespace is **`qf::pricing`**.
The same namespace contains the pricing classes and `PricingResult`.

## 1. High-level component architecture

```mermaid
flowchart TB
    Customer["Customer: terminal arguments"]
    subgraph Application["Application: apps/pricing_cli.cpp"]
        CLI["main / parse_arguments / parse_request"]
        RiskCLI["print_risk and price_at callback"]
        IVCLI["print_iv"]
        Output["Report formatting: stdout; exceptions: stderr, exit 1"]
    end
    subgraph API["API: include/qf/api and src/api"]
        Request["PricingRequest: market, product variant, model variant, simulation, method"]
        Interface["PricingEngine: abstract virtual price interface"]
        Engine["GBMPricingEngine: validation and product/method dispatch"]
    end
    subgraph Quant["Existing quantitative library"]
        Analytical["BlackScholesPricer / GeometricAsianPricer"]
        MC["MonteCarloPricer: European, Asian, barrier, antithetic, control variate"]
        Policies["GBM, RNGs, payoffs, PathGenerator, statistics"]
        Risk["RiskEngine: finite differences and parallel repricing"]
        IV["ImpliedVolatilitySolver"]
        Root["Selected local solver: Toms748 / Bisection / Newton"]
        Vega["BlackScholesGreeks::vega: analytical Newton derivative"]
        Result["PricingResult: price, standard error, confidence bounds, paths"]
        RiskResult["RiskReport: price, gradient, Hessian"]
    end
    Customer --> CLI
    CLI --> Request
    CLI -->|price command| Engine
    Request -->|input value| Engine
    Engine -.->|inherits| Interface
    Engine --> Analytical
    Engine --> MC
    MC --> Policies
    Analytical -->|double wrapped by dispatcher| Result
    MC --> Result
    CLI -->|risk command| RiskCLI
    RiskCLI -->|validate and retain base result| Engine
    RiskCLI -->|calculate with callback| Risk
    Risk -->|invoke supplied callback| RiskCLI
    RiskCLI -->|bumped PricingRequest| Engine
    Risk --> RiskResult
    CLI -->|iv command| IVCLI
    IVCLI -->|validate bracket prices| Engine
    IVCLI --> IV
    IV --> Root
    Root -->|evaluate objective lambda| IV
    IV -->|Newton derivative only| Vega
    IV -->|objective uses owned BlackScholesPricer| Analytical
    Result --> Output
    RiskResult --> Output
    IVCLI -->|implied volatility double| Output
    Output --> Customer
```

These are source-level layers, not separate services or separate API libraries.
CMake builds one `qf` library, the `pricing_cli` executable, two demonstration
executables and the registered tests. `qf` publicly links Boost headers and
Threads. Risk uses Boost.Asio and Boost.uBLAS; Eigen3 is not a current project
dependency. Python is used only by the CLI integration test. Representative pricing
and risk measurements are in [Numerical validation](numerical_validation.md).

### Implemented dispatch

| Product and CLI method | Existing operation | Concrete MC policies |
|---|---|---|
| European, `analytical` | `BlackScholesPricer::price` | None |
| European, `mc` | `MonteCarloPricer::price_european` | `GBM`, `NormalRNG`, `OnlineStatistics` |
| European, `antithetic` | `MonteCarloPricer::price_european_antithetic` | `GBM`, `AntitheticRNG<NormalRNG>`, `OnlineStatistics` |
| Geometric Asian, `analytical` | `GeometricAsianPricer::geometric_reference_price` | None |
| Arithmetic/geometric Asian, `mc` | `MonteCarloPricer::price_asian` | `GBM`, `NormalRNG`, `OnlineStatistics` |
| Arithmetic Asian, `cv` | Geometric reference, then `price_asian_control_variate` | `GBM`, `NormalRNG`, `OnlineBivariateStatistics` |
| All four barrier types, `mc` | `MonteCarloPricer::price_barrier` | `GBM`, `NormalRNG`, `OnlineStatistics` |

Analytical pricers return `double`; the dispatcher constructs
`PricingResult{value, 0.0, value, value, 0}`. Simulation pricers return an existing
`PricingResult` directly. Unsupported combinations throw; there is no automatic
fallback. Barrier pricing retains continuous Brownian-bridge survival weighting,
including the initial spot, with zero rebate. Grid points do not turn it into a
discrete-only monitoring algorithm.

## 2. UML class diagrams

The class view is divided into three diagrams to avoid a single dense graph.
Repeated names denote the same existing types in different views.

### 2.1 Request, domain values and engine contract

```mermaid
classDiagram
    direction TB
    class PricingEngine {
        <<Abstract>>
        +price(request)* PricingResult
    }
    class GBMPricingEngine {
        +price(request) PricingResult
    }
    class PricingRequest {
        <<struct>>
        +MarketState market
        +OptionProduct product
        +ModelSpecification model
        +optional~MonteCarloSettings~ simulation
        +PricingMethod method
    }
    class PricingResult {
        <<struct>>
        +double price
        +double standard_error
        +double ci_lower
        +double ci_upper
        +size_t paths
    }
    class MonteCarloSettings {
        <<struct>>
        +size_t paths
        +uint64_t seed
    }
    class PricingMethod {
        <<Enumeration>>
        Analytical
        MonteCarlo
        MonteCarloAntithetic
        MonteCarloControlVariate
    }
    class MarketState {
        <<struct>>
        +double spot
        +double volatility
        +double rate
        +double carry
    }
    class OptionProduct {
        <<variant_alias>>
    }
    class EuropeanOption {
        -OptionType type_
        -double strike_
        -double maturity_
        +type() OptionType
        +strike() double
        +maturity() double
    }
    class AsianOption {
        -OptionType _type
        -AverageType _average_type
        -double _strike
        -double _maturity
        -vector~double~ _observation_times
        +average_type() AverageType
        +observation_times() vector~double~
    }
    class BarrierOption {
        -OptionType _type
        -BarrierType _barrier_type
        -double _strike
        -double _barrier
        -double _maturity
        -vector~double~ _observation_times
        +barrier_type() BarrierType
        +barrier() double
        +observation_times() vector~double~
    }
    class ModelSpecification {
        <<variant_alias>>
    }
    class GBMParameters {
        <<struct>>
    }
    class HestonParameters {
        <<struct>>
        +double initial_variance
        +double mean_reversion
        +double long_run_variance
        +double vol_of_vol
        +double correlation
    }
    PricingEngine <|-- GBMPricingEngine
    PricingEngine ..> PricingRequest : const reference input
    PricingEngine ..> PricingResult : returns by value
    PricingRequest *-- MarketState : market value
    PricingRequest *-- OptionProduct : product value
    PricingRequest *-- ModelSpecification : model value
    PricingRequest *-- "0..1" MonteCarloSettings : optional value
    PricingRequest ..> PricingMethod : method value
    OptionProduct ..> EuropeanOption : variant alternative
    OptionProduct ..> AsianOption : variant alternative
    OptionProduct ..> BarrierOption : variant alternative
    ModelSpecification ..> GBMParameters : variant alternative
    ModelSpecification ..> HestonParameters : declared but rejected
```

`OptionProduct` is exactly `std::variant<EuropeanOption, AsianOption,
BarrierOption>`. `ModelSpecification` is exactly
`std::variant<GBMParameters, HestonParameters>`. `GBMParameters` is an empty
selection tag: volatility, rate and carry are taken from `MarketState`.
`HestonParameters` is only an existing data declaration; there is no Heston model
or pricing path, and the dispatcher rejects that alternative.

The products are unrelated value types; they do not inherit from a common option
base. All their getters are `const`. Asian/barrier schedules are owned vectors;
`observation_times()` returns a **const reference**, abbreviated in the diagrams.
`OptionType` has `Call`/`Put`; `AverageType` has `Arithmetic`/`Geometric`;
`BarrierType` has the four in/out, up/down alternatives.
The CLI spelling is `arithmetic`.

`PricingRequest::method` defaults to `Analytical`. `PricingResult` has zero
member defaults. `MonteCarloSettings` is an aggregate without member defaults.
It contains only `paths` and `seed`; there is no `time_steps` member. The CLI
converts `--steps` or `--times` into the product's `observation_times()` vector,
which is read directly by the path-dependent pricers.

### 2.2 Pricing, simulation and policy types

```mermaid
classDiagram
    direction TB
    class GBMPricingEngine
    class BlackScholesPricer {
        +price(market, option) double
    }
    class GeometricAsianPricer {
        +geometric_reference_price(market, option) double
    }
    class MonteCarloPricer {
        <<template>>
        -Model const& _model
        -RNG& _rng
        +MonteCarloPricer(model, rng)
        +price_european(...) PricingResult
        +price_european_antithetic(...) PricingResult
        +price_asian(...) PricingResult
        +price_asian_control_variate(...) PricingResult
        +price_barrier(...) PricingResult
    }
    class GBM {
        -double volatility_
        -double rate_
        -double carry_
        +GBM(volatility, rate, carry)
        +evolve(spot, dt, z) double
    }
    class NormalRNG {
        -std::mt19937_64 engine_
        -std::normal_distribution~double~ distribution_
        +NormalRNG(seed)
        +next() double
    }
    class AntitheticRNG {
        <<template>>
        -RNG& rng_
        +AntitheticRNG(rng)
        +next_pair()
    }
    class PathGenerator {
        <<template>>
        -Model const& _model
        -RNG& _rng
        +generate(initial_spot, times) vector~double~
        +for_each_step(initial_spot, times, observer) void
        +for_each_interval(initial_spot, times, observer) void
    }
    class OnlineStatistics {
        -size_t count_
        -double mean_
        -double m2_
        +add(x) void
        +count() size_t
        +mean() double
        +variance() double
        +standard_error() double
        +confidence_interval_95()
    }
    class OnlineBivariateStatistics {
        +add(x, y) void
        +count() size_t
        +mean_x() double
        +mean_y() double
        +variance_x() double
        +variance_y() double
        +covariance() double
        +regression_residual_sum_squares() double
    }
    class Payoff {
        <<Abstract>>
    }
    class CallPayoff {
        -double strike_
    }
    class PutPayoff {
        -double strike_
    }
    class PricingResult
    note for MonteCarloPricer "Template parameters: Model, RNG, Statistics"
    note for PathGenerator "Template parameters: Model, RNG"
    note for AntitheticRNG "Template parameter: RNG"
    GBMPricingEngine ..> BlackScholesPricer : local analytical pricer
    GBMPricingEngine ..> GeometricAsianPricer : analytical or control reference
    GBMPricingEngine ..> MonteCarloPricer : concrete template instantiation
    GBMPricingEngine ..> GBM : constructs local model
    GBMPricingEngine ..> NormalRNG : constructs with request seed
    GBMPricingEngine ..> AntitheticRNG : local wrapper for antithetic method
    MonteCarloPricer --> GBM : Model binding - borrowed _model
    MonteCarloPricer --> NormalRNG : RNG binding - borrowed _rng for plain MC and CV
    MonteCarloPricer --> AntitheticRNG : RNG binding - borrowed _rng for antithetic MC
    AntitheticRNG --> NormalRNG : non-owning rng_ in CLI instantiation
    MonteCarloPricer ..> OnlineStatistics : Statistics binding - local accumulator
    MonteCarloPricer ..> OnlineBivariateStatistics : Statistics binding - local CV accumulator
    MonteCarloPricer ..> PathGenerator : local helper for Asian and barrier paths
    PathGenerator --> GBM : non-owning _model in CLI instantiation
    PathGenerator --> NormalRNG : non-owning _rng in path-dependent methods
    MonteCarloPricer ..> CallPayoff : concrete Payoff argument
    MonteCarloPricer ..> PutPayoff : concrete Payoff argument
    MonteCarloPricer ..> PricingResult : returns by value
    Payoff <|-- CallPayoff
    Payoff <|-- PutPayoff
```

`MonteCarloPricer<Model, RNG, Statistics>`, `PathGenerator<Model, RNG>` and
`AntitheticRNG<RNG>` are **class templates**, not subclasses of model/RNG/statistics
interfaces. Their references do not own the model or RNG. The dispatcher keeps
the model, RNG and optional antithetic wrapper alive throughout the pricing call.
`NormalRNG` owns its engine and normal
distribution by value. Statistics and path generators are local objects inside
the pricing methods, not members owned by `MonteCarloPricer`.

The Asian and barrier MC paths use `for_each_step` and `for_each_interval`
callbacks respectively, updating running aggregates without storing complete
simulated paths. `OnlineStatistics` uses Welford's algorithm for the running
mean and sample variance; `OnlineBivariateStatistics` updates covariance online.
This streaming design avoids retaining paths and payoff samples; for a fixed
observation schedule, working memory per pricing call does not grow with the
number of simulated paths.
Control-variate MC uses the geometric reference price and bivariate moments.
Antithetic MC uses `next_pair()` and receives `paths / 2` pairs; the returned
`PricingResult.paths` remains the requested **total** paths.
Fitted Asian CV requires at least three paths and a statistics policy exposing
`regression_residual_sum_squares()`. The built-in bivariate policy accumulates it
stably online. CV uses residual degrees of freedom and a Boost.Math Student-t
interval; with one fixing it instead uses the known arithmetic/geometric payoff
identity after simulating the requested paths. See the
[CV numerical treatment and regressions](numerical_validation.md#b-asian-options).

### Runtime versus static polymorphism

Dynamic polymorphism provides a common API contract for callers. Static
polymorphism binds model, RNG and statistics policies at compile time, exposing
concrete types for compiler specialization and inlining. Variant dispatch
connects runtime product selection to product-specific compiled code.

| Mechanism | Actual implementation |
|---|---|
| Runtime polymorphic contract | `PricingEngine::price` is pure virtual; `GBMPricingEngine` overrides it. The CLI constructs and calls `GBMPricingEngine` directly; an API test also calls through `PricingEngine&`. |
| Runtime polymorphic payoff contract | `Payoff` declares a pure virtual `operator()(double) const`; `CallPayoff` and `PutPayoff` override it. |
| Template-based policies | MC is instantiated with concrete model, RNG and statistics types. Payoff-accepting member functions are templates and receive concrete call/put types in the dispatcher. There is no type-erased RNG/model/statistics interface. |
| Variant dispatch | `std::visit` selects the active product at runtime. Its visitor is instantiated for the actual product types; `if constexpr` selects the corresponding compiled code. `PricingMethod` controls method selection at runtime. |

The existence of virtual methods does not imply that the CLI selects numerical
algorithms through a hierarchy of pricer objects or a factory. It does not.

### 2.3 Risk engine and result structures

```mermaid
classDiagram
    direction LR
    class RiskEngine {
        -uint64_t _seed
        -boost::asio::thread_pool _pool
        +RiskEngine(seed, number_of_threads)
        +calculate(market, maturity, bumps, price_function) RiskReport
    }
    class thread_pool {
        <<external>>
    }
    class RiskReport {
        <<struct>>
        +double price
        +RiskVector gradient
        +RiskMatrix hessian
    }
    class RiskVector {
        <<type_alias>>
    }
    class RiskMatrix {
        <<type_alias>>
    }
    class MarketState
    note for RiskEngine "calculate has a PriceFunction template parameter.\nThe CLI supplies a callable, not an owned engine."
    RiskEngine *-- thread_pool : mutable pool owned by value
    RiskEngine ..> MarketState : input and reconstructed bumped states
    RiskEngine ..> RiskVector : input bumps and local states
    RiskEngine ..> RiskReport : returns by value
    RiskReport *-- RiskVector : gradient value
    RiskReport *-- RiskMatrix : hessian value
```

`thread_pool` is the existing **`boost::asio::thread_pool`**. Its ownership is
composition; it is not an application-defined pool. The `_pool` member is
`mutable`, and the destructor joins it.

`RiskVector` is `boost::numeric::ublas::c_vector<double, 5>`; `RiskMatrix` is
`boost::numeric::ublas::c_matrix<double, 5, 5>`. The gradient and both Hessian axes
use **spot, volatility, rate, carry, maturity**, in that order. `RiskReport` does
not inherit from or contain `PricingResult`; the CLI retains that base pricing
result separately for option-price MC statistics.

`RiskEngine` is a non-template class with a **templated `calculate` member**.
It accepts a callable rather than owning a `PricingEngine`. The CLI callback
connects it to local `GBMPricingEngine` instances. The existing
`BlackScholesGreeks` class provides static analytical functions and is used as a
reference in `test_risk_engine.cpp`; the CLI's named risk Greeks are obtained from
`RiskReport`, not by calling `BlackScholesGreeks`.

At the library boundary, construction rejects zero worker threads before the
pool is constructed. `calculate` requires finite positive spot, volatility and
maturity; finite rates/carry may be zero or negative. Bumps must be finite and
positive, keep both states distinct and in-domain, and leave the existing
difference denominators representable. Validation precedes even the base callback.
Callback prices must also be finite. Nonfinite base prices are rejected directly;
nonfinite bumped prices use the existing worker-exception propagation.

### 2.4 Implied volatility and numerical solvers

```mermaid
classDiagram
    direction TB
    class ImpliedVolatilitySolver {
        -BlackScholesPricer _pricer
        -double _tolerance
        -uintmax_t _max_evaluations
        -IVSolverMethod _method
        +ImpliedVolatilitySolver(tolerance, max_evaluations, method)
        +solve(option, market, option_market_price, lower, upper) double
    }
    class IVSolverMethod {
        <<Enumeration>>
        Toms748
        Bisection
        Newton
    }
    class BlackScholesPricer {
        +price(market, option) double
    }
    class BlackScholesGreeks {
        +vega(strike, maturity, market) double$
    }
    class Toms748 {
        -double _tolerance
        -uintmax_t _max_evaluations
        +Toms748(tolerance, max_evaluations)
        +solve(f, a, b) double
    }
    class Bisection {
        -double _tolerance
        -size_t _max_iterations
        +Bisection(tolerance, max_iterations)
        +solve(function, lower, upper) RootResult
    }
    class Newton {
        -double _tolerance
        -size_t _max_iterations
        -double _derivative_tolerance
        +Newton(tolerance, max_iterations, derivative_tolerance)
        +solve(function, derivative, initial_guess) RootResult
    }
    class RootResult {
        <<struct>>
        +double root
        +size_t iterations
        +bool converged
    }
    class MarketState
    class EuropeanOption
    ImpliedVolatilitySolver *-- BlackScholesPricer : owned _pricer
    ImpliedVolatilitySolver ..> IVSolverMethod : selection stored by value
    ImpliedVolatilitySolver ..> Toms748 : constructs local solver when selected
    ImpliedVolatilitySolver ..> Bisection : constructs local solver when selected
    ImpliedVolatilitySolver ..> Newton : constructs local solver when selected
    ImpliedVolatilitySolver ..> BlackScholesGreeks : static Vega in Newton derivative
    ImpliedVolatilitySolver ..> MarketState : input and checked trial copies
    ImpliedVolatilitySolver ..> EuropeanOption : const reference input
    Bisection ..> RootResult : returns by value
    Newton ..> RootResult : returns by value
```

`ImpliedVolatilitySolver` owns only its `BlackScholesPricer` subobject. It stores
the selected `IVSolverMethod`, tolerance and budget, then constructs exactly one
local solver in the corresponding `switch` branch of `solve`. The three solver
relationships are therefore creation dependencies, **not composition or
inheritance**. There is no shared abstract root-solver interface.

Each numerical solver is a non-template class with a templated `solve` member.
The IV objective copies `MarketState`, checks the candidate volatility against
the admissible interval, and returns the Black-Scholes price minus the observed
price. Newton also receives a derivative lambda calling the static analytical
`BlackScholesGreeks::vega`; `Newton` itself knows nothing about Black-Scholes.
The CLI's risk Greeks still come from `RiskReport`, not this derivative helper.

| Selected CLI solver | Existing return contract | IV adaptation and behavior |
|---|---|---|
| `toms748` (default) | `double` | Calls the existing Boost.Math `toms748_solve` wrapper, checks bracket width and returns its midpoint. Nonconvergence throws. |
| `bisection` | `RootResult` | Requires a bracket; IV checks `converged` before returning `root`. |
| `newton` | `RootResult` | Starts at the bounds' midpoint and uses analytical Vega. IV checks convergence and the returned root; invalid trial states or roots throw. |

Newton is not safeguarded and can step outside the interval. No fallback or
hybrid algorithm is implemented. The shared CLI flag `--max-evaluations` counts
function evaluations for TOMS 748 and iterations for Bisection/Newton. Their
stopping criteria also differ: TOMS 748 checks bracket width, Bisection checks
price residual or half-width, and Newton checks price residual or step size.

## 3. Pricing sequence: European plain Monte Carlo

This is a real supported request. The alternative routes are listed in the
dispatch table above.

```sh
pricing_cli price --type call --spot 100 --strike 100 --maturity 1 \
  --volatility 0.20 --rate 0.05 --carry 0.02 --method mc --paths 100000 --seed 42
```

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant CLI as apps/pricing_cli.cpp
    participant Request as request: PricingRequest
    participant Engine as engine: GBMPricingEngine
    participant Model as model: GBM
    participant RNG as rng: NormalRNG
    participant Payoff as temporary CallPayoff
    participant MC as pricer: MonteCarloPricer
    participant Stats as local OnlineStatistics
    Customer->>CLI: main(argc, argv) with price command
    CLI->>CLI: parse_arguments(argc, argv)
    CLI->>CLI: parse_request(args, false) and parse_product(...)
    CLI->>Request: Aggregate initialization of market, product, model, simulation, method
    Note over Request: EuropeanOption, GBMParameters tag, paths=100000, seed=42, MonteCarlo
    CLI->>CLI: print_price(request)
    CLI->>Engine: price(request)
    Engine->>Engine: Validate GBM, method, product and numerical inputs
    Engine->>Engine: std::visit product and select European MC branch
    Note over Engine: Concrete engine call#59; PricingEngine is the inherited contract
    Engine->>Model: GBM(volatility, rate, carry)
    Engine->>RNG: NormalRNG(seed)
    Engine->>MC: MonteCarloPricer(model, rng)
    Engine->>Payoff: CallPayoff(strike)
    Engine->>MC: price_european(payoff, spot, maturity, rate, paths)
    MC->>Stats: OnlineStatistics()
    loop Each requested path
        MC->>RNG: next()
        RNG-->>MC: z
        MC->>Model: evolve(initial_spot, maturity, z)
        Model-->>MC: terminal_spot
        MC->>Payoff: operator()(terminal_spot)
        Payoff-->>MC: terminal payoff
        MC->>Stats: add(discount_factor * payoff)
    end
    MC->>Stats: mean(), standard_error(), confidence_interval_95()
    Stats-->>MC: Mean, sampling error, confidence bounds
    MC-->>Engine: PricingResult by value
    Engine->>Engine: Validate finite result fields
    Engine-->>CLI: PricingResult by value
    CLI->>CLI: print_report_header / print_report_value / print_mc_estimate
    CLI-->>Customer: Price, standard error, 95% interval and total paths#59; exit 0
```

The loop uses one terminal GBM draw per path, without an observation grid.
In the antithetic route it uses pairs of
opposite draws instead. Asian/barrier routes use the product observation grid.
If parsing, validation or pricing throws, `main` catches `std::exception`, prints
an error to stderr and returns 1; no success report is printed.

## 4. Risk sequence: cached base result and parallel repricing

The example uses European MC risk; Asian and barrier risk use the same callable
connection and their existing selected pricing paths. `with_maturity` rebuilds
products with observation times held at the same fractions of maturity. For an
unchanged maturity it returns the original product value.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant CLI as print_risk in CLI
    participant BaseEngine as engine: GBMPricingEngine
    participant Risk as risk: RiskEngine
    participant Pool as boost::asio::thread_pool
    participant Callback as shared CLI price_at lambda
    participant Repricer as local_engine: GBMPricingEngine per job
    Customer->>CLI: risk command parsed into PricingRequest
    CLI->>BaseEngine: price(request)
    BaseEngine-->>CLI: base_result: PricingResult
    CLI->>CLI: Retain result#59; validate bumps and threads
    CLI->>Risk: RiskEngine(seed, threads)
    CLI->>Risk: calculate(market, maturity, bumps, price_at)
    Risk->>Risk: Validate inputs, then initialize report and construct base / up / down states
    Risk->>Callback: price_at(base_market, maturity, _seed)
    Callback->>Callback: Match original state and seed
    Callback-->>Risk: Cached base_result.price without another simulation
    Risk->>Pool: boost::asio::post for 10 single-factor jobs
    par Workers execute the queued jobs
        loop Each of 10 single-factor jobs
            Pool->>Callback: price_at(bumped_market, bumped_maturity, _seed)
            Callback->>Callback: with_maturity and simulation.seed = common_seed
            Callback->>Repricer: price(bumped PricingRequest)
            Note over Repricer: Existing dispatch and numerical algorithm#59; local RNG with common seed
            Repricer-->>Callback: PricingResult
            Callback-->>Pool: Scalar price
            Pool->>Risk: Store price_up/down and count down latch
        end
    and Calling thread waits
        Risk->>Risk: single_bump_done.wait()
    end
    Risk->>Risk: Central gradient and diagonal Hessian entries
    Risk->>Pool: boost::asio::post for 40 mixed-state jobs
    par Workers execute the queued jobs
        loop Each of 40 mixed-state jobs
            Pool->>Callback: price_at(mixed_market, mixed_maturity, _seed)
            Callback->>Callback: Rebuild product and copy simulation settings with common seed
            Callback->>Repricer: price(bumped PricingRequest)
            Repricer-->>Callback: PricingResult
            Callback-->>Pool: Scalar price
            Pool->>Risk: Store mixed price and count down latch
        end
    and Calling thread waits
        Risk->>Risk: mixed_done.wait()
    end
    Risk->>Risk: Mixed central differences#59; assign symmetric Hessian entries
    Risk-->>CLI: RiskReport by value
    CLI->>CLI: Validate finite report values
    CLI->>CLI: Print price and MC statistics from base_result
    CLI->>CLI: Map gradient/Hessian to named Greeks and print full Hessian
    CLI-->>Customer: Pricing and risk report#59; exit 0
```

The worker loops show logical batches; individual jobs may overlap up to the
configured thread count. Latches separate the single-factor and mixed batches.
Worker exceptions are captured under a mutex, each worker counts down its latch,
and the calling thread rethrows the captured exception after waiting. The CLI
then uses its ordinary error handling.

There are **51 scalar pricing callbacks**: one base, ten single-factor and forty
mixed states. The CLI computes the base `PricingResult` once before `calculate`;
the base callback reuses it. Thus there are **50 bumped repricings plus the one
original pricing calculation**, not a second base Monte Carlo run. The captured
base result is read-only; each bumped request constructs its own RNG using the
same seed. No shared RNG or writable result cache is used by the workers.

| Displayed quantity | Existing report field |
|---|---|
| Option price | `report.price`, originating from cached `base_result.price` |
| Price standard error, 95% confidence interval, paths | Original `base_result` fields only |
| Delta | `report.gradient(0)` |
| Gamma | `report.hessian(0, 0)` |
| Vega | `report.gradient(1)` |
| Rho | `report.gradient(2)` |
| Carry sensitivity | `report.gradient(3)` |
| dPrice/dMaturity | `report.gradient(4)` |
| Theta per year | `-report.gradient(4)` |
| Full second-order risk | `report.hessian(i, j)` in the five-factor order |

No standard errors or confidence intervals for Greeks/Hessian entries are
computed or displayed. Analytical risk reports omit MC statistics. Output is
formatted with nine decimal places; formatting is outside the numerical classes.

## 5. Implied-volatility sequence: default TOMS 748 path

The CLI's IV request is European and analytical. Its market volatility initially
holds the lower bracket bound, not an estimated or calibrated volatility.
`print_iv` checks bracket prices through `GBMPricingEngine` before invoking the
existing IV solver. During root finding the solver calls its own
`BlackScholesPricer` directly; it does not call the dispatcher on every trial.
The sequence below shows the default `--solver toms748` selection. Bisection
uses the same objective directly; Newton adds the analytical Vega callback.
Their distinct result contracts and dependencies are shown in section 2.4.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant CLI as print_iv in CLI
    participant Engine as engine: GBMPricingEngine
    participant Solver as ImpliedVolatilitySolver
    participant Toms as local Toms748 in solve
    participant Boost as boost::math::tools::toms748_solve
    participant Objective as objective lambda in solve
    participant BS as owned BlackScholesPricer
    Customer->>CLI: iv command parsed into European PricingRequest
    CLI->>CLI: Select IVSolverMethod from --solver (default Toms748)
    CLI->>CLI: Validate observed price, bounds, tolerance and evaluation budget
    CLI->>Engine: price(request at lower volatility)
    Engine-->>CLI: Lower-bound analytical PricingResult
    CLI->>Engine: price(upper_request at upper volatility)
    Engine-->>CLI: Upper-bound analytical PricingResult
    CLI->>CLI: Check observed price is inside supplied price bracket
    CLI->>Solver: ImpliedVolatilitySolver(tolerance, max_evaluations, method)
    CLI->>Solver: solve(option, market, option_market_price, lower_volatility, upper_volatility)
    Solver->>Toms: Construct selected local solver(tolerance, max_evaluations)
    Solver->>Toms: solve(objective, lower_volatility, upper_volatility)
    Toms->>Boost: toms748_solve(f, a, b, tolerance, evaluations)
    loop Objective evaluations until tolerance or budget condition
        Boost->>Objective: f(trial_volatility)
        Objective->>Objective: market_at checks interval, copies market and replaces volatility
        Objective->>BS: price(trial_market, option)
        BS-->>Objective: European Black-Scholes price
        Objective-->>Boost: Trial price minus option_market_price
    end
    Boost-->>Toms: Root bracket pair
    Toms->>Toms: Check bracket width#59; return std::midpoint
    Toms-->>Solver: double volatility
    Solver-->>CLI: double volatility
    CLI->>CLI: Check finite result and format report
    CLI-->>Customer: Selected method, observed price and implied volatility#59; exit 0
```

`Toms748` throws if the returned bracket does not satisfy its absolute-width
criterion. Boost may also reject a bracket. Those errors reach the CLI's
`std::exception` handler. The CLI selects Bisection or Newton only when requested;
there is no automatic fallback between solvers. No Monte Carlo calibration,
Heston path or additional IV result class is used by this flow.

## 6. Interface documentation

The class and sequence views above describe the implemented interfaces and
ownership relationships. Exact C++ declarations and implementation files remain
in the private project and are not included in this public documentation.

## 7. Source index and boundaries of this documentation

| Private implementation area | Role |
|---|---|
| CLI | Parsing, aggregate request construction, base-result reuse, risk callback, IV bracket checks, report helpers and error handling |
| Request, interface, concrete header, dispatcher | Actual API contract, variants, method enumeration and dispatch paths |
| MarketState, ModelSpecification, GBM | Market values, model alternatives and GBM evolution interface |
| European, Asian, Barrier | Product value types, enums and owned schedules |
| Black-Scholes, geometric Asian, MC, result | Existing pricing interfaces, template algorithms and result fields |
| Normal RNG, antithetic RNG, path generator | RNG ownership, reference lifetimes and callback traversal |
| Statistics, bivariate statistics, payoff interface, payoffs | Concrete policies and the separate virtual payoff contract |
| Risk engine, report, analytical Greeks | Finite differences, batching, result aliases and analytical test references |
| IV solver, TOMS wrapper, Bisection, Newton, RootResult | Owned analytical pricer, selected local root solver, result adaptation and Vega callback |
| CMake, API tests, product tests, CLI tests, risk tests | Build targets and executable evidence for the documented relationships |

No relationships are inferred merely from directory names.
`RiskEngine::calculate` is defined in its header; there is no
`src/risk/RiskEngine.cpp` in the current source tree.

### Architectural consistency notes

| Observation from the current source | Consequence for the diagrams and API |
|---|---|
| API files are in `api/`, but use namespace `qf::pricing`. | Directory layers do not introduce a `qf::api` namespace or a second pricing hierarchy. |
| `ModelSpecification` declares `HestonParameters`; the dispatcher accepts only `GBMParameters`. | The Heston data alternative is shown as declared and rejected, not as an implemented model. |
| CMake links Boost headers and Threads; risk matrices use Boost.uBLAS. Eigen3 is not required or installed by CI. | The diagrams show the current Boost-based risk types. |
| Asian and barrier products own their schedules and expose const-reference getters. | Barrier options, `OptionProduct` and `PricingRequest` support ordinary copy and move assignment without exposing mutable schedule access. |
| Root-solver return types and stopping criteria differ. | The IV wrapper adapts the existing contracts; the three solvers do not implement a common interface. |

The earlier documentation was inconsistent with the current code: it included
a removed `MonteCarloSettings::time_steps`, the old `Arithmatic` spelling and an
owned IV `_solver`, and described Bisection/Newton as unselected library-only
components. Those descriptions are corrected here and in the README. No source
or build configuration was changed to reconcile these documentation differences.

### Mermaid compatibility and verification limits

The diagrams use ordinary `flowchart`, `classDiagram` and `sequenceDiagram`
fences supported by [GitHub's Mermaid integration](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams).
They avoid newer namespace, shape and participant-creation extensions. Template
parameters containing commas are written in prose/signature blocks rather than
Mermaid generic class labels, following the documented
[class-diagram limitations](https://mermaid.js.org/syntax/classDiagram.html).
Parallel batches use the documented
[sequence-diagram notation](https://mermaid.js.org/syntax/sequenceDiagram.html).

The Mermaid CLI is not on the current PATH, and no local Mermaid parser was
used. The diagrams were checked against the official notation and reviewed
structurally; source symbols, relationship endpoints and local links were also
checked. Actual Mermaid parsing and SVG rendering were not run locally. GitHub
rendering therefore remains a visual verification step, rather than a claimed
completed render. The original Mermaid verification limits above are separate from the five
rendered PNG views included in this package.
