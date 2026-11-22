# Aerial Object Detection and Inference Benchmarking

An aerial object-detection study comparing inference speed, accuracy, model size and detection-message payloads on an Apple M4 Pro CPU.

**ONNX FP32 at 960 × 960 achieved 0.213 validation mAP50–95 at 22.2 ms mean prediction latency, compared with 0.169 at 22.9 ms for PyTorch FP32 at 640 × 640.** The larger input improved accuracy while keeping the observed mean latency close to the smaller PyTorch configuration on this machine.

[Results](docs/measurements/benchmarks.csv) · [Methodology](docs/METHODOLOGY.md) · [Reproduce](docs/REPRODUCE.md)

![Stock COCO model on the left and VisDrone fine-tuned model on the right, on the same saved drone frame.](docs/assets/before-after.png)

Stock COCO weights (left) and the fine-tuned VisDrone checkpoint (right) on the same [source frame](docs/assets/source-frame.jpg). This is a qualitative example; the models use different class vocabularies. Footage: Jordan Koschei / ConnectHV, CC0. See [source details](data/SOURCES.md).

## Experiment

The study compares PyTorch FP32, ONNX FP32 and calibrated ONNX INT8 at two input sizes. All six configurations use the same YOLO26 nano checkpoint, fine-tuned on VisDrone for 50 epochs at image size 640, batch 16 and seed 0. Training used MPS; validation and timing used the Apple M4 Pro CPU. The 960 configurations change inference size only.

The experiment also compares the payload needed to send detection messages with two video baselines. The laptop provides a controlled place to compare options before testing an intended deployment device.

## Results

Accuracy uses all 548 VisDrone validation images. Timing uses 100 predictions of one saved validation image after 10 warm-up calls, with 8 model intra-op threads. It includes preprocessing, inference, postprocessing and the common configuration/shape checks; image-file loading, warm-up and console logging are excluded.

| Format | Actual input H × W | mAP50–95 | mAP50 | Pedestrian AP50–95 | Mean ms (Apple M4 Pro CPU) | p95 ms (Apple M4 Pro CPU) | File MB |
|---|---:|---:|---:|---:|---:|---:|---:|
| PyTorch FP32 | 640 × 640 | 0.169 | 0.308 | 0.137 | 22.9 | 24.6 | 5.444 |
| ONNX FP32 | 640 × 640 | 0.169 | 0.308 | 0.137 | 10.9 | 11.6 | 9.861 |
| ONNX INT8 | 640 × 640 | 0.153 | 0.287 | 0.119 | 12.2 | 13.2 | 3.045 |
| PyTorch FP32 | 960 × 960 | 0.213 | 0.375 | 0.199 | 38.5 | 39.1 | 5.444 |
| ONNX FP32 | 960 × 960 | 0.213 | 0.375 | 0.199 | 22.2 | 25.4 | 10.071 |
| ONNX INT8 | 960 × 960 | 0.194 | 0.348 | 0.171 | 23.0 | 23.9 | 3.255 |

AP values are fractions; higher is better. Latency is the full prediction call; lower is better. File sizes use decimal MB. Runtime comparisons use matching input shapes within each size group, with guards that stop a run if the shapes differ.

## Speed versus accuracy

![Mean full-call latency on the Apple M4 Pro CPU versus VisDrone validation mAP50–95, with every format and input size labelled.](docs/assets/speed-vs-accuracy.png)

At 640, ONNX FP32 measured 10.9 ms against PyTorch's 22.9 ms, with matching validation accuracy to the reported precision. Increasing the ONNX input to 960 raised mean latency to 22.2 ms and mAP50–95 from 0.169 to 0.213. Pedestrian AP50–95 rose from 0.137 to 0.199. The mean at 960 was close to PyTorch's mean at 640 on this CPU. The comparison does not establish statistical equivalence or equal tail latency.

## File size versus accuracy

![Serialized model size in decimal MB versus VisDrone validation mAP50–95, with every format and input size labelled.](docs/assets/file-size-vs-accuracy.png)

I expected INT8 to run faster. At 640, it reduced the ONNX file from 9.861 MB to 3.045 MB (69.1%), but mean latency rose from 10.9 to 12.2 ms and mAP50–95 fell from 0.169 to 0.153. ONNX FP32 was also faster at 960 on this CPU.

The size comparison uses the two ONNX formats. The PyTorch checkpoint has a different storage representation; see the [precision and file-size notes](docs/METHODOLOGY.md#precision-and-file-size).

## Detection-message payloads

The fine-tuned PyTorch model processed 353 frames from a 640 × 1138 preview clip. Payload rates use the clip's recorded 24 frames/s. The comparison includes the original compressed MP4 and independently encoded JPEG frames at OpenCV's default quality of 95.

| Payload | Average bytes/frame | Average kB/s | Baseline ÷ detection payload |
|---|---:|---:|---:|
| Detection messages | 166.5 | 4.0 | — |
| Average MP4 file bytes per second | — | 63.3 | 15.8× |
| JPEG-frame payload bandwidth | 219896.0 | 5,277.5 | 1320.8× |

Detection messages contain classes, confidence scores and pixel-coordinate boxes. These rates describe payload size for one clip, exclude transport overhead, and do not measure network traffic or processing throughput. There were 78 empty messages; smaller payloads can reflect missed objects. The demo clip's detection accuracy was not measured.

See the [payload methodology](docs/METHODOLOGY.md#payload-calculations) and [saved measurements](docs/measurements/summary.json) for the calculations and underlying values.

## Scope and next steps

The first follow-up is to repeat the timings on varied images. The current timings use one frame, and crowded scenes could take a different amount of time to process. I would also evaluate with the official VisDrone toolkit to make the accuracy figures easier to compare across studies. The current scores use Ultralytics on converted VisDrone labels.

I would then train at 960, try a model larger than nano to see whether the gain in accuracy is worth the added latency, and time inference without external non-maximum suppression.

Deployment measurements would need the intended hardware and footage from its camera. Accuracy on VisDrone may differ from what the same detector achieves in that environment. On a supported NVIDIA device, I would compare runtimes including TensorRT and their quantization options, then measure sustained latency, heat, power, memory and end-to-end camera-to-link behavior. Those measurements are still open.

## Licensing

This repository's code is licensed under the GNU Affero General Public License v3.0 (AGPL-3.0); see [LICENSE](LICENSE).

Ultralytics is also distributed under [AGPL-3.0](https://github.com/ultralytics/ultralytics/blob/main/LICENSE) and offers a separate [Enterprise License](https://www.ultralytics.com/license). The repository's code license does not change the separate dataset and footage terms below.

VisDrone is a research dataset. As recorded in [data/SOURCES.md](data/SOURCES.md), the linked upstream README did not include a standalone license grant. Dataset images, annotations and model weights are not included in this repository. Review the upstream terms before reuse or redistribution.

The separate ConnectHV demo footage is CC0, as documented in [data/SOURCES.md](data/SOURCES.md). That statement does not apply to VisDrone.
