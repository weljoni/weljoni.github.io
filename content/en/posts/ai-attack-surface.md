---
title: "Why map the AI attack surface?"
date: 2026-08-26
draft: false
tags: ["AI Security", "Attack Surface", "Threat Modeling"]
categories: ["Research Notes"]
description: "How a systematic AI/Agent attack-surface taxonomy helps threat modeling and defense."
ShowToc: true
---

## Disclaimer

> This article and related materials are for **security research, education, and defensive engineering only**.  
> **Do not** use them to attack, probe, or exploit any system, model, service, or data without authorization.

## Why a map helps

Modern LLM apps connect tools, memory, retrieval, and multi-agent workflows. The attack surface is no longer “just the prompt” — it spans:

- Input manipulation (prompt injection, jailbreak, multimodal, guardrail bypass)
- Training & model supply chain
- Privacy & confidentiality
- Agent goal hijacking / tool abuse
- RAG vector-store poisoning
- Output & downstream integration
- Runtime & availability
- Recon & evaluation meta-attacks

A navigable taxonomy helps in three practical ways:

1. **More complete threat models** — fewer blind spots  
2. **Clearer defense priorities** — secure entry points and high-privilege actions first  
3. **Shared vocabulary** — engineering and security teams reason on one map  

## Interactive mind map

Use the top **中文 / English** switch; only one language is shown below. Pan and zoom supported:

**[Open interactive mind map →](/projects/ai-attack-surface/)**

Source repository:

- GitHub: [weljoni/ai-attack-surface](https://github.com/weljoni/ai-attack-surface)
