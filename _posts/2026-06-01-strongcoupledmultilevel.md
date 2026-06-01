---
layout: post
title: "Modeling life beyond standard Machine Learning algorithms"
date: 2026-06-01 00:00:00
description: "Why biological systems require multi-scale, strongly coupled, and perturbation-aware models, and how mathematical modeling and AI can move beyond standard CV and NLP paradigms."
tags: ["multi-scale", "strong-coupled","highly-dynamic"]
categories: multi-scale, strong-coupled
---

Life is multi-scale, strongly coupled, and highly dynamic. This is why modeling biology is fundamentally different from modeling images, language, or many other standard machine learning domains.

In computer vision, a model often receives an image and predicts a label, bounding box, segmentation mask, or visual representation. In NLP, a model reads a sequence of tokens and predicts the next token, retrieves information, classifies sentiment, or follows instructions. These tasks are extremely rich, but the input-output structure is usually relatively well defined: pixels to labels, tokens to tokens, documents to embeddings, prompts to responses.

Biology is different. A nucleotide change may alter RNA splicing. A splicing change may alter protein isoforms. Protein isoforms may change molecular interactions. These interactions may shift cellular state. Cellular state may reshape tissue microenvironments. Tissue context may then feed back into gene expression, RNA processing, and cellular behavior. The system is not a static mapping from genotype to phenotype. It is a nested, nonlinear, feedback-driven dynamical system.

This distinction matters because many machine learning methods developed for CV and NLP assume, implicitly or explicitly, that the task can be reduced to learning a high-dimensional function from input to output. Biological modeling often requires something more difficult: learning how interventions propagate through a living system.

## 1. Multi-scale biology: the variables do not live in the same space

Multi-scale modeling means that biological variables exist across very different levels of organization.

At the molecular scale, we have DNA sequence, RNA motifs, splice sites, RNA secondary structure, protein domains, post-translational modifications, and molecular binding events. At the cellular scale, we have gene expression, chromatin accessibility, cell identity, cell cycle, signaling state, metabolism, and perturbation response. At the tissue scale, we have spatial organization, cell-cell communication, extracellular matrix, vascularization, inflammation, and local microenvironments. At the organismal scale, we have development, aging, disease progression, treatment response, and evolutionary constraint.

These levels are not simply different resolutions of the same data. They are different coordinate systems. A nucleotide sequence is discrete and symbolic. Gene expression is quantitative and noisy. Spatial transcriptomics is structured by geometry. Cell-cell communication is relational. Disease phenotype is often sparse, delayed, and context-dependent.

This is a major difference from standard CV and NLP.

In CV, although images can contain multiple scales, the scales usually remain within one perceptual field: edges, textures, objects, scenes. In NLP, words, phrases, sentences, and documents form hierarchical structures, but they still live inside a shared linguistic token space. In biology, however, the hierarchy crosses physical, chemical, temporal, and organizational boundaries. A base-pair substitution and a neurodevelopmental phenotype are not just "short-range" and "long-range" tokens. They are separated by several mechanistic layers.

Therefore, biological foundation models cannot simply be larger versions of language models. They need mechanisms for connecting heterogeneous biological scales.

## 2. Strong coupling: biological modules cannot be modeled independently

Strong coupling means that biological subsystems interact so tightly that they cannot be accurately modeled as independent modules.

In weakly coupled systems, one can model component A, model component B, and then combine them with a relatively simple interaction term. Many engineering systems can be approximated this way. Biology often cannot.

Consider alternative splicing. Exon inclusion is influenced by splice donor and acceptor strength, local sequence motifs, intronic context, RNA secondary structure, RNA-binding proteins, transcriptional kinetics, chromatin state, cell type, developmental stage, and disease context. These factors do not contribute independently. A motif may be functional only when the relevant RBP is expressed. A variant may be benign in one tissue but pathogenic in another. A splicing change may alter the expression or function of a regulator, creating feedback.

This is why biology is not simply "sequence modeling with a different alphabet." DNA, RNA, and protein sequences are biological texts, but they are texts embedded in physical systems. Their meaning depends on cellular context, molecular concentration, spatial organization, and evolutionary history.

This also distinguishes biological modeling from mainstream NLP. In NLP, the meaning of a token is context-dependent, but the context is still mostly represented inside a sequence or document. In biology, the "context" may include cell type, chromatin state, tissue niche, age, perturbation history, and environmental exposure. Biological context is not just surrounding tokens. It is a living state.

## 3. Dynamics: biological models must predict change, not only representation

Modern CV and NLP models are often evaluated on representation quality, classification accuracy, generation quality, or instruction following. Biology certainly needs representation learning, but representation is not enough.

A useful biological model should answer questions such as:

- What happens if we mutate this splice site?
- What happens if we knock down this RNA-binding protein?
- What happens if we perturb this gene in a specific neuronal cell type?
- What happens if the same variant occurs in a different tissue?
- What happens after 6 hours, 24 hours, or 7 days?

These are not merely prediction questions. They are intervention questions.

This is where biological modeling intersects with dynamical systems, causal inference, perturbation modeling, and active learning. The goal is not only to describe the current state of a system, but also to simulate how the system changes after an intervention.

This is especially important for AI4Science. In many commercial ML applications, a model can be useful even if it only captures stable statistical regularities. In biology, a model that performs well on observational data may still fail when used for experimental design. The central question is not only, "Can the model predict what we have already measured?" It is also, "Can the model tell us what to perturb next?"

## 4. Mathematical modeling: from functions to coupled dynamical systems

From a mathematical perspective, biological systems are better described as coupled dynamical systems than as isolated input-output functions.

One natural formalism is differential equations. Ordinary differential equations, stochastic differential equations, delay differential equations, and partial differential equations can describe how biological states evolve over time. In gene regulation, for example, the expression level of each gene can be modeled as a function of transcriptional activation, repression, degradation, noise, and external perturbation. Neural differential equation models and single-cell dynamical models, such as scDiffEq, extend this idea by learning continuous cell-state transitions from high-dimensional data.

Another formalism is graph modeling. Gene regulatory networks, protein-protein interaction networks, splicing regulatory networks, cell-cell communication networks, and spatial tissue neighborhoods can all be represented as graphs. Graph neural networks are useful not only because they process graph-shaped data, but because they allow biological information to propagate along known or inferred relationships. Recent models such as GRN-aware RNA or single-cell foundation models attempt to inject gene regulatory structure into deep learning architectures, compensating for the limitations of purely data-driven embeddings.

A third formalism is probabilistic and energy-based modeling. Biological states can often be understood as distributions over possible configurations. Alternative splicing can be viewed as competition among isoforms. Cell identity can be viewed as an attractor in a high-dimensional state space. Perturbation can be viewed as deformation of an energy landscape. This perspective is useful because biological outputs are rarely independent labels. They are constrained configurations.

A fourth concept is coarse-graining. Multi-scale biological modeling cannot track every molecular event all the way to organism-level phenotype. The question is which lower-level variables can be compressed into effective higher-level variables without losing predictive power. This resembles ideas from statistical physics, but in biology the correct coarse-grained variables are often unknown and context-specific. Learning them is part of the scientific problem.

## 5. AI modeling: foundation models as cross-scale interfaces

Over the past three years, biological AI has moved rapidly from task-specific predictors toward foundation models for molecules, cells, tissues, and perturbations.

In single-cell biology, models such as scGPT, Geneformer, and scFoundation attempt to learn general representations of cells from large-scale transcriptomic datasets. They treat genes as tokens and cells as contexts, enabling tasks such as cell type annotation, gene module inference, perturbation prediction, and transfer across datasets.

However, this analogy to NLP has limits. A cell is not a sentence. Gene expression values are not words. The order of genes in a matrix is not equivalent to word order in language. More importantly, the gene expression profile is a snapshot of a biological state, not the full mechanism that produced it. Therefore, single-cell foundation models need additional biological structure: gene regulatory networks, chromatin accessibility, spatial context, lineage information, and perturbation data.

In spatial biology, models such as Nicheformer and related spatial transcriptomics foundation models begin to incorporate tissue neighborhoods and cell-cell communication. This is a major step beyond standard expression modeling. For tissues such as the brain, spatial context is indispensable. A neuronal cell state cannot be fully understood without considering local circuits, glial interactions, developmental stage, and disease microenvironment.

In RNA biology and splicing, recent models such as SpliceTransformer, Spliformer-v2, BigRNA, AIDO.RNA, TrASPr+BOS, and related RNA foundation models point toward a new generation of context-aware RNA models. These models move beyond canonical splice-site recognition. They aim to predict tissue-specific splicing, variant effects, RNA structure-function relationships, and even design sequences with desired regulatory outcomes.

For researchers working on splicing usage prediction in neuronal cells, this shift is especially relevant. A future splicing model should not only read the local sequence around a 5' splice site. It should also understand exon-intron architecture, RBP motifs, RNA structure, cell type, neuronal state, tissue context, evolutionary constraint, and perturbation history.

In other words, the task is not simply sequence-to-PSI regression. It is context-conditioned, multi-scale, perturbation-aware biological modeling.

## 6. Why biology is harder than standard CV and NLP

It is tempting to say that biological AI is now following the same path as NLP: pretrain on massive data, scale model size, fine-tune for downstream tasks, and eventually obtain general biological intelligence. This analogy is useful, but incomplete.

There are at least six major differences.

1. **Biological data are generated by experiments, not scraped from the internet.** Experimental design, batch effects, measurement noise, protocol differences, and missing modalities strongly shape the data distribution.

2. **Biological labels are often indirect.** A measured phenotype may be several causal steps away from the molecular event of interest.

3. **Biological systems are intervention-sensitive.** A model trained on observational data may not generalize to perturbations.

4. **Biological data are sparse relative to the size of the underlying state space.** The number of possible genotypes, cell states, perturbations, time points, and tissue contexts is astronomically large.

5. **Biological mechanisms impose constraints.** Molecules obey chemistry. Cells obey physical limits. Tissues obey spatial organization. Evolution imposes selection. These constraints are not optional priors; they are part of the system.

6. **Evaluation is more difficult.** In NLP, a model can often be judged by benchmark performance, human preference, or downstream task accuracy. In biology, the strongest evaluation may require new experiments. The real benchmark is whether the model can guide perturbations that work in the lab.

This is why biological AI should not blindly imitate CV and NLP. It can borrow architectures, optimization methods, scaling laws, and representation learning strategies, but it must also incorporate mechanistic priors, causal structure, uncertainty estimation, and experimental feedback loops.

## 7. Recent direction: from predictors to perturbable world models

A major trend from 2023 to 2026 is the move from static prediction toward perturbation-aware modeling.

In single-cell perturbation modeling, methods such as GEARS, CellOT, CPA, scGen, PDGrapher, STATE, and newer perturbation foundation models attempt to predict how cells respond to genetic or chemical perturbations. These models are important because they address one of the core problems of biology: how changes propagate through coupled molecular networks.

However, recent benchmarks also show that large neural models do not automatically outperform simpler baselines in all perturbation settings. Cross-cell-type generalization, unseen perturbations, combinatorial perturbations, and out-of-distribution biological contexts remain difficult. This suggests that scale alone is insufficient. Biological models need structure.

In RNA and splicing, the field is moving toward tissue-aware, variant-aware, and design-aware models. SpliceTransformer and Spliformer-v2 focus on tissue-specific splicing and variant effect prediction. BigRNA and AIDO.RNA expand the scope of RNA representation learning. TrASPr+BOS connects prediction with sequence design, making it possible to ask not only "what will this sequence do?" but also "what sequence should we design to produce this splicing outcome?"

In spatial and tissue modeling, models such as Nicheformer, PULSAR, SToFM, and pathology-spatial transcriptomics models attempt to connect molecular profiles with tissue organization. This matters because many biological functions emerge only at the multicellular level. A cell's behavior is not determined solely by its internal gene expression program; it is also shaped by its neighbors.

These directions point toward a larger goal: biological world models. A biological world model should not merely embed data. It should simulate state transitions, predict interventions, represent uncertainty, and guide experiments.

## 8. Toward a multi-scale splicing world model

For splicing biology, an ambitious but concrete goal would be a multi-scale splicing world model.

Such a model would take DNA or RNA sequence as the molecular input. It would represent splice-site strength, exon-intron architecture, RNA motifs, RNA structure, and RBP binding. It would condition predictions on cell type, tissue, developmental stage, disease state, and perturbation context. It would output PSI, isoform usage, transcript abundance, protein consequence, and possibly downstream cellular phenotype.

Most importantly, it would be perturbable. We could ask:

- What happens if this 5' splice site is weakened?
- What if a downstream intronic motif is mutated?
- What if an RBP is knocked down in neuronal cells?
- What if the same sequence is placed in a different cellular context?
- What if we want to design a sequence that increases exon inclusion only in one tissue?

This type of model would require more than a Transformer. It may need a hybrid architecture combining sequence models, graph neural networks, neural differential equations, causal representation learning, Bayesian optimization, and active learning. Sequence models would capture local regulatory grammar. Graph models would represent regulatory interactions. Dynamical systems would describe state transitions. Causal models would separate intervention from correlation. Active learning would close the loop with experiments.

This is where mathematics and AI meet. Mathematics defines state variables, constraints, coupling structures, and intervention operators. AI learns high-dimensional approximations from real biological data. Experiments provide the final grounding.

## 9. Conclusion: biology needs models that remain valid after perturbation

Life is multi-scale because molecular, cellular, tissue, and organismal processes exchange information across levels. Life is strongly coupled because changes at one level can propagate through feedback networks. Life is dynamic because these relationships change over time, context, and environment.

Therefore, the next generation of biological AI should not aim only to maximize benchmark accuracy. It should aim to remain biologically meaningful under perturbation.

This is the major distinction between biological machine learning and many standard CV/NLP settings. In CV and NLP, representation and generation are often the central products. In biology, representation is only the beginning. The real goal is intervention: to predict, design, perturb, and understand living systems.

For AI4Science, the challenge is not simply to build a larger model of biological data. The challenge is to build models that can reason across scales, respect coupling, capture dynamics, quantify uncertainty, and help design the next experiment.

We are not fitting a static function.

We are trying to learn a living system.

## References and source notes

This blog draft is organized around recent directions from 2023 to 2026 in biological foundation models, single-cell and spatial transcriptomics, perturbation modeling, RNA/splicing modeling, and AI4Science. Representative work discussed includes AI virtual cell and digital organism perspectives, scGPT, Geneformer, scFoundation, Nicheformer, PULSAR, scDiffEq, GEARS, CellOT, CPA, scGen, PDGrapher, STATE, SpliceTransformer, Spliformer-v2, BigRNA, AIDO.RNA, TrASPr+BOS, Splam, and minisplice.

Some of the cited directions include preprints from bioRxiv or arXiv and should be clearly marked as preprints before formal publication. For a public-facing version, numerical claims about model performance should be added only where they are directly supported by the original papers.

