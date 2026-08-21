# Datasets

The datasets for **“Cross-Workload Flow-Size Prediction for AI Data Center Networks”** are distributed separately because of their size.

## Download

Download the archive from [Google Drive](https://drive.google.com/file/d/1PUQBHa7oqHzD-HoWrY8vRRS6PpUoRL98/view?usp=sharing), then extract it into this directory while preserving the archive's directory structure.

## Contents and splitting protocol

The paper uses Flux traces together with GPT fine-tuning and updated Spark KMeans, PageRank, and SGD traces collected with INTFusion. The dataset contains multiple tabular files per workload; each file represents an independent workload run.

For reproducing the paper's split, assign complete files to training, validation, and test sets in a 70/20/10 ratio. Do not randomly split individual rows, because that can place flows from the same run in multiple partitions and leak run-specific information.

The target is `flow_size0`. It must be excluded from the input features. Historical flow-size fields such as `flow_size1` through `flow_size5` refer only to preceding flows and may be used as input. The paper scales every feature column and the target by the corresponding maximum calculated from the training partition only.

See the [top-level README](../README.md) and the revised paper for the model inventory and complete methodology.
