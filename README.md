# RECoT

# Installation

```bash
conda create -n recot python=3.8.0 -y && conda activate recot
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

# Prepare Data

You can download all our processed data by running

```bash
./download/processed_data.sh
```

The data will be downloaded in `processed_data/{dataset_name}/`. If you're just looking for just dev/test data we used in the paper, it's `processed_data/{dataset_name}/{dev|test}_subsampled.jsonl`.

<summary>Follow these steps if you want to generate all processed data from scratch again.</summary>

```bash
# 1. Download raw data:
## raw data will be in raw_data/{dataset_name}/
./download/raw_data.sh

# 2. Process raw data files in a single standard format
## processed data will be in processed_data/{dataset_name}/
python processing_scripts/process_hotpotqa.py
python processing_scripts/process_2wikimultihopqa.py


# 4. Subsample the processed datasets.
## Note (i) dev processing has to be done before test.
## (ii) because of randomness it may create different samples that what we used,
## so consider using the released data if the goal is reproduction.
## (iii) sampled data will be in processed_data/{dataset_name}/{dev|test}_subsampled.jsonl
python processing_scripts/subsample_dataset_and_remap_paras.py hotpotqa dev
python processing_scripts/subsample_dataset_and_remap_paras.py hotpotqa test
python processing_scripts/subsample_dataset_and_remap_paras.py 2wikimultihopqa dev
python processing_scripts/subsample_dataset_and_remap_paras.py 2wikimultihopqa test


# 5. Attach reasoning steps and supporting para annotations
## to the preprocessed (train) data files.
## To do this, you'll set up elasticsearch server, index all dataset corpuses.
## See 'Prepare Retriever and LLM Servers' section in the readme.
python prompt_generator/attach_data_annotations.py hotpotqa
python prompt_generator/attach_data_annotations.py 2wikimultihopqa
```


# Prepare Prompts

All our prompts are available in `prompts/` directory. If you're using these prompts outside of this codebase, note that `# METADATA: ...` lines need to be ignored at runtime from it.

If you want to generate them from scratch, run

```bash
python prompt_generator/generate_prompts.py {dataset_name} --task_name qa # hotpotqa, 2wikimultihopqa
```

Note though that because of random sampling to select distractors, some of the regenerated prompts may be different. So if you're goal is to reproduce the experiments, use the released ones.

# Prepare Retriever and LLM Servers

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


Start the elasticsearch server on port 9200 (default), and then start the retriever server as shown here. You can change the elasticsearch port in `retriever_server/serve.py` if needed.

```bash
uvicorn serve:app --port 8000 --app-dir retriever_server
```

Next, index the wikipedia corpuses for the datasets. Make sure you've downloaded `raw_data` and `processed_data`.

```bash
python retriever_server/build_index.py {dataset_name} # hotpotqa, 2wikimultihopqa
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
- SYSTEM: choose from (`recot`, `recot_qa`)
- MODEL: choose from (`codex`, `flan-t5-xxl`, `flan-t5-xl`, `flan-t5-large`, `flan-t5-base`, `none`)
- DATASET: choose from (`hotpotqa`, `2wikimultihopqa`)
``bash
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
