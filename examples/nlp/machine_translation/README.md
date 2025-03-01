# NMT Distillation


This is a guide explaining how to use NMT distillation, including Hinton-style, DistilBERT-style, sequence-level interpolation, and hybrid distillation in NeMo.

# Neural Machine Translation

Let $\bf x=[x_1 \dots x_{|\mathcal{S}|}]$ and $\bf y=[y_1\dots y_{|\mathcal{T}|}]$ be the source and target sentence, respectively. The task of NMT is to model $p(\bf y|\bf x)$, whereby for a encoder-decoder attentional sequence-to-sequence architecture, the encoder transforms the input sequence into a continuous representation. Then the decoder auto-regressively predicts the conditional distribution of each target word conditioned on the source sentence $\bf x$ and all previous decoded target outputs $\bf y_{<t}$ using an attention mechanism in a beam search. For NMT, we minimize NLL on a parallel training set of $N$ sentences:

$
\begin{aligned}
\mathcal{L}_{\text{NLL}}(\theta)&=-\sum_{n=1}^N \log p(\bf y^{(n)}|\bf x^{(n)})\\&=-\sum_{n=1}^N\sum_{t=1}^T\log p(\bf y^{(n)}_t | \bf y_{<t}^{(n)},h_{t-1}^{(n)},\text{Att}(\text{Enc}(\bf x^{(n)}),y^{(n)}_{<t},h_{t-1}^{(n)});\theta)
\end{aligned}
$

where $\bf h_{t-1}^{(n)}$ is the (t-1)-th decoder hidden state and $\bf y_{<t}$ are the preceding decoded words. The attention mechanism computes the context vector as a weighted sum of the encoder outputs $\text{Enc}(\bf x^{(n)})$ where the weights, i.e. attention scores, are computed by computing the similarity between the decoder hidden state at time $t$ and all other encoder hidden states. Modern NMT systems use Transformers (resp. multi-layer Bi-directional RNNs) as encoders.

# Hinton-style World-Level Distillation for NMT [Hinton et al., 2015; Kim & Rush, 2016]

Since NMT models minimize $\mathcal{L}_{\text{WORD-NLL}}$ at each decoder output step, if we have a teacher model, we can use knowledge distillation for multi-class cross entropy as presented in Hinton et al. 2015 where we minimize the cross entropy between the student and teacher distributions:

$$
\mathcal{L}_{\text{WORD-KD}}=-\frac{1}{N}\sum_{n=1}^N \sum_{t=1}^{|\mathcal{T}|}\sum_{k=1}^{|\mathcal{V}|} \bf{q}(y_t^{(n)}=k|\bf x^{(n)}, \bf y_{<t}^{(n)}) \log p(y_t^{{(n)}}=k|x^{(n)}, \bf y_{<t}^{(n)})
$$

for $\mathcal{V}$ the vocabulary (classes) set, $\mathcal{T}$ the set of all possible target sequences, and $N$ the number of examples in the parallel transfer dataset (usually smaller than the original training dataset). In the case of NMT, we use compute a softmax of the logits over the vocabulary to determine this probability, namely (ignoring reference to the time step): 

$$
q_{k}:=q(y=k|\bf x; \theta_T)=\frac{\exp(z_k)}{\sum_{j=1}^{|\mathcal{V}|}\exp(z_j)}
$$
for $z_k$ the teacher's logit for class $k$
and

$$
p_{k}:=p(y=k|\bf x; \theta)=\frac{\exp(w_k)}{\sum_{j=1}^{|\mathcal{V}|}\exp(w_j)}
$$

for $w_k$ the student's logit for class $k$.

Assuming the pre-trained teacher is parameterized by $\theta_T$ and the student is parameterized by $\theta$, then we interpolate between NLL and word-level KD with a mixing parameter $\alpha$:

$$
\mathcal{L}(\theta;\theta_T)=(1-\alpha)\mathcal{L}_{\text{NLL}}(\theta)+\alpha\mathcal{L}_{\text{WORD-KD}}(\theta; \theta_T)
$$

Note, this does not necessarily have to be a convex combination, i.e.

$$
\mathcal{L}(\theta;\theta_T)=\alpha_{\text{NLL}}\mathcal{L}_{\text{NLL}}(\theta)+\alpha_{\text{WORD-KD}}\mathcal{L}_{\text{WORD-KD}}(\theta; \theta_T)
$$

According to Hinton et al. 2015, tempering the student and teacher softmax distributions with a temperature $\tau$ produces a softer distribution over classes. In particular, we transfer the knowledge to the student model by training it with the tempered softmax teacher distribution​ as a target for every example in the transfer set, e.g. the tempered student softmax is:

$$
p_k^{\tau}=p^{\tau}(y=k|\bf x; \theta)=\frac{\exp(w_k/\tau)}{\sum_{j=1}^{|\mathcal{V}|}\exp(w_j/\tau)}
$$

We use a temperature when training the student model, but we must set it to temperature 1 when validating. While you can experiment with various temperatures, for NMT, we generally found temperatures [0.5, 1.0] to work best with $\alpha_{\text{WORD-KD}}=1.0$.

While we have been computing the categorical cross entropy over the target vocabulary between the student's predictions and teacher's pseudo-labels, using KL divergence is a better way to find a student distribution $p$ that is “close” to the teacher distribution $q$ becauase it does not require the degenerate one-hot encoded target distribution like cross entropy. In particular, "relative probabilities of incorrect answers tell us a lot about
how the [teacher] model tends to generalize" which KL divergence takes into account. In particular, let $\mathcal{X}$ be the probability space over which distributions $p$ and $q$ are defined, then the KL divergence (or relative entropy) is:

$$
D_{\text{KL}}(p||q)=\sum_{x\in\mathcal{X}}p(x)\log q(x)\\
=\sum_{x\in X}p(x)\log p(x)-\sum_{x\in\mathcal{X}}p(x)\log q(x)\\
=\mathcal{H}(p,q)-H(p)
$$
which is a metricized notion of distance between the distributions which has an information-theoretic interpretation, for $\mathcal{H}(\cdot,\cdot)$ cross entropy and $\mathcal{H}(\cdot)$ entropy.

Since we are given the training dataset, entropy $H(p)$ is constant so instead of minimizing cross entropy, we minimize the KL divergence between the tempered teacher probabilities and tempered student probabilities such that:

$$
\mathcal{L}_{\text{WORD-KD}}(\theta;\theta_T)=D_{\text{KL}}(q^{\tau}||p^{\tau}).
$$

# EXAMPLE

Suppose you have a pretrained model, say in `/nemo_models/teacher_24_6_de_en/AAYNBase.nemo`. You may train a 3x3 layer encoder-decoder DE->EN student NMT model with word-level distillation using $\alpha_{\text{NLL}}=1.0$ and $\alpha_{\text{WORD-LEVEL}}=1.0$ on WMT21 and validated on WMT13, WMT14, WMT18, WMT19, WMT20 as follows (note, in your own training using tarred training datasets is highly recommended). Label-smoothing is not recommended when training the teacher or student when performing distillation [Muller et al. 2019] Important: your student tokenizer must be the same as your teacher tokenizer, otherwise you will get errors so it is safest to specify it:

```
# Hyperparams
TOKENS_IN_BATCH=8000
LEARNING_RATE=4e-4
STEPS=150000
WARMUP_STEPS=15000

# Distillation
DISTILLATION_LOSS_WEIGHT=1.0
STUDENT_TRAIN_LOSS_WEIGHT=0.0
COSINE_LOSS_WEIGHT=0.0
TEMPERATURE=1.0

python enc_dec_nmt_distillation.py \
	--config-path=conf \
	--config-name=aayn_base_distillation \
	do_training=true \
	trainer.gpus=1 \
	~trainer.max_epochs \
	+trainer.max_steps=${STEPS} \
	+trainer.val_check_interval=1000 \
	+trainer.accumulate_grad_batches=1 \
	model.src_language=de \
	model.tgt_language=en \
	model.beam_size=4 \
	model.max_generation_delta=512 \
	model.label_smoothing=0.0 \
	model.encoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.decoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.encoder.hidden_size=256 \
	model.encoder.inner_size=1024 \
	model.encoder.num_attention_heads=4 \
	model.encoder.num_layers=3 \
	model.encoder.ffn_dropout=0.1 \
	model.encoder.pre_ln=true \
	model.encoder_tokenizer.vocab_size=32000 \
	model.decoder_tokenizer.vocab_size=32000 \
	model.decoder.pre_ln=true \
	model.decoder.num_layers=3 \
	model.decoder.hidden_size=256 \
	model.decoder.inner_size=1024 \
	model.decoder.num_attention_heads=4 \
	model.decoder.ffn_dropout=0.1 \
	model.train_ds.shard_strategy=scatter \
	model.train_ds.use_tarred_dataset=true \
    model.train_ds.tar_files=[/tarred_data/wmt21_tarred_dataset/parallel.batches.tokens.8000._OP_0..4697_CL_.tar] \
    model.train_ds.metadata_file=[/tarred_data/wmt21_tarred_dataset/metadata.tokens.8000.json] \
	model.train_ds.tokens_in_batch=${TOKENS_IN_BATCH} \
	model.validation_ds.src_file_name=[/data/newstest2020-en-de.clean.tok.ref,/data/newstest2019-en-de.clean.tok.ref,/data/newstest2018-en-de.clean.tok.ref,/data/newstest2014-en-de.clean.tok.ref,/data/newstest2013-en-de.clean.tok.ref] \
	model.validation_ds.tgt_file_name=[/data/newstest2020-en-de.clean.tok.src,/data/newstest2019-en-de.clean.tok.src,/data/newstest2018-en-de.clean.tok.src,/data/newstest2014-en-de.clean.tok.src,/data/newstest2013-en-de.clean.tok.src] \
	~model.test_ds \
	model.optim.lr=$LEARNING_RATE \
	+model.optim.sched.warmup_steps=$WARMUP_STEPS \
  	~model.optim.sched.warmup_ratio \
	model.distillation.model_path=/nemo_models/teacher_24_6_de_en/AAYNBase.nemo \
	model.distillation.distillation_loss_weight=${DISTILLATION_LOSS_WEIGHT} \
	model.distillation.student_train_loss_weight=${STUDENT_TRAIN_LOSS_WEIGHT} \
    model.distillation.cosine_loss_weight=${COSINE_LOSS_WEIGHT} \
	model.distillation.temperature=${TEMPERATURE} \
	model.distillation.distill_encoder=True \
    model.distillation.distilbert_initialization=False \
	+exp_manager.explicit_log_dir=/results \
	+exp_manager.resume_if_exists=True \
	+exp_manager.resume_ignore_no_checkpoint=True \
	+exp_manager.create_checkpoint_callback=True \
	+exp_manager.checkpoint_callback_params.monitor=val_sacreBLEU \
	+exp_manager.checkpoint_callback_params.save_top_k=1 \
	+exp_manager.checkpoint_callback_params.mode=max \
	+exp_manager.checkpoint_callback_params.always_save_nemo=True
```

# DistilBERT-style Distillation [Chaumond et al., 2020]

In 'DistilBERT, a distilled version of BERT' [Chaumond et al., 2020], the student model was instantiated from the student encoder-decder by sampling 1 of every n layers in the BERT encoder. We implement a similar initialization where we sample 1 of every $n_{\text{enc}}$ layers from the encoder and 1 of every $n_{\text{dec}}$ layers from the decoder, where the number of teacher encoder (resp. decoder) layers must be divisible by the number of student encoder (resp. decoder layers). For example, if we go from a 24x6 teacher->3x3 student: sample 1 every 8 from the encoder & 1 every 2 from the decoder​. We use a triple loss linear combination of negative log-likelihood loss, knowledge distillation loss, and cosine embedding loss given by:

$$
\mathcal{L}=\alpha_{\text{NLL}}\mathcal{L}_{\text{NLL}}+\alpha_{\text{KD}}\mathcal{L}_{\text{KD}}+\alpha_{\cos}\mathcal{L}_{\cos}
$$ 
where $\mathcal{L}_{\cos}(\bf h_s, \bf h_t)=1-\frac{\bf h_s\cdot \bf h_t}{||\bf h_s||||\bf h_t||}$ is the cosine embedding loss between the hidden states vectors of student and teacher models. You can activate this DistilBERT-style activation by merely making `COSINE_LOSS_WEIGHT` non-zero setting and `model.distillation.distilbert_initialization=True` if you want DistilBERT initialization. When we ran ablation studies investigating the effect of each of these losses, it was found that while the word-level KD and NLL substantially improved performance, the cosine embedding loss did not substantively improve performance. We do not recommend DistilBERT initialization for NMT, but for other applications such as language modeling, its effect could be different.

# Sequence-level Distillation [Kim & Rush, 2016]

While standard word-level distillation allows us to transfer the local word-level distributions, it is better for the students to learn the teacher's knowledge at the sequence-level. The sequence-level distribution over all possible sequences $\bf y\in\mathcal{T}$ is:

$$
p(\bf y|\bf x)=\prod_{t=1}^{|\mathcal{T}|}p(y_t|\bf x, \bf y_{<t}).
$$

For sequence-level knowledge distillation, we use the teacher distribution $q(\bf y|\bf x)$ over all possible sequences, to compute the cross entropy loss 

$$
\mathcal{L}_{\text{SEQ-KD}}=-\sum_{\bf y\in\mathcal{T}}\bf{q}(\bf y|\bf x)\log p(\bf y|\bf x)
$$

or, respectively, the KL-divergence loss:

$$
\mathcal{L}_{\text{SEQ-KD}}=D_{\text{KL}}(\bf{q}(\bf y|\bf x)|| p(\bf y|\bf x)).
$$

However, this sum involves exponentially-many terms, so we may approximate the teacher distribution with its mode:

$$
q(\bf y|\bf x)\sim \mathbb{1}\{\bf y=\argmax_{y \in \mathcal{T}} q(\bf y|\bf x)\}.
$$

Finding the mode is also expensive so we approximate it with beam search:

$$
\mathcal{L}_{\text{SEQ-KD}}\approx -\sum_{t\in\mathcal{T}}\mathbb{1}\{\bf y=\hat{\bf y}\}\log p(\bf y|\bf x)\\
=-\log p(\bf y = \hat{\bf y}| \bf x)
$$
for $\hat{\bf y}$ the output of beam search. The pipeline for sequence-level distillation is straightforward:
1. Train the teacher model.
2. Use the teacher to run beam search over the transfer dataset (which can be the same as the training dataset) to get a teacher-generated translation dataset. Note, we can even get predictions on a monolingual dataset so you can use unlabeled data for this step if you would like.

For this step, use the pre-trained teacher to run beam search over the transfer dataset to get a teacher-generated "soft translation" dataset via `translate_dpp.py` on NGC. You should comment out the `model.replace_beam_with_sampling(topk=args.topk)` line if you wish to use beam search, otherwise it will default to top-k sampling. Note, while you may want use greedy-sampling or top-k sampling because it is more efficient, in our experiments, this did not perform as well as the beam search method. 

Below, I am specifying a path to the teacher-tokenized WMT21 dataset since we will use the German monolingual data as our transfer dataset (but, the transfer and training sets need not match):

```
#!/bin/bash
INSTANCE=dgx1v.32g.8.norm
PROJECT=backtranslation-de-en-wmt21
DATAID=84118
WORKSPACE=wmt_translate_models
set -e
ngc batch run --name "translation_de_en_wmt21" --preempt RUNONCE \
    --image "nvcr.io/nvidia/pytorch:21.03-py3" \
    --ace nv-us-west-2 \
    --instance $INSTANCE \
    --commandline "export DEBIAN_FRONTEND=noninteractive && nvidia-smi && apt-get update && apt-get install -y libsndfile1 ffmpeg && \
    pip install wandb==0.10.21 && pip install Cython && wandb login ${WANDBLOGIN} && \
    git clone https://github.com/NVIDIA/NeMo.git && cd NeMo && \
    git checkout main && ./reinstall.sh && \
    cp -R /data/* /raid/ && \
    python examples/nlp/machine_translation/translate_ddp.py \
        --model=/raid/nemo_models/teacher_24_6_de_en/AAYNBase.nemo \
        --text2translate=/raid/wmt21_de_en_yttm_tokens_8000/parallel.batches.tokens.8000._OP_0..4194_CL_.tar \
        --src_language de \
        --tgt_language en \
        --metadata_path /raid/wmt21_de_en_yttm_tokens_8000/metadata.tokens.8000.json \
        --twoside \
        --result_dir /results" \
    --result /results/ \
    --org nvidian \
    --team ac-aiapps \
    --datasetid $DATAID:/data/ \
    --workspace $WORKSPACE:/models/
```

Once you obtain the translations across GPU ranks, first concatenate the `originals.txt` and `translations.txt` as follows:

```
cat rank*/originals.txt > originals.txt
cat rank*/translations.txt > translations_all.txt
```

Next, you need to perform tokenization and punctuation normalization: 

```
cat originals.txt | perl mosesdecoder/scripts/tokenizer/normalize-punctuation.perl -l de | perl mosesdecoder/scripts/tokenizer/tokenizer.perl -l de -no-rescape -threads 12 > originals.de.clean.tok.txt
cat translations_all.txt | perl mosesdecoder/scripts/tokenizer/normalize-punctuation.perl -l en | perl mosesdecoder/scripts/tokenizer/tokenizer.perl -l en -no-rescape -threads 12 > translations_all.en.clean.tok.txt
```

Then use the `create_tarred_parallel_dataset.py` to create the tarred data set of teacher-generated translations `beam_teacher_predictions_de_en_8k_tokens`:

```
python create_tarred_parallel_dataset.py \
    --src_fname data/originals.de.clean.tok.txt \
    --tgt_fname data/translations_all.en.clean.tok.txt \
    --out_dir data/beam_teacher_predictions_de_en_8k_tokens \
    --clean \
    --encoder_tokenizer_name yttm \
    --encoder_tokenizer_model=/raid/wmt21_de_en_yttm_tokens_8000/shared_tokenizer.32000.BPE.model \
    --decoder_tokenizer_model=/raid/wmt21_de_en_yttm_tokens_8000/shared_tokenizer.32000.BPE.model \
    --encoder_tokenizer_vocab_size 32000 \
    --decoder_tokenizer_name yttm \
    --decoder_tokenizer_vocab_size 32000 \
    --max_seq_length 512 \
    --min_seq_length 1 \
    --tokens_in_batch 8000 \
    --num_batches_per_tarfile 100
```

3. Train the student with NLL on this teacher-generated dataset:

```
python /code/examples/nlp/machine_translation/enc_dec_nmt_distillation.py \
	--config-path=conf \
	--config-name=aayn_base_distillation \
	do_training=true \
	trainer.num_nodes=${SLURM_JOB_NUM_NODES} \
	trainer.gpus=${SLURM_GPUS_PER_NODE} \
	~trainer.max_epochs \
	+trainer.max_steps=${STEPS} \
	+trainer.val_check_interval=1000 \
	+trainer.accumulate_grad_batches=1 \
	model.src_language=de \
	model.tgt_language=en \
	model.beam_size=4 \
	model.max_generation_delta=512 \
	model.label_smoothing=0.0 \
	model.encoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.decoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.encoder.hidden_size=1024 \
	model.encoder.inner_size=4096 \
	model.encoder.num_attention_heads=16 \
	model.encoder.num_layers=3 \
	model.encoder.ffn_dropout=0.1 \
	model.encoder.pre_ln=true \
	model.encoder_tokenizer.vocab_size=32000 \
	model.decoder_tokenizer.vocab_size=32000 \
	model.decoder.pre_ln=true \
	model.decoder.num_layers=3 \
	model.decoder.hidden_size=1024 \
	model.decoder.inner_size=4096 \
	model.decoder.num_attention_heads=16 \
	model.decoder.ffn_dropout=0.1 \
	model.train_ds.use_tarred_dataset=true \
	model.train_ds.tar_files=/tarred_data/beam_teacher_predictions_de_en_8k_tokens/parallel.batches.tokens.8000._OP_0..4660_CL_.tar \
	model.train_ds.metadata_file=/tarred_data/beam_teacher_predictions_de_en_8k_tokens/metadata.tokens.8000.json \
	model.train_ds.shard_strategy=scatter \
	model.train_ds.tokens_in_batch=${TOKENS_IN_BATCH} \
	model.validation_ds.src_file_name=[/data/newstest2020-en-de.clean.tok.ref,/data/newstest2019-en-de.clean.tok.ref,/data/newstest2018-en-de.clean.tok.ref,/data/newstest2014-en-de.clean.tok.ref,/data/newstest2013-en-de.clean.tok.ref] \
	model.validation_ds.tgt_file_name=[/data/newstest2020-en-de.clean.tok.src,/data/newstest2019-en-de.clean.tok.src,/data/newstest2018-en-de.clean.tok.src,/data/newstest2014-en-de.clean.tok.src,/data/newstest2013-en-de.clean.tok.src] \
	~model.test_ds \
	model.optim.lr=$LEARNING_RATE \
	+model.optim.sched.warmup_steps=$WARMUP_STEPS \
  	~model.optim.sched.warmup_ratio \
	model.distillation.model_path=/nemo_models/teacher_24_6_de_en/AAYNBase.nemo \
	model.distillation.distillation_loss_weight=${DISTILLATION_LOSS_WEIGHT} \
	model.distillation.student_train_loss_weight=${STUDENT_TRAIN_LOSS_WEIGHT} \
    model.distillation.cosine_loss_weight=${COSINE_LOSS_WEIGHT} \
	model.distillation.temperature=${TEMPERATURE} \
	model.distillation.distill_encoder=True \
    model.distillation.distilbert_initialization=False \
	+exp_manager.explicit_log_dir=/results \
	+exp_manager.resume_if_exists=True \
	+exp_manager.resume_ignore_no_checkpoint=True \
	+exp_manager.create_checkpoint_callback=True \
	+exp_manager.checkpoint_callback_params.monitor=val_sacreBLEU \
	+exp_manager.checkpoint_callback_params.save_top_k=1 \
	+exp_manager.checkpoint_callback_params.mode=max \
	+exp_manager.checkpoint_callback_params.always_save_nemo=True
```

# Sequence-Level Interpolation

We can train the student on a mixture of the original training dataset ($\mathcal{L}_{\text{NLL}}$) and the teacher generated dataset ($\mathcal{L}_{\text{SEQ-KD}}$):

$$
\mathcal{L}=\alpha_{\text{SEQ-NLL}}\mathcal{L}_{\text{SEQ-NLL}}+\alpha_{\text{SEQ-KD}}\mathcal{L}_{\text{KD}}\\
=-\alpha_{\text{SEQ-NLL}}\sum_{t\in\mathcal{T}}q(\bf y|\bf x)\log p(\bf y|\bf x)
$$
which is called sequence-level interpolation. Using the beam search modal approximation, this is given by:

$$
\mathcal{L}=-\alpha_{\text{SEQ-NLL}}\log \bf{p}(\bf y|\bf x)-\alpha_{\text{KD}} p(\hat{\bf y}|\bf x).
$$

Like sequence-level distillation, we need to generate the teacher-generated dataset to get pseudo-labels. Then we augment the training dataset with this teacher-generated data by sampling examples from datasets with different probabilities (which is the same as example weighting in the limit that we train for a long time). This added noise from the pseudo-labels helps SGD optimization. We can also view this as a form of regulairzation and, in the case of NMT, there are many ways to translate a source sentence so this added noise signals that there are many valid translations, thereby reducing the chance the model overfits. We run sequence-level interpolation on the WMT21 dataset and teacher-generated translations of WMT21 as follows: 

```
# Hyperparams
TOKENS_IN_BATCH=8000
LEARNING_RATE=4e-4
STEPS=150000
WARMUP_STEPS=15000

# Distillation
DISTILLATION_LOSS_WEIGHT=1.0
STUDENT_TRAIN_LOSS_WEIGHT=1.0
COSINE_LOSS_WEIGHT=0.0
TEMPERATURE=1.0

python enc_dec_nmt_distillation.py \
	--config-path=conf \
	--config-name=aayn_base_distillation \
	do_training=true \
	trainer.gpus=1 \
	~trainer.max_epochs \
	+trainer.max_steps=${STEPS} \
	+trainer.val_check_interval=1000 \
	+trainer.accumulate_grad_batches=1 \
	model.src_language=de \
	model.tgt_language=en \
	model.beam_size=4 \
	model.max_generation_delta=512 \
	model.label_smoothing=0.0 \
	model.encoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.decoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.encoder.hidden_size=1024 \
	model.encoder.inner_size=4096 \
	model.encoder.num_attention_heads=16 \
	model.encoder.num_layers=3 \
	model.encoder.ffn_dropout=0.1 \
	model.encoder.pre_ln=true \
	model.encoder_tokenizer.vocab_size=32000 \
	model.decoder_tokenizer.vocab_size=32000 \
	model.decoder.pre_ln=true \
	model.decoder.num_layers=3 \
	model.decoder.hidden_size=1024 \
	model.decoder.inner_size=4096 \
	model.decoder.num_attention_heads=16 \
	model.decoder.ffn_dropout=0.1 \
	model.train_ds.use_tarred_dataset=true \
	model.train_ds.tar_files=[/tarred_data/beam_teacher_predictions_de_en_8k_tokens/parallel.batches.tokens.8000._OP_0..4660_CL_.tar,/tarred_data/wmt21_de_en_yttm_tokens_8000/parallel.batches.tokens.8000._OP_0..4194_CL_.tar] \
	model.train_ds.metadata_file=[/tarred_data/beam_teacher_predictions_de_en_8k_tokens/metadata.tokens.8000.json,/tarred_data/wmt21_de_en_yttm_tokens_8000/metadata.tokens.8000.json] \
	model.train_ds.shard_strategy=scatter \
	model.train_ds.concat_sampling_technique=random \
	model.train_ds.concat_sampling_probabilities=[${TEACHER_PREDICTIONS_PROB},${GROUNDTRUTH_DATASET_PROB}] \
	model.train_ds.tokens_in_batch=${TOKENS_IN_BATCH} \
	model.validation_ds.src_file_name=[/data/newstest2020-en-de.clean.tok.ref,/data/newstest2019-en-de.clean.tok.ref,/data/newstest2018-en-de.clean.tok.ref,/data/newstest2014-en-de.clean.tok.ref,/data/newstest2013-en-de.clean.tok.ref] \
	model.validation_ds.tgt_file_name=[/data/newstest2020-en-de.clean.tok.src,/data/newstest2019-en-de.clean.tok.src,/data/newstest2018-en-de.clean.tok.src,/data/newstest2014-en-de.clean.tok.src,/data/newstest2013-en-de.clean.tok.src] \
	~model.test_ds \
	model.optim.lr=$LEARNING_RATE \
	+model.optim.sched.warmup_steps=$WARMUP_STEPS \
  	~model.optim.sched.warmup_ratio \
	model.distillation.model_path=/nemo_models/teacher_24_6_de_en/AAYNBase.nemo \
	model.distillation.distillation_loss_weight=${DISTILLATION_LOSS_WEIGHT} \
	model.distillation.student_train_loss_weight=${STUDENT_TRAIN_LOSS_WEIGHT} \
    model.distillation.cosine_loss_weight=${COSINE_LOSS_WEIGHT} \
	model.distillation.temperature=${TEMPERATURE} \
	model.distillation.distill_encoder=True \
    model.distillation.distilbert_initialization=False \
	+exp_manager.create_wandb_logger=True \
	+exp_manager.wandb_logger_kwargs.name=${EXPNAME} \
	+exp_manager.wandb_logger_kwargs.project=${PROJECT} \
	+exp_manager.explicit_log_dir=/results \
	+exp_manager.resume_if_exists=True \
	+exp_manager.resume_ignore_no_checkpoint=True \
	+exp_manager.create_checkpoint_callback=True \
	+exp_manager.checkpoint_callback_params.monitor=val_sacreBLEU \
	+exp_manager.checkpoint_callback_params.save_top_k=1 \
	+exp_manager.checkpoint_callback_params.mode=max \
	+exp_manager.checkpoint_callback_params.always_save_nemo=True
```

# Hybrid Distillation

The so-called "hybrid distillation" combines sequence-level interpolation and Hinton-style word-level distillation as follows:

$
\begin{aligned}
\mathcal{L}&=\alpha_{\text{SEQ-NLL}}\mathcal{L}_{\text{SEQ-NLL}}+\alpha_{\text{SEQ-KD}}\mathcal{L}_{\text{SEQ-KD}}+\alpha_{\text{WORD-KD}}\mathcal{L}_{\text{WORD-KD}}\\
&= -\alpha_{\text{SEQ-NLL}}\log \bf{p}(\bf y|\bf x)-\alpha_{\text{SEQ-KD}}\sum_{\bf y\in\mathcal{T}}q(\bf y|\bf x)\log p(\bf y|\bf x)+\alpha_{\text{WORD-KD}}D_{\text{KL}}(q||p)
\end{aligned}
$

which can be approximated with beam search as:

$$
\mathcal{L}\approx-\alpha_{\text{SEQ-NLL}}\log \bf{p}(\bf y|\bf x)-\alpha_{\text{SEQ-KD}} p(\hat{\bf y}|\bf x)+\alpha_{\text{WORD-KD}}D_{\text{KL}}(q||p).
$$

# Pipeline

The hybrid distillation pipeline is illustrated in the Figure below.

![pipeline](https://user-images.githubusercontent.com/35356586/131922482-852f4131-c4ef-470a-baf4-6dc16a38334e.png)

1. First, we obtain the teacher-generated translations dataset by using the teacher to run beam search on the (possibly unlabeled) transfer dataset as we did in previous sections. 

2. Then we combine the teacher-generated translations with the (labeled) parallel dataset using sampling, as before.

3. Finally, we train the student with NLL and word-level KD loss on this augmented transfer dataset:

```
# Hyperparams
TOKENS_IN_BATCH=8000
LEARNING_RATE=4e-4
STEPS=150000
WARMUP_STEPS=15000

# Distillation
DISTILLATION_LOSS_WEIGHT=1.0
STUDENT_TRAIN_LOSS_WEIGHT=1.0
COSINE_LOSS_WEIGHT=0.0
TEMPERATURE=1.0

# Mixing probabilities
TEACHER_PREDICTIONS_PROB=0.34
GROUNDTRUTH_DATASET_PROB=0.66

python enc_dec_nmt_distillation.py \
	--config-path=conf \
	--config-name=aayn_base_distillation \
	do_training=true \
	trainer.gpus=1 \
	~trainer.max_epochs \
	+trainer.max_steps=${STEPS} \
	+trainer.val_check_interval=1000 \
	+trainer.accumulate_grad_batches=1 \
	model.src_language=de \
	model.tgt_language=en \
	model.beam_size=4 \
	model.max_generation_delta=512 \
	model.label_smoothing=0.0 \
	model.encoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.decoder_tokenizer.tokenizer_model=/tokenizer/shared_tokenizer.32000.BPE.model \
	model.encoder.hidden_size=1024 \
	model.encoder.inner_size=4096 \
	model.encoder.num_attention_heads=16 \
	model.encoder.num_layers=3 \
	model.encoder.ffn_dropout=0.1 \
	model.encoder.pre_ln=true \
	model.encoder_tokenizer.vocab_size=32000 \
	model.decoder_tokenizer.vocab_size=32000 \
	model.decoder.pre_ln=true \
	model.decoder.num_layers=3 \
	model.decoder.hidden_size=1024 \
	model.decoder.inner_size=4096 \
	model.decoder.num_attention_heads=16 \
	model.decoder.ffn_dropout=0.1 \
	model.train_ds.use_tarred_dataset=true \
	model.train_ds.tar_files=[/tarred_data/beam_teacher_predictions_de_en_8k_tokens/parallel.batches.tokens.8000._OP_0..4660_CL_.tar,/tarred_data/wmt21_de_en_yttm_tokens_8000/parallel.batches.tokens.8000._OP_0..4194_CL_.tar] \
	model.train_ds.metadata_file=[/tarred_data/beam_teacher_predictions_de_en_8k_tokens/metadata.tokens.8000.json,/tarred_data/wmt21_de_en_yttm_tokens_8000/metadata.tokens.8000.json] \
	model.train_ds.shard_strategy=scatter \
	model.train_ds.concat_sampling_technique=random \
	model.train_ds.concat_sampling_probabilities=[${TEACHER_PREDICTIONS_PROB},${GROUNDTRUTH_DATASET_PROB}] \
	model.train_ds.tokens_in_batch=${TOKENS_IN_BATCH} \
	model.validation_ds.src_file_name=[/data/newstest2020-en-de.clean.tok.ref,/data/newstest2019-en-de.clean.tok.ref,/data/newstest2018-en-de.clean.tok.ref,/data/newstest2014-en-de.clean.tok.ref,/data/newstest2013-en-de.clean.tok.ref] \
	model.validation_ds.tgt_file_name=[/data/newstest2020-en-de.clean.tok.src,/data/newstest2019-en-de.clean.tok.src,/data/newstest2018-en-de.clean.tok.src,/data/newstest2014-en-de.clean.tok.src,/data/newstest2013-en-de.clean.tok.src] \
	~model.test_ds \
	model.optim.lr=$LEARNING_RATE \
	+model.optim.sched.warmup_steps=$WARMUP_STEPS \
  	~model.optim.sched.warmup_ratio \
	model.distillation.model_path=/nemo_models/teacher_24_6_de_en/AAYNBase.nemo \
	model.distillation.distillation_loss_weight=${DISTILLATION_LOSS_WEIGHT} \
	model.distillation.student_train_loss_weight=${STUDENT_TRAIN_LOSS_WEIGHT} \
    model.distillation.cosine_loss_weight=${COSINE_LOSS_WEIGHT} \
	model.distillation.temperature=${TEMPERATURE} \
	model.distillation.distill_encoder=True \
    model.distillation.distilbert_initialization=False \
	+exp_manager.explicit_log_dir=/results \
	+exp_manager.resume_if_exists=True \
	+exp_manager.resume_ignore_no_checkpoint=True \
	+exp_manager.create_checkpoint_callback=True \
	+exp_manager.checkpoint_callback_params.monitor=val_sacreBLEU \
	+exp_manager.checkpoint_callback_params.save_top_k=1 \
	+exp_manager.checkpoint_callback_params.mode=max \
	+exp_manager.checkpoint_callback_params.always_save_nemo=True
```


# Low Data Regime
Hinton et. al proposed that "soft targets allow the student to generalize well from only 3% of the training set". Using only 5% of ground labels and pseudo labels (with a 1:2 mixture) at a temperature of 1 with hybriid distillation, we saw similar performance to training on the entire transfer dataset and teacher-generated dataset. This leads us to propose the heuristic illustrated below, whereby you can use a large (unlabeled) monolingual corpus to get pseudo-labels from the teacher. Then you can concatenate this with a small (labeled) parallel dataset to obtain a "noisy" parallel dataset. Then we can train the student on this augmented dataset with word-level (or sequence-level NLL since they are equivalent) as well as word-level distillation using the teacher model.

![low_data](https://user-images.githubusercontent.com/35356586/131950223-a224f112-50d7-44df-87a6-47615801a70b.png)
