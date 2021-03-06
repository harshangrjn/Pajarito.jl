[![Build Status](https://travis-ci.org/JuliaOpt/Pajarito.jl.svg?branch=master)](https://travis-ci.org/JuliaOpt/Pajarito.jl) [![codecov.io](https://codecov.io/github/JuliaOpt/Pajarito.jl/coverage.svg?branch=master)](https://codecov.io/github/JuliaOpt/Pajarito.jl?branch=master) [![Pajarito](http://pkg.julialang.org/badges/Pajarito_0.5.svg)](http://pkg.julialang.org/?pkg=Pajarito&ver=0.5)

# Pajarito

Pajarito is a **mixed-integer convex programming** (MICP) solver package written in [Julia](http://julialang.org/).

MICP problems are convex except for restrictions that some variables take binary or integer values. Pajarito supports both **mixed-integer conic programming**, which encodes nonlinearities using a small number of predefined cones, and more traditional **convex mixed-integer nonlinear programming**, which encodes nonlinearities with smooth functions and uses their derivatives.

Pajarito solves MICP problems by constructing sequential lifted polyhedral outer-approximations of the convex feasible set to leverage the power of MILP solvers. The algorithm has theoretical finite-time convergence under reasonable assumptions. Pajarito accesses state-of-the-art MILP solvers and continuous conic or nonlinear programming (NLP) solvers through the MathProgBase interface.  

## Installation

Pajarito can be installed through the Julia package manager:
```
julia> Pkg.add("Pajarito")
```

## Usage

Pajarito has two entirely separate algorithms depending on the form in which the input is provided. The first algorithm is the **derivative-based nonlinear** (NLP) algorithm, where the approach used is analogous to that of [Bonmin](https://projects.coin-or.org/Bonmin) (and will perform similarly, with the primary advantage of being able to easily swap-in various mixed-integer and convex subproblem solvers that Bonmin does not support). The second algorithm is the **conic** algorithm, which is the new approach proposed in the publications describing Pajarito. The conic algorithm currently supports MICP with second-order, rotated second-order, (primal) exponential, and positive semidefinite cones. MISOCP and MISDP are two established sub-classes of MICPs that Pajarito can solve. 

There are several convenient ways to model MICPs in Julia and access Pajarito:

|             | [JuMP][JuMP-url]  | [Convex.jl][convex-url]  | [MathProgBase][mpb-url]  |
|-------------|-------------------|--------------------------|--------------------------|
| NLP model   | [X][JuMP-nlp-url] |                          | [X][mpb-nlp-url]         |
| Conic model | X                 | X                        | [X][mpb-conic-url]       |

[mpb-nlp-url]: http://mathprogbasejl.readthedocs.io/en/latest/nlp.html
[mpb-conic-url]: http://mathprogbasejl.readthedocs.io/en/latest/conic.html
[JuMP-url]: https://github.com/JuliaOpt/JuMP.jl
[JuMP-nlp-url]: http://jump.readthedocs.io/en/latest/nlp.html
[convex-url]: https://github.com/JuliaOpt/Convex.jl
[mpb-url]: https://github.com/JuliaOpt/MathProgBase.jl

JuMP and Convex.jl are algebraic modeling interfaces, while MathProgBase is a lower-level interface for providing input in raw callback or matrix form. Convex.jl is perhaps the most user-friendly way to provide input in conic form, since it transparently handles conversion of algebraic expressions. JuMP supports general nonlinear smooth functions, e.g. by using `@NLconstraint`. JuMP also supports conic modeling, but requires cones to be explicitly specified, e.g. by using `norm(x) <= t` for second-order cone constraints and `@SDconstraint` for positive semidefinite constraints. Note that a problem provided in conic form may solve faster than an equivalent problem in NLP form because conic form naturally encodes extended formulations; however, [Hijazi et al.](http://www.optimization-online.org/DB_FILE/2011/06/3050.pdf) suggest manual reformulation techniques which achieve many of the algorithmic benefits of conic form.

Pajarito may be accessed through MathProgBase from outside Julia by using the experimental [cmpb](https://github.com/mlubin/cmpb) interface which provides a C API to the low-level conic input format. The [ConicBenchmarkUtilities](https://github.com/mlubin/ConicBenchmarkUtilities.jl) package provides utilities to read files in the [CBF](http://cblib.zib.de/) format.

## MIP and continuous solvers

The algorithm implemented by Pajarito itself is relatively simple, and most of the hard work is performed by the MIP outer-approximation model solver and the continuous convex subproblem models solver. **The performance of Pajarito depends on these two types of solvers.** For best performance, use commercial solvers.

The mixed-integer solver is specified by using the `mip_solver` option to `PajaritoSolver`, e.g. `PajaritoSolver(mip_solver=CplexSolver())`. You must first load the Julia package which provides the mixed-integer solver, e.g. `using CPLEX`. This solver is typically a mixed-integer linear solver, but if a conic problem has both second-order cones and other nonlinear cones, or if it has PSD cones, then the MIP solver can be an MISOCP solver and Pajarito can put second-order cones in the outer-approximation model.

The continuous convex solver is specified by using the `cont_solver` option, e.g. `PajaritoSolver(cont_solver=IpoptSolver())`. When given input in derivative-based nonlinear form, Pajarito requires a derivative-based nonlinear solver, e.g. [Ipopt](https://projects.coin-or.org/Ipopt) or [KNITRO](http://www.ziena.com/knitro.htm). When given input in conic form, the convex subproblem solver can be *either* a conic solver which supports the cones used *or* a derivative-based solver like Ipopt. If a derivative-based solver is provided in this case, then Pajarito will switch to the derivative-based algorithm by using the [ConicNonlinearBridge](https://github.com/mlubin/ConicNonlinearBridge.jl) (which does not support PSD cones). Note that using derivative-based solvers for conic problems can cause numerical instability because conic problems are not always smooth.

For conic models, the predefined cones are listed in the [MathProgBase](http://mathprogbasejl.readthedocs.io/en/latest/conic.html) documentation. The following conic solvers (**O** means open-source) can be used by Pajarito to solve mixed-integer conic models with any mixture of the corresponding cones: 

|                        | Second-order | Rotated second-order | Positive semidefinite | Primal exponential |
|------------------------|--------------|----------------------|-----------------------|--------------------|
| [CSDP][csdp-url] **O** |              |                      | X                     |                    | 
| [ECOS][ecos-url] **O** | X            | X                    |                       | X                  | 
| [SCS][scs-url] **O**   | X            | X                    | X                     | X                  | 
| [Mosek][mosek-url]     | X            | X                    | X                     |                    | 

[csdp-url]: https://github.com/JuliaOpt/CSDP.jl
[ecos-url]: https://github.com/JuliaOpt/ECOS.jl
[mosek-url]: https://github.com/JuliaOpt/Mosek.jl
[scs-url]: https://github.com/JuliaOpt/SCS.jl

MIP and continuous solver parameters must be specified through their corresponding Julia interfaces. For example, to turn off the output of Ipopt solver, use `cont_solver=IpoptSolver(print_level=0)`.

## Pajarito solver options

The following options can be passed to `PajaritoSolver()` to modify its behavior (see [solver.jl](https://github.com/mlubin/Pajarito.jl/blob/master/src/solver.jl) for default values; **C** means conic algorithm only):

  * `log_level::Int` Verbosity flag: 0 for quiet, 1 for basic solve info, 2 for iteration info, 3 for detailed timing and cuts and solution feasibility info
  * `timeout::Float64` Time limit for algorithm (in seconds)
  * `rel_gap::Float64` Relative optimality gap termination condition
  * `mip_solver_drives::Bool` Let MIP solver manage convergence ("branch and cut")
  * `mip_solver::MathProgBase.AbstractMathProgSolver` MIP solver (MILP or MISOCP)
  * `mip_subopt_solver::MathProgBase.AbstractMathProgSolver` **C** MIP solver for suboptimal solves (with appropriate options already passed)
  * `mip_subopt_count::Int` **C** Number of times to use `mip_subopt_solver` between `mip_solver` solves
  * `round_mip_sols::Bool` **C** Round integer variable values before solving subproblems
  * `use_mip_starts::Bool` **C** Use conic subproblem feasible solutions as MIP warm-starts or heuristic solutions
  * `cont_solver::MathProgBase.AbstractMathProgSolver` Continuous solver (conic or nonlinear)
  * `solve_relax::Bool` **C** Solve the continuous conic relaxation to add initial subproblem cuts
  * `solve_subp::Bool` **C** Solve the continuous conic subproblems to add subproblem cuts
  * `dualize_relax::Bool` **C** Solve the conic dual of the continuous conic relaxation
  * `dualize_subp::Bool` **C** Solve the conic duals of the continuous conic subproblems
  * `soc_disagg::Bool` **C** Disaggregate SOC cones
  * `soc_abslift::Bool` **C** Use SOC absolute value lifting
  * `soc_in_mip::Bool` **C** Use SOC cones in the MIP model (if `mip_solver` supports MISOCP)
  * `sdp_eig::Bool` **C** Use PSD cone eigenvector cuts
  * `sdp_soc::Bool` **C** Use PSD cone eigenvector SOC cuts (if `mip_solver` supports MISOCP)
  * `init_soc_one::Bool` **C** Use SOC initial L_1 cuts
  * `init_soc_inf::Bool` **C** Use SOC initial L_inf cuts
  * `init_exp::Bool` **C** Use Exp initial cuts
  * `init_sdp_lin::Bool` **C** Use PSD cone initial linear cuts
  * `init_sdp_soc::Bool` **C** Use PSD cone initial SOC cuts (if `mip_solver` supports MISOCP)
  * `scale_subp_cuts::Bool` **C** Use scaling for subproblem cuts
  * `scale_subp_factor::Float64` **C** Fixed multiplicative factor for scaled subproblem cuts
  * `viol_cuts_only::Bool` **C** Only add cuts violated by current MIP solution
  * `prim_cuts_only::Bool` **C** Add primal cuts, do not add subproblem cuts
  * `prim_cuts_always::Bool` **C** Add primal cuts and subproblem cuts
  * `prim_cuts_assist::Bool` **C** Add subproblem cuts, and add primal cuts only subproblem cuts cannot be added
  * `cut_zero_tol::Float64` **C** Zero tolerance for cut coefficients
  * `prim_cut_feas_tol::Float64` **C** Absolute feasibility tolerance used for primal cuts (set equal to feasibility tolerance of `mip_solver`) 

**Pajarito is not yet numerically robust and may require tuning of parameters to improve convergence.** If the default parameters don't work for you, please let us know.

Note:
  * For the conic algorithm, Pajarito usually returns a solution constructed from one of the conic solver's feasible solutions. Since the conic solver is not subject to the same feasibility tolerances as the MIP solver (which should match the absolute feasibility tolerance `prim_cut_feas_tol`), Pajarito's solution will not necessarily satisfy `prim_cut_feas_tol`.
  * MIP solver integrality tolerance should typically be tightened, for example to `1e-8`, for improved Pajarito performance.
  * `viol_cuts_only` defaults to `true` on the MIP-solver-driven algorithm and `false` on the iterative algorithm.

## Bug reports and support

Please report any issues via the Github **[issue tracker]**. All types of issues are welcome and encouraged; this includes bug reports, documentation typos, feature requests, etc. The **[Optimization (Mathematical)]** category on Discourse is appropriate for general discussion.

[issue tracker]: https://github.com/mlubin/Pajarito.jl/issues
[Optimization (Mathematical)]: https://discourse.julialang.org/c/domain/opt

## We need your challenging MICP problems

Mixed-integer convex programming is an active area of research, and we are seeking out hard benchmark instances. Please get in touch either by opening an issue or privately if you would like to share any hard instances to be used as benchmarks in future work. Challenging problems will help us determine how to improve Pajarito.

## References

If you find Pajarito useful in your work, we kindly request that you cite the following [paper](http://dx.doi.org/10.1007/978-3-319-33461-5_9) ([arXiv preprint](http://arxiv.org/abs/1511.06710)):

    @Inbook{LubinYamangilBentVielma2016,
    author="Lubin, Miles
    and Yamangil, Emre
    and Bent, Russell
    and Vielma, Juan Pablo",
    editor="Louveaux, Quentin
    and Skutella, Martin",
    title="Extended Formulations in Mixed-Integer Convex Programming",
    bookTitle="Integer Programming and Combinatorial Optimization: 18th International Conference, IPCO 2016, Li{\`e}ge, Belgium, June 1-3, 2016, Proceedings",
    year="2016",
    publisher="Springer International Publishing",
    address="Cham",
    pages="102--113",
    isbn="978-3-319-33461-5",
    doi="10.1007/978-3-319-33461-5_9",
    url="http://dx.doi.org/10.1007/978-3-319-33461-5_9"
    }

The paper describes the motivation of Pajarito and is recommended reading for advanced users.
