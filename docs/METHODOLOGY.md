# Measurement methodology

[Project overview](../README.md) · [Reproduction commands](REPRODUCE.md)

## Model and validation

I chose YOLO26 nano because both the training laptop and a small drone have limited compute and power. VisDrone offered training images closer to the intended aerial view than ordinary ground-level photographs. A 10-epoch run checked that the training pipeline worked and reached 0.253 validation mAP50. A separate 50-epoch run reached 0.308. I stopped at 50 because it fit the time available. Whether training had converged or more epochs would help remains open.

All six configurations use the checkpoint from the 50-epoch run, fine-tuned at image size 640, batch 16 and seed 0. Training used MPS. The available Apple M4 Pro CPU provided a common platform for validation and timing. A drone computer with a different hardware and runtime combination could rank the configurations differently.

Validation covers all 548 images in the VisDrone validation split. Metrics come from Ultralytics on converted VisDrone labels, rather than the official challenge evaluation toolkit. They do not measure accuracy on the separate demo clip.

The 960 configurations change inference size only. Training at 960 would be a separate experiment. A fixed training seed does not guarantee identical MPS training results.

## Input shape and runtime controls

Within each input-size group, runtime comparisons use the same checkpoint and matching actual input shapes. Shape guards stop a run if the tensor dimensions differ from the required square input. The shared configuration and shape checks are included in the timed prediction call.

The saved detection head uses external non-maximum suppression (NMS). All exports and measurements kept that mode fixed so a change in postprocessing would not confound the runtime comparison. NMS-free inference could change the timings and remains untested.

The runs use 8 model intra-op threads. OpenCV's GCD backend did not honor the requested positive thread limit. Its reported setting was consistent across the matrix; the model thread count is not a total-process thread limit.

## Timing

Each row uses 10 warm-up calls followed by 100 predictions of one saved validation image on a quiet laptop. I [started the timer](../scripts/timing.py#L9-L28) after loading the image because a camera supplies frames in memory. Resizing stays inside the timed call because each frame still needs it. The matrix measures the full prediction call, including preprocessing, inference, postprocessing and the common configuration and shape checks.

Console logging was disabled for the timed runs. Turning it off left mean latency unchanged to one decimal place in an earlier check. The small change in p95 could have been run-to-run noise.

This setup compares repeated prediction calls on one image. It does not establish representative latency across the validation set, sustained performance under thermal load, or complete video-pipeline throughput. The timing loop's reciprocal-latency FPS value is not a measurement of a running video pipeline.

The p95 follows the timing loop's sorted-sample indexing convention: the zero-based sample at `int(0.95 * N)`. It describes the recorded samples and is not a worst-case guarantee.

The 22.2 ms mean for ONNX FP32 at 960 and 22.9 ms for PyTorch FP32 at 640 are observed results on the Apple M4 Pro CPU. They do not establish statistical equivalence, equal tail latency, or performance on another processor.

### Historical comparison

An earlier PyTorch run averaged 16.7 ms with a 384 × 640 input. The 640 × 640 PyTorch run in the final matrix averaged 22.9 ms. The checkpoint and measurement wrapper also changed, so the difference cannot be attributed to image shape alone. The final matrix compares formats at matching input shapes.

## Precision and file size

File sizes use decimal MB. Quantization's size tradeoff compares ONNX INT8 with ONNX FP32 at the same input size.

The PyTorch checkpoint is stored differently from an ONNX graph. Its FP32 label describes execution precision, so its file size is not an equal-format compression baseline. INT8 uses quantized weighted operations with float input/output and some remaining float operations.

At input size 640, INT8 reduced the ONNX file size by 69.1%, while mAP50–95 fell from 0.169 to 0.153. INT8 did not reduce mean latency on the Apple M4 Pro CPU in either input-size group.

## Payload calculations

The same fine-tuned PyTorch model processed 353 frames from a 640 × 1138 preview clip on the Apple M4 Pro CPU. I [calculated the JPEG and detection-message rates](../scripts/bandwidth.py#L52-L56) by averaging the encoded bytes per frame for each format and multiplying by the clip's recorded 24 frames/s, which gives the payload rate at playback speed. Rates use decimal kB/s.

- **Detection messages:** the script's JSON list of classes, confidence scores and pixel-coordinate boxes, including an empty list when nothing is detected.
- **MP4 baseline:** the whole clip file size divided by decoded-frame duration, including container contents. Duration is the decoded frame count divided by recorded FPS.
- **JPEG baseline:** each frame encoded independently using OpenCV's default quality of 95.

Encoding every frame independently as a JPEG takes more bytes for this clip than sending the compressed MP4, so the choice of video baseline changes the ratio substantially. Both estimates exclude transport overhead. Detection messages retain selected information from each frame, and actual traffic over a radio link remains unmeasured.

There were 78 empty detection messages among 353 frames. The largest message was 553 bytes. The clip was not scored for accuracy, so an empty message alone cannot tell us whether a frame had no relevant objects or the detector missed them.

The [saved summary](measurements/summary.json) contains the underlying values. The MP4 comparison was calculated from the original measurement without rerunning inference.

## Qualitative comparison

The before/after image compares stock COCO weights with the final VisDrone checkpoint on the same saved [source frame](assets/source-frame.jpg). The fine-tuned model returned more boxes on that frame. COCO and VisDrone use different class vocabularies, so the figure illustrates the change in detections. Accuracy is measured separately on the validation split.

Footage is credited to Jordan Koschei / ConnectHV under CC0. Provenance is in [data/SOURCES.md](../data/SOURCES.md).

## Deployment scope and reproduction

No target drone computer or radio link was tested. Heat, power, memory under sustained inference, deployment latency and actual camera/link behavior remain unmeasured.

The [reproduction guide](REPRODUCE.md) provides commands and environment requirements. The repository contains selected measurement summaries and images. Model weights, datasets and full original run logs remain local and ignored; the guide identifies those original records. Library versions, configuration and artifact identifiers are recorded in [summary.json](measurements/summary.json).
