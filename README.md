# Generating Natural Language Proofs with Verifier-Guided Search

![Task](images/nlproofs.jpg)

Code for the paper:  

[Generating Natural Language Proofs with Verifier-Guided Search](https://arxiv.org/abs/2205.12443)      
Conference on Empirical Methods in Natural Language Processing (EMNLP), 2022  
[Kaiyu Yang](https://yangky11.github.io/), [Jia Deng](https://www.cs.princeton.edu/~jiadeng/), and [Danqi Chen](https://www.cs.princeton.edu/~danqic/)   


## Quick Links

  - [Requirements](#requirements)
  - [Data Preprocessing](#data-preprocessing)
  - [EntailmentBank Experiments](#entailmentbank-experiments)
  - [RuleTaker Experiments](#ruletaker-experiments)
  - [Bugs or Questions](#bugs-or-questions)
  - [Citation](#citation)
  - [Credits](#credits)


## Requirements

1. Download and install [Miniconda Python 3](https://docs.conda.io/en/latest/miniconda.html) (Anaconda should also work).
1. Clone this repo and `cd` into its root.
1. Install Python dependencies: `conda env create -f nlproofs.yaml`. You may need to edit [nlproofs.yaml](./nlproofs.yaml) according to your system, e.g., use a different CUDA version. If you have trouble running the installation command, you may also manually install the packages in [nlproofs.yaml](./nlproofs.yaml) in whatever way that works for you. 
1. Activate the conda environment: `conda activate nlproofs`, and prepend the root of this repo to the `PYTHONPATH` environment variable.

## Data Preprocessing

1. Download the v3_May6_2022 version of [EntailmentBank](https://allenai.org/data/entailmentbank) (MD5: 9cb91896325157cee1f35616be0be179) and unzip it as `./data/entailment_trees_emnlp2021_data_v3/`.  
1. Download the OWA version of [RuleTaker](https://allenai.org/data/proofwriter) (MD5: bf490364bca241bb5ff9f0ab0c78b71a) and unzip it as `./data/proofwriter-dataset-V2020.12.3/`.
1. Run `python check_data.py` to check.
1. Run `python preprocess_ruletaker.py` to preprocess the RuleTaker dataset.


## EntailmentBank Experiments

We use [Lightning CLI](https://pytorch-lightning.readthedocs.io/en/1.6.2/common/lightning_cli.html) to create scripts for training, validation, and testing: [prover/main.py](prover/main.py) and [verifier/main.py](verifier/main.py) for the prover and the verifier, respectively. They take arguments from the command line as well as YAML configuration files. Please run `python main.py --help` or refer to the documentation of [Lightning CLI](https://pytorch-lightning.readthedocs.io/en/1.6.2/common/lightning_cli.html) for details. 

We provide YAML files for our hyperparameters and experimental settings in [./prover/](./prover/) and [./verifier/](./verifier/). We run all experiments on a single NVIDIA A6000 GPU with 48GB memory. For running them on GPUs with smaller memory, you may have to change `batch_size` and `accumulate_grad_batches`. On newer GPUs, `--trainer.precision bf16` may lead to significant speedup and memory savings. I have not tested those features thoroughly, so please use them at your own discretion. Note that [pretrained T5 models do not play well with fp16](https://github.com/huggingface/transformers/issues/10830).


### Training

#### Prover

First, `cd` into [./prover/](./prover). Then run `python main.py fit --help` to see how to use the training script. Below are example commands used in our experiments:
```bash
python main.py fit --config cli_task1_single_shot_t5-large.yaml  # Train a single-shot prover on Task 1 of EntailmentBank.
python main.py fit --config cli_task1_stepwise_t5-large.yaml     # Train a stepwise prover on Task 1 of EntailmentBank.
python main.py fit --config cli_task2_single_shot_t5-large.yaml  # Train a single-shot prover on Task 2 of EntailmentBank.
python main.py fit --config cli_task2_stepwise_t5-large.yaml     # Train a stepwise prover on Task 2 of EntailmentBank.
```

The training script saves hyperparameters, model checkpoints, and other information to `./prover/lightning_logs/EXP_ID/`, where `EXP_ID` is an arbitrary experiment ID that will be printed by the training script.

#### Verifier

First, `cd` into [./verifier/](./verifier). Then run `python main.py fit --help` to see how to use the training script. Below are example commands used in our experiments:
```bash
python main.py fit --config cli_entailmentbank_task1.yaml  # Train a verifier on Task 1 of EntailmentBank.
python main.py fit --config cli_entailmentbank_task2.yaml  # Train a verifier on Task 2 of EntailmentBank.
```

The training script saves hyperparameters, model checkpoints, and other information to `./verifier/lightning_logs/EXP_ID/`.

### Validation and Testing

Once training completes, we use the model checkpoint to predict on the validation and testing data. `cd` into [./prover/](./prover) and run `python main.py validate --help` and `python main.py test --help` to see how to use the script for validation and testing. Assume we have a prover checkpoint `PATH_TO_PROVER_CKPT` and a verifier checkpoint `PATH_TO_VERIFIER_CKPT`, below are example commands:
```bash
python main.py validate --config cli_task2_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT                                                                                                     # Validate the stepwise prover without verifier-guided search on Task 2 of EntailmentBank.
python main.py validate --config cli_task2_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --model.verifier_weight 0.5 --model.verifier_ckpt PATH_TO_VERIFIER_CKPT --model.proof_search true   # Validate NLProofS (stepwise prover + verifier-guided search).
python main.py validate --config cli_task2_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --model.verifier_weight 1.0 --model.verifier_ckpt PATH_TO_VERIFIER_CKPT --model.proof_search true   # Validate NLProofS w/o prover score.
python main.py test --config cli_task2_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --model.verifier_weight 0.5 --model.verifier_ckpt PATH_TO_VERIFIER_CKPT --model.proof_search true       # Test NLProofS (stepwise prover + verifier-guided search).
python main.py test --config cli_task1_single_shot_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT                                                                                                   # Test the single-shot prover on Task 1 of EntailmentBank.
python main.py test --confing cli_task2_single_shot_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --data.path_test ../data/entailment_trees_emnlp2021_data_v3/dataset/task_3/test.jsonl            # Test the single-shot prover (trained on Task 2) on Task 3 of EntailmentBank.
```

Validation and testing results are saved as `./prover/lightning_logs/EXP_ID/results_val.tsv` and `./prover/lightning_logs/EXP_ID/results_test.tsv`. They are the input to the [EntailmentBank's official evaluation code](https://github.com/allenai/entailment_bank/tree/71385b6d7cc42ac394006bc2fe84d5bd1117f9ac) for calculating the evaluation metrics.

### Test Results and Model Checkpoints

Slide right to see download links in the tables below.

#### Task 1

| Model         | Leaves-F1       | Leaves-AllCorrect      | Steps-F1      | Steps-AllCorrect       | Intermediates-F1       | Intermediates-AllCorrect       | Overall-AllCorrect       | Model checkpoints | Validation predictions | Test predictions  | 
| ------------- | -------- | ------- | --------------- | ------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- |
| NLProofS           | 97.6 | 90.0 | 54.8 | 41.8 | 72.0 | 39.7 | 38.2 | [prover](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/stepwise/epoch%3D499-step%3D10500.ckpt), [verifier](https://huggingface.co/kaiyuy/NLProofS/resolve/main/verifier/entailmentbank_task1/epoch%3D49-step%3D36300.ckpt) | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/nlproofs/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/nlproofs/results_test.tsv) |
| Stepwise prover    | 98.8 | 98.5 | 54.8 | 41.5 | 71.9 | 38.5 | 36.8 | The `prover` above       | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/stepwise/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/stepwise/results_test.tsv) |
| Single-shot prover | 98.2 | 82.7 | 51.8 | 40.9 | 66.7 | 36.5 | 34.7 | [prover](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/single_shot/epoch%3D399-step%3D8400.ckpt)               | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/single_shot/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task1/single_shot/results_test.tsv) |

#### Task 2

| Model         | Leaves-F1       | Leaves-AllCorrect      | Steps-F1      | Steps-AllCorrect       | Intermediates-F1       | Intermediates-AllCorrect       | Overall-AllCorrect       | Model checkpoints | Validation predictions | Test predictions  | 
| ------------- | -------- | ------- | --------------- | ------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- |
| NLProofS           | 90.3 | 60.6 | 48.6 | 35.6 | 70.3 | 39.4 | 34.4 | [prover](), [verifier](https://huggingface.co/kaiyuy/NLProofS/resolve/main/verifier/entailmentbank_task2/epoch%3D49-step%3D36300.ckpt) | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/nlproofs/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/nlproofs/results_test.tsv) |
| Stepwise prover    | 90.3 | 57.1 | 48.6 | 35.6 | 70.1 | 38.5 | 33.8 | The `prover` above       | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/stepwise/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/stepwise/results_test.tsv) |
| Single-shot prover | 85.9 | 44.7 | 41.3 | 29.1 | 62.5 | 31.5 | 27.7 | [prover](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/single_shot/epoch%3D399-step%3D8400.ckpt)               | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/single_shot/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task2/single_shot/results_test.tsv) |

#### Task 3

Results on Task 3 are produced by evaluating Task 2 models zero-shot on Task 3 data (by changing `--data.path_val` and `--data.path_test`).

| Model         | Leaves-F1       | Leaves-AllCorrect      | Steps-F1      | Steps-AllCorrect       | Intermediates-F1       | Intermediates-AllCorrect       | Overall-AllCorrect       | Model checkpoints | Validation predictions | Test predictions  | 
| ------------- | -------- | ------- | --------------- | ------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- | ---------------- |
| NLProofS           | 43.9 | 9.1 | 10.6 | 6.8 | 42.4 | 15.9 | 6.8 | Same as Task 2 | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/nlproofs/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/nlproofs/results_test.tsv) |
| Stepwise prover    | 42.8 | 7.4 | 9.3 | 5.9 | 42.1 | 15.0 | 5.9 | Same as Task 2 | [results_val.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/stepwise/results_val.json) | [results_test.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/stepwise/results_test.json) |
| Single-shot prover | 40.5 | 4.4 | 9.1 | 3.8 | 35.3 | 7.9 | 3.8 | Same as Task 2 | [results_val.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/single_shot/results_val.tsv) | [results_test.tsv](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/entailmentbank_task3/single_shot/results_test.tsv) |


## RuleTaker Experiments

### Training

#### Prover

Training on RuleTaker is similar to training on EntailmentBank but with different configuration files. Run the following commands in [./prover/](./prover): 
```bash
python main.py fit --config cli_ruletaker_single_shot_t5-large.yaml  # Train a single-shot prover on D0–D3 of RuleTaker (OWA).
python main.py fit --config cli_ruletaker_stepwise_t5-large.yaml     # Train a stepwise prover on D0–D3 of RuleTaker (OWA).
```

#### Verifier

Training the verifier is also similar. Run the following commands in [./verifier/](./verifier): 
```bash
python main.py fit --config cli_ruletaker.yaml  # Train a verifier on D0–D3 of RuleTaker (OWA).
```

### Validation and Testing

`cd` into [./prover/](./prover). Assume we have a prover checkpoint `PATH_TO_PROVER_CKPT` and a verifier checkpoint `PATH_TO_VERIFIER_CKPT`.
```bash
python main.py validate --config cli_ruletaker_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --model.verifier_weight 0.5 --model.verifier_ckpt PATH_TO_VERIFIER_CKPT --model.proof_search true --trainer.limit_val_batches 1.0  # Validate NLProofS on D0–D3 of RuleTaker (OWA).
python main.py test --config cli_ruletaker_stepwise_t5-large.yaml --ckpt_path PATH_TO_PROVER_CKPT --model.verifier_weight 0.5 --model.verifier_ckpt PATH_TO_VERIFIER_CKPT --model.proof_search true  # Test NLProofS on D0–D3 of RuleTaker (OWA).
```

Note the `--trainer.limit_val_batches 1.0` above. By default, we use only 200 batches for RuleTaker validation (see [./prover/cli_ruletaker_stepwise_t5-large.yaml](prover/cli_ruletaker_stepwise_t5-large.yaml) and [./prover/cli_ruletaker_single_shot_t5-large.yaml](prover/cli_ruletaker_single_shot_t5-large.yaml)), but here we want to use all batches.

Validation and testing results are saved as `./prover/lightning_logs/EXP_ID/results_val.json` and `./prover/lightning_logs/EXP_ID/results_test.json`. Run the following command for final evaluation:
```bash
python evaluate.py ruletaker --path-val PATH_TO_VAL_RESULTS --path-test PATH_TO_TEST_RESULTS
```

### Test Results and Model Checkpoints


| Model         | Answer accuracy       | Proof accuracy     | Model checkpoints | Validation predictions | Test predictions  | 
| ------------- | -------- | -------- | --------------- | ------------- | ----------------- |
| NLProofS           | 99.3 | 99.2 | [prover](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/stepwise/epoch%3D19-step%3D16940.ckpt), [verifier](https://huggingface.co/kaiyuy/NLProofS/resolve/main/verifier/ruletaker/epoch%3D49-step%3D93000.ckpt) | [results_val.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/nlproofs/results_val.json) | [results_test.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/nlproofs/results_test.json) |
| Stepwise prover    | 68.7 | 91.3 | The `prover` above       | [results_val.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/stepwise/results_val.json) | [results_test.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/stepwise/results_test.json) |
| Single-shot prover | 56.3 | 72.6 | [prover](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/single_shot/epoch%3D19-step%3D16680.ckpt)               | [results_val.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/single_shot/results_val.json) | [results_test.json](https://huggingface.co/kaiyuy/NLProofS/resolve/main/prover/ruletaker/single_shot/results_test.json) |


## Bugs or Questions

If you have any questions related to the code or the paper, feel free to email [Kaiyu](https://yangky11.github.io/). If you encounter any problems when using the code, or want to report a bug, you can open an issue. Please try to specify the problem with details so we can help you better and quicker!


## Citation

```bibtex
@inproceedings{yang2022nlproofs,
  title={Generating Natural Language Proofs with Verifier-Guided Search},
  author={Yang, Kaiyu and Deng, Jia and Chen, Danqi},
  booktitle={Conference on Empirical Methods in Natural Language Processing (EMNLP)},
  year={2022}
}
```

## Credits
 
* The code is formatted using [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black).

