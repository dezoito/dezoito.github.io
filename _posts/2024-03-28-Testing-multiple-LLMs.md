---
layout: post
comments: true
title: Testing Nultiple LLMs in one Pass
excerpt_separator: <!--more-->
---

> _**What is the best** model for for story telling?_

> _I want to use LLMs to generate RPG scripts, **which model** should I use?_

Users making such inquiries can usually receive good advice, but when you need to compare different (perhaps similar) models, the process can **take a lot of effort**, and the **results can remain inconclusive**.

In this article we explore some ways to automate the process, **testing several models in a single operation**, visually inspecting results and comparing data such as **generated tokens** and **thoroughput**.

<!--more-->

## Getting started

The automation process we propose requires a few moving parts:

- [Ollama](https://ollama.com/) installed and running (either on localhost or on a server the user has access to).
- [Ollama Grid Search](https://github.com/dezoito/ollama-grid-search) needs to be installed in the users machine.

Both dependencies above are **free** and can be installed in all major platforms.

Finally:

- The models for comparison need to have been pulled into the Ollama server

## Running tests

In this example, I'm building a [RAG Fusion](https://arxiv.org/abs/2402.03367) agent, and I need to decide between three models, which is the best at generating alternative, complementary queries to a prompt such as:

```
Consider the initial prompt:

"List the most relevant information we have on John Doe".

The person above may or may not be connected to criminal activities.
Write a list of three concise prompts that can complement the query above.

```

**[Ollama Grid Search](https://github.com/dezoito/ollama-grid-search)** is a Large Large Language Model testing desktop application. It let's us select the models we want to use, from a list of those installed on the server:

![Model Selection](https://raw.githubusercontent.com/dezoito/dezoito.github.io/master/public/images/ogs-model-selector.png)

After selecting the three models above, we can enter the prompt

![Prompt input](https://raw.githubusercontent.com/dezoito/dezoito.github.io/master/public/images/ogs-prompt.png)

## Conclusion