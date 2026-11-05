# Offline relaxed adaptive comparison

**Current contact-edge replay:** use commit
`7178cefe7df7c7aef5bf25ee17ed0cb8560e6f61` and the
[contact-edge commands below](#contact-edge-representative-explicit-opt-in-same-number-of-strips).
The earlier `5b1abff` commands retain the historical unchanged-C placement.

This opt-in script writes local preview files. It imports no ROS node, robot
client, motion executor, or controller. Existing planner defaults, product A,
and the existing adaptive C pose convention are unchanged.

The source revision is recorded in `index.json` as `source_commit`, together
with before/after SHA256 hashes of input files and the implementation call
chain. Keep that revision and the input packet with the output when copying a
run to another PC. The reproducible CLI implementation is pinned to research
commit **`5b1abff0a0d893f188afbfac5796a8701c855e20`**. The commands below use
that source in a separate offline worktree; they do not replace an installed
application checkout or deploy anything to equipment.

## Actual A and new C

A uses `replay_registered_product` and the actual product raster sampler,
registered strip selection, converter, and full `SurfacePlanningSession`.
The default sweep is **15, 20, 25, 30, 35, 40, 45, 50 mm**. For nominal
15/20 mm only, the sampler receives the requested spacing while the canonical
session audit uses 25 mm, its existing lower bound. Both numbers and the
measurement exception are recorded. Nipple confirmation, piece ordering, and
force handling remain enabled. A nipple-confirmation failure is retained as a
failed canonical case; no substitute planner is used.

`--diagnostic-a-if-nipple-unconfirmed` is an explicit opt-in, default OFF.
Only the exact canonical `ValueError` stating that the nipple is not confirmed
on the registered surface triggers a separate **A_diagnostic** row. The
canonical failure remains in the index and its own JSON. The diagnostic repeats
the same actual sampler, registered strip selector, and converter, preserving
all per-strip quality gates, but omits the full surface session. It has no
canonical session audit, nipple confirmation, force profile, or session ordering.
Its output is measurement-only, explicitly not product-ready. Other errors do
not trigger this diagnostic. No diagnostic prepares robot execution.

With the default `--boundary-policy legacy`, C calls the unchanged placement of
`plan_adaptive_raster` with **supplements=False** and
explicit `alpha` (0.5 or 1) and indentation (3, 5, or 8 mm). No new strips,
spacing correction, endcap pass, or attitude limit is added. The legacy placement wrapper changes only poses:

- tool Z = smooth signed outward normal;
- tool X = normalized projection of global HOME +Y into its tangent plane;
- tool Y = tool Z cross tool X.

XYZ and force arrays are preserved exactly. Waypoint pose dictionaries agree
with the pose arrays. The existing C normal/height/curvature estimator and
raster placement are untouched. Preview output remains `execution_ready=false`.
The selected research representative is **alpha=0.5, indentation=8 mm,
supplements=False**. The commands below pass these values explicitly; no product
default changes. This selection prioritizes a less dense raster for the offline
comparison. It is not acceptance of residual holes or certification of physical
indentation, force, or contact.

## Real-PC input contract

For a raw packet, provide an NPZ loaded with `allow_pickle=False`:

| Key | Shape / meaning |
| --- | --- |
| `points` | finite N×3 positions, metres, one common base frame |
| `normals` | finite N×3 nonzero signed surface-outward vectors in that frame |
| `nipple` | finite length-3 annotation in that frame, metres |
| `roi_boundary_base_m` | finite K×3 boundary, K≥3, in that frame, metres |
| `roi_mask` | optional native HOME H×W uint8 mask; selected pixels are 255 |
| `home_depth_m` | optional native HOME H×W optical depth in metres; 0/NaN are missing |
| `nipple_pixel` | optional native integer (u,v); otherwise annotation is projected |
| `point_unit` | optional scalar string, must equal `m` when supplied |
| `base_frame` | optional scalar string, must match HOME JSON when supplied |

The HOME JSON contains `width`, `height`, 3×3 `camera_k`, rigid 4×4
`base_from_camera`, and `base_frame`. `base_from_camera` maps native optical
coordinates (X right, Y down, Z forward) into the common base frame.
Optional `preferred_direction_base` is a finite length-3 longitudinal vector;
its HOME tangent projection defines HOME +Y. Otherwise HOME +Y is negative
camera Y; HOME outward +Z is negative camera Z; HOME +X = +Y cross +Z.
All reported axes are rows expressed in the common base frame. The camera
origin defines the HOME origin. No pixels, positions, or units are silently
rescaled. Raw normal lengths are explicitly normalized; their signs are kept.

Raw PN are passed through the existing single-HOME product surfel preparation
and quality/visibility selector. A and C share the resulting valid registered
PN. This reconstructs product surfels and **does not recreate original capture
lineage or persisted registration quality**. Original supplied normal lengths
are validated, then product preparation refits its own normals. This distinction
is saved in `preparation`.

For an original recorded run, supply `--registered-run` (and its trusted profile
if needed) instead. Its verified persisted registered context is authoritative;
raw refusion is not performed. A supplied HOME JSON must agree with that
context. The packet still supplies the nipple, ROI, and optional native mask
and HOME depth. Output and cache must be outside that immutable run directory.

## Commands

Create an offline worktree from a clone containing the pinned commit:

```bash
git worktree add --detach /data/autosonix-offline-research 5b1abff0a0d893f188afbfac5796a8701c855e20
cd /data/autosonix-offline-research
git rev-parse HEAD
```

Run from that worktree root using the validated product Python/native environment.
The following environment was used for this source checkout's focused tests;
replace installation-specific paths on the real PC with its validated equivalents:

```bash
export AUTOSONIX_GRAPH_BACKEND=cpp OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1
export PYTHONPATH="/home/sewon/.cache/raster-contact-review-20261103/native-overlay:Integration/tools:Integration/src/autosonix_scan_core:Integration/src/dimension_converter:Integration/src/autosonix_registration:Integration/src/rb_data_handler:/home/sewon/.cache/hybrid-main-e9d03e9/build/python"
```

The primary same-input comparison runs all eight A spacings and one explicit C
case together, using the selected alpha=0.5 / indentation=8 mm representative:

```bash
/home/sewon/.cache/hybrid-mpfb-validation/venv/bin/python Integration/tools/replay_adaptive_relaxed.py \
  --mode all --input /data/packet.npz --home /data/home.json \
  --output-dir /data/results/relaxed-same-input-off-01 --cache /data/cache/packet-surfels \
  --alpha 0.5 --indentation-mm 8 --center-pass off \
  --diagnostic-a-if-nipple-unconfirmed
```

The command explicitly requests separate diagnostic A measurements when the
canonical nipple gate refuses; omit that flag to keep only canonical A rows.
All A and C cases use the same packet, camera frame, source hash fence, and
prepared common PN cache. Repeat with `--center-pass on` and a different fresh
output directory for ON. Use `--mode a` to run only the A sweep or `--mode c`
with explicit parameters to run only C. Repeat `--spacing-mm 25 --spacing-mm 40`
to restrict A to selected values; a restricted sweep is labeled by its actual
rows and cannot imply a full eight-spacing comparison.

Only if the measured base-eight A range does not bracket the unchanged C
contact length, request a **targeted auxiliary A measurement** explicitly.
`--allow-extended-a-measurement` (default OFF) permits whole nominal millimetres
from 5 through 200. The actual sampler receives that nominal spacing; the
standard full-session audit is clipped to 25–50 mm. For example, nominal
10 mm uses audit 25 mm and nominal 100 mm uses audit 50 mm. Unsupported GUI
spacing, requested nominal spacing, actual audit spacing, and opt-in status are
recorded separately. Canonical nipple confirmation remains enabled; an
explicit diagnostic row still requires the independent diagnostic flag.
No extra A values are selected automatically, and C is never changed.

If C falls below the shortest measured A contact length, a targeted 100 mm
auxiliary measurement can test a shorter A raster on the same packet:

```bash
/home/sewon/.cache/hybrid-mpfb-validation/venv/bin/python Integration/tools/replay_adaptive_relaxed.py \
  --mode all --input /data/packet.npz --home /data/home.json \
  --output-dir /data/results/relaxed-targeted-a100-off-01 --cache /data/cache/packet-surfels \
  --alpha 0.5 --indentation-mm 8 --center-pass off \
  --spacing-mm 100 --allow-extended-a-measurement \
  --diagnostic-a-if-nipple-unconfirmed
```

If C instead exceeds the longest measured A contact length, use a targeted
10 mm measurement in its own fresh output directory:

```bash
/home/sewon/.cache/hybrid-mpfb-validation/venv/bin/python Integration/tools/replay_adaptive_relaxed.py \
  --mode all --input /data/packet.npz --home /data/home.json \
  --output-dir /data/results/relaxed-targeted-a10-off-01 --cache /data/cache/packet-surfels \
  --alpha 0.5 --indentation-mm 8 --center-pass off \
  --spacing-mm 10 --allow-extended-a-measurement \
  --diagnostic-a-if-nipple-unconfirmed
```

That index labels its single requested auxiliary A measurement; it does not
claim an eight-point curve. To retain the base-eight curve and the chosen
auxiliary spacing in one comparison index, explicitly repeat all eight
`--spacing-mm` values plus the targeted value in a fresh run. Match decisions
use only actual measurements present in that index, with no extrapolation.

For original registered data, add `--registered-run /data/recorded-run` and,
when needed, `--trusted-profile /data/trusted-profile.json`; `--home` becomes
optional. A valid raw preparation cache may be reused with the same packet,
HOME calibration, producer source, and native binary. Outputs are never replaced.

## Outputs and motion measurements

`index.json` records every case and failure, source/input hashes, common-frame
normalization, preparation provenance, and the exact checked-out source commit.
The index's `length_match` includes every requested A spacing and its actual
contact length (or explicit failure), all separate diagnostic A rows, the C
target length, shorter/longer A neighbours by **measured length**, and the nearest A with signed
`C_minus_A_length_m`. Nominal spacings for the bracket are retained even if
lengths are not monotonic with spacing. An exact budget match means absolute
length difference ≤1 mm; its tolerance is recorded. Matching prefers a
successful canonical A at each nominal spacing; only when
that canonical row failed may its explicitly requested diagnostic measurement
be selected. Every selected row identifies its measurement source. Denominators
state canonical successes/failures, diagnostic attempts/successes/selections,
and the final number of measured A spacings; diagnostic rows never replace or
increase the canonical success denominator. Missing neighbours and out-of-range
C lengths remain unbracketed. No A path or hole count is interpolated. Without an independent reference,
no coverage score or coverage interpolation is produced; measured lengths alone
are not a coverage result. A-only/C-only
runs report that their comparison counterpart is unavailable.

Successful canonical A/C case JSON contains actual geometry, poses, forces,
and waypoint dictionaries; diagnostic A has no invented force profile. All
case artifacts retain selection metadata and the preview contract when available. Missing reference contact/coverage
remains unscored; this script does not infer reference anatomy.

For new C, heading is the HOME-XY azimuth of tool X, unwrapped within each
fragment. For unchanged A, the equivalent longitudinal axis is tool Y and the
outward axis is negative tool Z. Tilt is the angle of the outward axis to
positive HOME Z. Full SO(3), heading change, and tilt change use **actual 10 mm
arc windows** with SLERP, a 0.5 mm start grid, and original waypoint-aligned
window endpoints. Windows never cross fragment gaps. Fragments shorter than
10 mm have no change window; their absolute delivered-point tilt is still
reported. Degenerate longitudinal XY projections are counted as unmeasured.
No short-edge rate normalization or attitude rate cap is applied. Lifts,
transfers, robot timing, and physical force/contact feasibility are not measured.


## Contact-edge representative: explicit opt-in, same number of strips

The current main candidate remains **C α=0.5 / planning δ=8 mm / no supplements**.
The boundary change is available only with `--boundary-policy contact_edge`;
the command-line and planner default remain `legacy`. The new policy moves
existing outer references using transverse smooth-surface distance and the
local effective contact halfwidth, then repositions the existing interior
references using C's contact-width/slope rule. It does not add strips. The
central nipple-pass reference is retained. The case's `planner_metadata` saves
placement diagnostics, original/new references and unresolved boundary
residuals. Constant-X lines cannot touch every point of a varying ROI boundary
simultaneously; unresolved locations are not silently treated as covered.

Use the exact edge-replay commit recorded with the published report in a
separate offline worktree. The historical commit near the top of this document
reproduces the earlier relaxed study only; it does not contain this new flag.
The updated executable records the actual checked-out commit and source hashes
in every replay's `index.json`.

```bash
git worktree add --detach /data/autosonix-contact-edge 7178cefe7df7c7aef5bf25ee17ed0cb8560e6f61
cd /data/autosonix-contact-edge
git rev-parse HEAD
```

```bash
/home/sewon/.cache/hybrid-mpfb-validation/venv/bin/python Integration/tools/replay_adaptive_relaxed.py \
  --mode all --input /data/packet.npz --home /data/home.json \
  --output-dir /data/results/contact-edge-off-01 --cache /data/cache/packet-surfels \
  --alpha 0.5 --indentation-mm 8 --center-pass off \
  --boundary-policy contact_edge --diagnostic-a-if-nipple-unconfirmed
```

The default A measurement sweep remains 15/20/25/30/35/40/45/50 mm. Repeat with
`--center-pass on` and a fresh output directory for ON. The explicit diagnostic
flag preserves canonical failures and appends separate sampler-only A rows; it
never makes a rejected canonical case successful. To reproduce the earlier C
placement on exactly the same input, repeat with `--boundary-policy legacy` in
another fresh output directory. No output overwrites or robot entry points are
used.

Every replay now writes **`comparison.html` and `comparison.json`**, even when
no reference surface is available. The static HTML contains a table of every
normal/diagnostic/failed A and C case, a measured A length curve with C's budget,
shorter/longer/nearest A measurements, and actual HOME-XY nearest-A/C output
paths. The nearest path is explicitly not an interpolated or exact same-length
A path; its signed length difference is listed. If C is outside the measured A
range, the comparison remains `outside_A_length_range`. Targeted extra A
measurements remain manual opt-ins as documented above. The report never
changes C or adds strips to achieve a comparable budget.

### Optional independent reference for coverage curves

A reference surface is not required for planning. Without one, the HTML says
`coverage_unmeasured_without_independent_reference`. Do not infer coverage from
path length, a planning height model, or incomplete observed PN alone.

If an independently prepared reference exists, append
`--reference-npz /data/reference.npz`. This NPZ must contain:

| Key | Meaning |
| --- | --- |
| `vertices` | finite N×3 metre coordinates in the packet base frame |
| `faces` | nonempty integer M×3 triangle vertex indices, consistently outward winding |
| `roi_face_mask` | boolean M-vector, at least one true, common evaluation ROI |
| `domain_face_mask` | optional boolean M-vector containing ROI; context allowed for surface-distance propagation, with disconnected/folded sheets separated beforehand |
| `point_unit` | required scalar string `m` |
| `base_frame` | required scalar string exactly matching authoritative HOME |

This reference is included in input SHA-256 checks before and after the replay
and evaluation. It is transformed into the same HOME frame and passed **only
to the evaluator**, never to C or product A. Its truth/measurement provenance
must be kept with the data; the tool labels it user-supplied independent
reference and never assumes it is GT. Mask consistency and finite/integer
geometry are validated. Clinical validity, scan-domain labeling and physical
contact are not certified by this file format.

The report evaluates the **same C geometry planned at δ=8** at evaluation
δ=3/5/8 mm. For each A/C case it reports contact-model coverage, maximum
numerical surface hole, and projection residual. It plots actual measured
A coverage versus path length and C as a point. If any canonical A succeeds,
the coverage curve uses **canonical A only**. Diagnostic A is used for a
**diagnostic-only** coverage curve only when no canonical A succeeds. The two
sources are never combined into one coverage envelope; `coverage_curve_source`
and exclusions are explicit in JSON/HTML. The actual case table and length
bracket retain every normal and diagnostic measurement separately. Only
coverage is interpolated, using the cumulative-best measured A envelope of
that single source, **only inside its measured length range**.
No extrapolation or interpolation of holes, poses or paths is performed.
The existing isotropic contact-disk model is an indentation assumption, not
measured probe pressure or a robot load command.
