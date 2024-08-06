# RECoT
# <h1 align="center"> Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions </h1>

This is the repository for our ACL 2023 paper ["Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions"](https://arxiv.org/abs/2212.10509).

![IRCoT Main Figure](ircot.jpg?raw=true)

# Installation

```bash
conda create -n ircot python=3.8.0 -y && conda activate ircot
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

# Prepare Data

You can download all our processed data by running

```bash
./download/processed_data.sh
```

The data will be downloaded in `processed_data/{dataset_name}/`. If you're just looking for just dev/test data we used in the paper, it's `processed_data/{dataset_name}/{dev|test}_subsampled.jsonl`.

<details>
<summary>Follow these steps if you want to generate all processed data from scratch again.</summary>

```bash
# 1. Download raw data:
## raw data will be in raw_data/{dataset_name}/
./download/raw_data.sh

# 2. Process raw data files in a single standard format
## processed data will be in processed_data/{dataset_name}/
python processing_scripts/process_hotpotqa.py
python processing_scripts/process_2wikimultihopqa.py
python processing_scripts/process_musique.py
python processing_scripts/process_iirc.py

# 4. Subsample the processed datasets.
## Note (i) dev processing has to be done before test.
## (ii) because of randomness it may create different samples that what we used,
## so consider using the released data if the goal is reproduction.
## (iii) sampled data will be in processed_data/{dataset_name}/{dev|test}_subsampled.jsonl
python processing_scripts/subsample_dataset_and_remap_paras.py hotpotqa dev
python processing_scripts/subsample_dataset_and_remap_paras.py hotpotqa test
python processing_scripts/subsample_dataset_and_remap_paras.py 2wikimultihopqa dev
python processing_scripts/subsample_dataset_and_remap_paras.py 2wikimultihopqa test
python processing_scripts/subsample_dataset_and_remap_paras.py musique dev
python processing_scripts/subsample_dataset_and_remap_paras.py musique test
python processing_scripts/subsample_dataset_and_remap_paras.py iirc dev
python processing_scripts/subsample_dataset_and_remap_paras.py iirc test

# 5. Attach reasoning steps and supporting para annotations
## to the preprocessed (train) data files.
## To do this, you'll set up elasticsearch server, index all dataset corpuses.
## See 'Prepare Retriever and LLM Servers' section in the readme.
python prompt_generator/attach_data_annotations.py hotpotqa
python prompt_generator/attach_data_annotations.py 2wikimultihopqa
python prompt_generator/attach_data_annotations.py musique
python prompt_generator/attach_data_annotations.py iirc
```

</details>

You'll also need `raw_data` if you want to build elasticsearch indices and run retriever or odqa systems.

```bash
./download/raw_data.sh
```

The data will be downloaded in `raw_data/{dataset_name}/`.


# Prepare Prompts

All our prompts are available in `prompts/` directory. If you're using these prompts outside of this codebase, note that `# METADATA: ...` lines need to be ignored at runtime from it.

If you want to generate them from scratch, run

```bash
python prompt_generator/generate_prompts.py {dataset_name} --task_name qa # hotpotqa, 2wikimultihopqa, musique, iirc
python prompt_generator/generate_prompts.py iirc --task_name no_context_open_retrieval
```

Note though that because of random sampling to select distractors, some of the regenerated prompts may be different. So if you're goal is to reproduce the experiments, use the released ones.

# Prepare Retriever and LLM Servers

<details>
<summary> First, install Elasticsearch 7.10. </summary>

### Install on Mac (option 1)
```
# source: https://www.elastic.co/guide/en/elasticsearch/reference/current/brew.html
brew tap elastic/tap
brew install elastic/tap/elasticsearch-full # if it doesn't work: try 'brew untap elastic/tap' first: untap>tap>install.
brew services start elastic/tap/elasticsearch-full # to start the server
brew services stop elastic/tap/elasticsearch-full # to stop the server
```

### Install on Mac (option 2)
```
# source: https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-darwin-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-darwin-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.10.2-darwin-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.10.2-darwin-x86_64.tar.gz
cd elasticsearch-7.10.2/
./bin/elasticsearch # start the server
pkill -f elasticsearch # to stop the server
```

### Install on Linux

```
# source: https://www.elastic.co/guide/en/elasticsearch/reference/8.1/targz.html
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.10.2-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.10.2-linux-x86_64.tar.gz
cd elasticsearch-7.10.2/
./bin/elasticsearch # start the server
pkill -f elasticsearch # to stop the server
```

Check out the reference sources if you run into problems installing it.

</details>

Start the elasticsearch server on port 9200 (default), and then start the retriever server as shown here. You can change the elasticsearch port in `retriever_server/serve.py` if needed.

```bash
uvicorn serve:app --port 8000 --app-dir retriever_server
```

Next, index the wikipedia corpuses for the datasets. Make sure you've downloaded `raw_data` and `processed_data`.

```bash
python retriever_server/build_index.py {dataset_name} # hotpotqa, iirc, 2wikimultihopqa, musique
```

After indexing you can check the number of documents in each index by running `curl localhost:9200/_cat/indices`. You should have 4 indices, one for each dataset, called `{dataset}-wikipedia`. Make sure they match up to the statistics given in the paper. You should expect to see the following sizes: HotpotQA (5,233,329), 2WikiMultihopQA (430,225), MuSiQue (139,416), and IIRC (1,882,415).

Next, if you want to use flan-t5-* models, start the llm_server by running:

```bash
MODEL_NAME={model_name} uvicorn serve:app --port 8010 --app-dir llm_server # model_name: flan-t5-xxl, flan-t5-xl, flan-t5-large, flan-t5-base
```

If you want to use openai models (e.g., codex in our experiments), you don't need to start it. In that case, you just need to set the environment variable `OPENAI_API_KEY`.

If you start retriever and/or llm_server on a different host or port, update them in `.retriever_address.jsonnet` and `.llm_server_address.jsonnet` before running retrieval/odqa systems.


# Run Retrieval and ODQA Systems

First, download dataset repositories for official evaluation: `./download/official_eval.sh`.

Next, set the variables:

- SYSTEM: choose from (`ircot`, `ircot_qa`, `oner`, `oner_qa`, `nor_qa`)
- MODEL: choose from (`codex`, `flan-t5-xxl`, `flan-t5-xl`, `flan-t5-large`, `flan-t5-base`, `none`)
- DATASET: choose from (`hotpotqa`, `2wikimultihopqa`, `musique`, `iirc`)

The systems ending with `_qa` are for ODQA and others are for retrieval. The `ircot` and `ircot_qa` are proposed systems and others are baselines (see NoR, OneR in paper). For `oner`, choose model to be `none`, not otherwise.

Now you can run the system using (language) model and dataset of your choice by running:

```bash
./reproduce.sh $SYSTEM $MODEL $DATASET
```

This script runs several things one after the other: instantiating experiment configs with HPs, running predictions for them on the dev set, picking up the best HP, making experiment config with the best HP, running it on the test set, and summarizing the results with mean and std.

If you prefer to have more control, you can also run it step-by-step as follows:


```bash
# Instantiate experiment configs with different HPs and write them in files.
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 1
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 2
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 3
## if you make a change to base_configs, the above steps need to be rerun to
## regenerate instantiated experiment configs (with HPs populated)

# Run experiments for different HPs on dev set
python runner.py $SYSTEM $MODEL $DATASET predict --prompt_set 1
## predict command runs evaluation at the end by default. If you want to run evaluation
## separately after prediction, you can replace predict with evaluate here.

# Show results for experiments with different HPs
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set 1
## Not necessary as such, it'll just show you the results using different HPs in a nice table.

# Pick the best HP and save the config with that HP.
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 1 --best
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 2 --best
python runner.py $SYSTEM $MODEL $DATASET write --prompt_set 3 --best

# Run the experiment with best HP on test set
python runner.py $SYSTEM $MODEL $DATASET predict --prompt_set 1 --best --eval_test --official
python runner.py $SYSTEM $MODEL $DATASET predict --prompt_set 2 --best --eval_test --official
python runner.py $SYSTEM $MODEL $DATASET predict --prompt_set 3 --best --eval_test --official
## predict command runs evaluation at the end by default. If you want to run evaluation
## separately after prediction, you can replace predict with evaluate here.

# Summarize best test results for individual prompts and aggregate (mean +- std) of them)
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set 1 --best --eval_test --official
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set 2 --best --eval_test --official
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set 3 --best --eval_test --official
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set aggregate --best --eval_test --official
## The mean and std in the final command is what we reported in the paper.
```

**DISCLAIMER:** Our Codex-based experiments were done when it was free. Now it has been deprecated. You can do these experiments with other OpenAI completion modes, or other open/commercial models (see notes below). But keep track of the cost, as it may add up quickly doing these experiments.

# Download Predictions

If you do not want to run models but just see the predictions/outputs they generated, you can download them by running

```bash
./download/predictions.sh
```

This will not only download the final best-HP predictions on the test set but also all HP-tuning predictions on the dev set. They will be stored in

```bash
predictions/{system_name}_{model_name}_{dataset_name}____{prompt_set_id}___{hp_descriptor}/
```.

Once done, you can run any of the above given "summary" commands to show a report. Again, the 2 most useful of them are.

```bash
# to show results for all HPs on dev set. Could help guestimate what HPs for a new model:
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set 1
# to show the result of best HP on the test set:
python runner.py $SYSTEM $MODEL $DATASET summarize --prompt_set aggregate --best --eval_test --official
```

The HP-tuning one is particularly useful if want to reproduce any of our experiments with your setup and/or model. Sample output:

```text
****************************************
Experiment Name: ircot_qa_codex_hotpotqa
****************************************
python run.py summarize ircot_qa_codex_hotpotqa --instantiation_scheme ircot_qa --prompt_set 1 --evaluation_path processed_data/hotpotqa/dev_subsampled.jsonl

bm25_retrieval_count  distractor_count         metric_value
0                     2              "1"  60.5 | 64.6 | 61.4 |   100
1                     2              "2"  60.9 | 64.0 | 62.8 |   100
2                     2              "3"  60.1 | 63.7 | 61.2 |   100
3                     4              "1"  55.6 | 58.8 | 57.3 |   100
4                     4              "2"  57.6 | 61.1 | 58.7 |   100
5                     4              "3"  55.3 | 58.8 | 56.7 |   100
6                     6              "1"  55.6 | 59.1 | 56.8 |   100
7                     6              "2"  52.2 | 55.6 | 53.5 |   100
8                     6              "3"  48.9 | 52.0 | 50.2 |   100
9                     8              "1"  53.5 | 56.4 | 54.7 |   100
10                    8              "2"  52.7 | 55.6 | 53.6 |   100
...
```

# Running IRCoT (QA) using a Different Dataset or LLM

Each experiment (system, model, data combination) in this project corresponds to an experiment config in `base_configs/...jsonnet`. Find the experiment closest to your use case and change the model, dataset and related information in it as per your need.

If you've changed the dataset, you'll need to ensure the Elasticsearch index of that name is available (see processing-notes and setting-up-retriever for it).

If you've changed the model, you'll need to ensure that the model of that name is implemented and available in the code. If you want to try out a different OpenAI completion model, it'd just involve configuring the `engine` variable and setting the `model_tokens_limit` in [here](https://github.com/StonyBrookNLP/ircot/blob/main/commaqa/models/gpt3generator.py). Chat-based API isn't readily supported yet, but shouldn't be much work if you're interested. If you're interested in open LLMs, like Llama, MPT, etc, you can set up OpenAI-complaint FastChat server as shown [here](https://github.com/lm-sys/FastChat/blob/main/docs/openai_api.md), and make necessary changes in the base_config/ and you should be good to go.

If you're stuck anywhere in this process, open an issue with your specific choice of data/model, and I can help you to get there.

**NOTE:** If you are trying to reproduce any of our experiments with your code and/or model, note that the choice of HP can make a huge difference in the result. See the sample output here, for example. In some experiments, the difference in scores between best and worst HP is over 20 pts. So do not skip the used HP tuning. If it is getting slow/expensive to try out all HPs, at least you might want to look at our HP exploration summary report above to guestimate what is likely to work the best.

# Acknowledgment

This code is heavily based on [CommaQA](https://github.com/allenai/CommaQA), which provides a way to build complex/multi-step systems involving agents. All modeling-related code for IRCoT project is in `commaqa/inference/ircot.py`, and all experiment configs (without HPs instantiated) for this project are in `base_configs/`.

# Citation

If you find this work useful, consider citing it:

```bib
@inproceedings{trivedi-etal-2023-interleaving,
    title = "Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions",
    author = "Trivedi, Harsh  and
      Balasubramanian, Niranjan  and
      Khot, Tushar  and
      Sabharwal, Ashish",
    editor = "Rogers, Anna  and
      Boyd-Graber, Jordan  and
      Okazaki, Naoaki",
    booktitle = "Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)",
    month = jul,
    year = "2023",
    address = "Toronto, Canada",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2023.acl-long.557",
    doi = "10.18653/v1/2023.acl-long.557",
    pages = "10014--10037",
}
```
