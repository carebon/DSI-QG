
#2023-11-07 ben

This work is expand of DSI-Transform https://github.com/ArvinZhuang/DSI-transformers/tree/main, but not offical, but helpful.

when i try to test this code, i find some tips.

1.Hugging Face https://huggingface.co/datasets/natural_questions is not usually work,especially when u need to use a proxy serve. if so, u need to try donwloading dataset and model to disk and cite it when used.https://github.com/ArvinZhuang/DSI-QG/issues/11


# DSI-QG
The official repository for [Bridging the Gap Between Indexing and Retrieval for Differentiable Search Index with Query Generation](https://arxiv.org/pdf/2206.10128.pdf),
Shengyao Zhuang, Houxing Ren, Linjun Shou, Jian Pei, Ming Gong, Guido Zuccon and Daxin Jiang.

##2023-11-02
When i use this code. i have some question, a most important is i cannt use Hugging Face dataset and models online.i try to down dataset to disk (https://blog.51cto.com/u_15719772/6104170), also successed process by process_marco.py. But when i Run run.py, faled to 

`Traceback (most recent call last):
  File "/-----/DSI-transform/lib/python3.8/site-packages/transformers/tokenization_utils_base.py", line 1724, in from_pretrained
    resolved_vocab_files[file_id] = cached_path(
  File "/-----/DSI-transform/lib/python3.8/site-packages/transformers/file_utils.py", line 1921, in cached_path
    output_path = get_from_cache(
  File "/-------/DSI-transform/lib/python3.8/site-packages/transformers/file_utils.py", line 2177, in get_from_cache
    raise ValueError(
ValueError: Connection error, and we cannot find the requested files in the cached path. Please try again or make sure your Internet connection is on.
`

another question is Pytorch and Cuda and cudnn, my GUP is single RTX3090. i make sure it can be use.

`pip install torch==1.7.1+cu110 torchvision==0.8.2+cu110 torchaudio==0.7.2 -f https://download.pytorch.org/whl/torch_stable.html`

2023-11-02
## Installation

`pip install -r requirements.txt`
> Note: The current code base has been tested with a GPU cluster with 8 Tesla V100 gpus.
> We also use [wandb](https://wandb.ai/site) cloud-based logger which you may need to register and login first.



## Data Preparing
Simply run `bash get_data.sh`. 

The script will automatically download and create training and evaluation datasets for XORQA 100k and MS MARCO 100k dataset in `\data` folder.

## Training
We take XORQA 100k and t5/mt5-base as a training example. You can simply change the dataset and model config in the run arguments to run other experiments.

The retrieval Hits score results on dev set will be logged on wandb logger during training.

### Train Original DSI model

Recall that the original DSI model directly take document text as input during indexing. Run the following command to train a DSI model with the original document corpus.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "DSI" \
        --model_name "google/mt5-base" \
        --run_name "XORQA-100k-mt5-base-DSI" \
        --max_length 256 \
        --train_file data/xorqa_data/100k/xorqa_DSI_train_data.json \
        --valid_file data/xorqa_data/100k/xorqa_DSI_dev_data.json \
        --output_dir "models/XORQA-100k-mt5-base-DSI" \
        --learning_rate 0.0005 \
        --warmup_steps 100000 \
        --per_device_train_batch_size 16 \
        --per_device_eval_batch_size 8 \
        --evaluation_strategy steps \
        --eval_steps 1000 \
        --max_steps 1000000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 1000 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 1 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False \
        --metric_for_best_model Hits@10 \
        --greater_is_better True

```


### Train DSI-QG model
#### Step 1:
Our DSI-QG model requires a query generation model to generate potentially-relevant queries to
represent each candidate documents. Run the following command to train a mT5-large cross-lingual query generation model.
> Note, if you plan to run DSI-QG experiments on MS MARCO dataset, you can skip this step and directly use off-the-shelf [docTquery-t5](https://github.com/castorini/docTTTTTquery) model with huggingface model name `castorini/doc2query-t5-large-msmarco` for query generation in step 2.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "docTquery" \
        --model_name "google/mt5-large" \
        --run_name "docTquery-XORQA" \
        --max_length 128 \
        --train_file data/xorqa_data/100k/xorqa_docTquery_train_data.json \
        --valid_file data/xorqa_data/100k/xorqa_docTquery_dev_data.json \
        --output_dir "models/xorqa_docTquery_mt5_large" \
        --learning_rate 0.0001 \
        --warmup_steps 0 \
        --per_device_train_batch_size 4 \
        --per_device_eval_batch_size 4 \
        --evaluation_strategy steps \
        --eval_steps 100 \
        --max_steps 2000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 100 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 4 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False

```
If you do not want to train the QG model yourself, we also provide our trained QG models on [huggingface](https://huggingface.co/ielabgroup/xor-tydi-docTquery-mt5-large).

#### Step 2:
We then run the query generation for all the documents in the corpus: 
> Note: set the `--model_path` to the best checkpoints. Remove `--model_path` argument and set `--model_name castorini/doc2query-t5-large-msmarco` for MS MARCO dataset.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task generation \
        --model_name google/mt5-large \
        --model_path models/xorqa_docTquery_mt5_large/checkpoint-xxx \
        --per_device_eval_batch_size 32 \
        --run_name docTquery-XORQA-generation \
        --max_length 256 \
        --valid_file data/xorqa_data/100k/xorqa_corpus.tsv \
        --output_dir temp \
        --dataloader_num_workers 10 \
        --report_to wandb \
        --logging_steps 100 \
        --num_return_sequences 10
```
the `--num_return_sequences` is the argument for how many queries per langurage to generate for representing each candidate document. In this example, we use 10 generated queries each language per document.

#### Step 3:

Finally, we can train DSI-QG with query-represented corpus:

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "DSI" \
        --model_name "google/mt5-base" \
        --run_name "XORQA-100k-mt5-base-DSI-QG" \
        --max_length 32 \
        --train_file data/xorqa_data/100k/xorqa_corpus.tsv.q10.docTquery \
        --valid_file data/xorqa_data/100k/xorqa_DSI_dev_data.json \
        --output_dir "models/XORQA-100k-mt5-base-DSI-QG" \
        --learning_rate 0.0005 \
        --warmup_steps 100000 \
        --per_device_train_batch_size 32 \
        --per_device_eval_batch_size 32 \
        --evaluation_strategy steps \
        --eval_steps 1000 \
        --max_steps 1000000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 1000 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 1 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False \
        --metric_for_best_model Hits@10 \
        --greater_is_better True \
        --remove_prompt True
```
