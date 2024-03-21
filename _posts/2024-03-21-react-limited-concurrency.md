---
layout: post
comments: true
title: Limited Concurrency for Multiple API calls in React
excerpt_separator: <!--more-->
---

Sometimes you have a list of items that need to be processed by an API, like a list of documents to be summarized by a Large Language Model, or a list of terms that trigger scraping jobs.

If these tasks are expensive (i.e.: they demand a lot from the server), or if there are rate limits, you may need to use a Limited Concurrency pattern to stop the server from being flooded with several simultaneous requests.

This article explains how this was solved in React, using [tanstack-query](https://tanstack.com/query/latest).

<!--more-->
