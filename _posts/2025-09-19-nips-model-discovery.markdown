---
layout: post
title:  "Automated Model Discovery via Multi-modal & Multi-step Pipeline"
date:   2025-10-14 22:21:59 +00:00
image: /images/model_discovery.png
categories: publications
author: "Lee Jung-Mok"
venue: "NeurIPS 2025, IPIU 2025 <ystrong>(Best Poster Awarded!!)</ystrong>"
authors: "<strong>Lee Jung-Mok</strong>, Nam Hyeon-Woo, Moon Ye-Bin, Junhyun Nam, Tae-Hyun Oh"
subtitle: "Automatic Model Discovery using VLM agents"
arxiv: https://arxiv.org/abs/2509.25946
code: https://github.com/kaist-ami
website: https://github.com/kaist-ami
---

We present a multi-modal & multi-step pipeline for effective automated model discovery, using the multimodal LLM agents. We model the interpretation of time-series data using Gaussian Process kernel discovery. 

<blockquote>
  <p>
    Automated model discovery is the process of automatically searching and identifying the most appropriate model for a given dataset over a large combinatorial search space. Existing approaches, however, often face challenges in balancing the capture of fine-grained details with ensuring generalizability beyond training data regimes with a reasonable model complexity. In this paper, we present a multi-modal & multi-step pipeline for effective automated model discovery. Our approach leverages two vision-language-based modules (VLM), AnalyzerVLM and EvaluatorVLM, for effective model proposal and evaluation in an agentic way. AnalyzerVLM autonomously plans and executes multi-step analyses to propose effective candidate models. EvaluatorVLM assesses the candidate models both quantitatively and perceptually, regarding the fitness for local details and the generalibility for overall trends. Our results demonstrate that our pipeline effectively discovers models that capture fine details and ensure strong generalizability. Additionally, extensive ablation studies show that both multi-modality and multi-step reasoning play crucial roles in discovering favorable models.
  </p>
</blockquote>
