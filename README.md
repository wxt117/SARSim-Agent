<div align="center">

# [SARSim-Agent](https://github.com/wxt117/SARSim-Agent)

### A Task-Driven and Quality-Assured Agent Framework for SAR Data Simulation and Generation

[![CI](https://github.com/wxt117/SARSim-Agent/actions/workflows/ci.yml/badge.svg)](https://github.com/wxt117/SARSim-Agent/actions/workflows/ci.yml)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](pyproject.toml)
[![License: MIT](https://img.shields.io/badge/License-MIT-2EA44F.svg)](LICENSE)
[![Research code](https://img.shields.io/badge/status-research%20release-F59E0B.svg)](#release-scope)

**Xuanting Wu, Fan Zhang, Fei Ma, Ling Guan, Guochun Ma, Deliang Xiang, Yongsheng Zhou**

College of Information Science and Technology, Beijing University of Chemical Technology<br>
National Key Laboratory of Scattering and Radiation

[Overview](#overview) | [Installation](#installation) | [Quickstart](#quickstart) | [Results](#paper-results) | [Citation](#citation)

</div>

SARSim-Agent is a task-driven framework for synthetic aperture radar (SAR) data simulation, generation, and augmentation. Given a natural-language request and SAR inputs, it reads the dataset, checks its metadata, selects suitable tools, records the execution steps, and evaluates the generated samples. It connects existing SAR tools into a repeatable workflow under the available computing budget.

<p align="center">
  <img src="docs/assets/saga-b747-workflow.jpg" width="100%" alt="SARSim-Agent workflow for full-azimuth B747 SAR generation">
</p>

<p align="center"><sub><b>End-to-end SARSim-Agent workflow.</b> A full-azimuth B747 example covering dataset interpretation, tool selection, generation, output checks, and limited repair (paper Fig. 2).</sub></p>

## Overview

SAR augmentation decisions depend on more than visual realism. Dataset layouts are often laboratory-specific; metadata such as class, azimuth, depression angle, resolution, band, and polarization may be encoded in paths, filenames, sidecars, or annotations. Generation methods also differ in applicability, computational cost, controllability, and failure modes.

SARSim-Agent connects five steps:

1. **Read and check the data.** Scan images, labels, and metadata stored in different formats. Build and validate a dataset schema: rules for reading fields such as category, polarization, and viewing angle from files and folders.
2. **Select suitable tools.** Match the task and input data to simulation or generation methods, taking account of their requirements, computing cost, and results from previous runs.
3. **Record the workflow.** Store each step's inputs, parameters, outputs, and dependencies in an executable recipe. A directed acyclic graph (DAG) specifies which steps depend on earlier outputs.
4. **Check the outputs.** Evaluate image quality, data distributions, SAR artifacts, duplicates, and data leakage. Run downstream task evaluation when requested.
5. **Repair and reuse results.** Make a limited number of corrections when checks fail, and save reports that can guide later planning.

LLM assistance is optional. An LLM can help interpret requests and propose dataset schemas or plans. Explicit rules check these proposals before execution.

<p align="center">
  <img src="docs/assets/saga-architecture.png" width="100%" alt="Overall architecture of SARSim-Agent">
</p>

<p align="center"><sub><b>Overall architecture.</b> Solid paths are deterministic execution and validated data flow; dashed paths are optional LLM-assisted proposals that must pass validation (paper Fig. 4).</sub></p>

## Key Features

| Capability | What SARSim-Agent provides |
| --- | --- |
| Dataset reading | Inspection and validation of image files, labels, metadata, and dataset layouts |
| Tool selection | Task interpretation, input compatibility checks, and planning within resource limits |
| Workflow execution | Recorded steps and dependencies, checkpoints, input/output tracking, and standardized reports |
| SAR data checks | Quality, distribution, artifact, duplicate, leakage, and metadata-slice evaluation |
| Task evaluation | Reports that distinguish generated files from measured downstream improvement |
| Feedback | Limited repair, run continuation, and saved results for later planning |

## Installation

Python 3.10 or newer is required. The included Conda file creates an environment named `sd3`:

```bash
git clone https://github.com/wxt117/SARSim-Agent.git
cd SARSim-Agent
conda env create -f environment.yml
conda activate sd3
python -m saga --help
```

The project is named **SARSim-Agent**; the Python package and CLI remain **`saga`** in this release. Use `python -m saga` in the commands below.

If `sd3` already exists, update it instead:

```bash
conda env update -n sd3 -f environment.yml
conda activate sd3
```

Install development and plotting dependencies when running tests or paper experiment scripts:

```bash
python -m pip install -e ".[dev,plots]"
```

## Quickstart

The public demo is CPU-only and self-contained. It creates a small synthetic SAR-like fixture, profiles the format, applies deterministic SAR-aware augmentation, and evaluates the generated samples. No private dataset, model checkpoint, GPU, or API key is required.

```bash
# 1. Create 12 SAR-like chips with filename and sidecar metadata.
python examples/make_demo_dataset.py

# 2. Profile the dataset and ground a validated schema.
python -m saga profile-dataset \
  --root examples/demo_dataset \
  --request "augment the vehicle SAR chips while preserving polarization metadata" \
  --output outputs/demo/profile

python -m saga bridge-format \
  --profile outputs/demo/profile \
  --output outputs/demo/profile

# 3. Materialize 12 deterministic augmentations.
python -m saga traditional-augment \
  --dataset examples/demo_dataset \
  --output outputs/demo/augmentation \
  --target-count 12 \
  --run

# 4. Run the quality observer.
python -m saga evaluate-output \
  --input outputs/demo/augmentation/generated_images \
  --output outputs/demo/evaluation \
  --expected-count 12
```

To inspect an end-to-end plan from one request, run:

```bash
python -m saga agent-run \
  --request "use traditional SAR augmentation on examples/demo_dataset and generate 12 samples" \
  --dataset-root examples/demo_dataset \
  --output outputs/demo/agent \
  --no-memory
```

`agent-run` and computationally expensive skills default to dry-run, which writes the plan without running the generation backend. Add `--run` to execute the selected workflow after checking its parameters and backend paths.

### Optional LLM assistance

To enable LLM-assisted request interpretation or planning, supply the API key through an environment variable:

```bash
export DEEPSEEK_API_KEY="your-api-key"

python -m saga agent-run \
  --request "profile my SAR dataset and propose a feasible augmentation recipe" \
  --dataset-root /path/to/dataset \
  --output outputs/llm-plan \
  --llm-config configs/llm/deepseek_v4pro.yaml \
  --use-llm-intent \
  --use-llm-planner
```

Never commit credentials to a configuration file.

## Skills and Backends

The release includes a deterministic framework and local skills, plus adapters for specialized research backends.

| Category | Included capabilities | Availability |
| --- | --- | --- |
| Data understanding | Profiling, format inspection, schema bridging, validation, metadata captions | Runs from this repository |
| Local augmentation | Traditional SAR augmentation, preprocessing, pseudocolor, target-background composition | Runs from this repository |
| Evaluation | Quality, distribution, SAR artifacts, duplicates, leakage, metadata slices | Runs from this repository |
| Agent runtime | Intent routing, planning, recipe DAGs, execution, repair, memory, reporting | Runs from this repository |
| Learned generation | GAN, diffusion LoRA, style transfer, GeoDiff-SAR, background generation | Adapter; external code/weights required |
| Physics and 3D | RaySAR, view sweeps, model-to-POV conversion, Gaussian splatting | Adapter; external tools/assets required |
| Downstream evaluation | Classification workflow and standardized evidence reports | Adapter; evaluator project/data required |

Backend configuration examples live in [`configs/skills/`](configs/skills/). External projects, checkpoints, datasets, and CUDA environments are not vendored and remain subject to their own licenses. See the [skill catalog](docs/skill_catalog.md) for inputs, limitations, and command examples.

## Paper Results

The manuscript evaluates dataset interpretation, request interpretation and tool selection, workflow execution, failure detection and repair, downstream task improvement, and the effect of individual components.

Across the reported benchmark of **8 SAR task groups x 12 seeds**, Full SARSim-Agent achieved:

| Metric | Full SARSim-Agent | Interpretation |
| --- | ---: | --- |
| Aggregate downstream score | **60.1 +/- 4.5** | Equal-weight macro-average of task-appropriate Accuracy, macro-F1, or mAP |
| Matched gain over no augmentation | **+7.4 +/- 1.2 pp** | Improvement over the matched no-augmentation baseline |
| Invalid-sample rate | **1.0%** | Generated samples rejected as invalid |
| Evidence-qualified improvement rate (Lv5) | **94.8%** | Runs supported by downstream evaluation evidence |

These values combine task-appropriate metrics and should not be interpreted as one homogeneous accuracy measure. Fixed methods perform strongly only on favorable task groups; SARSim-Agent instead selects strategies according to task, data, cost, and observed evidence.

<p align="center">
  <img src="docs/assets/saga-results.png" width="100%" alt="Downstream augmentation results across eight SAR task groups">
</p>

<p align="center"><sub><b>Downstream augmentation benefit.</b> Overall score, task-level gains, evidence levels, benefit-risk behavior, and gain distributions across tasks and seeds (paper Fig. 14).</sub></p>

### Qualitative Evidence

SARSim-Agent reports not only generated samples but also the evidence used to evaluate them. The examples below visualize residual artifacts, azimuth coverage, content retention, and target-mask consistency across several skill families.

<p align="center">
  <img src="docs/assets/saga-qualitative-cases.png" width="620" alt="Qualitative SARSim-Agent outputs with observer evidence">
</p>

<p align="center"><sub><b>Representative decision chains.</b> Input or prior, SARSim-Agent result, and corresponding observer evidence (paper Fig. 15). These examples provide traceability rather than blinded perceptual evaluation.</sub></p>

## Release Scope

This repository provides the SARSim-Agent framework, a small public demo, and adapters for external SAR tools.

**Reproducible from the repository:** dataset profiling, schema bridging and validation, recipe serialization, traditional augmentation, target-background composition, observers, report validation, and controlled benchmark logic.

**Requires external or private assets:** the private datasets used in the manuscript, model weights, CUDA-specific environments, and the source projects behind LoRA, GeoDiff-SAR, GAN, style transfer, Gaussian splatting, RaySAR, and downstream evaluator adapters.

The repository therefore does not claim artifact-level reproduction of private-data or checkpoint-dependent results. The exact boundary is documented in [Reproducibility](docs/reproducibility.md).

## Repository Structure

```text
saga/          core framework, agent runtime, skills, observers, and CLI
configs/       dataset schemas, policies, LLM settings, and backend adapters
examples/      self-contained public demo dataset generator
experiments/   scripts corresponding to the manuscript experiments
tests/         unit and integration tests with generated fixtures
docs/          architecture, protocol, skill, runtime, and reproducibility notes
results/       release-level result notes
```

Useful technical references:

- [Architecture](docs/architecture.md) - component boundaries and data flow
- [Agent runtime](docs/agent_runtime.md) - planning, recipes, execution, and repair
- [Protocol](docs/protocol.md) - normalized objects and skill contracts
- [Data format strategy](docs/data_format_strategy.md) - heterogeneous dataset handling
- [Reproducibility](docs/reproducibility.md) - public and private artifact boundary
- [Contributing](CONTRIBUTING.md) - development workflow and pull requests

## Development

```bash
conda activate sd3
python -m pytest
python scripts/check_publication.py
```

The publication check rejects credential-like strings, absolute machine-local paths, and tracked files larger than 100 MB.

## Citation

Please use the following provisional citation for the associated manuscript. Journal metadata will be added when available:

```bibtex
@misc{wu2026sarsimagent,
  title  = {SARSim-Agent: A Task-Driven and Quality-Assured Agent Framework for SAR Data Simulation and Generation},
  author = {Wu, Xuanting and Zhang, Fan and Ma, Fei and Guan, Ling and Ma, Guochun and Xiang, Deliang and Zhou, Yongsheng},
  year   = {2026},
  note   = {Research manuscript},
  url    = {https://github.com/wxt117/SARSim-Agent}
}
```

Machine-readable metadata are provided in [`CITATION.cff`](CITATION.cff).

## License

SARSim-Agent's original code is released under the [MIT License](LICENSE). External datasets, model weights, and third-party backends retain their respective licenses and terms.

