# SCOPING: <library name> v0.1
## 1. What it does today
A 2D finite-element solver for the unsteady, incompressible Navier–Stokes
equations, written in C++. It is currently set up for the lid-driven cavity
at Re = 100 on a 20×20 element mesh (441 nodes).

**Equations.** Non-dimensional incompressible Navier–Stokes in primitive
variables (u, v, p).

**Discretisation.** Bilinear Q1 quadrilateral elements, equal-order
interpolation for u and p, 3×3 Gauss quadrature. The momentum weak form is
derived by the method of weighted residuals (Galerkin), with a GLS-type
stabilisation term applied to the convective term only, so in practice it
is SUPG-like. The pressure-gradient, viscous and continuity terms are plain
Galerkin.

Stabilisation parameter:

    τ = 1 / √( (2/dt)² + (2|u_b|/h)² + (4/(Re·h²))² )

h = dx = 0.05 is a global constant, and |u_b| is the nodal speed at node b,
not the speed at the Gauss point. With the current settings τ ≈ 5×10⁻⁴,
and the 2/dt term dominates.

**Pressure.** Pressure is found by solving a pressure-stabilised,
Poisson-type equation driven by the current velocity divergence
(ε = 0.01, [role of ε: TODO]). The pressure matrix K0 is inverted once at
start-up (dense `FullPivLU`). After that, each step applies K0⁻¹ to the
divergence vector as a matrix–vector product.

**Time stepping.** Explicit Forward Euler with a lumped (diagonal) mass
matrix, dt = 0.001, 3002 steps. Each step:
1. Update velocity explicitly by dividing the assembled right-hand side by
   the lumped mass.
2. Re-impose the Dirichlet velocity values.
3. Compute pressure from the divergence, using K0⁻¹.

No nonlinear (Newton) iteration is needed. Estimated stability limits are
dt ≤ 0.05 (convective CFL) and dt ≤ 0.0625 (viscous), so dt = 0.001 has a
margin of about 50×. These are estimates and have not been checked by a run.

**Boundary conditions.** Lid: u = 1, v = 0. Other walls: u = v = 0. The two
top corners use u = 0.5. Dirichlet values are overwritten after each
explicit update. Pressure is not pinned (see Section 6, issue 1).

**Validation.** Not yet done. There is no comparison with reference data
(e.g. Ghia et al., 1982) in the repository yet.

## 2. Code map

All code is in `NS_FEM.cpp` (435 lines). Except for three helper functions,
everything is inside `main()`.

| Block (lines) | What it does | Decision | Why |
|---|---|---|---|
| Parameters (44–67) | Gauss points/weights and Ndt, dt, Re as globals; mesh size, domain, ε and derived values in `main()`; h = `le` = dx | Rewrite → `Config` struct | Parameters are split between globals and `main()`, and h is one global value. |
| Boundary predicates (30–41) | Tests whether a node index lies on the left/right/bottom/top wall of the structured grid | Keep → move into `Mesh` | Correct for node indices. Line 263 wrongly calls them with element indices. |
| Mesh generation (70–90) | Node coordinates, Q1 connectivity | Rewrite → `Mesh` class (nodes, elements, tagged boundary nodes and edges) | Assembly, BCs, forces and output all need it. |
| Shape functions (328–375) | `maping()`: Q1 basis, derivatives, Jacobian determinant; `summed_multiplication()` | Keep → `Element` module; return a named struct | It returns `{Ny, Nx, N}` in that order, which makes it easy to swap x and y by mistake. |
| Static assembly (91–130) | Assembles K0 (Laplacian), K1 and K2 (∂/∂x, ∂/∂y), M/dt; lumps M by row sum | Rewrite → sparse matrices; store the lumped mass as a vector | Dense NT×NT storage, and the lumped mass is kept as a full matrix. |
| Pressure operator (132–135, 216) | Dense `FullPivLU` inverse of K0; p = K0⁻¹·RHS_cont | Rewrite → pin one node, factorise once (sparse LDLᵀ), solve each step | K0 is singular (issue 1), and the dense inverse does not scale. |
| Convective + SUPG assembly (163–198) | Rebuilds a dense K3 every step; weight N_a + τ(u·∇N_a) | Rewrite → element-level contribution to the RHS; τ computed once per Gauss point using the Gauss-point velocity | Allocates NT² entries every step, and τ is recomputed 16× per Gauss point from the speed at node b. |
| RHS assembly (199–209) | Continuity and momentum right-hand sides | Rewrite → sparse matrix–vector products | A dense double loop costs O(NT²) per step. |
| Initial BCs + update + BCs (141–156, 220–240) | Updates interior velocity explicitly, then overwrites Dirichlet values | Rewrite → one `applyVelocityBCs()` function | The same BC block appears twice. |
| Time loop (160–320) | Assemble → pressure → velocity → output | Keep → reduce to calls to the blocks above | The algorithm is unchanged; only the structure changes. |
| Force routine (259–316) | Wall force on the bottom wall | Rewrite | Three bugs; see Section 6, issue 2. |
| VTK writer (377–435) | Legacy ASCII VTK output for u, v, p | Keep → `io` module | Works. File names should come from `Config`. |
| Leftovers | Unused includes (`cassert`, `numeric`, `stdexcept`); `omp.h` used only for the timer; per-step `cout` (161, 214, 218–219); stale message "Matrix inversion complete" (323); "10x10" file names (249, 304) | Delete / fix | Noise, and misleading labels. |

Proposed layout: `config.hpp`, `mesh.{hpp,cpp}`, `element.{hpp,cpp}`,
`assembly.{hpp,cpp}`, `bc.{hpp,cpp}`, `forces.{hpp,cpp}`, `io.{hpp,cpp}`,
`main.cpp` (time loop only).## 3. Public API v0.1 (5-8 calls)
Python sketch: mesh = ...; solver = ...; solver.run(...); fields = ...
## 4. Out of scope for v0.1 (at least 3)
## 5. Gaps to close, in priority order
## 6. Known issues
- Convergence table is non-monotonic: my diagnosis is ...
- Pressure null space: my diagnosis is ...
- Wall-force integral: my diagnosis is ...
