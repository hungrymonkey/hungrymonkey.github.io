---
layout: post
title: Developing a pro bono auction application with Cloudflare Workers and Fauna
date: 2021-08-10 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: 2021-08-10-faundb-cloudflare-banner.png # Add image post (optional)
tags: [cloudflare, faunadb, CRUD, javascript, database, serverless] # add tag
---

## Introduction

I would like to thank Cloudflare Workers and Fauna for providing a generous free tier allocation to allow me to develop, test, and deploy a simple auction application for the 2021 Asian American Architect and Engineer yearly fundraiser.

For my volunteer work, the AAa/e organizers asked me to develop an auction website for their 2021 annual fundraiser. Although I agree the project is feasible within the allotted time, the organizers provided vague answers to hosting resources because architects tend to work long hours and the silent auction was one of many events to be hosted at the 2021 fundraiser. As a result, I decided to evaluate various serverless products to reduce deployment, development, and hosting resources. Although I never used their products before, Cloudflare stands out because their engineering team chronicled their troubles and strives to contribute to the Linux ecosystem. In addition, I decided to choose their preferred database partner with a decent sized free tier because I wanted to avoid compatibility problems. Yikes, I added two new tools to learn on a time constrained project.

![AAa/e silent auction]({{site.baseurl}}/assets/img/2021-08-10-auction-website.png)

As a basic overview, an auction site must have a frontend, backend, and database. Since this project only needs to serve a few hundred people, I can forgo features such as caches and various other features to improve scalability. Hence, this project can be completed within the free tier. Nevertheless, these tools have benefits and caveats as mentioned below.

## Backend

Instead of VMs, Cloudflare uses V8 javascript processes called workers for an isolated environment. This architecture allows an always active runtime process that spawns workers in milliseconds. Cloudflare allows you to avoid finding workarounds for cold starts in comparison to VM based serverless solutions. For VM based serverless functions, the cloud provider only provides an instance when the user touches your code. The initialization process can be quite long because the cloud provider must download all dependencies from scratch and evaluate the code onto a fresh instance. This provision instance is temporary and will likely disappear after 10-15 minutes of inactivity which triggers another cold start. For example, AWS cold starts can be at least 100 ms for scripting languages such as Python and shoot up to seconds for compiled languages like Java.

As expected, Cloudflare tooling allows anyone to set up an environment in less than an hour. In fact, creating and uploading a GET endpoint that outputs “Hello World” has the same amount of command as running a c compile to output a basic binary. In order to support the simple workflow and tools, Cloudflare scoped out many features such as dependency management in order to encapsulate code as a single file. As a result, maintainers advocate end developers to use solutions such as webpack in order to combine javascript files. Naturally, you should turn off optimization in order to read the output in the Cloudflare debugger.

![Cloudflare Development Environment]({{site.baseurl}}/assets/img/2021-08-10-cloudflare-editor.jpg)

The V8 engine introduces many restrictions on tooling and language usage. Although the web has introduced new features such Web Assembly, Cloudflare developers choose Node.js in favor of the V8 javascript engine only. In my testing, I attempted to develop in Python and Javascript.

### Javascript

In Javascript, Node.js dominates the back development ecosystem such that the vast majority of libraries only support the Node.js API. Unfortunately, Cloudflare only supports HTTPS requests and can only use browser compatible API. Cloudflare workers cannot benefit from Express, Password.js, or the Mongodb driver because those libraries either expect the Node.js runtime, or use a protocol other than regular HTTPS calls. As a workaround, the community created their libraries in order to emulate those functionality.

#### Webpack.config.js

In order to allow imports, it is advisable to set up webpack in order to combine the code into one javascript file.

```javascript
const webpack = require("webpack")
const path = require('path')

module.exports = {
 target: "webworker",
    entry: "./src/index.js", // inferred from "main" in package.json
 resolve: {
  extensions: ['.ts', '.tsx', '.js'],
  plugins: []
 },
 optimization: {
  minimize: false,
   },
 output: {
  path: __dirname + "/dist",
  publicPath: "dist",
  filename: "worker.js"
 }
}
```

#### Routes

To add routes to the endpoint, Cloudflare allows the user to access calling information by attaching a listener. The listener returns a JSON payload that can be passed into a router and allows different features within the same codebase. Each event has basic information to construct a router library.

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
  console.log(event)
})
```

```json
{
  "request": {
    "fetcher": {},
    "redirect": "manual",
    "headers": {},
    "url": "https://app-api.username.workers.dev/foo",
    "method": "GET",
    "bodyUsed": false,
    "body": null
  },
  "type": "fetch"
}
```

### Python

In contrast, Cloudflare workers' requirement to compile down to Javascript has neutered and alienated Python from the rest of the ecosystem. Wranglers uploads Python by converting the code into Javascript with Transcrypt. The way that Python is separate from its runtime is similar to the scars of the fabled 2to3 transition. In any programming language, there is a concept of moving the executable to another environment and observing the same output which is called an application binary interface. For example, a Windows C binary can run on all systems that speak the Window C API. This ABI has been emulated on other OS through projects such as WINE is not an emulator and Transgaming. Contrary to compiled languages similar to C, scripting language's ABI is the code itself. Code tends to have more concepts and a larger surface area than the binary itself. As a consequence, any major changes to runtime can reclassify the language as something different despite the similarities. While writing Transcrypt code, I felt as if I was writing Javascript code in Python.

#### Language differences

In the example below, I attempted to iterate the dictionary with `keys()` but the compiler complained. I had to use `items()` instead.

```python
#Uncaught (in response) TypeError: Cannot read property 'py_items' of undefined
#
#worker.js:3517 Uncaught (in promise) TypeError: Cannot read property 'py_items' of undefined
#    at VM3 worker.js:3517
#    at handle_post_echo (VM3 worker.js:3521)
#    at async handleRequest (VM3 worker.js:3534)

str([(k,form_data[k]) for k in form_data.keys()])
```

#### Package Imports

Transcrypt cannot find single file imports within the local directory. In order to move the code into different files, you layout your source tree as a package and import the code into your main handler.

## Database

Fauna introduces the Fauna Query Languages for its serverless systems. FQL is not a declarative language and has attributes closer to NoSql database. For a basic auction site, I created a basic schema within a week for my auction.

### Tables

Faunadb divides tables into collections. Unlike SQL databases, collections never impose a schema which allows developers to have the flexibility to add or remove fields within each entry or document without any type of data migration.

### Fields

Moreover, Faunadb duck tapes the schema with `Index`. Once an index is made, you can only delete and wait until Faunadb cleans up the index before the name can be reused. index displays an arbitrary lens into your dataset. Each lens can be modified to suit your needs. All the data transformation will be done in FQL.

### Progamming

Deep down, FQL is a Turing Complete Functional Language. FQL formats all queries to be shaped as an Abstract Syntax Tree with nested JSON statements. Once anyone masters the keyword `Let`, variable assignment, FQL feels no different from any functional language with routines such as `Map` or `Filter`. As an added benefit, an entire query represents an ACID compliant transaction.

### a. Largest Bid

As an example below, here is an FQL query that finds the max value by selecting the first element in a sorted index and passing the unique id to be formatted into the final output with a series of unions.

```FQL
Map(
  Max(
    Filter(
      Paginate(Match(Index("bid_by_amount_desc"))),
      Lambda("Y", 
      Let({ auctionRef: Select(1, Var("Y"))},
        Equals(Select(["data", "name"], Get(Var("auctionRef"))), "dummy_auction_name")
      )
    )
  )),
  Lambda(
    "X", Let({
      auction: Get(Select(1, Var("X"))), bid: Get(Select(2, Var("X")))
    },
    Merge(
      Merge(Select(["data"], Var("bid")), {"ts": Select("ts", Var("bid"))}),
      Merge( 
        {"auction_end": Select(["data", "auction_end"], Var("auction")) },
        {"auction_name": Select(["data", "name"], Var("auction")) }
      ),
    )
  )
  )
 )
)
CreateIndex({
  name: "bid_by_amount_desc",
  source: Collection("bid"),
  values:  [
    { field: [ "data", "amount" ], reverse: true },
    { field: [ "data", "auctionRef" ] },
    { field: [ "ref" ] },
  ]
})
```

#### b. Create Bid

Within FQL, anyone can script various failure modes and custom outputs such that your query can display custom information on how it fails. This query forbids duplicate bids and returns the current last bid. This query can be further improve by allowing developers can discern the bugs with custom error codes.

```FQL
Let(
  {
    maxBid: Max(
      Filter(
        Paginate(Match(Index("bid_by_amount_desc"))),
        Lambda(
          "Y",
          Let(
            { auctionRef: Select(1, Var("Y")) },
            Equals(Select(["data", "name"], Get(Var("auctionRef"))), "dummy")
          )
        )
      )
    )
  },
  Let(
    {
      amount: Select(["data", 0, 0], Var("maxBid")),
      bid: Get(Select(["data", 0, 2], Var("maxBid"))),
      timestamp: Now()
    },
    If(
      GT(Var("amount"), 700),
      Var("bid"),
      If(
        Equals(Select(["data", "email"], Var("bid")), "dummy4@example.com"),
        Var("bid"),
        Create(Collection("bid"), {
          data: {
            email: "dummy4@example.com",
            name: "dummy4",
            amount: 700,
            timestamp: Var("timestamp"),
            auctionRef: Select(
              "ref",
              Get(Match(Index("auction_by_name"), "dummy"))
            )
          }
        })
      )
    )
  )
)
```

### Costs

Regrettably, costs can be difficult to predict. A single query can contain many write or read ops. Since FQL is a functional language, calculating costs can be similar to reasoning out the big O notation.

#### a. Problem

Within the documented SQL to FQL example below, `Match` and `Get` imposes one read ops each. Combining Get with Map turns a from a constant time algorithm into linear consumption of read ops. Pretend that dept has 5 documents. The total number of read ops will be 6 because this query will execute 5 `Get` and 1 `Match`.

```FQL
Map(
  Paginate(
    Match(Index("all_depts"))
  ),
  Lambda("X", GET(Var("X")))
)

CreateIndex({
  name: "all_depts",
  source: Collection("dept")
})
```

#### b. Solution

As a workaround, Fauna suggests all users should create specialized indexes that also expose even more document data than the unique ID. This workaround would pollute the namespace such that your application may be filled with arbitrary index names similar to SQL Views.

```FQL
Paginate(Match(Index("all_depts_with_fields")))

CreateIndex({
  name: "all_depts_with_fields",
  source: Collection("dept")
  values:  [
    { field: ['data', 'dname'] }, 
    { field: ['data', 'deptno'] },
    { field: ['ref'] }]
})
```

As a result, it is not that difficult to consume 7 or more read ops per web page load. Futhermore, all those reads are accumulated at the end of the month.

## Conclusion

As usual, I am grateful for Faunadb and Cloudflare for providing such generous free tiers to allow me to develop and deploy an auction application within a month from start to finish. Thank you.

* <https://egghead.io/lessons/faunadb-reducing-the-number-of-read-ops-in-a-query-using-indexes>
* <https://mikhail.io/serverless/coldstarts/aws/languages/>
* <https://docs.fauna.com/fauna/current/api/fql/functions/let?lang=shell>
* <https://developers.cloudflare.com/workers/learning/how-workers-works>
* <https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/>
* <https://pages.awscloud.com/rs/112-TZM-766/images/2020_0316-SRV_Slide-Deck.pdf>
* <https://blog.cloudflare.com/using-webpack-to-bundle-workers/>
* <https://github.com/cloudflare/python-worker-hello-world>
* <https://fauna.com/blog/getting-started-with-fauna-and-cloudflare-workers>
* <https://blog.cloudflare.com/partnership-announcement-db/>
* <https://github.com/cloudflare/wrangler/issues/543>
* <https://github.com/cloudflare/worker-template-router>
* <https://blog.cloudflare.com/using-webpack-to-bundle-workers/>
* <https://blog.cloudflare.com/code-everywhere-cloudflare-workers/>
