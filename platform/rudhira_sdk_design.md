# Rudhira SDK: Platform Architecture & Design

**Vision**: Build the **"Stripe for Cancer Immunology"** — a developer-first platform where researchers can go from `pip install rudhira` to predicting neoantigens in 10 lines of code.

---

## 1. The "Chai Model" Strategy

### Open Source SDK (Free Tier) → Cloud API (Paid Tier)

```
┌─────────────────────────────────────────────────┐
│  Open Source Python Library (GitHub/PyPI)       │
│  • Local CPU-only inference                     │
│  • Sample datasets & tutorials                  │
│  • Community support                            │
└─────────────────┬───────────────────────────────┘
                  │
                  │ (User grows beyond local limits)
                  ↓
┌─────────────────────────────────────────────────┐
│  Rudhira Cloud API (Paid SaaS)                  │
│  • GPU-accelerated inference                    │
│  • Unlimited patients/samples                   │
│  • Premium models (Chai-1, proprietary GNNs)    │
│  • Priority support & consultations             │
└─────────────────────────────────────────────────┘
```

**Conversion Funnel**:
1. **Free User**: PhD student runs 3 samples locally → Publishes a paper citing Rudhira.
2. **Paid User**: Their PI's lab wants to analyze 500 samples → Buys API credits.
3. **Enterprise**: Biotech company integrates Rudhira into their pipeline → Annual contract.

---

## 2. SDK Architecture (Microservices)

Each step of the pipeline = A **separate microservice** exposed via the SDK.

### Microservice Breakdown

| Service Name | Function | Free Tier | Paid Tier | Tech Stack |
|:-------------|:---------|:----------|:----------|:-----------|
| **`ingestion-service`** | Parse FASTQ/VCF/Clinical data | ✅ Unlimited | ✅ Unlimited | FastAPI + Polars |
| **`variant-caller`** | WGS/WES → Somatic mutations | ❌ (Local only) | ✅ GPU (Parabricks) | Docker + NVIDIA Clara |
| **`neoantigen-predictor`** | MHC binding prediction | ✅ 10 samples/month | ✅ Unlimited | PyTorch + MHCflurry 2.0 |
| **`immune-simulator`** | Digital Twin (GNN/Chai-1) | ❌ | ✅ Paid only | JAX + Chai-lab |
| **`visualization`** | 3D protein folding viewer | ✅ Limited | ✅ Full features | PyMOL + Plotly |

---

## 3. SDK Usage Examples

### Example 1: Basic Neoantigen Prediction (Free Tier)

```python
# Installation
# pip install rudhira-bio

import rudhira

# Initialize with local mode (CPU, free)
client = rudhira.Client(mode="local")

# Load patient data
patient = client.load_patient(
    vcf_file="patient_001.vcf",
    rna_bam="patient_001_rna.bam",
    hla_type=["A*02:01", "B*44:02"]
)

# Predict neoantigens (runs locally)
candidates = patient.predict_neoantigens(
    top_n=20,
    min_affinity=500  # IC50 < 500 nM
)

# Results
print(candidates.to_dataframe())
# Output:
#   mutation    peptide      mhc_allele  binding_score
#   BRAF_V600E  TEVWHHTEE    A*02:01     12.3
#   KRAS_G12D   KLVVVGADGV   B*44:02     45.7
```

### Example 2: Cloud API with GPU Acceleration (Paid)

```python
import rudhira

# Initialize with API key (paid tier)
client = rudhira.Client(
    api_key="rk_live_abc123...",
    accelerator="gpu"  # Uses A100s on GCP
)

# Run full pipeline in the cloud
job = client.submit_pipeline(
    fastq_r1="s3://my-bucket/tumor_R1.fastq.gz",
    fastq_r2="s3://my-bucket/tumor_R2.fastq.gz",
    normal_bam="s3://my-bucket/normal.bam",
    hla_typing=True,  # Auto-detect HLA
    include_spatial=True  # Add spatial transcriptomics
)

# Check status
print(job.status)  # "running"

# Wait for completion (async)
results = job.wait()

# Get comprehensive report
results.generate_report(output="patient_001_report.pdf")
```

### Example 3: Microservice-Level API (Advanced)

```python
# Direct access to individual microservices
from rudhira.services import NeoantigenPredictor, ImmunSimulator

# Step 1: Predict neoantigens
predictor = NeoantigenPredictor(model="mhcflurry-2.0")
peptides = predictor.predict(
    mutations=["BRAF_V600E", "KRAS_G12D"],
    hla_alleles=["A*02:01"]
)

# Step 2: Validate with Immune Simulator (Chai-1)
simulator = ImmunSimulator(model="chai-1", backend="cloud")
for peptide in peptides:
    # Fold the peptide-MHC-TCR complex
    structure = simulator.fold_complex(
        peptide=peptide.sequence,
        mhc="HLA-A*02:01",
        tcr="TRAV12-1/TRBV7-2"  # Patient's TCR
    )
    
    # Predict binding energy
    energy = simulator.compute_binding_energy(structure)
    print(f"{peptide.sequence}: {energy} kcal/mol")
```

---

## 4. Microservices Deployment Architecture

### Cloud Infrastructure (Kubernetes + Serverless)

```
┌─────────────────────────────────────────────────┐
│          API Gateway (FastAPI)                   │
│          • Authentication (API keys)             │
│          • Rate limiting                         │
│          • Request routing                       │
└─────────────┬───────────────────────────────────┘
              │
              ├──────────────────────────────────────┐
              │                                      │
              ↓                                      ↓
┌────────────────────────┐          ┌────────────────────────┐
│  Ingestion Service     │          │  Variant Caller        │
│  (AWS Lambda)          │          │  (K8s Pod + A100 GPU)  │
│  • Parse VCF/FASTQ     │          │  • Parabricks WGS      │
│  • Validate HLA        │          │  • Auto-scale 0-10     │
└────────────────────────┘          └────────────────────────┘
              │                                      │
              ↓                                      ↓
┌────────────────────────┐          ┌────────────────────────┐
│  Neoantigen Predictor  │          │  Immune Simulator      │
│  (GCP Cloud Run)       │          │  (GCP TPU Pods)        │
│  • MHCflurry inference │          │  • Chai-1 folding      │
│  • 100ms latency       │          │  • Graph Neural Nets   │
└────────────────────────┘          └────────────────────────┘
              │                                      │
              └──────────────┬───────────────────────┘
                             ↓
              ┌────────────────────────┐
              │  Results Store         │
              │  (PostgreSQL + S3)     │
              │  • Patient metadata    │
              │  • PDB structures      │
              └────────────────────────┘
```

### Technology Stack

| Layer | Technology | Rationale |
|:------|:-----------|:----------|
| **SDK (Client)** | Python 3.10+ | Universal in bioinformatics |
| **API Gateway** | FastAPI + Uvicorn | Async, type-safe, auto-docs |
| **Orchestration** | Kubernetes (GKE) | Auto-scaling for GPU jobs |
| **Serverless** | AWS Lambda / GCP Cloud Run | Cost-efficient for light tasks |
| **GPU Compute** | NVIDIA A100 (GCP/AWS) | Parabricks, Chai-1 |
| **TPU Compute** | Google TPU v5e | Large GNN models |
| **Storage** | S3 (raw data) + PostgreSQL (metadata) | Compliance-ready |
| **Monitoring** | Datadog + Sentry | Real-time error tracking |

---

## 5. Monetization Strategy

### Pricing Tiers

#### **Free Tier** (Open Source)
- **Local inference only** (user's laptop/HPC).
- **10 samples/month** via Cloud API.
- **Community support** (GitHub Discussions).
- **Limitations**: No GPU, no Chai-1, no Spatial Transcriptomics.

#### **Professional** ($499/month)
- **Unlimited samples**.
- **GPU acceleration** (A100 instances).
- **Chai-1 access** for structure prediction.
- **Email support** (48h response).

#### **Enterprise** (Custom pricing, $50k+/year)
- **On-premises deployment** (air-gapped).
- **Dedicated infrastructure** (VPC).
- **Custom model training** on proprietary data.
- **SLA guarantees** (99.9% uptime).
- **White-glove onboarding** (CSM + PhD support).

---

## 6. SDK Distribution & Developer Experience

### Open Source Repository Structure

```
rudhira/
├── rudhira/              # Core SDK
│   ├── __init__.py
│   ├── client.py         # Main API client
│   ├── models/           # ML models
│   │   ├── mhcflurry.py
│   │   ├── chai_wrapper.py
│   ├── services/         # Microservice clients
│   │   ├── ingestion.py
│   │   ├── variant_calling.py
│   │   ├── neoantigen.py
│   ├── utils/
│   │   ├── vcf_parser.py
│   │   ├── visualization.py
├── examples/             # Jupyter notebooks
│   ├── quickstart.ipynb
│   ├── advanced_pipeline.ipynb
├── tests/
├── docs/                 # MkDocs documentation
├── docker/               # Dockerfiles for local services
├── setup.py
├── pyproject.toml
└── README.md
```

### Developer Onboarding Flow

1. **Install**: `pip install rudhira-bio`
2. **Sign Up**: `rudhira auth login` → Creates free API key.
3. **Quickstart**: Run example notebook → Predicts neoantigens for sample patient in 5 minutes.
4. **Upgrade**: Hit 10-sample limit → Prompted to upgrade to Pro.

---

## 7. Competitive Differentiation

| Feature | Rudhira | Gritstone (Closed) | Public Tools (GATK/MHCflurry) |
|:--------|:--------|:-------------------|:------------------------------|
| **Ease of Use** | `pip install` | Enterprise sales only | Requires 10+ tools |
| **Speed** | GPU-accelerated (1h WGS) | Unknown | 24h+ CPU |
| **Digital Twin** | Chai-1 integration | Proprietary | None |
| **Cost** | $499/month | $100k+ contracts | Free (but slow) |
| **Deployment** | Cloud or on-prem | Cloud only | DIY |

---

## 8. Next Steps to Build This

### Phase 1: SDK MVP (Weeks 1-8)
1. Build `rudhira-bio` PyPI package with **Neoantigen Predictor** only.
2. Deploy simple FastAPI backend on GCP Cloud Run.
3. Implement API key auth + rate limiting.
4. Write 3 example notebooks.

### Phase 2: Cloud Services (Weeks 9-16)
1. Add **Variant Caller** microservice (Parabricks on K8s).
2. Integrate **Chai-1** for structure prediction.
3. Build usage dashboard for customers.

### Phase 3: Enterprise Features (Weeks 17-24)
1. On-premises deployment package (Helm charts).
2. Custom model fine-tuning API.
3. HIPAA compliance certifications.

---

## Appendix: Example API Response

```json
{
  "job_id": "job_abc123",
  "status": "completed",
  "patient_id": "PT-001",
  "results": {
    "mutations_detected": 342,
    "neoantigens": [
      {
        "rank": 1,
        "mutation": "BRAF_V600E",
        "peptide": "TEVWHHTEE",
        "mhc_allele": "A*02:01",
        "binding_affinity_ic50": 12.3,
        "immunogenicity_score": 0.89,
        "structure_pdb_url": "https://api.rudhira.bio/structures/abc123.pdb"
      }
    ],
    "digital_twin": {
      "tcr_repertoire_diversity": "high",
      "predicted_response_rate": 0.67
    }
  }
}
```
