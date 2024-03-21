---
layout: post
comments: true
title: Limited Concurrency for Multiple API calls in React
excerpt_separator: <!--more-->
---

Sometimes you have a list of several items that need to be processed by an API, like a list of documents to be summarized by a Large Language Model, or a list of terms that trigger scraping jobs.

If these tasks are expensive (i.e.: they demand a lot from the server), or if there are rate limits, you may need to use a Limited Concurrency pattern to stop the server from being flooded with simultaneous requests.

This article explains how this was solved in React, using [tanstack-query](https://tanstack.com/query/latest).

<!--more-->

## The Problem

In developing [Ollama Grid Search](https://github.com/dezoito/ollama-grid-search), a tool that allows users to test an AI prompt on a combinations of models and parameters, we had two requirements that relate specifically with the topic of this article:

1. Each test run could potentially generate dozens of expensive calls to the LLMs (and most likely those would be hosted locally), so we needed a way to stop the app from asynchronously sending all those calls to the API, potentially crashing the machine.

2. Users needed a way to re-run a single inference call (a combination of a model and several params), without having to perform an entire experiment again.

The **Tanstack-query** libray makes implementing requirement #2 trivial (by using a function called `refetchQueries()`) and, with a little help, also makes the first one easily attainable too!

<img src="https://raw.githubusercontent.com/dezoito/ollama-grid-search/main/screenshots/main.png" alt="Ollama Grid Search interface" width="720">

Inference calls are processed one at a time, but each combination of parameters can be requeried individually.

## The Solution (and credit to [anton](https://stackoverflow.com/users/8571434/anton) and [spaceoso](https://stackoverflow.com/users/6090489/spaceoso))

We were having a little trouble marrying and implementation of the Semaphore protocol with the library (and no AI was able to help), but luckly we came across this [StackOverflow answer](https://stackoverflow.com/a/76953016).

```js
const [selection, setSelection] = React.useState([])
const [noCompleted, setNoCompleted] = React.useState(0)

const results = useQueries(
  selection.map((item, i) => ({
    queryKey: ['something', item]
    queryFn: () => fetchItem(item)
    enabled: i <= noCompleted
    staleTime: Infinity
    cacheTime: Infinity
   })
)

const firstLoading = results.findIndex((r) => r.isLoading)

React.useEffect(() => {
  setNoCompleted(firstLoading)
}, [firstLoading])
```

The gist of it is:

1- Create an array of **disabled** queries.

2- Wait until one query is finished to start the next one.

While it didn't work right out of the box for our use case, it eventually led us to the correct implementation! Keep reading!

## An example

To better explain this concept we created a [React Limited Concurrency Demo Application](https://github.com/dezoito/react-limited-concurrency-queries):
