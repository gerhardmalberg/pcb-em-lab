# Validation status

Observed validation, most recently 2026-11-03. Historical checks are dated
separately below. This file records observed checks, not intended capabilities.

## Standalone regression, 2026-11-03

All 37 unit tests pass from an isolated copy of the maintained source files.
Both supplied CPU examples complete extraction, meshing, solving, postprocessing
and headless rendering using the installed images. The 3D example produces
Touchstone and fitted circuit exports. The copy contains no external hardware
checkout, prior outputs or stage caches. No image rebuild or new CUDA/GUI
validation was performed for this cleanup.

Local check records and logs are retained under
`results/validation/decoupling-2026-11-03/`. The examples remain teaching models
with the physical and numerical limitations described below.

## JSON configuration migration, 2026-11-01

All eight maintained environment/example manifests now use `.json`; their
contents and SHA-256 hashes are unchanged. Case parsing uses the standard
library and rejects YAML syntax. Documentation and VS Code tasks use the new
paths. Spack's required `spack.yaml` is generated from the JSON manifest only
inside build containers; JSON manifests and existing JSON locks remain the
maintained build inputs.

- All 37 lab tests pass, including parsing without site packages and invalid
  JSON/object rejection.
- The standalone 2D and 3D CPU pipelines complete through postprocessing.
  The 2D CUDA run uses an identical mesh; its differential-candidate L/C/Z
  values differ from CPU by at most 8.53e-9 relative. This is backend agreement
  on the teaching model, not mesh or physical-model qualification.
- Installed Spack accepts the renamed CPU manifest with its existing lock
  (75 resolved packages). Editable installation succeeds offline. A full
  source-image rebuild was not repeated for this migration.
- NVIDIA tooling generates and reads a temporary JSON CDI spec (three device
  aliases). Bootstrap scripts now configure JSON output and preserve backups
  of old generated specs. Existing host GPU configuration was not changed.

## Standalone installation and GUI example, 2026-11-01

The CPU, tools and CUDA images were rebuilt from the pinned recipes. Both full
`doctor` runs pass, including native tool interfaces, headless rendering, GUI
launchers, circuit handoffs and Palace references. The working graphics driver
was preserved. The FLTK bootstrap now defers configure to Spack using
`NOCONFIGURE=1`; KiCad remains 10.0.7 from the pinned snapshot.

| Current check | Observed result |
|---|---|
| Unit suite | 22 passed |
| Standalone 2D / 3D CPU pipelines | Complete, approximately 15.7 / 25.2 seconds with installed images; two ranks, one thread per rank |
| Stage cache | All four stages reused, approximately 0.25 seconds |
| Fresh repository-only CPU run | Passed in a copy excluding ignored files, caches and results; reused installed container images, not prior stage artifacts |
| GUI reopening | PCB Editor, FreeCAD, Gmsh 2D/3D, ParaView 2D/3D and Qucs-S passed with authenticated displays and preserved input hashes; screenshots inspected |
| Qucs-S actual FEM handoff | F2 simulation generated eight frequency points; maximum absolute S21 difference from Touchstone 1.57e-16 |
| VS Code workspace | Task menu and cached complete 2D CPU task exercised in an isolated profile; terminal screenshot inspected |
| CPU/CUDA reference agreement | Maximum relative differences 5.4871e-5 (BoundaryMode), 2.2266e-6 (Driven), 1.5916e-10 (Magnetostatic) |

The standalone cases themselves ran on CPU; GPU validation used the three
reference problems.

The 2D differential candidate (mode 2) is approximately 111.674 Ω. The other
mode has an ill-conditioned terminal-derived impedance and must not be treated
as a validated common-mode impedance. Both teaching cases use coarse meshes,
finite air domains and lossless materials/PEC copper without convergence studies.
The 3D circuit fit passes the implemented passivity/error checks with RMS error
0.03319; inspect the retained fitting diagnostics before reusing it. Neither case
qualifies a manufactured PCB. Field images use the normalization stated on them.

Evidence for the earlier setup is under ignored `results/validation/`: `cpu/doctor.json`,
`cuda/doctor.json`, `cuda/palace/references.json`,
`standalone-setup/progress.json`, `standalone-setup/clean-copy.json`,
`example-gui/checks.json` and `example-vscode/check.json`.
Native example outputs and provenance are under
`results/examples/differential-pair/{2d,3d}/`. The initial GUI authentication
failure is retained in progress history; after fixing the Xauthority hostname
match, all seven actual example GUI checks passed.

## Reusable runtime and numerical checks

Historical CPU Palace BoundaryMode, Driven and Magnetostatic reference checks
passed, with maximum relative CSV differences approximately 1.07e-4, 1.31e-5
and 5.65e-8. The container RC reference exercises Touchstone import, passive
fitting, floating-port ngspice, XSPICE, OpenVAF/OSDI and QucsatorRF handoffs.
These checks validate tool interfaces on reference problems; each physical model
still requires its own geometry, material and convergence assessment.

The runtime selection checks cover explicit CPU/CUDA selection, automatic CPU
selection when a requested GPU is unavailable, and explicit CUDA failure. Solver
failure never switches backends. CPU isolation checks verify absence of CUDA
package dependencies and visible NVIDIA devices. Evidence is retained under
`results/validation/backend-after-cuda.json` and `results/validation/cpu/`.

Reusable safeguards include:

- Explicit frequency-by-port reference-impedance arrays, including equal frequency
  and port counts; standard Touchstone reference impedances and preserved polarity.
- Configurable Delaunay/HXT meshing, CAD tolerance and drill faceting, with
  nonpositive tetrahedron quality rejected before solving. Meshing defaults to
  one thread because the pinned Gmsh build lacks OpenMP.
- Separate linear-solver and error-estimator tolerances. Approximate estimator
  diagnostics do not establish mesh convergence.
- Passive fitting with retained passivity/error diagnostics, and circuit checks
  that inspect logs as well as exit codes and numerical responses.
- Rootless containers using the caller's UID/GID and an init process; authenticated
  GUI forwarding and separate Python runtimes for binary tool modules.

The earlier CPU image export/import check preserved the image ID. Transfer to a
second physical server remains untested. See [server transfer](servers.md) for
storage guidance and the OpenVAF recompilation limitation.

## CUDA memory policy and diagnosis

A traced 18 MB `cudaMalloc` request failed with out-of-memory status before the
original illegal-access failure. HYPRE's initial Umpire pool was 4 GiB on this
6 GiB card; libCEED's returning error handler could not safely propagate the
allocation error through its success-assuming macro. The small source-controlled
adapter uses HYPRE's public pool-size API and aborts clearly on that CUDA
allocation error. See the pinned
[HYPRE pool defaults](https://github.com/hypre-space/hypre/blob/v3.2.0/src/utilities/umpire.c)
and [libCEED error macro](https://github.com/CEED/libCEED/blob/8a374e8d5d8d33fd19ce69a93026384ec1046a86/include/ceed/macros.h).

The CUDA image now uses direct MFEM allocations and a 256 MiB initial HYPRE
pool, which can grow. An impossible 8 TiB allocation verifies the explicit error
path; it does not trigger CPU fallback. The patch was compiled offline against
the existing image, and all 82 packages in its active Spack environment retain
the same hashes. Image lineage and adapter source/binary hashes are recorded in
`env/locks/cuda-75-12.9.1/image.json`. Future CUDA builds compile the same adapter.
The host driver and physical models were unchanged. Reassess this adapter when
upgrading HYPRE or libCEED; it addresses the pinned versions, not every release.

The unfiltered Compute Sanitizer log includes OpenMPI pointer-classification
API errors. Its
[pinned implementation](https://github.com/open-mpi/ompi/blob/v5.0.10/opal/mca/accelerator/cuda/accelerator_cuda.c)
tries `cuMemRetainAllocationHandle` and handles failure when a pointer is not a
VMM allocation. The separate invalid-memory-access check uses
`--report-api-errors no`; the unfiltered log is retained for inspection. This
filter does not disable memory-access checking. The final image completed the
full static run with zero reported memory errors. Evidence is collected
under `results/validation/cuda-small-pool/` and `results/validation/cuda/`.
