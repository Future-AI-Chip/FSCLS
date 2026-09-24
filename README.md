# FSCLS
FSCLS is a uniffed full-system cyclelevel simulation framework that enables heterogeneous SNN accelerators to be evaluated using common workloads, architecture-speciffc adaptation, and consistent measurement boundaries.
# SNN Accelerator Simulation and Comparison Framework

This repository contains a cycle-level simulation and validation platform for
SNN accelerator architectures. It combines a native SiBrain simulator with a
common workload pipeline for comparing three representative dataflows:

- **S-Compressed (SPC)**: weight-sparse broadcast execution;
- **SiBrain (SIB)**: event-driven spatiotemporal execution;
- **FireFly v2 (FIF)**: regular systolic streaming execution.

The project is intended for architecture analysis, cycle accounting,
microarchitectural validation, and controlled resource-sensitivity studies.
It is not an RTL implementation and should not be interpreted as a drop-in
replacement for the original hardware designs.


## Main workflow

The recommended order is:

```text
native simulator changes
        ↓
NativeArchitectureValidation
        ↓
ComparisonFramework smoke tests
        ↓
controlled sweeps and real-network replay
        ↓
normalized reports and figures
```

Do not use cross-architecture results as final evidence before the native
validation gate has been checked. The validation directory records whether a
result is a strict pass, an unsupported case, a diagnostic result, or a
reference-only comparison.

## Native SiBrain simulator

The native simulator is implemented in C++14 and models the SiBrain execution
path, including sparse detection, event dispatch, S2TP-PE execution, LIF/state
updates, on-chip buffers, and optional DRAMSim3-backed memory timing.

The simulator accepts CSV layer descriptions and Q8.8 input/weight files. It
also supports the common temporal input format used by
`ComparisonFramework`, together with configurable PE, buffer, SRAM, bandwidth,
frequency, and memory parameters.

### Build

Linux or WSL is recommended for the provided Makefile.

```bash
cd Sibrain_2
make CXX=g++ -j
```

To enable DRAMSim3 timing, build or provide the DRAMSim3 library and run:

```bash
make clean
make CXX=g++ USE_DRAMSIM3=1 -j
```

The default executable name is `sibrain_sim.exe`; it can be changed with
`BIN`, for example:

```bash
make BIN=sibrain_sim CXX=g++ -j
```

### Example invocation

```bash
./sibrain_sim \
  --csv src/convnet_sibrain.csv \
  --weights /path/to/weights_q88.txt \
  --input /path/to/input_q88.txt \
  --no-dramsim3
```

For a common temporal workload, add `--common-mode`. Hardware parameters can
be supplied through a JSON configuration or command-line options such as
`--pe`, `--buffer-size`, `--dram-width`, `--frequency`, and the buffer/SRAM
configuration options implemented in `src/main.cc`.

## ComparisonFramework

`ComparisonFramework/` provides a shared experiment contract for the three
simulators:

1. generate or import a common workload;
2. produce a reference result;
3. adapt the workload to each native simulator;
4. run the three simulators;
5. verify layer outputs and source hashes;
6. normalize cycle and timing fields;
7. analyze sweeps or real-network results.

The common workload uses the following conventions:

- input spikes: TCHW, unsigned 8-bit values;
- weights: OIHW, signed 8-bit values;
- accumulator and membrane state: 32-bit integers;
- LIF: leak, threshold comparison, and subtraction reset as defined by the
  common contract;
- pooling: binary OR in common mode;
- bias and batch normalization: disabled in the common integer replay path.

### Minimal smoke test

```bash
cd ComparisonFramework

python3 workload_generator/generate_workload.py \
  --case-id smoke_single_conv \
  --preset single_conv \
  --T 4 --C 4 --H 8 --W 8 --threshold 8

python3 runners/run_all_smoke.py \
  --case-id smoke_single_conv \
  --memory-mode dram \
  --paths configs/sim_paths.json

python3 results/normalize_results.py \
  --case-id smoke_single_conv \
  --paths configs/sim_paths.json \
  --out-dir results/normalized
```

The exact executable locations are configured in
`ComparisonFramework/configs/sim_paths.json`. A successful run should record
functional validation, valid cycle fields, source-hash agreement, and a valid
comparable on-chip cycle field for each supported simulator.

### Controlled experiments

After smoke and native validation pass, the framework can run experiments for:

- weight and input-spike sparsity;
- PE-array scaling;
- on-chip buffer capacity;
- on-chip bandwidth;
- timestep and shape sensitivity;
- real-network layer replay.

Use the corresponding runners in `ComparisonFramework/runners/` and keep
generated CSV files and reports under the results directories. Do not manually
edit normalized result files; regenerate them from the simulator outputs.

## NativeArchitectureValidation

`NativeArchitectureValidation/` is a prerequisite gate for architecture
claims. It checks:

- functional correctness and layer-dump consistency;
- sparse-weight and sparse-input behavior;
- timestep support and unsupported boundaries;
- shape and PE-tail behavior;
- memory and cycle-accounting consistency;
- FireFly clock-domain and parallelism assumptions;
- comparable on-chip timing boundaries.

Run the pilot validation first:

```bash
cd NativeArchitectureValidation

python3 runners/run_smoke_validation.py \
  --comparison-framework ../ComparisonFramework \
  --out-prefix native_smoke_pilot \
  --max-cases 6 \
  --scompressed-timeout 900
```

Then inspect the generated gate and reports under
`NativeArchitectureValidation/reports/`. A `CHECK` or `NATIVE_UNSUPPORTED`
result is not automatically a simulator failure, but it must be explained
before the corresponding result is used as final evidence.

## Real-network data

The framework supports validated Conv/LIF replay from imported network data,
including BrainCog and sparse LTH/u-Ticket workloads. The reported aggregate
is a cycle result over the admitted common convolution-layer subset; it should
not automatically be described as complete end-to-end network latency.

Training checkpoints, datasets, generated traces, DRAM traces, and large
result directories are intentionally treated as external or generated data.
They should be added to GitHub only when redistribution and storage are
appropriate.

## Known boundaries

- The native SiBrain common path currently supports the validated timestep
  range documented by the validation plans; extended timestep cases may be
  reported as unsupported.
- Native wall cycles and comparable on-chip cycles answer different questions
  and must not be mixed without stating the measurement boundary.
- DRAMSim3-enabled results require the configured memory backend and should
  not be relabeled from fixed-latency fallback results.
- Results from older generated directories are evidence for debugging unless
  they are rerun or matched to the current source snapshot.

## Documentation

- [`SIMULATION_PLATFORM_COMMAND_MANUAL.md`](SIMULATION_PLATFORM_COMMAND_MANUAL.md)
  — detailed commands and execution order;
- [`ComparisonFramework/README.md`](ComparisonFramework/README.md)
  — common workload, adapters, runners, and experiment pipeline;
- [`NativeArchitectureValidation/README.md`](NativeArchitectureValidation/README.md)
  — native validation rules and gate criteria;
- [`Sibrain_2/scripts/README_training.md`](Sibrain_2/scripts/README_training.md)
  — training and export-related helpers.

## License and third-party components

This repository includes or references third-party simulators, RTL projects,
datasets, and paper artifacts. Check the license file in each third-party
subdirectory before redistributing those components. The license for the
project-specific code should be added at the repository root before public
release.
