# Infrastructure Cost Rudhira

## Cost Breakdown

| Stage | Step | Machine | vCPU | Memory (GB) | Pricing (per hour): Compute | Est. Compute Time (hours) | Compute Cost | Storage | Est. Data | Cost for Storage | Total Est. Cost | Cost for 6 Months | Storage Calc |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Cohort and metadata injestion | omics.m.xlarge | 4 | 16 | 0.2592 | 3 | 0.7776 | 0.023 | 0.5 | 0.0115 | 0.7891 | 0.8466 | 50samples*1mb per sample*storage cost(standard first 50 tb s3) |
| 1 | WES/WGS - Variant Calling and HLA Typing | omics.m.8xlarge | 32 | 128 | 2.0736 | 1200 | 2488.32 | 0.023 | 5503 | 126.569 | 2614.889 | 3247.734 | (3gb ref + (8gb WES+100gb WGS)*50sample + 100gb cram) * stroage cost(standard first 50tb cost) |
| 1 | WES/WGS - Variant Calling and HLA Typing (parallelized) | omics.m.8xlarge | 32 | 128 | 2.0736 | 150 | 311.04 | 0.023 | 5503 | 126.569 | 437.609 | 1070.454 | (3gb ref + (8gb WES+100gb WGS)*50sample + 100gb cram) * stroage cost(standard first 50tb cost) |
| 1 | Bulk RNA-seq: Expression and Antigen Presentation Signature | omics.m.4xlarge | 16 | 64 | 1.0368 | 100 | 103.68 | 0.023 | 554 | 12.742 | 116.422 | 180.132 | (11 gb per sample*50 sample + 4gb bam) |
| 1 | Bulk RNA-seq: Expression and Antigen Presentation Signature | omics.m.4xlarge | 16 | 64 | 1.0368 | 25 | 25.92 | 0.023 | 554 | 12.742 | 38.662 | 102.372 | (11 gb per sample*50 sample + 4gb bam) |
| 1 | TCR/BCR Sequencing | omics.m.2xlarge | 8 | 32 | 0.5184 | 50 | 25.92 | 0.023 | 150 | 3.45 | 29.37 | 46.62 | (2gb per sample * 50 sample)+(1gb pert sample final storage *30) |
| 1 | TCR/BCR Sequencing | omics.m.2xlarge | 8 | 32 | 0.5184 | 25 | 12.96 | 0.023 | 150 | 3.45 | 16.41 | 33.66 | (2gb per sample * 50 sample)+(1gb pert sample final storage *30) |
| 1 | scRNA-seq - Immune and Tumor Microenvironment | omics.m.16xlarge | 64 | 256 | 4.1472 | 110 | 456.192 | 0.023 | 400 | 9.2 | 465.392 | 511.392 | (4 gb raw data *50 samples) + (4 gb processed data *50) |
| 1 | scRNA-seq - Immune and Tumor Microenvironment | omics.m.16xlarge | 64 | 256 | 4.1472 | 50 | 207.36 | 0.023 | 400 | 9.2 | 216.56 | 262.56 | (4 gb raw data *50 samples) + (4 gb processed data *50) |
| 1 | Spatial Transcriptomics | omics.g6.16xlarge | 64 | 256 | 4.5857 | 40 | 183.428 | 0.023 | 150 | 3.45 | 186.878 | 204.128 | (10 sample *10 gb raw data per sample) + (10 sample * 5 gb processed data) |
| 1 | Spatial Transcriptomics | omics.g6.16xlarge | 64 | 256 | 4.5857 | 20 | 91.714 | 0.023 | 150 | 3.45 | 95.164 | 112.414 | (10 sample *10 gb raw data per sample) + (10 sample * 5 gb processed data) |
| 1 | Ligandome (Immunopeptidomics) | omics.g6.8xlarge | 32 | 128 | 2.7194 | 60 | 163.164 | 0.023 | 250 | 5.75 | 168.914 | 197.664 | (10 samples *20 gb raw data) + 50 gb processed data |
| 1 | Ligandome (Immunopeptidomics) | omics.g6.8xlarge | 32 | 128 | 2.7194 | 30 | 81.582 | 0.023 | 250 | 5.75 | 87.332 | 116.082 | (10 samples *20 gb raw data) + 50 gb processed data |

## Summary Costs (for 6 months)

| Metric | Cost |
|---|---|
| Sequential cost | 4388.5166 |
| Paralellized cost | 1698.3886 |
| Practically parallelized cost | 1327.6141 |
