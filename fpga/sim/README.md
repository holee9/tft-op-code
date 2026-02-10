# FPGA Simulation Environment

Testbenches and simulation scripts.

## Directory Structure

| Directory | Description |
|-----------|-------------|
| `tb/` | Testbench files |
| `scripts/` | Simulation scripts and Makefiles |

## Running Simulation

```bash
# Using ModelSim/Questa
cd scripts
./run_simulation.sh

# Or manually
vsim -c -do "run -all" tb_top
```

## Coverage Goals

- Code Coverage: 95%+
- Functional Coverage: 100% of features
- Assertion Coverage: All SVA assertions

## Status

*Testbench development in progress*
