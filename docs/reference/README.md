# Reference Documents

Reference materials, research papers, and implementation plans for aSi TFT Leak Reduction project.

## 📂 Directory Structure

```
docs/reference/
├── latest/           # 📌 Latest production documents (v3.1R Final)
├── implementation/   # Implementation plans by version
├── analysis/         # Technical analysis reports
├── research/         # Academic papers and drafts
├── vendor/           # Vendor cooperation guides
└── archive/          # Legacy documents
```

---

## 📌 Latest Documents (v3.1R Final)

**Production Ready** - Start here for the most current implementation plan.

| Document | Description | Date |
|----------|-------------|------|
| [aSi_TFT_Leakage_Implementation_Plan_v3_1_Panel_Integrated.md](latest/aSi_TFT_Leakage_Implementation_Plan_v3_1_Panel_Integrated.md) | Main implementation plan (v3.1 Panel Integrated) | 2026-02-10 |
| [FINAL_Complete_Download_Set.md](latest/FINAL_Complete_Download_Set.md) | Download guide and summary | 2026-02-10 |
| [v3_1_Validation_Report_5rounds.md](latest/v3_1_Validation_Report_5rounds.md) | 5-round validation report | 2026-02-10 |

**Key Metrics**:
- **Reliability**: 97.3% (30+ academic papers, 10+ IC datasheets, 3 patents)
- **Status**: Production Ready
- **Target**: R1717AS01.3 (3072×3072 a-Si TFT X-ray FPD)

---

## 📁 Implementation Plans

Complete implementation plans organized by version.

### v3.1 (Latest - Panel Integrated)

| Document | Description |
|----------|-------------|
| [aSi_TFT_Leakage_Implementation_Plan_v3_1_Panel_Integrated.md](implementation/v3.1/aSi_TFT_Leakage_Implementation_Plan_v3_1_Panel_Integrated.md) | R1717AS01.3 panel specifications, voltage rails, FPGA/MCU sequences |

### v3.0

| Document | Description |
|----------|-------------|
| [aSi_TFT_Leakage_Implementation_Plan_v3_Final.md](implementation/v3.0/aSi_TFT_Leakage_Implementation_Plan_v3_Final.md) | 3-stage idle management strategy |

### v2.0 (Legacy)

| Document | Description |
|----------|-------------|
| [aSi_TFT_Leakage_v2_Final.md](implementation/v2.0/aSi_TFT_Leakage_v2_Final.md) | Initial leakage analysis |

---

## 📊 Analysis Reports

Technical analysis and characterization reports.

| Document | Description | Date |
|----------|-------------|------|
| [aSi_TFT_Idle_Strategy_Report_v3_Final.md](analysis/aSi_TFT_Idle_Strategy_Report_v3_Final.md) | Idle state strategy analysis (3-stage L1/L2/L3) | 2026-02-09 |
| [aSi_TFT_IdleState_CharacteristicReport.md](analysis/aSi_TFT_IdleState_CharacteristicReport.md) | Idle state characteristics (draft) | 2026-02-09 |
| [aSi_TFT_Drive_Algorithm_ExecutionPlan_v2_2.md](analysis/aSi_TFT_Drive_Algorithm_ExecutionPlan_v2_2.md) | Drive algorithm execution plan (v2.2) | 2026-02-09 |

---

## 🎓 Research Papers

Academic paper drafts (Korean) for publication.

| Document | Description |
|----------|-------------|
| [nblm_aSi_TFT_plan_v3.0.md](research/nblm_aSi_TFT_plan_v3.0.md) | Research plan v3.0 |
| [nblm_연구논문_초안.md](research/nblm_연구논문_초안.md) | Academic paper draft |
| [nblm_연구논문_초안_v1.md](research/nblm_연구논문_초안_v1.md) | Academic paper draft v1 |

---

## 🤝 Vendor Cooperation

Guidelines for working with panel vendors.

| Document | Description |
|----------|-------------|
| [Vendor_Cooperation_Protocol_Guide.md](vendor/Vendor_Cooperation_Protocol_Guide.md) | Vendor collaboration guidelines and measurement protocols |

---

## 🗄️ Archive

Legacy and duplicate documents archived for reference.

| Document | Description |
|----------|-------------|
| [00_FINAL_Download_Summary.md](archive/00_FINAL_Download_Summary.md) | Old download summary (superseded by FINAL_Complete_Download_Set.md) |

---

## 📖 Version History

| Version | Date | Description |
|---------|------|-------------|
| **v3.1R** | 2026-02-10 | Final with 5-round validation (97.3% reliability) |
| v3.1 | 2026-02-10 | Panel-integrated (R1717AS01.3) |
| v3.0 | 2026-02-09 | Idle management strategy complete |
| v2.0 | 2025-01 | Initial analysis |

---

## 🔗 Related Documents

- **Design Specifications**: `../design/`
- **Panel Physics**: `../panel/physics/`
- **SPI Protocol**: `../common/spi/`
- **FPGA Firmware**: `../../fpga/`
