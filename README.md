# Reference-Fetcher

ReferenceFetcher is a simple-to-use JS library to fetch entity references.

[![Travis Build](https://img.shields.io/travis/wing-eu/reference-fetcher.svg?style=flat-square)](https://travis-ci.org/wing-eu/reference-fetcher/) [![Version](https://img.shields.io/npm/v/reference-fetcher.svg?style=flat-square)](https://github.com/wing-eu/lucmerceron/reference-fetcher/releases) [![Code Coverage](https://img.shields.io/codecov/c/github/wing-eu/reference-fetcher.svg?style=flat-square)](https://codecov.io/gh/wing-eu/reference-fetcher)

## Installation

```
npm install --save reference-fetcher
```

## Documentation

ReferenceFetcher is an algorithm useful for retrieving multiple referenced entity from a database.

Imagine we have this kind of database:

![Reddit Base](./reddit.png)

Posts is the key database table here. A post has 3 references; a tag, a creator and a subreddit which correspond to IDs that we can find in the corresponding database tables.

**Note:** ReferenceFetcher is made to work with only unique references up to now, do not use it to fetch an array of reference (i.e. a Post should have only **one** tag, **one** creator and **one** subreddit in our example). Array may come as a feature in the near future.

Our workflow is to get a particular set of Posts (to feed a page or whatever) and its references but we do not want the Back-End to populate them for us for three reasons:
 1. This could create a massive request that take times to compute with a considerable payload (hurting our [First Meaningful Paint](https://developers.google.com/web/tools/lighthouse/audits/first-meaningful-paint) time).
 2. It will probably contains duplication of the same entity (multiple posts can have the same creator, tag or subreddit). 
 3. We usually prefer to keep our Front-End stores as flat as possible. Using deeply nested data is often very difficult to use in JavaScript applications, especially for those using [Flux](http://facebook.github.io/flux/) or [Redux](http://redux.js.org/)
 
That's where ReferenceFetcher shines. Thanks to a configuration object, we can declare what references we want to fetch and it will call our Promises with the correct parameters plus taking care of duplication. Then it is up to us to populate correctly our store with our Promises results. At Wing we use Redux dispatch from our Promises (which are actions) result to populate our stores. Promises must **always** returns their results after the operation you have done.

```javascript
import referenceFetcher from 'reference-fetcher'

const configuration = {
  entity: 'posts',
  fetch: () => getPosts(),
  rootNoCache: true,
  refs: [{
    entity: 'subreddit',
    fetch: subredditId => getSubreddit(subredditId),
  }, {
    entity: 'subscribers',
    relationName: 'creator',
    batch: true,
    fetch: subscribersIds => getSubscribers(subscribersIds),
    refs: [{
      entity: 'stats',
      relationName: 'stat',
      batch: true,
      fetch: getStats,
    }]
  }, {
    entity: 'tag',
    noCache: true,
    fetch: getTag,
  }],
  sides: [{ // Optional: useful to retrieve entity by post ids
    entity: 'notLinked',  
    fetch: postIds => getNotLinkedEntity(postIds),
  }]
}

referenceFetcher(configuration)
```

Example of actions used:

```javascript
const getPosts = () => new Promise((resolve, reject) => {
  // Retrieve posts from your api
  fetch(`${APIURL}/posts`).then(response => {
    // Populate your store
    postsStore.push(...response.posts)
    // Return the response allowing to fetch underneath references
    // and chain our promise
    return response.posts
  })
})
const getSubreddit = subredditId => new Promise((resolve, reject) => {
  fetch(`${APIURL}/subreddits/${subredditId}`).then(response => {
    subredditsStore.push(response.subreddit)
    return response.subreddit
  })
})
const getSubscribers = subscribersId => new Promise((resolve, reject) => {
  // A POST method to get a batch of subscribers, because you can
  fetch(`${APIURL}/subscribers`, {
    method: 'POST',
    body: subscribersId
  }).then(response => {
    subscribersStore.push(...response.subscribers)
    return response.subscribers
  })
})
```

### Arguments
* entity (String): The name that will be used to retrieve data from the Promise result. These data will then be used to retrieve our underneath refs. Omit this argument if your API response contains directly the desired result.
* fetch (Promise): The Promise that will be called to get our particular `entity`. fetch is called with no parameters if it is the root Object (`Post` in our example) or is called multiple times with  a `String` representing the ID to fetch (`getSubreddit` in our example) or is called once with an `Array` of ID to fetch (`getSubscribers` in our example). **The promise must return its results if we want lower refs to work.**
* relationName (String): *Only for refs*  The name of the attribute in the parent object where we can find the ID to fetch. Default to `entity`.
* batch (Boolean): *Only for refs* Default value: false. Set to true if you want your `fetch`called only once with the array of ID to retrieve.
* optional (Boolean): *Only for refs* Default value: false. Set to true if the entity could not be in the object you fetched.
* noCache (Boolean): *Only for refs* Default value: false. Set to true if you want to bypass the already fetched watcher. It means that if the entity we want to retrieve was already retrieved earlier, ReferenceFetcher will still call our `fetch` with the unique IDs it finds in the parent `Object`.
* rootNoCache (Boolean): *Only for root* Default value: false. Set to true if you want to bypass the caching on the root fetch. It means that if we call our referenceFetcher with the same root fetch function, it will recall it each time we call the referenceFetcher
* sides (Array): An array composed of object of type: `({ entity: string, fetch: function })`. Used to fetch entities that are not linked to a parent. Fetch will be called with the result of the parent fetch function. Will use the cache.

### Returns
ReferenceFetcher returns nothing, its objective is only to correctly call the different Promises. We need to feed our store directly from the Promises.

## Change Log
This project adheres to [Semantic Versioning](http://semver.org/).

You can find every release documented on the [Releases](https://github.com/wing-eu/reference-fetcher/releases) page.

## License
MIT