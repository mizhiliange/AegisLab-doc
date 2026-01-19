---
title: Sharing Datasets
weight: 4
---

# Sharing Datasets

Make your datasets available to the research community.

## Overview

Sharing datasets enables:
- Reproducible research
- Algorithm benchmarking
- Community collaboration
- Advancing RCA research

## Preparation

### Validate Quality

Before sharing, ensure dataset quality:

```bash
cd rcabench-platform
./main.py validate-dataset trainticket-custom-001
```

### Document Dataset

Create comprehensive documentation:

```markdown
# TrainTicket Custom Dataset v1

## Overview

Custom fault scenarios for TrainTicket benchmark system.

## Statistics

- **Datapacks**: 100
- **Total size**: 3.5 GB
- **Benchmark**: TrainTicket (42 microservices)
- **Generation method**: Manual curation
- **Created**: 2026-01-18

## Fault Distribution

- Network delay: 35%
- Pod failure: 25%
- Memory pressure: 20%
- CPU stress: 15%
- JVM exceptions: 5%

## Usage

```bash
# Download dataset
./main.py download-dataset trainticket-custom-001

# Evaluate algorithm
./main.py eval batch simplerca trainticket-custom-001 0 99
```

## Citation

If you use this dataset, please cite:

```bibtex
@dataset{trainticket_custom_2026,
  title={TrainTicket Custom Fault Scenarios},
  author={Your Name},
  year={2026},
  publisher={AegisLab}
}
```

## License

MIT License
```

### Add Metadata

Update metadata.json with complete information:

```json
{
  "dataset_name": "trainticket-custom-001",
  "version": "1.0.0",
  "description": "Custom fault scenarios for TrainTicket",
  "authors": ["Your Name <email@example.com>"],
  "license": "MIT",
  "citation": "...",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "created_at": "2026-01-18T10:30:00Z",
  "tags": ["trainticket", "fault-injection", "rca"],
  "fault_types": ["network_delay", "pod_failure", "memory_pressure"],
  "statistics": {
    "total_traces": 1523400,
    "avg_error_rate": 0.42,
    "avg_traces_per_datapack": 15234
  }
}
```

## Sharing via JuiceFS

### Upload to Shared Storage

```bash
# Copy dataset to JuiceFS
cp -r data/trainticket-custom-001 /mnt/jfs/rcabench-platform-v2/

# Verify upload
ls /mnt/jfs/rcabench-platform-v2/trainticket-custom-001/
```

### Set Permissions

```bash
# Make dataset readable by all users
chmod -R 755 /mnt/jfs/rcabench-platform-v2/trainticket-custom-001/
```

### Register Dataset

Register dataset with AegisLab:

```python
from rcabench.openapi import ApiClient, Configuration, DatasetApi
from rcabench.openapi.models import DtoRegisterDatasetReq

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)
dataset_api = DatasetApi(client)

# Register dataset
request = DtoRegisterDatasetReq(
    name="trainticket-custom-001",
    benchmark="trainticket",
    datapack_count=100,
    description="Custom fault scenarios for TrainTicket",
    storage_path="/mnt/jfs/rcabench-platform-v2/trainticket-custom-001",
    version="1.0.0",
    license="MIT",
    tags=["trainticket", "fault-injection", "rca"]
)

response = dataset_api.register_dataset(request)
print(f"Dataset registered: {response.dataset_id}")
```

## Sharing via Download

### Create Archive

```bash
# Create compressed archive
tar -czf trainticket-custom-001.tar.gz data/trainticket-custom-001/

# Verify archive
tar -tzf trainticket-custom-001.tar.gz | head
```

### Upload to Storage

Upload to cloud storage or file sharing service:

```bash
# Example: Upload to S3
aws s3 cp trainticket-custom-001.tar.gz s3://aegislab-datasets/

# Example: Upload to Google Drive
gdrive upload trainticket-custom-001.tar.gz

# Example: Upload to Zenodo
# Use Zenodo web interface or API
```

### Provide Download Instructions

```markdown
## Download

Download the dataset from:

- **Direct download**: https://example.com/trainticket-custom-001.tar.gz
- **Mirror**: https://mirror.example.com/trainticket-custom-001.tar.gz

```bash
# Download and extract
wget https://example.com/trainticket-custom-001.tar.gz
tar -xzf trainticket-custom-001.tar.gz
```

**Checksum (SHA256)**:
```
abc123def456...
```
```

## Publishing

### Create GitHub Repository

```bash
# Create repository
mkdir trainticket-custom-dataset
cd trainticket-custom-dataset

# Add README
cat > README.md << 'EOF'
# TrainTicket Custom Dataset

Custom fault scenarios for TrainTicket benchmark system.

## Download

See [releases](https://github.com/username/trainticket-custom-dataset/releases) for download links.

## Usage

```bash
# Extract dataset
tar -xzf trainticket-custom-001.tar.gz

# Use with rcabench-platform
cd rcabench-platform
ln -s /path/to/trainticket-custom-001 data/
./main.py eval batch simplerca trainticket-custom-001 0 99
```

## Citation

```bibtex
@dataset{trainticket_custom_2026,
  title={TrainTicket Custom Fault Scenarios},
  author={Your Name},
  year={2026},
  url={https://github.com/username/trainticket-custom-dataset}
}
```

## License

MIT License
EOF

# Initialize git
git init
git add README.md
git commit -m "Initial commit"

# Create release with dataset
gh release create v1.0.0 trainticket-custom-001.tar.gz \
    --title "TrainTicket Custom Dataset v1.0.0" \
    --notes "Initial release"
```

### Publish to Zenodo

1. Go to https://zenodo.org
2. Create new upload
3. Upload dataset archive
4. Fill in metadata:
   - Title
   - Authors
   - Description
   - Keywords
   - License
5. Publish and get DOI

### Announce Dataset

Share on relevant platforms:

- Research mailing lists
- Twitter/social media
- Conference workshops
- Research papers

## Licensing

### Choose License

Common licenses for datasets:

**MIT License** (Permissive):
```
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this dataset and associated documentation files (the "Dataset"), to deal
in the Dataset without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Dataset, and to permit persons to whom the Dataset is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Dataset.

THE DATASET IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE DATASET OR THE USE OR OTHER DEALINGS IN THE
DATASET.
```

**CC BY 4.0** (Attribution):
- Requires attribution
- Allows commercial use
- Allows modifications

**CC BY-SA 4.0** (Attribution-ShareAlike):
- Requires attribution
- Allows commercial use
- Requires derivatives to use same license

### Add License File

```bash
# Add LICENSE file
cat > LICENSE << 'EOF'
MIT License

Copyright (c) 2026 Your Name

[Full license text]
EOF

git add LICENSE
git commit -m "Add license"
```

## Versioning

### Semantic Versioning

Use semantic versioning for datasets:

- **v1.0.0**: Initial release
- **v1.1.0**: Added 50 new datapacks
- **v1.0.1**: Fixed schema issues in 5 datapacks
- **v2.0.0**: Changed schema format (breaking change)

### Version Updates

When updating dataset:

```bash
# Create new version
cp -r trainticket-custom-001 trainticket-custom-001-v1.1.0

# Update metadata
cat > trainticket-custom-001-v1.1.0/metadata.json << 'EOF'
{
  "dataset_name": "trainticket-custom-001",
  "version": "1.1.0",
  "changelog": "Added 50 new datapacks with CPU stress faults",
  ...
}
EOF

# Create release
tar -czf trainticket-custom-001-v1.1.0.tar.gz trainticket-custom-001-v1.1.0/
gh release create v1.1.0 trainticket-custom-001-v1.1.0.tar.gz
```

## Best Practices

1. **Validate before sharing**: Ensure dataset quality
2. **Document thoroughly**: Provide clear usage instructions
3. **Choose appropriate license**: Consider how you want dataset used
4. **Use version control**: Track changes over time
5. **Provide checksums**: Enable verification of downloads
6. **Include citation**: Make it easy for others to cite
7. **Respond to issues**: Maintain dataset and address problems
8. **Archive permanently**: Use services like Zenodo for long-term preservation

## Community Guidelines

### Attribution

When using shared datasets:

```python
# Include attribution in code
"""
This evaluation uses the TrainTicket Custom Dataset v1.0.0
by Your Name (https://github.com/username/trainticket-custom-dataset)
Licensed under MIT License
"""
```

### Reporting Issues

If you find issues in a shared dataset:

1. Check if issue is already reported
2. Open GitHub issue with details:
   - Datapack ID
   - Issue description
   - Steps to reproduce
   - Suggested fix
3. Provide pull request if possible

### Contributing Improvements

To contribute to shared datasets:

1. Fork repository
2. Make improvements (fix errors, add documentation)
3. Submit pull request
4. Describe changes clearly

## Example: Complete Sharing Workflow

```bash
# 1. Validate dataset
./main.py validate-dataset trainticket-custom-001

# 2. Create documentation
cat > data/trainticket-custom-001/README.md << 'EOF'
[Dataset documentation]
EOF

# 3. Create archive
tar -czf trainticket-custom-001-v1.0.0.tar.gz data/trainticket-custom-001/

# 4. Calculate checksum
sha256sum trainticket-custom-001-v1.0.0.tar.gz > trainticket-custom-001-v1.0.0.tar.gz.sha256

# 5. Create GitHub repository
mkdir trainticket-custom-dataset
cd trainticket-custom-dataset
git init
# Add README, LICENSE, etc.

# 6. Create release
gh release create v1.0.0 ../trainticket-custom-001-v1.0.0.tar.gz \
    --title "TrainTicket Custom Dataset v1.0.0" \
    --notes "Initial release with 100 fault scenarios"

# 7. Register with AegisLab
python register_dataset.py

# 8. Announce
# Post on mailing lists, social media, etc.
```

## See Also

- [Quality Checks](../quality-checks): Validating dataset quality
- [Dataset Structure](../dataset-structure): Understanding dataset format
- [Algorithm Developers](../../algorithm-developers): Using datasets for evaluation
