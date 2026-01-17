# Per month cost estimate for Digital twin feature extractor pipeline

This document provides a detailed cost analysis for pipeline steps where AWS HealthOmics is the optimal choice, incorporating estimates from the engineering team and validating the use of GPU-accelerated instances (`omics.g` family).

---

## Executive Summary

AWS HealthOmics is a strong fit for **almost the entire pipeline (Steps 1-8)** due to recent support for **GPU instances (omics.g4/g5/g6)**. This allows us to unify CPU-heavy steps (WGS, RNA-seq) and GPU-heavy steps (scRNA-seq, Spatial) under one managed service.

**Key Benefits**:
- **Managed Orchestration**: Runs Nextflow/WDL without managing head nodes.
- **GPU Acceleration**: Native support for **NVIDIA L4/A10G** (`omics.g6`) enables Parabricks, RAPIDS, and Deep Learning tools.
- **Unified Data Plane**: No cross-service transfer fees (avoiding S3 \<\> EC2 costs).

**Estimated Monthly Cost** (100 samples/month): **~$7,410** (Infrastructure + Labor + Licenses)

---



---

## Primary Differences w.r.t Susmitha's Estimate

1.  **GPU Optimization (Huge Savings)**:
    *   **Susmitha**: Used CPU for *Step 6 (scRNA-seq)* → **~$9.12 / sample**.
    *   **Proposed Plan**: Switched to GPU (`omics.g6`) + RAPIDS → **~$0.90 / sample** (10x cheaper).

2.  **Ready2Run vs Custom (Predictability)**:
    *   **Susmitha**: Estimated custom CPU workflows for *Step 2 (WGS)* with a wide variance ($6 - $50).
    *   **Proposed Plan**: Standardized on **Ready2Run** for a guaranteed **$10.00** fixed cost.

---

## Detailed Pipeline Step Analysis (Revised)

### ✅ Step 1: Cohort Ingestion
- **Instance**: `omics.m.xlarge` (4 vCPU, 16 GB)
- **Runtime**: ~3 hours (Batch processing)
- **Cost**: **$0.79/sample**
- **Note**: Covers S3 download validation and metadata extraction.

### ✅ Step 2: WGS – Variant Calling
- **Recommendation**: **Ready2Run GATK-BP Germline**
- **Why**: Guaranteed **$10.00/sample** regardless of runtime.
- **Private Workflow**: Running Parabricks on `omics.g6` is possible but likely costs **$15-$20** (due to high GPU hourly rate + storage). Ready2Run is the simpler, cheaper "easy button".

### ✅ Step 3: Bulk RNA-seq
- **Instance**: `omics.m.4xlarge` (16 vCPU, 64 GB)
- **Runtime**: ~2 hours
- **Cost**: **$2.08/sample**
- **Tooling**: STAR + Salmon (CPU efficient).

### ✅ Step 4: Neoantigen Prediction (MHCflurry)
- **Instance**: `omics.g4dn.xlarge` or `omics.g6.xlarge` (GPU)
- **Why**: MHCflurry 2.0 requires GPU for efficient inference.
- **Cost**: ~$0.30/hr * 1 hr = **$0.30/sample**

### ✅ Step 5: TCR/BCR Sequencing
- **Instance**: `omics.m.2xlarge` (8 vCPU, 32 GB)
- **Runtime**: ~1 hour
- **Cost**: **$0.52/sample**

### ✅ Step 6: scRNA-seq (RAPIDS)
- **Instance**: `omics.g6.4xlarge` (16 vCPU, 64 GB, 1x L4 GPU)
- **Why**: Switch from colleague's CPU (`m.16xlarge`: $4.14/hr) to GPU (`g6.4xlarge`: ~$1.79/hr).
- **Benefit**: RAPIDS on GPU is 10-50x faster than CPU.
- **Estimated Cost**: 0.5h * $1.79 = **~$0.90/sample** (Significantly cheaper than colleague's $9.12 estimate).

### ✅ Step 7: Spatial Transcriptomics
- **Instance**: `omics.g6.16xlarge`
- **Pricing**: $4.5857/hr (Confirmed)
- **Runtime**: ~40 min
- **Cost**: **$3.06/sample**

### ✅ Step 8: Ligandome (Mass Spec)
- **Instance**: `omics.g6.8xlarge`
- **Pricing**: $2.7194/hr (Confirmed)
- **Runtime**: ~1-2 hours
- **Cost**: **~$3.40/sample**

---

## Consolidated Cost Estimates

AWS HealthOmics is the superior choice for the entire pipeline. All steps can now run within a single managed service and single VPC plane.

| Step | Recommended Service | Instance | Cost/Sample | Calculation Formula |
|:---|:---|:---|:---:|:---|
| **1. Ingestion** | HealthOmics Private | omics.m.xlarge | $0.79 | `($0.2592/hr * 3h) + ($0.023/GB * 0.5GB)` |
| **2. WGS** | **HealthOmics Ready2Run** | GATK-BP | **$10.00** | `Fixed Cost per Run` |
| **3. RNA-seq** | HealthOmics Private | omics.m.4xlarge | $2.08 | `($1.0368/hr * 2h) + Storage Overhead` |
| **4. Neoantigen** | HealthOmics Private | omics.g6.xlarge | $0.30 | `($1.0865/hr * 0.25h) + ($0.0004/GB * 50GB)` |
| **5. TCR/BCR** | HealthOmics Private | omics.m.2xlarge | $0.52 | `($0.5184/hr * 1h)` |
| **6. scRNA-seq** | HealthOmics Private | **omics.g6.4xlarge** | **$0.90** | `($1.7863/hr * 0.5h)` |
| **7. Spatial** | HealthOmics Private | omics.g6.16xlarge | $3.06 | `($4.5857/hr * 0.66h)` |
| **8. Ligandome** | HealthOmics Private | omics.g6.8xlarge | $3.40 | `($2.7194/hr * 1.25h)` |
| **Total** | | | **~$21.05 / sample** | |

> [!IMPORTANT]
> **Optimization Opportunity**: By switching **Step 6 (scRNA-seq)** from CPU (`m.16xlarge`) to GPU (`g6.4xlarge`) and using NVIDIA RAPIDS, we can reduce the cost from ~$9.12 to ~$0.90 per sample, a **10x reduction**.

---

## Storage & Data Transfer (Unchanged)

### Sequence Store
- **Active**: $0.005769/Gb/month
- **Archive**: $0.001154/Gb/month
- **Formula**: `(Total Gigabases * Price) * Days`

### Data Transfer
- **Internal**: $0 (HealthOmics to S3 in same region).
- **Recommendation**: Keep all data in `us-east-1`.

### Scenarios
1.  **HealthOmics → S3 (Same Region)**:
    - **Cost**: **$0.00**
    - **Use Case**: Moving BAMs/VCFs to a Data Lake for long-term storage or further AWS processing.
    - **Formula**: `Internal Transfer = $0`

2.  **Export Features to Internet/On-Prem (Data Transfer OUT)**:
    - **Rate**: ~$0.09 per GB (first 10 TB).
    - **Typical "Features" Payloads**:
        - VCFs (Variants): ~200MB/sample
        - Expression Matrix (RNA): ~50MB/sample (compressed)
        - Neoantigen List (Peptides): ~10MB/sample
    - **Total per Sample**: ~300 MB
    - **Cost for 100 Samples**: 300 MB * 100 = 30 GB * $0.09 = **~$2.70** (Negligible)
    - **Formula**: `(Data Size GB * $0.09)`
    - **Calculation**: `300 samples * 0.3 GB * $0.09 = $8.10`

3.  **Export Raw Data (Not Recommended)**:
    - **Payload**: BAM/CRAM files (~50GB/sample).
    - **Cost for 100 Samples**: 5 TB * $0.09 = **$450.00**.
    - **Recommendation**: Do *not* export raw data. Keep it in AWS; only export the processed feature matrices.
    - **Formula**: `(Data Size GB * $0.09)`

> [!TIP]
> **Budget Buffer**: Allocate **$10-$20/month** for miscellaneous downloads (reports, logs, small feature files). This covers practically all standard export needs.

## Labor & Software Costs

To execute this pipeline, we account for human capital and enterprise software licenses.

| Role / Software | Rate | Quantity | Monthly Cost | Formula |
|:---|:---|:---:|:---:|:---|
| **PhD Intern** | $60 USD / hour | 20 hrs/week | **$4,800** | `$60 * 20hrs * 4weeks` |
| **Gemini for Workspace** | ~$30 USD / user | 5 seats | **$150** | `$30 * 5 users` |
| **Total Labor & Software** | | | **$4,950** | |

> [!NOTE]
> **Why $60/hr for a PhD Intern?**
> The standard market rate for remote bioinformatics interns is ~$35–$45/hr. By budgeting **$60/hr**, we position ourselves to attract **top-tier talent** from elite institutions (Stanford, MIT, Broad Institute) with specialized expertise in **Neoantigen Prediction**, **Graph Neural Networks (GNNs)**, or **LLMs**. This premium ensures high-quality execution for this complex digital twin pipeline.

---

## Final Monthly Cost Estimate (100 Samples)

| Component | Cost per Sample | Samples | Monthly Total |
|:---|:---:|:---:|:---|
| **Compute (Steps 1-8)** | $21.05 | 100 | $2,105 |
| **Storage (Active)** | ~$0.75 | 100 | $75 |
| **Storage (Archive)** | ~$0.14 | 500 (accumulated) | $70 |
| **Data Transfer (Exports)** | ~$0.08 | 100 | $8 |
| **Buffer (Reruns)** | +10% | - | $210 |
| **Total Infrastructure** | - | - | **~$2,468** |
| | | | |
| **Labor & Licenses** | | | **$4,950** |
| **GRAND TOTAL** | | | **~$7,410** |

This total is remarkably efficient for a multi-omics digital twin pipeline, largely thanks to Ready2Run pricing and GPU acceleration.
