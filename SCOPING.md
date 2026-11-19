# SCOPING: [library name] v0.1
 
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
 
**Pressure.** Pressure comes from a Poisson-type equation driven by the
current velocity divergence:
 
    K0 · p = −(1/ε) · (K1·u + K2·v),      ε = 0.01
 
K0 is the Galerkin Laplacian (pure Neumann: there is no pressure Dirichlet
condition). K1 and K2 are the Galerkin ∂/∂x and ∂/∂y operators. ε only
scales the divergence source and adds nothing to K0. K0 is inverted once
at start-up (dense `FullPivLU`). After that, each step applies K0⁻¹ to the
right-hand side as a matrix–vector product. K0 is singular; see Section 6,
issue 1.
 
**Time stepping.** Explicit Forward Euler with a lumped (diagonal) mass
matrix, dt = 0.001, 3002 steps. Each step:
1. Assemble the momentum RHS from uⁿ and pⁿ.
2. Compute pⁿ⁺¹ from the divergence of uⁿ.
3. Update interior velocity: uⁿ⁺¹ = RHS / lumped mass.
4. Overwrite the Dirichlet velocity values.
So the momentum equation at step n uses pⁿ, which was computed from uⁿ⁻¹.
Pressure lags velocity by one step. There is no nonlinear (Newton)
iteration. Estimated stability limits are dt ≤ 0.05 (convective CFL) and
dt ≤ 0.0625 (viscous), so dt = 0.001 has a margin of about 50× (estimate,
not checked by a run).
 
**Boundary conditions.** Lid: u = 1, v = 0. Other walls: u = v = 0. The two
top corners use u = 0.5. Dirichlet values are overwritten after each
explicit update. Pressure is not pinned.
 
**Validation.** Not yet done. There is no comparison with reference data
(e.g. Ghia et al., 1982) in the repository.
 
 
## 2. Code map
 
All code is in `NS_FEM.cpp` (435 lines). Except for three helper functions,
everything is inside `main()`.
 
| Block (lines) | What it does | Decision | Why |
|---|---|---|---|
| Parameters (44–67) | Gauss points/weights and Ndt, dt, Re as globals; mesh size, domain, ε and derived values in `main()`; h = `le` = dx | Rewrite → `Config` struct | Parameters are split between globals and `main()`, and h is one global value. |
| Boundary predicates (30–41) | Tests whether a node index lies on the left/right/bottom/top wall of the structured grid | Keep → move into `Mesh` | Correct for node indices. Line 263 wrongly calls them with element indices. |
| Mesh generation (70–90) | Node coordinates, Q1 connectivity | Rewrite → `Mesh` class (nodes, elements, tagged boundary nodes and edges) | Assembly, BCs and output all need it. |
| Shape functions (328–375) | `maping()`: Q1 basis, derivatives, Jacobian determinant; `summed_multiplication()` | Keep → `Element` module; return a named struct; throw instead of `exit(1)` | It returns `{Ny, Nx, N}` in that order, which makes it easy to swap x and y by mistake. |
| Static assembly (91–130) | Assembles K0, K1, K2, M/dt; lumps M by row sum | Rewrite → sparse matrices; store the lumped mass as a vector | Dense NT×NT storage, and the lumped mass is kept as a full matrix. |
| Pressure operator (132–135, 216) | Dense `FullPivLU` inverse of K0; p = K0⁻¹·RHS_cont | Rewrite → pin one node, factorise once (sparse LDLᵀ), solve each step | K0 is singular (Section 6, issue 1), and the dense inverse does not scale. |
| Convective + SUPG assembly (163–198) | Rebuilds a dense K3 every step; weight N_a + τ(u·∇N_a) | Rewrite → element-level contribution to the RHS. **The τ formula stays as it is in v0.1.** | Allocates NT² entries every step. Moving τ to the Gauss point is deferred to v0.2 (Section 4) so v0.1 refactors can be checked against a fixed snapshot. |
| RHS assembly (199–209) | Continuity and momentum right-hand sides | Rewrite → sparse matrix–vector products | A dense double loop costs O(NT²) per step. |
| Initial BCs + update + BCs (141–156, 220–240) | Updates interior velocity explicitly, then overwrites Dirichlet values | Rewrite → one `applyVelocityBCs()` function | The same BC block appears twice. |
| Time loop (160–320) | Assemble → pressure → velocity → output | Keep → reduce to calls to the blocks above | The algorithm is unchanged; only the structure changes. |
| Force routine (259–316) | Wall force on the bottom wall | Remove from v0.1, rewrite in v0.2 | Three bugs (Section 6, issue 2). |
| VTK writer (377–435) | Legacy ASCII VTK output for u, v, p | Keep → `io` module | Works. File names come from `Config` (Section 6, issue 3). |
| Leftovers | Unused includes (`cassert`, `numeric`, `stdexcept`); `omp.h` used only for the timer; per-step `cout` (161, 214, 218–219); stale message "Matrix inversion complete" (323) | Delete | Noise, and misleading output. |
 
Proposed layout: `config.hpp`, `mesh.{hpp,cpp}`, `element.{hpp,cpp}`,
`assembly.{hpp,cpp}`, `bc.{hpp,cpp}`, `io.{hpp,cpp}`,
`main.cpp` (time loop only).
 
 
## 3. Public API v0.1
 
### What a user types
 
```python
import nsfem   # placeholder until the library is named
 
cfg = nsfem.Config(
    nx=20, ny=20, Lx=1.0, Ly=1.0,
    Re=100.0, dt=1e-3, n_steps=3000, epsilon=0.01,
    output_every=100, output_prefix="cavity",
)
 
mesh = nsfem.Mesh.structured(cfg)               # Q1 quads on a rectangle
 
# General boundary conditions by tag ...
bc = nsfem.DirichletBC(mesh)
bc.set(["left", "right", "bottom"], u=0.0, v=0.0)
bc.set("top", u=1.0, v=0.0)
bc.set_pressure_reference(node=0, value=0.0)
 
# ... or the same thing as a preset (also sets the top corners to u = 0.5)
bc = nsfem.presets.lid_driven_cavity(mesh, u_lid=1.0)
 
solver = nsfem.Solver(cfg, mesh, bc)
solver.run()              # all n_steps; writes VTK every output_every steps
# solver.step(100)        # or advance by a number of steps
 
u, v, p = solver.u, solver.v, solver.p     # numpy arrays, shape (n_nodes,)
x, y    = mesh.x, mesh.y
U = u.reshape(cfg.ny + 1, cfg.nx + 1)       # grid view, U[j, i]
 
y_c, u_c = solver.centreline("vertical")    # u along x = 0.5 (Ghia comparison)
x_c, v_c = solver.centreline("horizontal")  # v along y = 0.5
 
solver.write_vtk("cavity_final.vtk")
```
 
### Classes and functions behind it (C++)
 
1. `Config`: plain struct with all run parameters, with no globals.
2. `Mesh`: structured Q1 generator. Holds nodes, connectivity and tagged
   boundary nodes/edges. Exposes `x` and `y`.
3. `Element` (internal, not bound): Q1 shape functions, derivatives,
   Jacobian, Gauss rule.
4. `Assembler` (internal): sparse K0, K1, K2, lumped mass vector, and the
   convective + SUPG right-hand side per step.
5. `PressureSolver` (internal): pinned K0, factorised once, solved every
   step.
6. `DirichletBC`: velocity values by boundary tag + pressure reference
   node. `presets.lid_driven_cavity()` builds one.
7. `Solver`: owns the state (u, v, p). Provides `step(n)`, `run()`,
   `u`/`v`/`p`, `centreline()` and `write_vtk()`.
### Contract
 
- **Node ordering.** Node id = i + j·(nx+1), with i along x and j along y,
  so x varies fastest. Therefore `u.reshape(ny + 1, nx + 1)` (numpy's
  default C order) gives `U[j, i]`.
- **Pressure** is defined up to a constant. Values are reported relative
  to the reference node set in the BC object.
- **Corners and overlapping tags.** When two `set()` calls cover the same
  node, the last call wins.
- **`centreline()`** returns nodal values. It requires an even nx (for
  `"vertical"`) or an even ny (for `"horizontal"`) and raises otherwise.
- **Errors.** The C++ side throws exceptions derived from
  `std::runtime_error` (e.g. `nsfem::MeshError` for a non-positive
  Jacobian, `nsfem::ConfigError` for invalid input) instead of calling
  `exit(1)`. The bindings register them with `py::register_exception`.
- **Heat equation.** Not in v0.1; planned for v0.2 (Section 4).
### Config fields to bind (pybind11)
 
`nx, ny, Lx, Ly, x_min, y_min, Re, dt, n_steps, epsilon, output_every, output_prefix`
 
 
## 4. Out of scope for v0.1
 
| Item | Why it is out | What brings it back |
|---|---|---|
| Heat equation (temperature transport, Boussinesq coupling) | v0.1 is not validated yet. Adding a second field now would double the debugging surface. **Planned for v0.2.** | The cavity matches Ghia (Re = 100) within an agreed tolerance, and `DirichletBC` by tag works, so temperature BCs can reuse it. |
| Wall forces and the cylinder case | The current force routine has three bugs (Section 6, issue 2). | An edge-integral routine reproduces an analytic wall shear, e.g. plane Couette or Poiseuille flow, within 1%. |
| τ at the Gauss point with an element-wise h | It changes the numerics, so it would break the v0.1 regression snapshot. It is not needed at Re = 100 on a uniform mesh. | v0.1 is validated. The change then gets its own Ghia check and a new snapshot. |
| General `sample_line(start, end, n)` with Q1 interpolation | `centreline()` on nodes is enough for the Ghia comparison. | A test case needs profiles that do not lie on node lines (non-uniform meshes, the cylinder wake). |
| Unstructured meshes / Gmsh import | v0.1 only generates structured Q1 grids, and the boundary predicates assume structured indexing. | `Mesh` stores tagged boundary edges, no code path uses i/j indexing, and a test on distorted quads passes. |
| Outflow / Neumann velocity BCs | The cavity needs only Dirichlet BCs. Open boundaries change the pressure problem: K0 is no longer pure Neumann. | `DirichletBC` is stable, and a channel-flow test with a parabolic inlet and an outflow BC is defined. |
| Implicit / Newton time stepping | Explicit Euler is enough at Re = 100, with a dt margin of about 50×. | The explicit dt limit becomes the bottleneck, e.g. steady runs at Re ≥ 1000 or fine meshes where the viscous limit dt ~ Re·h² gets small. |
| Parallelism (OpenMP / MPI) | The real cost today is dense O(NT²) storage, not the lack of threads. | After the sparse rewrite, profiling shows assembly dominating the run time. |
| GPU acceleration | Today the solver keeps 7 dense NT×NT matrices (K0, K1, K2, M, M_lumped, K3, K0⁻¹). On a 4 GB GPU (T1200) that limits the mesh to about 90×90 nodes, and the same storage wall applies to CPU RAM. The problem is the data structure, not the hardware. | After the sparse rewrite, profiling shows assembly or the pressure solve dominating the run time on meshes ≥ 200×200. |
| 3D, higher-order elements | Each one is effectively a new library. | v0.2 is stable and a user asks for it. |
 
**Plan for the heat equation.** At the end of week 6, P1 ships flow only: the
incompressible cavity validated against Ghia et al. (Re = 100), with Python
bindings and tests. Heat transport (temperature field and Boussinesq
coupling) is v0.2, scheduled for weeks 7–8, before P3. P4's heat-exchanger
runs depend on v0.2, not v0.1. [Confirm the week numbers against the P-plan.]
 
 
## 5. Gaps to close, in priority order
 
**Capacity.** 8 h/week each, including a 2 h joint session, leaves 6 h of
individual work per week: 6 × 6 = **36 h each, 72 h in total**.
 
**What the regression test protects.** It does not protect correctness.
The current code cannot provide a stable reference, because its pressure
comes from inverting a singular K0, and u depends on p through the
momentum RHS. So the pressure fix (#1) comes first, and the snapshot (#2) is
taken from the fixed code. After that, #4, #5 and #7 are pure refactors:
same maths, same results up to round-off (tolerance: relative 1e-8 on u, v,
p after 3000 steps, tightened if it holds easily). The snapshot catches
accidental changes during these refactors. **Correctness is checked by the
Ghia comparison (#9, #11).** Any intentional change to the numerics (e.g.
τ in v0.2) gets a new snapshot only after it passes the Ghia check.
 
| # | Gap | Owner | Hours | Depends on |
|---|---|---|---|---|
| 1 | Fix the pressure singularity: pin one node, factorise K0 once (sparse LDLᵀ for K0 only; the rest stays dense for now) | Kareem | 4 | — |
| 2 | Snapshot A: u, v, p (relative to the reference node) and ‖K1·u + K2·v‖ at Re = 100, 20×20, from the fixed code | Ahmed | 1 | 1 |
| 3 | Build system: CMake, Eigen, GitHub Actions CI | Ahmed | 5 | — |
| 4 | Sparse rewrite: K0, K1, K2 sparse, lumped mass vector, convection + SUPG as element-level RHS. τ formula unchanged. Must match snapshot A. | Kareem | 7 | 1, 2 |
| 5 | Split into modules (Config, Mesh, Element, Assembler, PressureSolver, Solver); exceptions instead of `exit(1)`. Must match snapshot A. | Kareem | 9 | 4 |
| 6 | C++ regression test against snapshot A (Catch2 or GoogleTest), run in CI | Ahmed | 3 | 2, 3 |
| 7 | `DirichletBC` by tag + cavity preset + "last call wins" rule. Preset must reproduce snapshot A. | Kareem | 5 | 5 |
| 8 | pybind11 bindings: Config, Mesh, DirichletBC, Solver, numpy arrays, exception translation | Ahmed | 10 | 5 (headers frozen, end of week 3) |
| 9 | Validation: Ghia comparison at Re = 100 on 20×20 and 40×40, observed order of convergence, error reported as a number | Kareem | 6 | 1 (first pass); rerun after 7 |
| 10 | `centreline()` from nodal values | Ahmed | 1 | 5 |
| 11 | pytest suite: API, node-ordering contract, Ghia tolerance test | Ahmed | 5 | 8, 9, 10 |
| 12 | Packaging: `pip install .` from source (scikit-build-core), no wheels | Ahmed | 3 | 8 |
| 13 | README / docs: API, contract, honest claims | Both | 4 (2 + 2) | 9 |
 
| | Kareem | Ahmed | Total |
|---|---|---|---|
| Work | 33 h | 30 h | 63 h |
| Capacity | 36 h | 36 h | 72 h |
| Margin | +3 h | +6 h | +9 h |
 
**Deferred to v0.2 to make this fit (−10 h):** τ at the Gauss point (3 h),
general `sample_line` (2 h), example notebook (3 h), 80×80 mesh in the
convergence study (2 h).
 
**Critical path:** #1 → #4 → #5 → #8 → #11. If #4 or #5 slips, Ahmed's
spare hours go to #9's post-processing (Ghia table and plots), not into a
new feature.
 
 
## 6. Known issues
 
### Issue 1 — K0 is singular, so the pressure is undefined (lines 115, 135, 216)
 
**Diagnosis.** K0 is the Galerkin Laplacian with no pressure Dirichlet
condition (pure Neumann). Each row sums to zero, so the constant vector is
in its null space and K0 has rank NT − 1. Eigen's `FullPivLU::inverse()`
requires an invertible matrix, so `K_inv` comes from dividing by a pivot
that is zero up to round-off. Its entries are therefore huge and dominated
by round-off.
 
In exact arithmetic, the damage would stay in the constant mode, because
K1·1 = K2·1 = 0 (partition of unity), and a constant pressure offset exerts
no force on the velocity. But the offset is huge, so round-off of about
|p|·1e-16 in K1·p and K2·p does reach the momentum RHS.
 
**Status: hypothesis, not yet tested.** This is the likely cause of the
erratic pressure column in the README table, and if it is right, u and v
from the current code are not a stable reference either. The checks below
(gap #1) are what prove or disprove it.
 
**Compatibility of the RHS.** The sum of the RHS entries is
−(1/ε)·∫ ∇·u_h dΩ = −(1/ε)·∮ u_h·n ds. On the top wall u_h·n = v = 0. The
side walls are only partly zero: in each top side element, u_h·n varies
linearly from 0 to the corner value 0.5 (bilinear interpolation), which
gives a flux of 0.5·dy/2 out through the left wall and the same flux in
through the right wall. These cancel only because both corners have the
same value. So the pinned system has a valid solution for this BC set,
but a different BC set (e.g. unequal corner values) would break it.
 
**Fix.** Pin p = 0 at one node: replace that row and column of K0 with the
identity, and set that RHS entry to 0. Then factorise once with a sparse
LDLᵀ and solve each step. Report p relative to that node.
 
**Checks (part of gap #1).**
1. K0·1 ≈ 0 at round-off level. This confirms the null space.
2. max|p| from the current code is very large. This confirms that the
   inverse is dominated by round-off.
3. Σ RHS_cont ≈ 0 at round-off level, every step. This confirms
   compatibility. Keep it as an assertion in `PressureSolver`, so a BC
   set that breaks compatibility fails loudly.
4. After pinning: the residual of the original (unpinned) equations is at
   round-off level, and the README table's pressure column becomes smooth.
   If the column is still erratic, the hypothesis is wrong and issue 1
   needs a new diagnosis.
### Issue 2 — The wall-force routine is wrong in three ways (lines 259–316)
 
**Diagnosis.**
1. *Element selection.* Line 263 calls `onBottomBoundary(element, nnx, nny)`,
   a node predicate, with an element index. It selects elements 0–20, but
   the bottom row is elements 0–19, so element 20 (the first element of the
   second row) is included by mistake.
2. *Wrong integral.* It integrates the traction over the element area with
   3×3 Gauss points and the area Jacobian. A wall force is a line integral
   along the wall edge.
3. *Pressure index.* Line 286 reads `p_old[element]`, a nodal array indexed
   by element number.
**Fix.** Loop over e = 0 … Ex−1. On the bottom edge (η = −1), use 1D Gauss
points in ζ with the edge Jacobian ds = (dx/2)·dζ. Evaluate the velocity
gradients with `maping(…, ζ_g, −1)`. Interpolate p = Σ N_z·p(conn[z]).
Integrate t = σ·n with n = (0, −1).
 
**Caveat.** After issue 1 is fixed, p is defined up to the reference
value, so Fy on a closed-cavity wall shifts by p_ref·L. Only Fx (shear) is
independent of the reference.
 
**Status.** Out of scope for v0.1 (Section 4).
 
### Issue 3 — Stale output labels (lines 249, 304, 323)
 
**Diagnosis.** File names are hard-coded as `"fineTime05_10x10_"` but the run
is 20×20, and "Matrix inversion complete" is printed at the end of the run.
 
**Fix.** Build names from `Config`:
`<output_prefix>_<nx>x<ny>_<step>.vtk`. Delete the stale message.
 
**Follow-up.** Check which mesh each row of the README table really came
from before reusing any of those numbers.
 
### Other issues found while reading the code
 
4. **Pressure lags velocity by one step.** The momentum RHS at step n uses pⁿ,
   which was computed from uⁿ⁻¹. This is consistent with explicit
   stepping, but it is documented here so nobody "fixes" the order by
   accident.
5. **ε is independent of dt.** A classical projection uses 1/dt as the
   divergence scale. Here 1/ε = 100 while 1/dt = 1000, so the divergence
   is relaxed rather than removed each step. ‖K1·u + K2·v‖ is not
   monitored. It should be logged (snapshot A includes it) before ε is
   tuned.
6. **τ ≈ dt/2.** The 2/dt term dominates, so the amount of stabilisation
   follows dt, not the mesh. τ also uses the nodal speed and a global h.
   This is deferred to v0.2.
7. **Dense O(NT²) storage, and K3 is rebuilt every step.** This is fixed by
   gap #4.
8. **`exit(1)` on a non-positive Jacobian (line 361).** This is fixed by
   gap #5.
9. **Corner value u = 0.5.** This is a regularisation choice that affects
   the comparison with Ghia near the lid. State it in the validation
   report.
 
