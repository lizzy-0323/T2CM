# T2CM
## Structuring Meaningful Translation Between Programming Languages
### Dataset: 
Oda Y, Fudaba H, Neubig G, et al. Learning to generate pseudo-code from source code using statistical machine translation. 2015 30th IEEE/ACM International Conference on Automated Software Engineering (ASE).015;574-584.doi:10.1109/ASE.2015.36.
### Python library dependencies:
OpenNMT-py



# Quickstart

### Step 0: Install OpenNMT-py

```bash
pip install --upgrade pip
pip install OpenNMT-py
```

### Step 1: Prepare the data

To get started, we propose to download a toy English-German dataset for machine translation containing 10k tokenized sentences:

```bash
wget https://s3.amazonaws.com/opennmt-trainingdata/toy-ende.tar.gz
tar xf toy-ende.tar.gz
cd toy-ende
```

The data consists of parallel source (`src`) and target (`tgt`) data containing one sentence per line with tokens separated by a space:

* `src-train.txt`
* `tgt-train.txt`
* `src-val.txt`
* `tgt-val.txt`

Validation files are used to evaluate the convergence of the training. It usually contains no more than 5k sentences.

```text
$ head -n 2 toy_ende/src-train.txt
It is not acceptable that , with the help of the national bureaucracies , Parliament &apos;s legislative prerogative should be made null and void by means of implementing provisions whose content , purpose and extent are not laid down in advance .
Federal Master Trainer and Senior Instructor of the Italian Federation of Aerobic Fitness , Group Fitness , Postural Gym , Stretching and Pilates; from 2004 , he has been collaborating with Antiche Terme as personal Trainer and Instructor of Stretching , Pilates and Postural Gym .
```

We need to build a **YAML configuration file** to specify the data that will be used:

```yaml
# toy_en_de.yaml

## Where the samples will be written
save_data: toy-ende/run/example
## Where the vocab(s) will be written
src_vocab: toy-ende/run/example.vocab.src
tgt_vocab: toy-ende/run/example.vocab.tgt
# Prevent overwriting existing files in the folder
overwrite: False

# Corpus opts:
data:
    corpus_1:
        path_src: toy-ende/src-train.txt
        path_tgt: toy-ende/tgt-train.txt
    valid:
        path_src: toy-ende/src-val.txt
        path_tgt: toy-ende/tgt-val.txt
...

```

From this configuration, we can build the vocab(s), that will be necessary to train the model:

```bash
onmt_build_vocab -config toy_en_de.yaml -n_sample 10000
```

**Notes**:
- `-n_sample` is required here -- it represents the number of lines sampled from each corpus to build the vocab.
- This configuration is the simplest possible, without any tokenization or other *transforms*. See [other example configurations](https://github.com/OpenNMT/OpenNMT-py/tree/master/config) for more complex pipelines.

### Step 2: Train the model

To train a model, we need to **add the following to the YAML configuration file**:
- the vocabulary path(s) that will be used: can be that generated by onmt_build_vocab;
- training specific parameters.

```yaml
# toy_en_de.yaml

...

# Vocabulary files that were just created
src_vocab: toy-ende/run/example.vocab.src
tgt_vocab: toy-ende/run/example.vocab.tgt

# Train on a single GPU
world_size: 1
gpu_ranks: [0]

# Where to save the checkpoints
save_model: toy-ende/run/model
save_checkpoint_steps: 500
train_steps: 1000
valid_steps: 500

```

Then you can simply run:

```bash
onmt_train -config toy_en_de.yaml
```

This configuration will run the default model, which consists of a 2-layer LSTM with 500 hidden units on both the encoder and decoder. It will run on a single GPU (`world_size 1` & `gpu_ranks [0]`).

Before the training process actually starts, the `*.vocab.pt` together with `*.transforms.pt` can be dumped to `-save_data` with configurations specified in `-config` yaml file by enabling the `-dump_fields` and `-dump_transforms` flags. It is also possible to generate transformed samples to simplify any potentially required visual inspection. The number of sample lines to dump per corpus is set with the `-n_sample` flag.

For more advanded models and parameters, see [other example configurations](https://github.com/OpenNMT/OpenNMT-py/tree/master/config) or the [FAQ](FAQ).
### Step 3:Change the yaml file to run T2CM
To run T2CM model, you should use the following standard configure,and change the parameters to get a better score on valid set.

```yaml
...

#### General opts
save_model: foo
save_checkpoint_steps: 10000
valid_steps: 10000
train_steps: 200000

#### Batching
queue_size: 10000
bucket_size: 32768
world_size: 4
gpu_ranks: [0, 1, 2, 3]
batch_type: "tokens"
batch_size: 4096
valid_batch_size: 8
max_generator_batches: 2
accum_count: [4]
accum_steps: [0]

#### Optimization
model_dtype: "fp32"
optim: "adam"
learning_rate: 2
warmup_steps: 8000
decay_method: "noam"
adam_beta2: 0.998
max_grad_norm: 0
label_smoothing: 0.1
param_init: 0
param_init_glorot: true
normalization: "tokens"

#### Model
encoder_type: transformer
decoder_type: transformer
position_encoding: true
enc_layers: 6
dec_layers: 6
heads: 8
rnn_size: 512
word_vec_size: 512
transformer_ff: 2048
dropout_steps: [0]
dropout: [0.1]
attention_dropout: [0.1]
 ``` 
Here are what the most important parameters mean:

* `param_init_glorot` & `param_init 0`: correct initialization of parameters;
* `position_encoding`: add sinusoidal position encoding to each embedding;
* `optim adam`, `decay_method noam`, `warmup_steps 8000`: use special learning rate;
* `batch_type tokens`, `normalization tokens`: batch and normalize based on number of tokens and not sentences;
* `accum_count 4`: compute gradients based on four batches;
* `label_smoothing 0.1`: use label smoothing loss.

### Step 4: Translate

```bash
onmt_translate -model toy-ende/run/model_step_1000.pt -src toy-ende/src-test.txt -output toy-ende/pred_1000.txt -gpu 0 -verbose
```

Now you have a model which you can use to predict on new data. We do this by running beam search. This will output predictions into `toy-ende/pred_1000.txt`.

