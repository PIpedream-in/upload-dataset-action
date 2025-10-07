# Pipedream Upload Dataset Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-upload-dataset-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/upload-dataset)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/release/PIpedream-in/upload-dataset-action.svg)](https://github.com/PIpedream-in/upload-dataset-action/releases)


Upload datasets for ML training on Pipedream.

## Quick Start

```yaml
- uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./data/train.csv
    name: training-data
```

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Examples](#examples)
- [Best Practices](#best-practices)

---

## Overview

This GitHub Action uploads datasets to Pipedream ML Orchestrator for use in training jobs. It supports:

- ✅ **Multiple formats:** CSV, Parquet, JSON, directories
- ✅ **Automatic compression:** Directories are auto-compressed to tar.gz
- ✅ **Exclusion patterns:** Exclude unwanted files
- ✅ **Wait for upload:** Optionally wait for upload completion
- ✅ **Dataset versioning:** Multiple versions supported

---

## Prerequisites

### 1. Get API Key

Visit [https://pipedream.in/api-keys](https://pipedream.in/api-keys) to create an API key.

### 2. Add GitHub Secret

1. Go to your repository **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `PDTRAIN_API_KEY`
4. Value: Your API key (e.g., `sdk_xxxxx`)

---

## Usage

### Upload CSV File

```yaml
- uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./data.csv
    name: train-data
```

### Upload Directory (Auto-compressed)

```yaml
- uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./image-dataset
    name: cifar10
    description: "CIFAR-10 training images"
```

### Upload with Exclusions

```yaml
- uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./my-images
    name: image-classification
    exclude-patterns: "*.DS_Store,*.tmp,raw"
```

---

## Inputs

### Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| `api-key` | Pipedream API key | `${{ secrets.PDTRAIN_API_KEY }}` |
| `path` | Path to dataset file or directory | `./data.csv` or `./images/` |
| `name` | Dataset name | `training-data` |

### Optional Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `description` | None | Dataset description |
| `exclude-patterns` | `*.DS_Store,*.pyc,__pycache__,.git,.venv,node_modules` | Comma-separated patterns to exclude |
| `wait-for-upload` | `true` | Wait for upload to complete |
| `api-url` | `https://ml-orchestrator.pipedream.in` | Orchestrator API URL |

---

## Outputs

All outputs are available via `steps.<step-id>.outputs.<output-name>`.

| Output | Description | Example |
|--------|-------------|---------|
| `dataset-id` | Dataset ID | `ds_abc123-def456-7890` |
| `dataset-name` | Dataset name | `training-data` |
| `size-bytes` | Dataset size in bytes | `1048576` |
| `upload-url` | S3 URL where dataset was uploaded | `s3://...` |

### Using Outputs

```yaml
- uses: pipedream/upload-dataset@v1
  id: dataset
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./data.csv
    name: train-data

- name: Show dataset info
  run: |
    echo "Dataset ID: ${{ steps.dataset.outputs.dataset-id }}"
    echo "Size: ${{ steps.dataset.outputs.size-bytes }} bytes"

- uses: pipedream/train-ml@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    dataset: ${{ steps.dataset.outputs.dataset-id }}
    framework: pytorch
    code-path: ./training
```

---

## Examples

### Upload CSV on Release

```yaml
name: Upload Production Dataset

on:
  release:
    types: [published]

jobs:
  upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: pipedream/upload-dataset@v1
        id: dataset
        with:
          api-key: ${{ secrets.PDTRAIN_API_KEY }}
          path: ./data/production.csv
          name: production-dataset
          description: "Production dataset for release ${{ github.ref_name }}"

      - name: Comment on release
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.repos.updateRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              release_id: context.payload.release.id,
              body: context.payload.release.body + `\n\n📊 Dataset ID: \`${{ steps.dataset.outputs.dataset-id }}\``
            });
```

### Upload Image Directory

```yaml
- uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./datasets/images
    name: image-classification
    description: "Image classification dataset"
    exclude-patterns: "*.DS_Store,*.tmp,thumbs.db,raw"
```

### Upload and Train in Pipeline

```yaml
name: Upload and Train

on: [push]

jobs:
  upload-and-train:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Upload dataset
        id: dataset
        uses: pipedream/upload-dataset@v1
        with:
          api-key: ${{ secrets.PDTRAIN_API_KEY }}
          path: ./data/latest.parquet
          name: latest-data

      - name: Train model
        uses: pipedream/train-ml@v1
        with:
          api-key: ${{ secrets.PDTRAIN_API_KEY }}
          dataset: ${{ steps.dataset.outputs.dataset-id }}
          framework: pytorch
          code-path: ./training
          entry-point: train.py
```

### Conditional Upload (only on main)

```yaml
- name: Upload dataset
  if: github.ref == 'refs/heads/main'
  uses: pipedream/upload-dataset@v1
  with:
    api-key: ${{ secrets.PDTRAIN_API_KEY }}
    path: ./data/train.csv
    name: production-data
```

---

## Best Practices

### 1. Use Descriptive Names

```yaml
name: train-data-${{ github.ref_name }}-${{ github.run_number }}
# Good: Includes context about version/run
```

### 2. Add Descriptions for Production Datasets

```yaml
description: |
  Production training dataset for release ${{ github.ref_name }}
  Generated: ${{ github.event.head_commit.timestamp }}
  Commit: ${{ github.sha }}
```

### 3. Exclude Unnecessary Files

```yaml
exclude-patterns: "*.DS_Store,*.pyc,__pycache__,.git,.venv,node_modules,*.tmp,raw,thumbs.db"
```

### 4. Use Output in Subsequent Steps

```yaml
- uses: pipedream/upload-dataset@v1
  id: dataset
  # ...

- uses: pipedream/train-ml@v1
  with:
    dataset: ${{ steps.dataset.outputs.dataset-id }}  # Use uploaded dataset
```

### 5. Add Upload Summary

```yaml
- name: Create upload summary
  run: |
    echo "## Dataset Upload Summary" >> $GITHUB_STEP_SUMMARY
    echo "- **Dataset ID:** \`${{ steps.dataset.outputs.dataset-id }}\`" >> $GITHUB_STEP_SUMMARY
    echo "- **Name:** ${{ steps.dataset.outputs.dataset-name }}" >> $GITHUB_STEP_SUMMARY
    echo "- **Size:** $(numfmt --to=iec-i --suffix=B ${{ steps.dataset.outputs.size-bytes }})" >> $GITHUB_STEP_SUMMARY
```

---

## Supported File Formats

| Format | Extension | Notes |
|--------|-----------|-------|
| CSV | `.csv` | Comma-separated values |
| Parquet | `.parquet` | Columnar storage format |
| JSON | `.json` | JSON data |
| Directory | N/A | Auto-compressed to `.tar.gz` |

---

## Troubleshooting

### Error: "Dataset path does not exist"

**Cause:** The specified path is not found.

**Solution:** Ensure the file/directory exists and the path is correct:
```yaml
- uses: actions/checkout@v5  # Make sure to checkout first
- uses: pipedream/upload-dataset@v1
  with:
    path: ./data/train.csv  # Verify this path exists
```

### Upload is slow

**Cause:** Large dataset or slow network.

**Solution:**
- Use compressed formats (Parquet instead of CSV)
- Exclude unnecessary files
- Consider uploading only on specific branches/events

### Dataset already exists error

**Cause:** Dataset with the same name already exists.

**Solution:** Dataset names should be unique. Use versioning:
```yaml
name: train-data-v${{ github.run_number }}
```

---

## Complete Workflow Example

```yaml
name: Dataset Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'data/**'

jobs:
  upload-dataset:
    runs-on: ubuntu-latest
    outputs:
      dataset-id: ${{ steps.upload.outputs.dataset-id }}

    steps:
      - uses: actions/checkout@v5

      - name: Upload dataset
        id: upload
        uses: pipedream/upload-dataset@v1
        with:
          api-key: ${{ secrets.PDTRAIN_API_KEY }}
          path: ./data/training.parquet
          name: train-data-${{ github.run_number }}
          description: "Training data from commit ${{ github.sha }}"
          exclude-patterns: "*.DS_Store,*.tmp"

      - name: Create summary
        run: |
          echo "## Dataset Upload Summary" >> $GITHUB_STEP_SUMMARY
          echo "- **ID:** \`${{ steps.upload.outputs.dataset-id }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- **Name:** ${{ steps.upload.outputs.dataset-name }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Size:** $(numfmt --to=iec-i --suffix=B ${{ steps.upload.outputs.size-bytes }})" >> $GITHUB_STEP_SUMMARY

  train-model:
    needs: upload-dataset
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v5

      - uses: pipedream/train-ml@v1
        with:
          api-key: ${{ secrets.PDTRAIN_API_KEY }}
          dataset: ${{ needs.upload-dataset.outputs.dataset-id }}
          framework: pytorch
          code-path: ./training
          entry-point: train.py
```

---

## Support

- **Documentation:** [https://docs.pipedream.in](https://docs.pipedream.in)
- **CLI Tool:** [`pdtrain`](https://github.com/pipedream/pdtrain)
- **Email:** [hello@pipedream.in](mailto:hello@pipedream.in)

---

## License

MIT License - see [LICENSE](../LICENSE) for details.
