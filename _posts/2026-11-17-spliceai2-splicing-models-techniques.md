---
layout: post
title: "From SpliceAI to SpliceAI2: the technical toolbox of modern sequence-to-splicing models"
date: 2026-11-17 00:00:00
description: "A machine learning view of SpliceAI2 and recent splicing models (SpliceAI, Pangolin, DeltaSplice, Splam, SpliceTransformer, AlphaGenome, OpenSpliceAI), focusing on context length, output hierarchy, splice graphs with dynamic programming, target design, tissue conditioning, and variant scoring."
tags: ["RNA splicing", "SpliceAI2", "sequence-to-function", "variant effect prediction"]
categories: [RNA-biology, predictive models]
---

RNA splicing is one of the cleanest examples of a <span style="color: blue;">sequence-to-function problem</span> in biology. The input is a stretch of genomic DNA. The output is the set of introns that get removed, how often each one is removed, and which full-length mRNA isoforms are produced. Everything in between, including the spliceosome, hundreds of RNA-binding proteins, co-transcriptional kinetics and chromatin, is hidden from the model.

Since SpliceAI (2019), a whole family of deep learning models has tried to learn this mapping directly from sequence. In October 2026, Illumina released **SpliceAI2**, described in the manuscript *"A unified framework for quantitative splicing and transcript prediction"* (Jaganathan, Chen, Liu, Zhang et al.). On paper it is a large step: it reads about 200 kb of sequence, predicts splice site usage, splice junction usage and **complete transcripts**, and it is trained on 314,745 RNA-seq samples from ten mammalian species.

The question this post asks is not only:

```text
How accurate is SpliceAI2?
```

but rather:

```text
Which technical choices make a modern splicing model work,
and how do SpliceAI2 and its recent peers differ in those choices?
```

I first set up the prediction problem, then walk through the lineage of recent models, and then take SpliceAI2 apart component by component: backbone, output heads, the splice graph and its dynamic programming loss, training targets, optimization, variant scoring and tissue conditioning. At the end I summarize the shared design space and what still seems to be missing.

---

## 1. Splicing prediction as a hierarchy of outputs

For a gene with genomic sequence $x \in \{A,C,G,T\}^L$, splicing can be described at several levels of resolution. Each level is a different prediction target.

**Level 1: splice site identity.** For every position $p$, is it a donor (5' splice site), an acceptor (3' splice site), or neither?

$$
y_p \in \{\text{none}, \text{donor}, \text{acceptor}\}
$$

**Level 2: splice site usage.** How often is position $p$ used as a donor or acceptor?

$$
u^{d}_p, \; u^{a}_p \in [0, 1]
$$

**Level 3: splice junction usage.** How often is the intron from donor $i$ to acceptor $j$ removed?

$$
J_{ij} \in [0,1], \quad i < j
$$

**Level 4: transcript usage.** What fraction of the gene's mRNA molecules follow a specific intron chain $T$?

$$
p(T), \quad \sum_{T} p(T) = 1
$$

These levels are not independent. A junction needs a used donor and a used acceptor. A transcript is a compatible chain of non-overlapping junctions. Most of the history of splicing models can be read as a <span style="color: blue;">climb up this hierarchy</span>:

```text
SpliceAI (2019)          → level 1 (site identity)
Pangolin, DeltaSplice    → level 2 (site usage, quantitative)
Splam, AlphaGenome       → level 3 (junctions, pairwise)
SpliceAI2 (2026)         → level 4 (full transcripts), trained end-to-end on levels 2–4
```

For variant interpretation, every level gives a *delta*. If $f$ is the model and $x^{\text{ref}}, x^{\text{alt}}$ are the reference and alternate sequences, the variant effect is

$$
\Delta = f(x^{\text{alt}}) - f(x^{\text{ref}})
$$

computed at the relevant positions, pairs, or transcripts. The finer the level, the closer the prediction is to the biological consequence (for example a frameshift transcript or an exon skip) rather than a proxy for it.

---

## 2. The lineage: what each recent model contributed

Before going into SpliceAI2, it helps to see what each recent model added to the toolbox. I focus on technical ideas rather than leaderboard numbers.

### 2.1 SpliceAI (2019): long context with dilated residual convolutions

SpliceAI established the basic recipe:

* one-hot encoded DNA input;
* a deep stack of **residual blocks with dilated 1D convolutions**, so the receptive field grows quickly with depth;
* the largest model (SpliceAI-10k) sees 5,000 nt on each side of each position;
* a per-position **3-class softmax** (none / acceptor / donor);
* labels from annotated GENCODE transcripts;
* a variant **delta score**: the largest change in donor or acceptor probability within a ±50 nt window.

Its central lesson was that <span style="color: blue;">long-range context matters</span>. Splice site choice depends on exon definition, intron length, competing sites and distal regulatory elements, not only on the local 9-mer or 23-mer motifs used by MaxEntScan.

Its central limitation was that it learned **annotation**, a binary property, rather than **usage**, a quantitative property. A rarely used cryptic site and a constitutive site both look like "1" or "0".

### 2.2 Pangolin (2022): quantitative and tissue-aware

Pangolin kept a SpliceAI-like dilated convolutional backbone but changed the targets:

* trained on RNA-seq from four species (human, rhesus macaque, mouse, rat) and four tissues (heart, liver, brain, testis);
* per tissue, it predicts both the probability that a site is spliced and its **splice site usage** (SSU), a continuous value;
* tissues are handled as **separate output heads** in a multitask setup.

The new tools are *RNA-seq-derived quantitative targets* and *multi-species training as data augmentation*. Orthologous genes across mammals share a splicing code but differ in sequence, which acts like natural mutagenesis.

### 2.3 DeltaSplice (2024): bigger context and reference-informed prediction

DeltaSplice extended the receptive field to about 30 kb, with a residual CNN core and roughly an order of magnitude more parameters than SpliceAI. Its distinctive idea is a **dual-sequence mode**. To predict usage for a mutated sequence, the model also receives the **reference sequence together with its measured usage**, and predicts the target relative to that anchor.

```text
single mode:  x            → usage
dual mode:    (x_ref, u_ref, x_alt) → usage(x_alt)
```

This is a form of <span style="color: blue;">conditioning on a known baseline</span>. Variant effect prediction becomes closer to predicting a difference than predicting two absolute values and subtracting them.

### 2.4 Splam (2024): thinking in pairs

Splam, from the Salzberg and Pertea groups, is built for spliced alignment rather than variant scoring, but it made an important conceptual move. Instead of scoring donors and acceptors independently, it scores **a donor and an acceptor together** as a junction. The input is a short window around each end of the intron, concatenated, passed through a residual convolutional network. This allows Splam to filter spurious junctions from RNA-seq alignments.

The tool here is **pairwise prediction**. A junction is a property of two positions, not one.

### 2.5 SpliceTransformer (2024): attention for tissue-specific splicing

SpliceTransformer (SpTransformer) adds self-attention on top of convolutional features to predict **tissue-specific** splice site usage across many tissues, and applies the model to ClinVar variants to link tissue-specific splicing changes to disease. It represents the trend of moving from pure CNNs to hybrid conv–attention models for long sequences.

### 2.6 AlphaGenome (2025–2026): splicing inside a general sequence-to-function model

AlphaGenome from Google DeepMind is not a splicing-specific model. It takes **1 Mb** of DNA and predicts thousands of genomic tracks (expression, chromatin accessibility, histone marks, contact maps, and more) with a U-Net-like architecture: convolutional downsampling, a transformer at coarse resolution, and upsampling back to base-pair resolution.

For splicing, it has three relevant output types:

* splice site classification;
* splice site usage;
* **splice junctions**, predicted by a dedicated pairwise head that combines embeddings of candidate donor and acceptor positions and outputs junction read counts per tissue.

So AlphaGenome also reaches level 3 of the hierarchy, and it brings massive multimodal supervision. For variant scoring, its documentation recommends a **composite score** that adds the splice site, splice site usage and scaled junction effects.

### 2.7 OpenSpliceAI (2025): retrainable infrastructure

OpenSpliceAI is a PyTorch reimplementation of SpliceAI with a modular training pipeline. It reproduces SpliceAI's accuracy and makes it practical to retrain on new species or annotations. Its contribution is **infrastructure**, not architecture. This matters because many of the later gains come from data and targets, which only help if retraining is cheap.

A side note: Kuan-Hao Chao, first author of Splam and OpenSpliceAI, is also a co-author of SpliceAI2. The pairwise-junction idea and the retrainable-pipeline idea both reappear in SpliceAI2.

---

## 3. SpliceAI2 at a glance

SpliceAI2 combines most of the ideas above and adds one new piece: a **structured transcript layer**. At a high level:

```text
196,608 bp DNA + species one-hot
        ↓
dense projection to 256 channels
        ↓
20 multi-scale MetaFormer blocks (depthwise dilated conv + FFN)
        ↓                                   ↓
per-block splice site heads          per-block query/key heads
(averaged → sigmoid)                 on top-1,024 candidate sites
        ↓                                   ↓
donor/acceptor usage                 pairwise junction usage
over central 65,536 bp               (1,024 × 1,024, upper triangle)
        ↓                                   ↓
        └───────────── splice graph ────────┘
                         ↓
       dynamic programming (forward algorithm / Viterbi)
                         ↓
         transcript distribution and top-k isoforms
```

Key numbers from the methods:

| Component | SpliceAI2 setting |
| --- | --- |
| Input length | 196,608 bp (65,536 bp of flank on each side of the predicted window) |
| Output window | central 65,536 positions |
| Input channels | 4 (nucleotide) + 10 (species one-hot) |
| Backbone | 20 MetaFormer blocks, 256 channels |
| Candidate splice sites for junctions | top 1,024 positions |
| Parameters | about 13M |
| Short-read data | 314,745 RNA-seq samples, 10 mammals, ~67M junctions before filtering |
| Long-read data | 330 ENCODE samples (189 human, 141 mouse) |
| Training | 50 epochs, Muon + AdamW, bf16, 4×H100, ensemble of 2 |

Notice the parameter count. At about 13M parameters, SpliceAI2 is <span style="color: blue;">small by modern standards</span>, much smaller than AlphaGenome. Most of the improvement comes from data scale, target design and output structure, not from model size.

---

## 4. The backbone: a multi-scale convolutional MetaFormer

### 4.1 MetaFormer blocks

A MetaFormer is the abstract transformer skeleton with the attention layer replaced by any "token mixer":

$$
h' = h + \mathrm{TokenMixer}(\mathrm{Norm}(h)), \qquad
h'' = h' + \mathrm{ChannelMixer}(\mathrm{Norm}(h'))
$$

In SpliceAI2:

* **Token mixer**: batch normalization, then a **depthwise convolution** with kernel size 5 and a dilation rate that **doubles every two blocks**;
* **Channel mixer**: batch normalization, then a feed-forward network with expansion ratio 4 and ReLU.

This is essentially a ConvNeXt-style block for 1D sequences. Depthwise convolutions mix information only along the sequence, and the FFN mixes only across channels. That separation is cheaper than a dense convolution and works well with the transformer-style residual layout.

### 4.2 Multi-scale processing: pooling inside each block

The second ingredient is less common. Each block is **preceded by average pooling** and **followed by linear upsampling** back to base-pair resolution, and the **pooling factor doubles every four blocks**.

```text
block b:  h  → AvgPool(factor_b) → MetaFormer block → Upsample(factor_b) → h_b
factor_b: 1,1,1,1, 2,2,2,2, 4,4,4,4, 8,8,8,8, 16,16,16,16   (doubling every 4 blocks)
```

(The schedule above is illustrative. The paper states that the factor doubles every four blocks, but does not give the starting value.)

The dilation also doubles every two blocks. The two mechanisms multiply, so the deep blocks see very long ranges at low cost. Pooling by 16 means one convolution tap spans 16 bp, and with a large dilation a kernel of size 5 can span thousands of bases.

Why not a transformer over 196 kb? Self-attention over about $2 \times 10^5$ tokens at base resolution is very expensive. AlphaGenome solves this by downsampling heavily before attention. SpliceAI2 avoids global attention altogether in the backbone. <span style="color: blue;">Splicing is dominated by local motifs plus a few long-range dependencies</span>, which suits a convolution-only design with an exponentially growing receptive field.

### 4.3 Deep supervision through per-block heads

The splice site output is not read from the last layer only. **Each of the 20 blocks** has its own dense head. The outputs are cropped to the central 65,536 positions and averaged:

$$
\hat{u}_p = \sigma\left( \frac{1}{20} \sum_{b=1}^{20} W_b \, h_b(p) \right) \in [0,1]^2
$$

where the two channels are donor and acceptor usage.

This has several effects:

* shallow blocks capture local motif strength (the GT/AG dinucleotides, polypyrimidine tract, branch point);
* deep blocks capture long-range context (exon definition, competing sites);
* averaging in logit space makes the final prediction a **sum of evidence across scales**;
* every block receives a direct gradient, which stabilizes training of a 20-block network.

The same per-block averaging is used in the junction head.

---

## 5. The junction head: attention scores as edges

### 5.1 Selecting candidate sites

A 65,536 × 65,536 junction matrix would have more than 4 billion entries. SpliceAI2 avoids it by first selecting the **1,024 positions** with the highest predicted splice site usage, taking the maximum of donor and acceptor usage, and sorting them by coordinate. Junctions are only predicted among these candidates.

### 5.2 Bilinear scoring with genomic RoPE

For each block $b$, the features at the 1,024 candidate positions are layer-normalized and projected to queries and keys. **Rotary position embeddings (RoPE)** are applied using the **genomic coordinates** of the sites, not their rank among candidates:

$$
s^{(b)}_{ij} = \frac{\left\langle \mathrm{RoPE}(q_b(h_b(i)), \, x_i), \; \mathrm{RoPE}(k_b(h_b(j)), \, x_j) \right\rangle}{\sqrt{d}}
$$

The lower triangle including the diagonal is masked, so only $i < j$ (upstream donor, downstream acceptor on the sense strand) survives. The junction usage is

$$
\hat{J}_{ij} = \sigma\left( \frac{1}{20} \sum_{b=1}^{20} s^{(b)}_{ij} \right)
$$

This is attention without the softmax and without values. The query–key dot product is used directly as an **edge score**. RoPE makes the score depend on the relative distance $x_j - x_i$, which is exactly the intron length. Intron length is a strong biological prior for junction choice: very short introns are disfavored, and exon definition operates over typical exon sizes.

Note the **sigmoid**, not a softmax. Each junction is an independent Bernoulli-style prediction of usage. A donor can be paired with several acceptors at different levels, which is how alternative splicing appears in the data.

### 5.3 Teacher forcing for candidate selection

During training, the 1,024 candidate sites are taken from the **ground-truth** splice site usage, not from predictions. The paper gives two reasons:

* early in training, predictions are arbitrary, and candidates chosen from them would be meaningless;
* it avoids ever materializing the full junction matrix.

This is the same logic as teacher forcing in sequence models: <span style="color: blue;">decouple the selection step from the learning step</span> until the model is good enough to select for itself.

---

## 6. The splice graph and dynamic programming: the key new idea

### 6.1 Transcripts as paths

SpliceAI2 combines site and junction predictions into a **splice graph**:

* nodes are the candidate splice sites, ordered along the gene;
* a directed edge $i \to j$ represents an intron from donor $i$ to acceptor $j$;
* consecutive sites are also joined by an **exon continuation** edge with weight 0.

For implementation, node weights are folded into edges. The weight of an intron edge is the sum of three logits:

$$
w_{ij} = \mathrm{logit}(\hat{u}^{d}_i) + \mathrm{logit}(\hat{J}_{ij}) + \mathrm{logit}(\hat{u}^{a}_j)
$$

A transcript $T$ is a path through the graph: a chain of non-overlapping introns separated by exonic stretches. Its score is

$$
s(T) = \sum_{(i,j) \in T} w_{ij}
$$

and the model's transcript distribution is a Gibbs distribution over all paths:

$$
p(T) = \frac{e^{s(T)}}{Z}, \qquad Z = \sum_{T' \in \mathcal{T}} e^{s(T')}
$$

This is a **linear-chain / semi-Markov conditional random field** over splice sites, with neural potentials. It connects deep splicing models back to classical gene finders such as GENSCAN and to recent hybrid models like Tiberius and Helixer, which combine neural networks with HMMs.

### 6.2 Training with the forward algorithm

Long-read RNA-seq gives an observed transcript distribution $q(T)$. The transcript loss is cross-entropy:

$$
\mathcal{L}_{\text{tx}} = -\sum_{T} q(T) \log p(T) = -\sum_{T} q(T) \, s(T) + \log Z
$$

The first term only involves observed transcripts, so it is cheap. The second term sums over an **exponentially large** set of paths. It is computed exactly with a forward recursion. Let $\alpha_j$ be the total exponentiated score of all paths through the first $j$ sites:

$$
\alpha_{j+1} = \underbrace{\alpha_j}_{\text{exon continuation}} + \sum_{i \le j} \alpha_i \, e^{w_{i,j+1}}
$$

This takes $O(N^2)$ time in the number of candidate sites, which is affordable for $N = 1{,}024$. Because every step is differentiable, the gradient of $\log Z$ flows back through the junction head and the splice site head into the backbone. This means <span style="color: blue;">transcript-level supervision shapes site-level and junction-level predictions</span>.

In practice, the transcript loss is weighted by $10^{-5}$ because $\log Z$ is numerically large compared with the per-position BCE losses.

### 6.3 Decoding with Viterbi and k-best paths

At inference, the most likely transcript is found by replacing the sum with a max:

$$
\beta_{j+1} = \max\left( \beta_j, \; \max_{i \le j} \left( \beta_i + w_{i,j+1} \right) \right)
$$

with back-pointers to recover the path. Replacing the max with a **k-max** gives the $k$ best transcripts in one pass, which is how the paper shows the top three isoforms of a held-out gene (NOL12).

On held-out genes on chromosomes 21 and 22, SpliceAI2 exactly reconstructs the most abundant transcript for **82%** of genes. Without long-read supervision, the figure is **78%**; with the site output alone (the "1D" setting in Fig. 1c), it is **77%**. The headline is that short-read data already gets most of the way, because good site and junction potentials already imply good paths. Long reads add global consistency.

### 6.4 Why this matters

Previous models give you site and junction scores, and building isoforms from them is left to the user, usually with heuristics. SpliceAI2 makes the isoform an **explicit latent structure with a proper likelihood**. This makes several things possible:

* exact marginal probabilities of transcripts;
* a principled way to combine short-read (local) and long-read (global) supervision in one objective;
* direct prediction of the transcript consequence of a variant, for example an exon skip or intron retention that leads to a frameshift.

In my view, this is the most important technical contribution of the paper.

---

## 7. Training targets: normalization and the cube root

### 7.1 From junction counts to usage

The short-read targets come from STAR junction counts summed across all samples of a species. Within each gene:

* donor usage at $p$: sum of counts of all junctions using $p$ as donor;
* acceptor usage at $p$: sum of counts of all junctions using $p$ as acceptor;
* both are divided by the **largest such count in the gene**;
* junction counts are divided by the **same** value.

So usage is relative to the most-used site of the gene, not a PSI with a local denominator. This makes the targets comparable across a gene and keeps the site and junction outputs on the same scale.

### 7.2 Cube root: making rare sites count

All usage values are then passed through a **cube root**:

$$
\tilde{u} = u^{1/3}
$$

| raw usage $u$ | target $u^{1/3}$ |
| --- | --- |
| 0.001 | 0.10 |
| 0.01 | 0.22 |
| 0.1 | 0.46 |
| 1.0 | 1.00 |

Without the transform, a site used 0.1% of the time contributes almost nothing to a BCE loss. After it, it becomes a clear signal at 0.1. The ablation in the paper attributes much of SpliceAI2's gain to these **rarely used sites**. Their characterization depends on the very deep coverage obtained by pooling 314,745 samples, and their contribution to training depends on the cube root.

This links directly to the variant problem. A cryptic splice site activated by a rare variant often looks like a site that the reference sequence uses at a very low level. <span style="color: blue;">Learning the low-usage tail of the reference transcriptome is learning the space of cryptic sites.</span>

### 7.3 Losses

The full pretraining objective is

$$
\mathcal{L} = \mathcal{L}_{\text{BCE}}^{\text{site}} + \mathcal{L}_{\text{BCE}}^{\text{junction}} + 10^{-5} \, \mathcal{L}_{\text{tx}}
$$

with BCE against **soft targets** (the cube-rooted usage), restricted to positions within genes.

---

## 8. Data and optimization

### 8.1 Data curation at scale

The short-read data come from recount3 (human, mouse) and the NCBI SRA (eight other mammals: chimpanzee, baboon, two macaques, cow, sheep, pig, rat). Two details stand out:

* projects were classified as bulk, single-cell or ambiguous by **Llama 3.3**, using their titles and abstracts, and only bulk RNA-seq was kept. This is a nice example of using an LLM as a cheap metadata curator;
* reads were mapped with STAR using **annotation-free** indices, so junction discovery is not biased toward known annotation.

Junctions were then restricted to protein-coding genes, required at least one annotated end, and required the gene to reach at least 1,000 junction reads at its top junction.

Long-read data come from all ENCODE long-read samples. Isoforms were merged by **intron chain**, because the model only predicts splicing, not transcription start or polyadenylation sites.

### 8.2 Species as an input, not a head

The species is provided as a one-hot **input** channel, broadcast along the sequence. This differs from Pangolin's per-species output heads. The backbone sees which genome it is reading and can adjust, while all parameters remain shared.

### 8.3 Optimizer and schedule

* **Muon** for feed-forward weight matrices, **AdamW** for everything else;
* learning rate $10^{-3}$, dropped to $10^{-4}$ after 40 of 50 epochs, weight decay 0.01;
* bfloat16 mixed precision, gradient checkpointing, batch size 4 per device on 4 H100 GPUs;
* initialization from $\mathcal{N}(0, 0.01^2)$, biases at zero;
* two models trained identically form the final ensemble.

Muon orthogonalizes the update of each weight matrix (via Newton–Schulz iterations) and has become popular in LLM training. Seeing it in a genomics model is a sign that <span style="color: blue;">genomic deep learning is now borrowing optimization tricks directly from the LLM stack</span>.

### 8.4 Avoiding leakage

Human chromosomes 21 and 22 are held out. For the transcript reconstruction benchmark, the authors train **human-only** models, because orthologous genes from other species would leak the held-out human gene structure. This is a subtle point: multi-species training is fine for variant effects, but it can inflate reference-sequence benchmarks.

---

## 9. Variant scoring: getting the details right

Scoring a variant sounds simple: run reference and alternate, subtract. With a candidate-selection step, it is not.

If the reference and alternate alleles were scored independently, and each ensemble member chose its own top 1,024 sites, the junction matrices would not be aligned. A variant that creates a new site would put it in one candidate set but not the other.

SpliceAI2's solution:

1. run both alleles (196,608 bp centred on the variant) through the backbone of both ensemble members;
2. average donor and acceptor usage over the ensemble;
3. select **one shared candidate set**: the 1,024 positions with the largest usage, maximized over donor/acceptor **and over both alleles**;
4. compute junctions for both alleles on these shared sites, averaged over the ensemble;
5. report donor gain/loss, acceptor gain/loss and junction gain/loss, keeping the ten largest effects per category.

The **summary score** is the maximum across donor gain, donor loss, acceptor gain and acceptor loss. Interestingly, junction gain/loss effects were found to give **no net improvement** to the summary score and are kept only for visualization. Even though the model predicts junctions well, the site-level delta remains the most robust single number for prioritization.

---

## 10. Tissue-specific splicing: conditioning on RBP expression

### 10.1 Tissue as a biological signature, not an ID

Pangolin and AlphaGenome represent tissues as output heads or tracks. Each tissue gets its own parameters. SpliceAI2 instead fine-tunes a model that takes the tissue as an **input vector**: the expression of **147 RNA-binding proteins** (79 from RBPmap, 68 from oRNAment), as log2 TMM-CPM from GTEx, divided by 20 and broadcast along the sequence.

$$
f_\theta(x, \; r) \rightarrow \text{tissue-specific junction usage}, \qquad r \in \mathbb{R}^{147}
$$

This changes the meaning of "tissue". It becomes a **point in a regulator space**, and the model learns how sequence motifs interact with regulator abundance. It also means the model can, in principle, be queried with *any* RBP vector, including disease states or perturbations that it never saw as named tissues.

### 10.2 Architecture of the fine-tuning model

The pretrained MetaFormer blocks are kept, and two kinds of modules are added:

**Signature injection modules**, before each of the 20 blocks:

* **bidirectional FiLM (bi-FiLM)**: the RBP signature modulates the sequence features through feature-wise scale and shift, and the sequence stream also modulates the signature stream;
* a convolutional block with batch norm and a **squeeze-and-excitation** gate;
* the result is scaled by a **learnable scalar** and added residually to the sequence embedding.

The learnable residual scale makes the injection start near zero, so fine-tuning begins from the pretrained model's behaviour.

**A three-layer cross-attention decoder**:

* queries are sequence embeddings at **anchor splice sites**, plus a learned tissue embedding;
* RoPE with $\theta = 10^5$ (long wavelengths for kilobase distances), linear relative position biases and **QK-normalization**;
* output logits are soft-capped as $10 \tanh(x/10)$ for stability;
* for each anchor, partners on the wrong side (upstream of a donor, downstream of an acceptor) are masked, and a **softmax over positions** gives the distribution of junction usage for that anchor.

Note the switch from sigmoid to softmax. In pretraining, junction usage is normalized per gene. In the tissue model, it is **anchor-based**: for a given donor, what fraction of its junctions go to each acceptor? This is essentially PSI3/PSI5, the quantity that changes between tissues.

### 10.3 Losses for differences, not only values

Fine-tuning uses three losses:

* cross-entropy on anchor-based junction usage;
* weighted BCE for donor/acceptor classification;
* an **MSE contrastive loss on pairwise tissue differences**: the predicted difference between tissues A and B should match the measured difference.

The third loss is important. Most junctions are constitutive and identical across tissues. A model can achieve a low absolute error by ignoring tissues entirely. A loss on differences forces the model to <span style="color: blue;">spend capacity on the part of the signal that is actually tissue-specific</span>.

### 10.4 What it can and cannot do

Across 48 GTEx tissues and about 14.7 million tissue-pair comparisons on held-out chromosomes, predicted and observed differences correlate at Pearson $r = 0.44$, rising to 0.58 for cerebellum vs blood. Two technical findings are worth noting:

* **fine-tuning from the pretrained model clearly beats training from scratch**, so the shared splicing code transfers;
* the RBP signature only **marginally** beats a one-hot tissue ID. The biological representation is more interpretable and more flexible, but not much more accurate on GTEx.

The flexibility is shown by **counterfactual conditioning**. With identical DNA input, swapping in the RBP vector of SF3B1-mutant vs wild-type uveal melanoma, or of myotonic dystrophy at different MBNL1 levels, reproduces disease-associated splicing differences in held-out genes. In silico mutagenesis recovers RBFOX, QKI, HNRNPA2B1 and MBNL1 motifs, and the strength of these motifs tracks the expression of the corresponding RBP across tissues.

The honest limitation: for **tissue-specific variant effects**, the tissue model adds nothing beyond knowing the reference splicing levels in each tissue, the same conclusion reached by ABSplice. In other words, the model knows *which tissues use a junction differently*, but not yet *how a variant's effect depends on tissue* beyond that.

---

## 11. Benchmarks

All numbers below are from the SpliceAI2 manuscript, with essential splice dinucleotide variants excluded. They are reported by the developers; independent evaluation will matter.

| Benchmark | Metric | SpliceAI | Pangolin | AlphaGenome | SpliceAI2 |
| --- | --- | --- | --- | --- | --- |
| GTEx GT/AG cryptic sites (4,545 vs 573,967) | auPRC | 0.66 | 0.56 | 0.54 | **0.77** |
| GTEx GT/AG cryptic usage | Spearman ρ | 0.45 | 0.47 | 0.41 | **0.63** |
| GTEx sQTL fine-mapped vs null | auROC | 0.71 | 0.70 | 0.73 | **0.76** |
| GTEx sQTL effect size | Spearman ρ | 0.65 | 0.66 | 0.68 | **0.73** |
| OpenSplice MPRA (283,121 SNVs) | Spearman ρ | 0.55 | 0.53 | 0.49 | **0.59** |
| GEL GT/AG cryptic sites | auPRC | 0.48 | 0.40 | 0.36 | **0.59** |
| UK Biobank plasma protein | Pearson r | −0.463 | −0.475 | −0.481 | **−0.504** |

Some observations:

* the largest gain is on **cryptic site detection**, especially for deep intronic variants, which fits the cube-root and data-scale explanation in Section 7;
* AlphaGenome is strongest among baselines on the **sQTL** tasks, which involve common, often regulatory, variants;
* on the **MPRA**, all models are closer together, suggesting that exon-skipping in reporter constructs is a different and harder regime;
* AlphaGenome scores depended strongly on scorer configuration (40 configurations tested). A single generalist model exposes many possible scores, and choosing one is part of the evaluation.

In the Genomics England rare disease cohort, SpliceAI2-prioritized cryptic splice variants account for about **15% of the excess genetic burden**, and half of them lie more than 50 bp into introns, outside typical exome capture.

---

## 12. The shared design space

Placing all the models side by side, the choices fall along a few axes.

| Axis | SpliceAI | Pangolin | DeltaSplice | Splam | AlphaGenome | SpliceAI2 |
| --- | --- | --- | --- | --- | --- | --- |
| Context | 10 kb | 10 kb | ~30 kb | short windows | 1 Mb | ~197 kb |
| Backbone | dilated res-CNN | dilated res-CNN | res-CNN | res-CNN | U-Net + transformer | multi-scale conv MetaFormer |
| Highest output level | site | site usage | site usage | junction | junction | **transcript** |
| Targets | annotation | RNA-seq usage | RNA-seq usage | annotation / alignments | RNA-seq tracks | usage$^{1/3}$ + long-read isoforms |
| Species | human | 4 | 8 | human (+ others) | human, mouse | 10 (as input) |
| Tissue handling | none | output heads | none | none | output tracks | RBP-signature conditioning (fine-tune) |
| Structured output | none | none | none | pairwise | pairwise | splice graph + DP |

From this, I see six recurring techniques.

### 12.1 Long context, cheaply

Every model grows the receptive field with an exponential mechanism: dilation, pooling, or both. Only AlphaGenome uses attention over the long axis, and only at reduced resolution. <span style="color: blue;">For splicing, local resolution matters more than global attention</span>; the needed long-range signal is sparse.

### 12.2 Quantitative targets, reshaped

The move from annotation labels to RNA-seq usage was the first big step. The second was **reshaping** those targets (normalization by gene maximum, cube root) so that rare events carry gradient. Target engineering is as important as architecture.

### 12.3 Pairwise heads for junctions

Splam, AlphaGenome and SpliceAI2 all represent junctions as functions of **two positions**. SpliceAI2's version (top-k candidates + bilinear Q·K with genomic RoPE) is a light, scalable way to do it.

### 12.4 Structured prediction on top of neural potentials

SpliceAI2 brings back the **CRF/HMM view of gene structure**, now with learned potentials and end-to-end training. This idea is also appearing in gene finding (Tiberius, Helixer). It turns a set of local scores into a globally consistent object.

### 12.5 Context as an input rather than a head

Species one-hot and RBP signatures both turn context into an **input**. This favours a shared code with context-dependent modulation, matching the biology of a common spliceosome tuned by trans-acting factors, and it allows counterfactual queries.

### 12.6 Careful variant scoring

A shared candidate set across alleles, ensemble averaging before selection, and simple max-delta summary scores matter as much as the model. The AlphaGenome configuration sweep shows how much a score definition can move a benchmark.

---

## 13. What is still missing

The SpliceAI2 discussion is candid about the gaps, and I think they define the next generation of models.

**Context-dependent variant effects.** Tissue-specific splicing is predicted, but tissue-specific variant effects are not, beyond baseline usage. Single-cell data and perturbation screens of splicing factors may provide the missing supervision, if the sparse junction coverage of single-cell data can be handled.

**Beyond splicing alone.** Transcripts are merged by intron chain, so transcription start and polyadenylation are ignored. Long reads make it possible to model splicing and 3' end formation **jointly**, which the splice graph could naturally extend to with start and end nodes.

**Kinetics and order.** Splicing is co-transcriptional, and the order of intron removal and the elongation rate affect outcomes. Current models are static maps from sequence to steady-state usage. Data on splicing order and timing could turn them into **kinetic** models.

**Mechanism.** These models learn RBP motifs and branch point features, but the spliceosome itself remains a black box. In my [CryoDRGN-AI post]({% post_url 2026-06-07-cryodrgn-ai-heterogeneous-cryo-em-reconstruction %}), I discussed how heterogeneous cryo-EM reveals the conformational landscape of the spliceosome. A long-term goal could be connecting the two worlds: sequence-to-splicing models that tell us *what* is spliced, and structural landscapes that tell us *how* the machine gets there.

**Design.** A differentiable model with a transcript likelihood is an attractive objective for **RNA and DNA design**: optimizing a sequence to favour a specific isoform, to rescue a mis-spliced exon, or to design splice-switching antisense oligonucleotides. The splice graph gives a natural reward: the log-probability of the desired transcript.

---

## 14. Conclusion

SpliceAI2 is less a new architecture than a careful **integration of techniques** that the field has been accumulating:

```text
long context via dilation + multi-scale pooling
MetaFormer blocks with depthwise convolution
deep supervision through per-block heads
quantitative RNA-seq usage targets, cube-root transformed
massive multi-species data with species as input
pairwise junction head (top-k candidates, Q·K with genomic RoPE)
splice graph CRF with forward-algorithm training and Viterbi decoding
Muon + AdamW optimization, small ensemble
consistent variant scoring with shared candidates
tissue/disease conditioning on RBP expression (bi-FiLM, cross-attention decoder, difference loss)
```

Its main conceptual step is to treat the **transcript** as the prediction target, with an exact likelihood over an exponential space of isoforms, while still learning from abundant short-read data. Its main empirical lesson is that much of the gain comes from **data scale and target design**, especially from learning the long tail of rarely used splice sites, rather than from model size.

The broader trajectory is clear. Splicing models have gone from classifying positions, to quantifying sites, to pairing them into junctions, to assembling junctions into transcripts. The next step is probably to make these predictions **context-aware and dynamic**: in which cell, under which regulator state, at what time, and through which conformations of the spliceosome.

---

## References

1. Jaganathan, K., Chen, J., Liu, X., Zhang, Y. et al. A unified framework for quantitative splicing and transcript prediction (SpliceAI2). Illumina manuscript, 2026. [PDF](https://assets.illumina.com/content/dam/illumina-marketing/images/genomics-research/articles/spliceai2/SpliceAI2.pdf); [announcement](https://www.illumina.com/science/genomics-research/articles/introducing-spliceai2--the-next-generation-of-splicing-and-trans.html); [model weights](https://huggingface.co/collections/illumina-ai/spliceai2).

2. Jaganathan, K. et al. Predicting splicing from primary sequence with deep learning. *Cell*, 2019.

3. Zeng, T. and Li, Y. I. Predicting RNA splicing from DNA sequence using Pangolin. *Genome Biology*, 2022.

4. Xu, C. et al. Reference-informed prediction of alternative splicing and splicing-altering mutations from sequences (DeltaSplice). *Genome Research*, 2024.

5. Chao, K.-H., Mao, A., Salzberg, S. L. and Pertea, M. Splam: a deep-learning-based splice site predictor that improves spliced alignments. *Genome Biology*, 2024.

6. You, N. et al. SpliceTransformer predicts tissue-specific splicing linked to human diseases. *Nature Communications*, 2024.

7. Avsec, Ž. et al. Advancing regulatory variant effect prediction with AlphaGenome. *Nature*, 2026.

8. Chao, K.-H. et al. OpenSpliceAI provides an efficient modular implementation of SpliceAI enabling easy retraining across nonhuman species. *eLife*, 2025.

9. Wagner, N. et al. Aberrant splicing prediction across human tissues (ABSplice). *Nature Genetics*, 2023.

10. Yu, W. et al. MetaFormer is actually what you need for vision. *CVPR*, 2022.

11. Liu, J. et al. Muon is scalable for LLM training. *arXiv*, 2025.

12. Sarawagi, S. and Cohen, W. W. Semi-Markov conditional random fields for information extraction. *NeurIPS*, 2004.

13. Burge, C. and Karlin, S. Prediction of complete gene structures in human genomic DNA (GENSCAN). *Journal of Molecular Biology*, 1997.

14. Gabriel, L. et al. Tiberius: end-to-end deep learning with an HMM for gene prediction. *Bioinformatics*, 2024.

15. Perez, E. et al. FiLM: visual reasoning with a general conditioning layer. *AAAI*, 2018.
