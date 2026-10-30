# Engineering record: profiling a YOLOv12s browser-to-device video pipeline

> **Initial engineering record — technical details remain pending confirmation as marked below.** One implementation on one engineering unit. This is an engineering observation record, not a reproducible benchmark, a product specification, or a comparison between platforms.

## 1. Summary

In the implementation described here, the worker executes preprocessing, inference and postprocessing serially. Reported preprocessing times of **51.0–59.5 ms** are close to the worker inference-stage times of **59.5–62.7 ms**; postprocessing is **11.3–12.9 ms**. The preprocessing measurement is aggregated, so it does not establish which individual operation dominates.

One run completed **977 inferences in 130 seconds**, approximately **7.5 new inferences per second**. Input and output video are configured at **25 FPS**, but output frames may reuse the last detection result. The engineering notes report approximately **69% input frames dropped**. Output FPS is not inference throughput, and neither establishes capture-to-display latency.

The host architecture is **riscv64**. **YOLOv12s executes on a dedicated DFU accelerator, not on the RISC-V CPU.** This is not a RISC-V CPU inference test.

## 2. Development status, provenance and sources

SingNova is in development. This record concerns one engineering unit and one software implementation; it does not establish general availability, supported configurations, SDK access, or delivery commitments.

The measurements and implementation details were supplied by a colleague on the SingNova engineering team. Philip, affiliated with SingNova, compiled and edited the record; **the compiler is not the experimenter**. Text preparation used AI assistance. Engineering confirmation is required for revisions to technical claims.

Sources:

- Primary source: *Profiling a YOLOv12s video pipeline: preprocessing nearly matches inference time* — the ComputerVision manuscript and its [public discussion](https://www.reddit.com/r/computervision/comments/1xha6n7/profiling_a_yolov12s_video_pipeline_preprocessing/).
- Supplementary source: the RISC-V host/accelerator manuscript supplied for this review.
- Source review on **2026-10-30, Asia/Shanghai**: the visible public post was consistent with the primary manuscript. No comments were returned in the loaded comment DOM; no author corrections or engineering-confirmed technical answers were found in that view. This is a point-in-time observation, not a claim of exhaustive comment retrieval.

The primary manuscript governs differences in interpretation. In particular, the worker/runtime timing difference has **not been quantified or attributed with paired per-invocation traces**. No numerical overhead conclusion is carried over from the supplementary manuscript.

There is currently **no accompanying public code or complete reproduction package** for this record. No installation, execution, download or SDK instructions are provided. Discussion suggestions, if added later, must remain proposals until implementation and measurement are confirmed by engineering.

## 3. Test environment

| Item | Reported configuration or status |
|---|---|
| Unit | One engineering unit; exact hardware configuration and unit identifier pending confirmation |
| Host CPU architecture | riscv64 |
| Model execution device | Dedicated DFU accelerator, not the host CPU |
| OS | openEuler 24.03 LTS-SP2 |
| Kernel | Linux 6.6.118+ |
| Runtime | 1.0.3, model-specific plugin, explicit cache synchronization |
| Model | YOLOv12s |
| Model input | 1×3×640×640; BGR input for this setup |
| Model artifact, precision and quantization | Pending confirmation |
| Source video | 640×360 at 25 FPS |
| Decode | FFmpeg software decoding; hardware video decoding is excluded |
| Output | FFmpeg H.264 encode; output configured at 25 FPS |
| Source revision and dependency versions | Pending confirmation beyond the versions listed above |
| Measurement date | Pending confirmation; document review date is not the test date |

Hardware video decoding was described in the source as a separate development path. It was not used in these measurements, and no result or availability claim for that path is made here. The H.264 encoder implementation and whether encoding is hardware-assisted have not been confirmed.

## 4. Processing path

```text
Browser → WebRTC/WHIP → ZLMediaKit → FFmpeg software decode
        → preprocessing → DFU inference → postprocessing/overlay
        → FFmpeg H.264 encode → RTSP → WebRTC/WHEP → browser
```

Preprocessing includes colorspace conversion, scaling and black letterbox padding. Postprocessing includes tensor decoding, confidence filtering, non-maximum suppression (NMS) and box rendering.

The three worker stages currently run serially. When processing falls behind, the implementation drops older input frames. Output can reuse the last detection result between new inferences. This description does not establish where all media queues reside or which operations fall inside each timer.

## 5. Timing definitions

| Instrumentation level | Field or quantity | Meaning and boundary |
|---|---|---|
| Runtime | `kernel_ms` | Device execution time |
| Runtime | `total_ms` | Runtime inference call, including input/output copies and cache synchronization |
| Worker | `preprocess_ms` | Aggregated preprocessing stage; detailed operation and waiting boundaries remain unconfirmed |
| Worker | `infer_ms` | Worker inference-stage interval; not an alias for runtime `total_ms` or device `kernel_ms` |
| Worker | `post_ms` | Reported postprocessing-stage interval; exact instrumentation boundaries pending confirmation |
| Run-level throughput | Completed inferences / elapsed run time | New-inference rate for the reported run, not output video FPS |
| End-to-end | Capture-to-display latency | Not measured |

The runtime means, worker ranges and individual log sample below are **different statistical summaries and scopes**. They must not be subtracted as if they were paired observations. The worker timing is higher than the reported runtime-call mean, but the difference has not been quantified or attributed using paired per-invocation traces.

The preprocessing timer does not yet isolate colorspace conversion, scaling/padding, tensor preparation, copies, or waiting for frames or buffers. Its aggregate duration cannot be attributed to one operation or to the RISC-V instruction set.

## 6. Reported measurements

### Runtime means

| Runtime measurement | Reported mean |
|---|---:|
| Device execution: `kernel_ms` | 46.45 ms |
| Complete runtime call: `total_ms` | ~53 ms |

The averaging window, sample count, warm-up policy and relationship to the run-level counts below are pending confirmation.

### Worker stage ranges

| Pipeline measurement | Reported range |
|---|---:|
| `preprocess_ms` | 51.0–59.5 ms |
| `infer_ms` | 59.5–62.7 ms |
| `post_ms` | 11.3–12.9 ms |

These are source-reported ranges, not means, percentiles or confidence intervals. Their sampling window and range definition remain pending confirmation. Adding their endpoints does not establish measured end-to-end latency.

### Original individual runtime log sample

The following is **one invocation**, preserved from the supplied sources. Its `kernel_ms=50.570` is not the reported mean of 46.45 ms; its `total_ms=53.834` does not replace the reported mean of approximately 53 ms. Internal field names are preserved without expanding undocumented abbreviations.

```text
[DFU_PROFILE] infer_id=977 status=ok
input_copy_ms=1.677 input_cache_clean_ms=0.263
kernel_ms=50.570 result_pre_read_ci_ms=0.095
output_copy_ms=1.043 result_post_read_ci_ms=0.114
total_ms=53.834
input_bytes=2457600 result_bytes=1411200
```

### Throughput, frame handling and initialization

| Quantity | Reported result | Interpretation |
|---|---|---|
| Completed inferences | 977 in 130 seconds | About 7.5 new inferences/s; run timing boundaries pending confirmation |
| Source/input cap | 25 FPS | Not evidence that every input frame is inferred |
| Input frames dropped | Approximately 69% | Reported by engineering; counter definition and denominator pending confirmation |
| Configured output rate | 25 FPS | Can reuse the previous detection result; not 25 new inferences/s |
| First invocation after model switch | 1459.9 ms | Separately reported initialization-associated invocation; exact included work pending confirmation |
| Steady-state invocation reference | Approximately 60–61 ms | Source-reported comparison, not whole-pipeline latency |
| Capture-to-display latency | Not measured | No current frame-ID linkage from browser capture to returned result |

The drop figure is retained as reported, not recomputed from inference throughput. The initial invocation is not a universal cold-start result, whole-pipeline duration, or capture-to-display measurement.

## 7. Limitations

- One unit and one implementation do not establish general RISC-V performance, accelerator capability across models, production readiness, or competitor comparisons.
- Software decoding is part of this path; hardware video decode is not. No claim about a hardware-decoded version follows from these results.
- Missing hardware details, model precision, exact artifacts, revisions, test date and measurement methodology prevent a complete reproducibility claim.
- No accuracy evaluation or application-level detection-quality result is supplied. Timing alone cannot establish suitability for an application.
- Runtime means and worker ranges are unpaired; overhead attribution is unresolved.
- The preprocessing aggregate does not identify a dominant sub-operation.
- The run includes substantial input dropping and output-result reuse. Output smoothness cannot establish fresh-result rate or frame age.
- Browser capture, transport, queues, encoding and display have not been connected through end-to-end tracing. Processing time is not capture-to-display latency.
- No optimization discussed below has a confirmed speedup in this record.

## 8. Next questions to validate

1. **Split preprocessing traces:** identify colorspace conversion, scaling/padding, tensor preparation, copies, and frame/buffer waits; establish exact timer boundaries before attributing cost.
2. **Pair worker and runtime events:** link the same frame and invocation across `infer_ms`, `total_ms` and `kernel_ms`, using suitable clock domains, before quantifying differences.
3. **Establish safe scheduling experiments:** compare reducing preprocessing work with overlapping stages only after confirming buffer ownership, synchronization and safe reuse. Measure outcomes rather than projecting speedups from stage times.
4. **Trace frame identity:** preserve capture/result association through the latest-frame/drop-old path and investigate frame-age measurement without assuming synchronized browser and device clocks.
5. **Document throughput accounting:** confirm run start/end rules, initialization treatment, frame-drop counters, queue policies and output reuse behavior.
6. **Complete the environment record:** confirm hardware, artifacts, precision, dependency versions, source revision and test date before planning a reproducible release.

These are validation tasks, not implemented optimizations, approved delivery plans or performance promises. No additional model downloads, paid compute or deployments are implied.

## 9. Revision history

| Record revision | Date | Change | Evidence status |
|---|---|---|---|
| Initial record 0.1 | 2026-10-30 | Consolidated the two supplied manuscripts; checked the visible primary discussion; separated means, ranges and sample; retained dropping and latency limitations | Compiled from engineering-supplied notes; outstanding technical confirmations are marked above |

Future revisions should record the author of each confirmed correction, the supporting artifact or trace, the affected configuration, and whether the change is a measurement, implementation change, or interpretation correction. Editorial dates must not substitute for test dates.
