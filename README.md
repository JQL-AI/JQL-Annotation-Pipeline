# JQL-Annotation-Pipeline
This repository provides inference code for the data annotation pipeline described in [Judging Quality Across Languages: A Multilingual Approach to Pretraining Data Filtering with Language Models](https://arxiv.org/abs/2505.22232)

 The process involves:

1. Embedding text documents using a specified multilingual embedding model.
2. Passing these embeddings through multiple fine-tuned regression head to predict a score. (E.g. for educational value)


In this repository we provide two ways to run our pre-trained regression heads for educational content (available on [Huggingface](https://huggingface.co/Jackal-AI/JQL-Edu-Heads)).

In `src/utils/regression_head.py` we define `RegressionHead` for direct use of the JQL annotators (see example below). 

Additionally, in `src/datatrove_jql_annotator.py` we provide a [Datatrove](https://github.com/huggingface/datatrove) pipeline step `JQLAnnotator` ready for data curation at scale.
## Installation 

Clone repository 
```
git clone https://github.com/JQL-AI/JQL-Annotation-Pipeline.git
cd ./JQL-Annotation-Pipeline/
```

Install torch requirements for your specific CUDA driver (here CUDA 12.6). 

`pip install -r torch_requirements.txt --index-url https://download.pytorch.org/whl/cu126`

Install other dependencies: 

`pip install -r requirements.txt`

## Usage Example
A minimal usage example using our [datatrove](https://github.com/huggingface/datatrove) pipeline could look like the snippet below. This code will automatically load our pre-trained Edu annotators from [huggingface](https://huggingface.co/Jackal-AI/JQL-Edu-Heads).

When providing the `JQLAnnotator` with a `stats-writer` the pipeline will write only the meta-data of each document to a seperate directory in addition to the regular data writer. 
We found this to be particularly useful for annotation analysis without needing to load the actual text documents. 

```python 
from datatrove_jql_annotator import JQLAnnotator, stats_adapter
from datatrove.pipeline.readers import ParquetReader
from datatrove.executor import LocalPipelineExecutor
from datatrove.pipeline.writers import ParquetWriter, JsonlWriter


pipeline = [
            ParquetReader(
                data_folder='webdata_inputs', # Change to your input directory
                glob_pattern='*.parquet',
            ),          
            JQLAnnotator(
                stats_writer=JsonlWriter(
                    output_folder=f'jql_outputs/stats', # Change to your output directory
                    adapter=stats_adapter,
                    expand_metadata=True,
                    ),
            ),
            ParquetWriter(
                output_folder=f'jql_outputs/data' # Change to your output directory
            ),
    ]
stage = LocalPipelineExecutor(
    pipeline,
    tasks=2, # n_tasks to be executed across all machines
    local_tasks=2, # n_tasks to be executed on this machine (e.g. the number of available gpus)
    local_rank_offset=0, # determines the first rank/task of this machine (has to be adapted per machine)
    logging_dir=f'./logs/test/jql'
)
```


`JQLAnnotator` takes the following arguments and works as follows:


-   `embedder_model_id` (str): This specifies the identifier for the multilingual embedding model to be used. Examples include `'Snowflake/snowflake-arctic-embed-m-v2.0'`, `'Alibaba-NLP/gte-multilingual-base'`, or `'jinaai/jina-embeddings-v3'`. This model is responsible for converting text into numerical vector representations (embeddings). The default is `'Snowflake/snowflake-arctic-embed-m-v2.0'`.
-   `regression_head_checkpoints` (Optional[dict[str, str]]): This is a dictionary where each key is a custom name for a regression head, and the corresponding value is the file path to its PyTorch Lightning checkpoint (`.ckpt` file). The scores generated by these heads will be stored in the document metadata using these custom names (e.g., `score_MY_CUSTOM_HEAD_NAME`).
    -   If you set this to `None` (the default), the annotator will attempt to load default our pretrained JQL-Edu regression heads from [Huggingface](https://huggingface.co/Jackal-AI/JQL-Edu-Heads). **Important:** These default heads are only compatible if `embedder_model_id` is set to `'Snowflake/snowflake-arctic-embed-m-v2.0'`. If you use a different embedder model, you *must* provide your own `regression_head_checkpoints`.
-   `batch_size` (int): Defines the number of text samples to be processed together in a single batch during both the embedding and annotation phases. Larger batch sizes can lead to higher throughput but will also require more memory (RAM/VRAM). The default value is `1_000` which we found suitable on A100-80GB. If your data contains many long documents batch-size `1_000` will lead to OOM errors. In this case you can simply rerun the pipeline with reduced bath size. Datatrove will ensure that only unfinished tasks are attempted again.
-   `device_overwrite` (Optional[str]): Allows you to explicitly specify the device (e.g., `'cuda'`, `'cuda:0'`, `'cpu'`) on which the embedding and regression models should be loaded and where computations will be performed.
    -   If left as `None` (the default) and CUDA-enabled GPUs are available, the process will attempt to distribute the workload across all available GPUs based on the rank of the current process. If CUDA is not available, it will fall back to the CPU. If you provide a specific device string (e.g., `'cuda:0'`), all processes will use that designated device.
-   `stats_writer` (Optional[DiskWriter]): An optional instance of a `DiskWriter` (from the Datatrove library) or a compatible class. If provided, this writer will save document metadata (including the newly added scores) in additional files, separate from the actual document text. 
The default outputs will still contain all metadata.
This can be useful for analyzing scoring distributions or dataset characteristics without needing to load the full text content.

**How it works:**

1.  **Initialization**: When a `JQLAnnotator` step is created, it loads the specified `embedder_model_id` and the `regression_head_checkpoints`. If default heads are used, it downloads them from the Hugging Face Hub.
2.  **Embedding**: During the pipeline run, for each batch of documents, the text content is fed into the loaded embedder model to produce numerical embeddings.
3.  **Scoring**: These embeddings are then passed through each of the loaded regression heads. Each head outputs a raw score.
4.  **Annotation**: For each document in the batch, the scores from the different regression heads are added to its `metadata` dictionary. For instance, if a regression head checkpoint was provided with the key `'Edu-JQL-Gemma-SF'`, the corresponding score will be stored in `doc.metadata['score_Edu-JQL-Gemma-SF']`.
5.  **Stats Logging (Optional)**: If a `stats_writer` is configured, the metadata (including the new scores but excluding the text) for each document is written to a separate output. 
6.  **Output**: The documents, now augmented with the new scores in their metadata, are passed on to the next step in the pipeline.

Alternatively, you can use the JQL Annotators directly by modifing the code below 

```python 
from utils.regression_head import RegressionHead
from transformers.utils.hub import cached_file
from utils.embedder import get_embedder_instance
import torch

# load embedder
device = 'cuda'
embedder = get_embedder_instance('Snowflake/snowflake-arctic-embed-m-v2.0', device, torch.bfloat16)
# load JQL Edu annotation heads
regression_head_checkpoints = {
                'Edu-JQL-Gemma-SF': cached_file('Jackal-AI/JQL-Edu-Heads', 'checkpoints/edu-gemma-snowflake-balanced.ckpt'),
                'Edu-JQL-Mistral-SF': cached_file('Jackal-AI/JQL-Edu-Heads', 'checkpoints/edu-mistral-snowflake-balanced.ckpt'),
                'Edu-JQL-Llama-SF': cached_file('Jackal-AI/JQL-Edu-Heads', 'checkpoints/edu-llama-snowflake-balanced.ckpt'),
            }
regression_heads = {}
for name, path in regression_head_checkpoints.items():
    regression_heads[name] = RegressionHead.load_from_checkpoint(path, map_location=device).to(torch.bfloat16)

# Given a single document
doc = 'Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua'
embeddings = embedder.embed([doc])
scores = {}
with torch.no_grad():
    for name, regression_head in regression_heads.items():
        scores[f'score_{name}'] = regression_head(embeddings).cpu().squeeze(1)
```
## 📖 Citation
Please cite as
```bibtex
@article{ali2025judging,
    title     = {Judging Quality Across Languages: A Multilingual Approach to Pretraining Data Filtering with Language Models},
    author    = {
      Mehdi Ali,
      Manuel Brack,
      Max Lübbering,
      Elias Wendt,
      Abbas Goher Khan,
      Richard Rutmann,
      Alex Jude,
      Maurice Kraus,
      Alexander Arno Weber,
      Felix Stollenwerk,
      David Kaczér,
      Florian Mai,
      Lucie Flek,
      Rafet Sifa,
      Nicolas Flores-Herr,
      Joachim Köhler,
      Patrick Schramowski,
      Michael Fromm,
      Kristian Kersting
    },
    year      = {2025},
    journal   = {arXiv preprint arXiv:2505:22232}
  }
```
