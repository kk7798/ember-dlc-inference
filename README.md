# EMBER DeepLabCut inference on Pennsieve

Upload this folder's contents to the root of `ember-dlc-inference` on GitHub.
Pennsieve builds `Dockerfile` and uses `app.yml`. No training code, weights, local
launcher, or datasets belong in this repository.

## Pennsieve inputs

Supply the complete `dlc_project` produced by `ember-dlc-training` and media to
analyze under Pennsieve's `INPUT_DIR`. Folder names around the project are arbitrary.

```text
INPUT_DIR/
  dlc_project/
    config.yaml
    early_stopping.json
    training-datasets/...
    dlc-models-pytorch/.../train/
      pytorch_config.yaml
      early-stopping/best-validation.pt
  bees_test/
    CollectedData_EMBER.h5
    image.png
```

Inference discovers one model project. It reads
`best_checkpoint_relative_to_project` from `early_stopping.json`, resolves it
against the relocated project, and uses that exact checkpoint. The old absolute
training path is ignored. The best epoch, validation loss, checkpoint path and
SHA-256 are recorded in `OUTPUT_DIR/inference_run.json`.

The project is copied into `OUTPUT_DIR/model`. Only this working copy receives a
DLC-compatible snapshot alias. The processor explicitly selects that alias;
other numbered or best snapshots cannot override it. Missing or invalid training
metadata, missing weights, or a shuffle/training-fraction mismatch cause failure.
There is no automatic fallback to another checkpoint.

## Parameters

- `SINGLE_IMAGE=true` (default): process exactly one image. Labeled test data takes
  priority; select the alphabetically first labeled image and evaluate only its
  annotation row. Otherwise select the first external image. Additional images
  and all videos are skipped. The selected path is recorded in `inference_run.json`.
  Keep this enabled for the current smoke test; set false only for a full run.
- `CHECKPOINT_POLICY=best-validation` (default): use the training metadata above.
- `CHECKPOINT_POLICY=snapshot`: explicit compatibility mode for older training
  runs without early stopping; `SNAPSHOT_INDEX=-1` uses DLC's last ordered snapshot.
- `SHUFFLE=1`, `TRAININGSET_INDEX=0`: must match the trained model.
- `DEVICE=cpu`, `CPU_THREADS=4`, `BATCH_SIZE=1`: execution settings. GPU use requires
  GPU compute provisioned in Pennsieve as well as `DEVICE=cuda:0`.
- `PLOT_IMAGES=false`, `PCUTOFF=0.6`, `PCK_PIXELS=10`: optional overlays and evaluation.
- `DETECTOR_SNAPSHOT_INDEX=-1`: separate detector selection for top-down projects;
  early-stopping metadata selects the pose checkpoint only.

Labeled `<Task>_test` data must sit outside the model project and match its species
and skeleton. External PNG/JPEG images and supported videos need no annotations.
Training/validation folders and model assets are excluded from automatic discovery.
If multiple projects are staged, set `MODEL_DIR` to the intended project; relative
paths resolve under `INPUT_DIR`. `DATA_DIR` optionally restricts media discovery.

## Outputs

Pennsieve collects `OUTPUT_DIR`, containing `inference_run.json`, the working model,
prediction HDF5/CSV files, and optional image overlays and held-out evaluation.
Evaluation uses per-frame Hungarian animal matching and reports coverage, pixel
errors and PCK; these are not official EMBER benchmark mAP. Video slots do not
provide persistent animal identities. Use a fresh output location for each run.

The runtime is DeepLabCut 3.0.0/PyTorch. Local container checks do not establish
successful Pennsieve registration or scientific accuracy of a trained model.

## EMBER DANDI 000598 workflow

Use three Pennsieve steps, with GitHub-backed processors:

1. DANDI CLI download: `DANDI_ACTION=download`,
   `DANDI_URL=https://dandi.emberarchive.org/dandiset/000598/draft`.
2. Connect `downloaded_files` to this processor's `inference_bundle`.
3. Connect `inference_results` to DANDI CLI upload's `files` input.
   Set `DANDI_ACTION=upload`, `DATASET_ID=000598`,
   `DANDI_INSTANCE=ember-dandi`, `DESTINATION_PREFIX=inference-results`,
   and `UPLOAD_VALIDATION=require`. Supply `DANDI_API_KEY` through Pennsieve
   (prefer a secret/environment mapping); never commit it to GitHub.

The existing DANDI uploader stages files in a unique timestamp/ID folder and
writes `upload-receipt.json`; no archive deletion or synchronization is required.
Use the updated uploader that implements this behavior.

### Inspected archive state, 2026-11-09

The complete 1,468-asset listing for draft 000598 contains the trained project:

```text
workflow-results/20261023T195506Z-bd5f564ec9f54beba0460afd41e1c05a/dlc_project
```

Its `dlc_training_run.json` reports a completed one-epoch run with early stopping
**disabled**. Its saved weight is `snapshot-001.pt`; neither `early_stopping.json`
nor `best-validation.pt` is present. For this archived run explicitly set:

```text
CHECKPOINT_POLICY=snapshot
SNAPSHOT_INDEX=-1
SHUFFLE=1
TRAININGSET_INDEX=0
MODEL_DIR=000598/workflow-results/20261023T195506Z-bd5f564ec9f54beba0460afd41e1c05a/dlc_project
```

`MODEL_DIR` above assumes the DANDI download output retains the `000598/` root.
Confirm the download output layout in Pennsieve and adjust the relative path if
it differs. The manifest exposes MODEL_DIR and DATA_DIR for this purpose.

No held-out test data or external inference media is currently present. Only
`bees_train` and `bees_val` images exist, both in the source project and its
training-output copy. Add held-out `bees_test` images and matching annotations,
or external media, before running inference. Then set DATA_DIR to that downloaded
folder; its exact archive path has not yet been chosen. Do not use training or
validation images as a held-out test set.

For a future early-stopping training run, point MODEL_DIR at that run's complete
project and restore `CHECKPOINT_POLICY=best-validation`. Repeated full-dandiset
downloads include previous results, so explicit MODEL_DIR and DATA_DIR selections
prevent ambiguous model discovery and unintended reprocessing.

The configured topology is download → inference → upload; training is separate.
This workflow has not been executed on Pennsieve in this session.
