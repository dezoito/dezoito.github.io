---
layout: post
comments: true
title: Node and Azure - Asynchronous List Searches using Bing Search API
excerpt_separator: <!--more-->
---

This article explores the issue of running "simultaneous" searches for all entries in a list of terms (in reality, we are running searches [asynchronously](https://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean)).

I'm also trying to improve on Azure's Search API example a little bit.
<!--more-->

### Pre-requisites

1. You have to have an account on [Azure](https://portal.azure.com/) and a subscription key for the Bing Web Search Service  (See docs here: https://azure.microsoft.com/en-us/services/cognitive-services/bing-web-search-api/).

2. Node and npm installed (I was using Node 12.14 and npm 6.14 at the time of writting).


### The Problem
I have to run searches on dozen (sometimes hundreds of terms) on occasion, so the first step was to automate that, using Bing's excellent Web Search API.

The second step was taking advantage of Javascript's asynchronous nature so *all* these terms could be searched in parallel (hopefuly yealsing a faster response, instead of on-by-one).

### Code
The first iteration looks like this (I'll break it down and add complexity later):

```js
const CognitiveServicesCredentials = require('ms-rest-azure').CognitiveServicesCredentials;
const WebSearchAPIClient = require('azure-cognitiveservices-websearch');
const BING_SEARCH_API_KEY = '<<your_subscription_key_here>>';

// list of search terms/wish-list
const myList = ['Lagavulin Offerman Edition',
  'Laphroaig Lore',
  'Laphroaig PX CAsk',
  'Hibiki Japanese Harmony',
  'Jameson Caskmates',
  'Caol Ila Distillers Edition',
  'Aberlour A\'bunadh',
  'Nikka Coffey Whisky',
  'Glenfiddich Vintage Cask',
  'Glenfiddich 18',
  'Oban Little Bay'];


/* 
 * Defines logic for a single search against BING's API
 */
const asyncSearchBing = async (entity) => { 
  console.log(`Searching BING for ${entity}`);
  let credentials = new CognitiveServicesCredentials(BING_SEARCH_API_KEY);
  let webSearchApiClient = new WebSearchAPIClient(credentials);

  try {
    const response = await webSearchApiClient.web.search(`${entity}`, {
      market: "en-GB",
      location: 'lat: 55.953251, long: -3.188267',
      count: 5,
    })
    var data = {
      entity,
      source: "Bing",
      data: response["webPages"]  // return only webPage results
    };

  } catch (error) {
    console.error(`Error seaarching entity ${entity}`)
    console.error(error);
    var data = {
      entity,
      source: "Bing",
      data
    };
  }
  return data;
}


/* 
 * Runs async searches using the previously defined search functions
 */
const runSearch = async (list) => { 
  const promises = []
  promises.push(...list.map(async (entity) => asyncSearchBing(entity)));

  // executes all promisses asynchronously
  const getData = async () => {
    return await Promise.all( promises )
  }
  return await getData();
};

//executes searches
runSearch(myList)
  .then(results => console.log(results));
```
### Code Breakdown
The first 3 lines are boilerplate for setting up Bing's Web Search Client.

```js
const CognitiveServicesCredentials = require('ms-rest-azure').CognitiveServicesCredentials;
const WebSearchAPIClient = require('azure-cognitiveservices-websearch');
const BING_SEARCH_API_KEY = '<<your_subscription_key_here>>';
```

The `asyncSearchBing` function encapsulates the logic for searching a single keyword using Bing's service.

Bing's results are grouped into `properties` such as `images`, `news`, `videos` and `webpages` (we only want the latter).

Also, notice we define preferences for market and location and limit the results to 5 entries per term: 

```js
const asyncSearchBing = async (entity) => { 
  console.log(`Searching BING for ${entity}`);
  let credentials = new CognitiveServicesCredentials(BING_SEARCH_API_KEY);
  let webSearchApiClient = new WebSearchAPIClient(credentials);

  try {
    const response = await webSearchApiClient.web.search(`${entity}`, {
      market: "en-GB",
      location: 'lat: 55.953251, long: -3.188267',
      count: 5,
    })
    var data = {
      entity,
      source: "Bing",
      data: response["webPages"]// return only webPage results
    };

  } catch (error) {
    // error handling ....
    };
  }
  return data;
}
```

Next we define a way to "map" this function to each search term in our list, and then run all the searches in "parallel":

```js
const runSearch = async (list) => { 
  const promises = []
  promises.push(...list.map(async (entity) => asyncSearchBing(entity)));

  // executes all promisses asynchronously
  const getData = async () => {
    return await Promise.all( promises )
  }
  return await getData();
};
```

The `promises.push(...)` line was written that way so, in the future, we could run the search using more providers (or using functions with different logic).

Had we defined a `asyncSearchGoogle` function, similar to the one we did for Bing, we could modify our code to search each term on both services:

```js

  const promises = []
  promises.push(...list.map(async (entity) => asyncSearchBing(entity)));
  promises.push(...list.map(async (entity) => asyncSearchGoogle(entity)));

```
Finally, we just execute the searches:

```js
runSearch(myList);
```

Bing will, hopefully, give you search results like these:
```js
 [
  {
    entity: 'Glenfiddich 18',
    source: 'Bing',
    data: {
      _type: 'Web/WebAnswer',
      webSearchUrl: 'https://www.bing.com/search?q=Glenfiddich+18',
      totalEstimatedMatches: 343000,
      value: [Array]
    }
  },
  {
    entity: 'Oban Little Bay',
    source: 'Bing',
    data: {
      _type: 'Web/WebAnswer',
      webSearchUrl: 'https://www.bing.com/search?q=Oban+Little+Bay',
      totalEstimatedMatches: 117000,
      value: [Array]
    },
    ....
  }
]

```
### Improvements
Bing's free tier subscription imposes a limit of 3 searches per second, which is constantly broken by the code above, returning a "quota exceeded error" in some of the results.

One way to deal with that is to add a randomized pause before searching each keyword (or you can sign up for a paid subscription if you want **really** fast execution).

Our modified code blocks look like this:

```js

const CognitiveServicesCredentials = require('ms-rest-azure').CognitiveServicesCredentials;
const WebSearchAPIClient = require('azure-cognitiveservices-websearch');
const BING_SEARCH_API_KEY = '<<your_subscription_key_here>>';

// aux: generates pseudorandom numbers within a range
const randomIntFromInterval = (min, max) => { 
  return Math.floor(Math.random() * (max - min + 1) + min);
}

// aux: async sleep
const sleep = (milliseconds) => {
  return new Promise(resolve => setTimeout(resolve, milliseconds))
}

```
and

```js

const asyncSearchBing = async (entity) => { 
  console.log(`Searching BING for ${entity}`);
  let credentials = new CognitiveServicesCredentials(BING_SEARCH_API_KEY);
  let webSearchApiClient = new WebSearchAPIClient(credentials);

  try {
    // sleeps for a random interval before executing the search
    await sleep(randomIntFromInterval(600, 800));

    const response = await webSearchApiClient.web.search(`${entity}`, {
      market: "en-GB",
      location: 'lat: 55.953251, long: -3.188267',
      count: 5,
    })

    ....
  }

```
This "random pause" technique "slows down" execution just enough to stay within the quota, but it's also useful if you want a more "human" pattern.

Full code for this article can be seen in [this gist](https://gist.github.com/dezoito/42252dcfacafd20a13c8e0a31942a5cb).

## References
[Bing Search API Documentation](https://docs.microsoft.com/en-us/azure/cognitive-services/bing-web-search/)

[Quickstart: Use a Bing Web Search client library](https://docs.microsoft.com/en-us/azure/cognitive-services/bing-web-search/quickstarts/client-libraries?pivots=programming-language-javascript)

[How to make your JavaScript functions sleep](https://flaviocopes.com/javascript-sleep/)

[Javascript Spread Syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)









