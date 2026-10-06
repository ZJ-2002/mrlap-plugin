# mrlap plugin (WO-R-02, scaffold 2026-10-06)

Containerized **official R MRlap** — the release path for the A-family
sample-overlap-aware MR correction. The native Rust `mrlap` node carries
documented, unvalidated deviations from the official pipeline
(`crates/node-bundles/nodes-mr/src/mrlap.rs:10`: M definition, jackknife
blocks, fixed EUR panel, EAF palindrome handling) and is FORBIDDEN from
the formal chain until certified (node acceptance matrix row 3). This
family is row 4: `mrlap_official`.

## Layout

- `manifest.toml` — node kind `mrlap_official`: one edge per instance,
  three file inputs (exposure TSV, outcome TSV, instrument rsID list),
  four outputs (17-field summary TSV, RDS, print text, log), seed param
  default 1, LD/HM3 panel paths
- `scripts/mrlap.sh` — pure R script (harmonise.sh pattern) porting the
  frozen canonical `WOE4_mrlap_seed_rerun.R` (canonical v2): RNGkind +
  per-edge `set.seed(seed)` immediately before `MRlap()`,
  `do_pruning = FALSE` fixed, `user_SNPsToKeep` from port 2, defensive
  `g()` summary builder (absent quantity stays NA, never 0)
- `Dockerfile` — R 4.6.1 base; method packages pinned to the exact
  commits recorded by the seed1 acceptance host library (MRlap @660f026
  sha256:42e74c0c…, TwoSampleMR @d4df219/0.7.11 sha256:a203d741…,
  GenomicSEM @da95d431/0.0.5 sha256:cf5f1a7f…); support stack from the
  2026-09-13 CRAN snapshot (same as the twosamplemr family image);
  build-time stopifnot on all three versions
- `test_mrlap_official.sh` — parity gate: replays the three seed1 edges
  (XM/XY/MY) and compares every summary field against the frozen
  `WOE4_mrlap_summary_seed1.tsv`

## Deploy gates (open — DO NOT add to plugins.toml until all closed)

1. **Image digest pin**: build, push to `ghcr.io/ZJ-2002/mrlap-official`
   (cellphonedb precedent), then replace `reference =
   "localhost/mrlap-official:0.0.3.3"` in `manifest.toml` with the
   `@sha256:` digest. Digest-pinning is mandatory: the reference pins
   algorithm + R environment together.
2. **LD panel registration**: publish `~/tools/genetics/ldsc_assets`
   (211 MB: 22 per-chromosome LD-score gzips + M_5_50 +
   `w_hm3.noMHC.snplist`, the 45-file content identity established in
   the seed1 acceptance) as
   `wjixiang/catalog-mrlap-ldsc-eur-w-ld-hm3-no-mhc` and materialize it;
   the DAG cannot bind the panel until then.
3. **Container parity**: `MRLAP_CONTAINER=1 test_mrlap_official.sh`
   green against the digest-pinned image, plus a deployed-DAG reproduction
   of the three edges, before the matrix row flips to ACCEPTED.

## Boundaries

- MRlap standardizes `Z/sqrt(N)` internally and its HM3 IV subset differs
  from the conventional IVW sets — the summary columns must never be
  subtracted from or compared "better/worse" against `twosamplemr` family
  outputs; they are separate conventions (seed1 acceptance §"不应称为…").
- TwoSampleMR 0.7.11 here vs 0.7.9 in the twosamplemr family image are
  independently recorded environments; no cross-validation between them.
- Effects are on standardized scales; NA-vs-NA comparisons stay honest.
- M/Y frozen feeds carry duplicate rsID rows (multi-allele
  representations); the plugin preserves input bytes and does not
  constitute general QC acceptance.
