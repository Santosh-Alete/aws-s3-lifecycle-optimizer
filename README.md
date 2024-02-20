# AWS S3 Lifecycle Optimizer

Automated tool to identify S3 buckets without lifecycle policies and recommend optimal storage class transitions to reduce costs.

## Overview

This tool scans all S3 buckets across AWS accounts, analyzes object access patterns, and recommends lifecycle policies to transition objects to cheaper storage classes (Intelligent-Tiering, Glacier, Deep Archive). Perfect for FinOps teams looking to optimize S3 storage costs.

## Features

- **Multi-Account Support**: Scan S3 buckets across multiple AWS accounts
- **Lifecycle Policy Audit**: Identify buckets without lifecycle policies
- **Access Pattern Analysis**: Analyze S3 access logs to determine object usage
- **Cost Estimation**: Calculate potential savings from lifecycle transitions
- **Intelligent-Tiering Recommendations**: Identify candidates for automatic tiering
- **Glacier Migration**: Recommend objects for Glacier/Deep Archive
- **CSV Reports**: Generate detailed reports with savings estimates
- **Automated Remediation**: Optionally apply recommended lifecycle policies

## How It Works

```
┌─────────────────┐
│  Scan S3        │
│  Buckets        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Check Existing │
│  Lifecycle      │
│  Policies       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Analyze Object │
│  Age & Access   │
│  Patterns       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Calculate      │
│  Savings &      │
│  Recommend      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Generate       │
│  Report         │
└─────────────────┘
```

## Installation

```bash
git clone https://github.com/Santosh-Alete/aws-s3-lifecycle-optimizer.git
cd aws-s3-lifecycle-optimizer
pip install -r requirements.txt
```

## Configuration

Create `config.yaml`:

```yaml
aws:
  accounts:
    - account_id: "123456789012"
      name: "Production"
      role_arn: "arn:aws:iam::123456789012:role/S3AuditorRole"
    - account_id: "987654321098"
      name: "Development"
      role_arn: "arn:aws:iam::987654321098:role/S3AuditorRole"
  
  regions:
    - us-east-1
    - us-west-2

analysis:
  # Days before transitioning to different storage classes
  transition_rules:
    standard_to_ia: 30        # Standard → Standard-IA
    ia_to_intelligent: 90     # Standard-IA → Intelligent-Tiering
    to_glacier: 180           # → Glacier Flexible Retrieval
    to_deep_archive: 365      # → Glacier Deep Archive
  
  # Minimum object size for transitions (bytes)
  min_object_size: 128000     # 128 KB (IA minimum)
  
  # Cost per GB per month (update with current pricing)
  storage_costs:
    standard: 0.023
    standard_ia: 0.0125
    intelligent_tiering: 0.0025
    glacier: 0.004
    deep_archive: 0.00099

reporting:
  output_dir: "./reports"
  format: "csv"  # csv or json
```

## Usage

### Audit All Buckets

```bash
python s3_optimizer.py --mode audit --output s3-audit-report.csv
```

### Generate Recommendations

```bash
python s3_optimizer.py --mode recommend --output s3-recommendations.csv
```

### Apply Lifecycle Policies (Dry Run)

```bash
python s3_optimizer.py --mode apply --dry-run
```

### Apply Lifecycle Policies (Live)

```bash
python s3_optimizer.py --mode apply --bucket my-bucket-name
```

### Analyze Specific Account

```bash
python s3_optimizer.py --mode audit --account Production
```

## Example Output

```
🗂️  S3 Lifecycle Optimization Report
Generated: 2024-02-20 10:30:00

Account: Production (123456789012)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total Buckets: 45
Buckets without Lifecycle Policies: 12
Total Storage: 8.5 TB
Estimated Monthly Cost: $196.50

Optimization Opportunities:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 my-app-logs (2.1 TB)
   Current Cost: $48.30/month
   Recommendation: Transition to Intelligent-Tiering after 30 days
   Potential Savings: $43.05/month (89%)
   
📦 backup-data (1.8 TB)
   Current Cost: $41.40/month
   Recommendation: Transition to Glacier after 90 days
   Potential Savings: $34.20/month (83%)
   
📦 archive-2023 (950 GB)
   Current Cost: $21.85/month
   Recommendation: Move to Deep Archive immediately
   Potential Savings: $20.91/month (96%)

Total Potential Savings: $98.16/month ($1,178/year)
```

## Lifecycle Policy Examples

### Intelligent-Tiering (Recommended for Most Use Cases)

```json
{
  "Rules": [
    {
      "Id": "IntelligentTieringRule",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ]
    }
  ]
}
```

### Multi-Tier Archival Strategy

```json
{
  "Rules": [
    {
      "Id": "ArchivalStrategy",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER_IR"
        }
      ]
    }
  ]
}
```

## AWS Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation",
        "s3:GetBucketLifecycleConfiguration",
        "s3:GetBucketVersioning",
        "s3:GetBucketLogging",
        "s3:ListBucket",
        "s3:GetObjectAttributes",
        "s3:PutLifecycleConfiguration"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricStatistics"
      ],
      "Resource": "*"
    }
  ]
}
```

## Cost Savings Calculator

The tool estimates savings based on:
- Current storage class and size
- Object age distribution
- Access patterns (if S3 access logs available)
- AWS pricing for different storage classes

**Example Calculation:**
```
Bucket: my-logs (1 TB in Standard)
Current Cost: $23/month

Recommendation: Intelligent-Tiering after 30 days
- 30% frequently accessed → $6.90/month
- 70% infrequently accessed → $1.75/month
New Cost: $8.65/month

Monthly Savings: $14.35 (62%)
Annual Savings: $172.20
```

## Real-World Impact

At my previous organization:
- Identified 300+ buckets without lifecycle policies
- Implemented Intelligent-Tiering across 85% of storage
- Achieved $750K annual savings
- Reduced storage costs by 68% on average
- Automated lifecycle management for all new buckets

## Scheduling

Run weekly to catch new buckets:

```bash
# Weekly audit (every Monday at 8 AM)
0 8 * * 1 /usr/bin/python3 /path/to/s3_optimizer.py --mode audit --output /reports/s3-audit-$(date +\%Y\%m\%d).csv
```

## Contributing

Pull requests welcome! Please open an issue first.

## License

MIT License

## Author

Santosh Alete  
Cloud FinOps Engineer  
[LinkedIn](https://www.linkedin.com/in/santosh-r-alete-09260a27b/) | [GitHub](https://github.com/Santosh-Alete)
