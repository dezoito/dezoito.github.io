---
layout: post
comments: true
title: Grid Search on Large Language Models using Ollama and Rust
excerpt_separator: <!--more-->
---

This article explores the process of optimizing Large Language Models (LLMs) for specific applications through grid search.

Our goal is to streamline parameter tuning for enhanced inference efficiency, complementing prompt engineering efforts, using models hosted in an [Ollama](https://www.ollama.ai) instance and an interface built in Rust.

> [01-MAR-2024] Note, this article was written before we released a Desktop version of **[Ollama Grid Search](https://github.com/dezoito/ollama-grid-search)**!

<!--more-->

## Why use LLMs

I recently built a [multilabel classifier](https://en.wikipedia.org/wiki/Multi-label_classification) API - you post text to the endpoint and it returns an array of zero or more topics related to the text, selected from a taxonomy of subjects.

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-multilabel-classifier_n.png?raw=true)

Due to the classified nature of the data, ALL steps had to be performed "on premises".

Since external APIs were out of the question, this required efforts on a few different fronts:

- Collect and analyze labeled data
- Deal with inbalanced classes
- Choose a multi-language model (I used a variation of BERT)
- Experiment with hyperparams
- Train model with optimized params (done in PyTorch)
- Build "sliding window" inference
- Build API inference server with FastAPI and Docker
- Pre/Post processing for the inputs and response
- Monitoring

But now I can use Ollama to conveniently host and serve different LLM models.

For each task, all I have to do is carefully craft a prompt, make a POST request, and Ollama will return a nicely formatted response.

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-llm-service.png?raw=true)

No need to build Fast API servers, muck with datasets or choose and train models.

Right?

Not so fast.

## The Caveats

While the conclusions above sound promising, the fact is that LLMs tend (for now) to consume sigificantly more resources - and take longer to process - compared to the previous method.

In addition, to get the expected results, you **still** have to experiment with some factors:

- Model selection
- Prompt engineering
- Pre/Post processing
- **Model Parameter Tuning**

There are a ton of resources on the first three factors, so let's focus on the last.

## Parameter Tuning

Ollama let's you define a [comprehensive list of parameters](https://github.com/jmorganca/ollama/blob/main/docs/modelfile.md#parameter) when creating or prompting LLMs, but I found it difficult to understand how some of them affected the model performance.

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-ollama-params.png?raw=true)

More importantly, trying different combinations and annotating results was a tedious and time consuming process, so I made **[Ollama Grid Search](https://github.com/dezoito/ollama-grid-search)**, a Rust based tool that automates prompt testing, iterating over combinations of parameters and allowing the user to visually inspect results and stats.

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-tool-view.png?raw=true)

Please check the [project's README](https://github.com/dezoito/ollama-grid-search) for a more in-depth look at the tool and its usage.

## Experiments

Consider the task of generating "**clear and concise**" summaries for different document types in a database.

After deciding on a particular model, we selected 4 different types of commonly used documents that vary in style.

These differences caused the quality of the results to vary wildly, using the default params, so the question was:

<blockquote>
Is there a set of model params that yields good results for all document types?
</blockquote><br/>

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-script-params.png?raw=true)

For each document we performed an experiment, using the same prompt but iterating over the possible combinations of parameters.

At a quick glance, you can tell which configurations generated good summaries for that particular prompt/model, but it’s still hard to determine which ones were good across all types of documents:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-results.png?raw=true)

A way to visually detect the best performing parameters was to manually annotate results on a grid:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-spreadsheet.png?raw=true)

Simply, the image above displays a spreadsheet where each cell represents a combination of parameters.

If that combination generated good results for a particular document type, we added its name to the cell.

Cells were collored according to the number of documents, producing a "heatmap" and clearly indicating that the params in cells 18, 20 and 43 generalized best for this application.

We can even repeat the same process, across different models:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/grid-spreadsheets.png?raw=true)

In the image above we test three different models:

- dolphin2.2-mistral:7b-q4_0 (default SYSTEM prompt)
- dolphin2.2-mistral:latest (default SYSTEM prompt)
- dolphin2.2-mistral:latest (custom SYSTEM prompt)

We can clearly see that, in this case, the quantized model did not perform as well as the other ones.

## Conclusion

In conclusion, the process above, assisted by **[Ollama Grid Search](https://github.com/dezoito/ollama-grid-search)**, was very useful in determining the optimal model parameters for a given task, but also to test different models and prompts, to visually compare results, and make an informed decision.
