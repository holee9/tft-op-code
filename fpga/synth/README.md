# FPGA Synthesis

Synthesis scripts, constraints, and reports.

## Contents

- Vivado project files
- Timing constraints (SDC)
- Synthesis reports
- Resource utilization

## Synthesis Guidelines

1. Clean previous build: `./clean.sh`
2. Run synthesis: `./synthesize.sh`
3. Check timing: `./check_timing.sh`
4. Review reports: `reports/`

## Timing Targets

| Clock | Frequency | Margin |
|-------|-----------|--------|
| system | 50 MHz | +20% |
| spi | 10 MHz | +20% |

## Status

*Synthesis setup in progress*
