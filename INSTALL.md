# Installing Loss Prevention Pipelines into ViPPET

This guide describes how to install the Loss Prevention pipeline YAML files into
an existing [ViPPET](https://github.com/open-edge-platform/edge-ai-libraries/tree/main/tools/visual-pipeline-and-platform-evaluation-tool) installation.

## Pipeline Files

The following pipeline definitions are provided:

| File                             | Description                                                                                         |
| -------------------------------- | --------------------------------------------------------------------------------------------------- |
| `loss-prevention-retail.yaml`    | Object detection (YOLO 11n) + classification (EfficientNet B0) for retail loss prevention workloads |
| `loss-prevention-face-reid.yaml` | Face detection (face-detection-retail-0004) + re-identification (face-reidentification-retail-0095) |
| `loss-prevention-detection.yaml` | Object detection only (YOLO 11n) for benchmarking                                                   |

Each file includes CPU, GPU, NPU, and GPU\_NPU variants (detection-only has CPU, GPU, NPU).

## Prerequisites

- A working ViPPET installation (see [ViPPET Get Started](https://github.com/open-edge-platform/edge-ai-libraries/blob/main/tools/visual-pipeline-and-platform-evaluation-tool/docs/user-guide/get-started.md))
- Docker Compose services running or ready to start
- Required models installed via ViPPET's model management

## Installation Steps

### Step 1: Locate the ViPPET Pipelines Directory

If you installed ViPPET from source (cloned the repository):

```bash
cd edge-ai-libraries/tools/visual-pipeline-and-platform-evaluation-tool
PIPELINES_DIR=vippet/pipelines
```

If you set up ViPPET using the quick-start (downloaded compose files):

```bash
# The pipelines directory is baked into the vippet container image.
# You will need to rebuild the image after adding files (see Step 3).
PIPELINES_DIR=vippet/pipelines
```

### Step 2: Copy the Pipeline YAML Files

Copy the loss-prevention pipeline files into the ViPPET pipelines directory:

```bash
cp configs/vippet/loss-prevention-retail.yaml      $PIPELINES_DIR/
cp configs/vippet/loss-prevention-face-reid.yaml    $PIPELINES_DIR/
cp configs/vippet/loss-prevention-detection.yaml     $PIPELINES_DIR/
```

### Step 3: Install Required Models

The pipelines require the following models. Use ViPPET's model management to
install them, or ensure they are present under `shared/models/output/`:

| Model                                    | Path in Container                                                                                 | Source                                  |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------- |
| YOLO 11n (INT8)                          | `/models/output/public/yolo11n/INT8/yolo11n.xml`                                                  | Ultralytics YOLO 11n, quantized to INT8 |
| EfficientNet B0 (INT8)                   | `/models/output/object_classification/efficientnet-b0/INT8/efficientnet-b0.xml`                   | EfficientNet B0, quantized to INT8      |
| face-detection-retail-0004 (FP16)        | `/models/output/omz/face-detection-retail-0004/FP16/face-detection-retail-0004.xml`               | OpenVINO Model Zoo                      |
| face-reidentification-retail-0095 (FP16) | `/models/output/omz/face-reidentification-retail-0095/FP16/face-reidentification-retail-0095.xml` | OpenVINO Model Zoo                      |

To install models using ViPPET's built-in model manager:

```bash
make install-models-force
```

Then select the required models from the interactive prompt.

Alternatively, if the models are already present on the host under `shared/models/output/`,
they will be automatically mounted into the container at `/models/output/`.

### Step 4: Prepare Input Videos

Download the sample video from [google drive](https://drive.google.com/drive/folders/1yg6cfxAASvCGwzJeAufn3XTxbhCf2ihi?usp=drive_link).

Then, place your test video files in the ViPPET shared videos directory:

```bash
cp your-video.mp4 shared/videos/items-in-basket.mp4
cp your-video.mp4 shared/videos/face-reidentification.mp4
```

The `filesrc location=` paths in the pipeline YAML files reference `/videos/input/` inside the
container, which maps to `shared/videos/` on the host. Update the filenames in the YAML
if your video files have different names.

### Step 5: Rebuild and Restart ViPPET

If you installed from source:

```bash
make build run
```

If using pre-built images, rebuild only the vippet service to pick up the new pipeline files:

```bash
docker compose build vippet
docker compose up -d
```

### Step 6: Verify the Pipelines

1. Open ViPPET in your browser at `http://localhost` (or `http://<HOST-IP>`)
2. The new pipelines should appear in the pipeline selection dropdown:
   - **Loss Prevention - Retail Detection & Classification**
   - **Loss Prevention - Face Re-identification**
   - **Loss Prevention - Object Detection**
3. Select a pipeline and device variant (CPU/GPU/NPU/GPU\_NPU)
4. Click **Run** to execute the pipeline and observe performance metrics

## Customization

### Changing the Input Video

Edit the `filesrc location=` parameter in the YAML file to point to a different video:

```yaml
filesrc location=/videos/input/your-custom-video.mp4 !
```

### Adjusting Model Paths

If your models are stored in a different location, update the `model=` parameters
in each variant's `pipeline_description` to match your model paths.

### Tuning Inference Parameters

Key parameters that can be adjusted per variant:

| Parameter            | Description                    | Default                                                  |
| -------------------- | ------------------------------ | -------------------------------------------------------- |
| `batch-size`         | Inference batch size           | `1`                                                      |
| `inference-interval` | Run inference every N frames   | `3`                                                      |
| `threshold`          | Detection confidence threshold | `0.5`                                                    |
| `nireq`              | Number of inference requests   | `2` (CPU/GPU), `4` (NPU)                                 |
| `ie-config`          | OpenVINO throughput streams    | `CPU_THROUGHPUT_STREAMS=2` or `GPU_THROUGHPUT_STREAMS=2` |
