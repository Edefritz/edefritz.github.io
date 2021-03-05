---
layout: post
title: Throttle a series of fetch requests in JavaScript
categories: [NodeJS, JavaScript, Throttle]
---

# Throttle a series of fetch requests in JavaScript

Let's say you need to make API requests to process a huge array of data. With JavaScript's asynchronous nature it is easy to make a lot of requests in parallel. 

```js
import fetch from "node-fetch";

const data = [{ id: 1 }, { id: 2 }, [+1000 more objects]];

const fetchFromApi = (id) => {
  const url = `https://example.com/api/my-resource/${id}`;

  const response = fetch(url)
    .then((x) => x.json())
    .catch((error) => console.log(error));
  return response;
};

for (const i of data) {
  fetchFromApi(i.id).then((result) => // do something with result);
}

```
### HTTP code 429: Too many requests
However, most API providers don't like if you flood them with too many requests at the same time.
What you usually would get in return is a HTTP error code 429. If you check the documentation there might be a limitation of let's say maximum 5 requests per second.
But even if it is an internal API that isn't that restricted you might want to reduce the amount of parallel requests.

### Wait for the response before making another request?
What you could do is introducing a blocking structure to wait for the response of the previous call, before making another one using JavaScripts async/await syntax.

```javascript
import fetch from "node-fetch";

const data = [{ id: 1 }, { id: 2 }, [+1000 more objects]];

const fetchFromApi = async (id) => {
  const url = `https://example.com/api/my-resource/${id}`;

  const response = fetch(url)
    .then((x) => x.json())
    .catch((error) => console.log(error));
  return response;
};

for (const i of data) {
  const response = await fetchFromApi(i.id);
  // do something with result
}
```
While this would take longer to run, it would not solve the issue. The API might respond very quickly and you would still reach the limit of 5 requests per second.
On the other hand if the API responds slowly, you wouldn't benefit of parallelism at all, which would make the whole operation take longer than needed.

### Semaphore to the rescue
Using a throttling mechanism would be the more elegant way to deal with this issue. In computer science there's the concept of a [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming)) which describes a way to control access to a common resource by multiple processes.
There is a [library](https://github.com/vercel/async-sema) which implements that and allows you to limit the maximum parallel requests. The code would look something like this:

```javascript
import fetch from "node-fetch";
import {RateLimit} from "async-sema";

// configure a limit of maximum 5 requests / second
const limit = RateLimit(5);

const data = [{ id: 1 }, { id: 2 }, [+1000 more objects]];

const fetchFromApi = (id) => {
  const url = `https://example.com/api/my-resource/${id}`;

  // use the configured throttle here
  const response = fetch(url)
    .then((x) => x.json())
    .catch((error) => console.log(error));
  return response;
};

for (const i of data) {
  // checks if limit is reached
  await limit()
  fetchFromApi(i.id).then((result) => console.log(result));
}
```