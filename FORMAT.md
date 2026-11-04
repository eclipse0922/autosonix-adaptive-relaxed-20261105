# Actual XYZ/RPY public schema

Index schema: adaptive_relaxed_public_v1. Case payload schema: actual_xyz_rpy_v1.

Every `lineN` is N×3 actual XYZ metres. Every `rpyN` is the matching N×3 actual fixed-axis xyz Euler radians. Join them columnwise to recover every actual XYZRPY waypoint in its original order. Separate pieces remain separate. No point decimation, resampling, coordinate quantization or rounding is applied. Float64 bytes, including signed zero, are compared against original source XYZ and RPY after reopening the written gzip.

The source PN normals are not separately exported. Original source PN arrays remain private in the local NPZ identified by npz_sha256. C uses positive tool Z as outward; A and the explicitly labeled diagnostic A use their legacy negative tool Z. Derive the correct signed axis from each case frame_policy and actual RPY; this does not assert identity with the omitted source PN normals.

Arrays use float64-byte-shuffle-base64-v1 with shape, dtype and SHA256 over canonical little-endian float64 bytes. Reverse byte-plane transposition before interpreting float64. Repeated arrays may reference an earlier byte-identical array in the same payload. strip_ids and supplement_flags preserve piece labels. array_manifest verifies shape, dtype and bytes. Source hashes and written payload hashes remain in source-payload-manifest.json.

GT meshes, raw source NPZ, diagnostic grids and source PN normal columns are not public. Full metrics and provenance are in index.json.gz; tabular metrics are in metrics.csv.gz. Null means unassessed.

## Scientific case index packaging

Small index.json.gz files may contain cases inline. Large index.json.gz is a metadata manifest with cases=[] and case_parts_format=adaptive_relaxed_case_parts_v1. Each ordered case_parts entry contains asset, sha256 (compressed file bytes), and case_count. Download every listed data/index-part-NNN.json.gz, verify its compressed SHA256, decompress JSON, and concatenate its cases arrays in listed order to recover all scientific cases without field omission or rounding. Each part has schema adaptive_relaxed_case_part_v1. The manifest case_count is the complete union count. Actual geometry case assets remain byte-identical. Static HTML lists all part links; JavaScript is not required for core tables or downloads. Every published asset must be smaller than 24 MiB.
