# Advanced Usage

- [Understanding Catalogs](#understanding-catalogs) (searching, filtering, paginating)
- [Understanding Cinemeta](#understanding-cinemeta)
- [Adding Stream Results to Cinemeta Items](#adding-stream-results-to-cinemeta-items)
- [Getting Metadata from Cinemeta](#getting-metadata-from-cinemeta)
- [Resolving Movie / Series names to IMDB ID](#resolving-movie--series-names-to-imdb-id)
- [Using User Data in Add-ons](#using-user-data-in-add-ons)
- [Using Internal Links in Add-ons](#using-internal-links-in-add-ons)
- [Proxying Other Add-ons](#proxying-other-add-ons)
- [Crawler (Scraping) Add-ons](#crawler--scraping-add-ons)


## Understanding Catalogs

The `catalog` resource in Stremio add-ons can be used to:
- show one or more catalogs in the Board and Discover pages, these responses can also be filtered and paginated
- show search results from catalogs

Let's first look at how `catalog` is declared in the [manifest](./api/responses/manifest.md):
```json
{
  "resources": ["catalog"],
  "catalogs": [
    {
      "id": "testcatalog",
      "type": "movie"      
    }
  ]
}
```

This is normally all you'd need to make a standard catalog, but it won't support filtering, paginating and it won't be searchable.


### Searching in Catalogs

To state that your catalog supports searching, you'd need to set it in `extraSupported`:

```json
catalogs: [
  {
    "id": "testcatalog",
    "type": "movie",
    "extra": [
      {
        "name": "search",
        "isRequired": false
      }
    ]
  }
]
```

But then, what if you want your catalog to support only search (as in, not show in the Board or Discover pages at all)?

Then you'd need to state that your catalog supports only searching, and you can do that with:

```json
catalogs: [
  {
    "id": "testcatalog",
    "type": "movie",
    "extra": [
      {
        "name": "search",
        "isRequired": true
      }
    ]
  }
]
```

Once you've set `search` in `extraSupported`, your catalog handler will receive `args.extra.search` as the search query (if it is a search request), so here's an example of a search response:

```javascript
const meta = {
  id: 'tt1254207',
  name: 'Big Buck Bunny',
  year: 2008,
  poster: 'https://image.tmdb.org/t/p/w600_and_h900_bestv2/uVEFQvFMMsg4e6yb03xOfVsDz4o.jpg',
  posterShape: 'regular',
  banner: 'https://image.tmdb.org/t/p/original/aHLST0g8sOE1ixCxRDgM35SKwwp.jpg',
  type: 'movie'
}
addon.defineCatalogHandler(function(args) {
  if (args.id == 'testcatalog') {
    // this is a request to our catalog id
    if (args.extra && args.extra.search) {
      // this is a search request
      if (args.extra.search == 'big buck bunny') {
        // if someone searched for "big buck bunny" (exact match)
        // respond with our meta item
        return Promise.resolve({ metas: [meta] })
      } else {
        // otherwise with no meta item
        return Promise.resolve({ metas: [] })
      }
    } else {
      // this is a standard catalog request
      // just respond with our meta item
      return Promise.resolve({ metas: [meta] })
    }
  } else {
    // this is not a request for our catalog
    return Promise.resolve({ metas: [] })
  }
})
```


### Filtering in Catalogs

Maybe you would like your catalog to be filtered by `genre`, in this case, we'll set that in the catalog definition:

```json
catalogs: [
  {
    "id": "testcatalog",
    "type": "movie",
    "extra": [
      {
        "name": "genre",
        "options": [ "Drama", "Action" ],
        "isRequired": false
      }
    ]
  }
]
```

Now we'll receive `genre` in our catalog handler:

```javascript
const meta = {
  id: 'tt1254207',
  name: 'Big Buck Bunny',
  year: 2008,
  poster: 'https://image.tmdb.org/t/p/w600_and_h900_bestv2/uVEFQvFMMsg4e6yb03xOfVsDz4o.jpg',
  posterShape: 'regular',
  banner: 'https://image.tmdb.org/t/p/original/aHLST0g8sOE1ixCxRDgM35SKwwp.jpg',
  type: 'movie'
}
addon.defineCatalogHandler(function(args) {
  if (args.id == 'testcatalog') {
    // this is a request to our catalog id
    if (args.extra && args.extra.genre) {
      // this is a filter request
      if (args.extra.genre == "Action") {
        // in this example we'll only respon with our
        // meta item if the genre is "Action"
        return Promise.resolve({ metas: [meta] })
      } else {
        // otherwise with no meta item
        return Promise.resolve({ metas: [] })
      }
    } else {
      // this is a standard catalog request
      // just respond with our meta item
      return Promise.resolve({ metas: [meta] })
    }
  } else {
    // this is not a request for our catalog
    return Promise.resolve({ metas: [] })
  }
})
```


## Pagination in Catalogs

If we want our catalogs to be paginated, we can use `skip` as follows:

```json
catalogs: [
  {
    "id": "testcatalog",
    "type": "movie",
    "extra": [
      {
        "name": "skip",
        "isRequired": false
      }
    ]
  }
]
```

Optionally, we can also set the steps in which the catalog will request the next items, for example:

```json
catalogs: [
  {
    "id": "testcatalog",
    "type": "movie",
    "extra": [
      {
        "name": "skip",
        "options": ["0", "100", "200"],
        "isRequired": false
      }
    ]
  }
]
```

This is not a requirement though, as if we don't set `options` Stremio will request `skip` once it comes to the end of your catalog.

Here's an example of using `skip`:

```javascript
// we only have one meta item
const meta = {
  id: 'tt1254207',
  name: 'Big Buck Bunny',
  year: 2008,
  poster: 'https://image.tmdb.org/t/p/w600_and_h900_bestv2/uVEFQvFMMsg4e6yb03xOfVsDz4o.jpg',
  posterShape: 'regular',
  banner: 'https://image.tmdb.org/t/p/original/aHLST0g8sOE1ixCxRDgM35SKwwp.jpg',
  type: 'movie'
}

const metaList = []

// but we'll make an array that includes our meta 60 times
for (let i = 0; i++; i < 60) {
  metaList.push(meta)
}

addon.defineCatalogHandler(function(args) {
  if (args.id == 'testcatalog') {
    // this is a request to our catalog id
    if (args.extra && args.extra.skip) {
      // this is a skipped catalog request
      // we'll slice our meta list using
      // skip as the starting point
      return Promise.resolve({ metas: metaList.slice(args.extra.skip, 20) })
    } else {
      // this is a standard catalog request
      // we'll respond with the first 20 items
      return Promise.resolve({ metas: metaList.slice(0, 20) })
    }
  } else {
    // this is not a request for our catalog
    return Promise.resolve({ metas: [] })
  }
})
```


## Understanding Cinemeta

Cinemeta is the primary add-on that Stremio uses to show Movie, Series and Anime items. Other add-ons can choose to create their own catalogs of items or respond with streams to the Cinemeta items.

Cinemeta uses IMDB IDs for their metadata, to understand it's pattern:
- `tt0111161` is the meta ID (and stream request ID) of a movie
- `tt3107288` is the meta ID of a series, and `tt3107288:1:1` is the request ID for streams for season 1, episode 1 of the series with the `tt3107288` ID


## Adding Stream Results to Cinemeta Items

To add only stream results to Cinemeta items, you will first need to state that your add-ons id prefix is `tt` (as for IMDB IDs).

Add these to your [manifest](./api/responses/manifest.md):
- `resources: ["stream"]`
- `idPrefixes: ["tt"]`

Now here is an example of returning stream responses for Cinemeta items:

```javascript
addon.defineStreamHandler(function(args) {
  if (args.type === 'movie' && args.id === 'tt1254207') {
    // serve one stream for big buck bunny
    const stream = { url: 'http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4' }
    return Promise.resolve({ streams: [stream] })
  } else if (args.type === 'series' && args.id === 'tt8633518:1:1') {
    // return one stream for the series Weird City, Season 1 Episode 1
    // (free Youtube Originals series)
    const stream = { id: 'yt_id::fMnq5v8yZp4' }
    return Promise.resolve({ streams: [stream] })
  } else {
    // otherwise return no streams
    return Promise.resolve({ streams: [] })
  }
})
```


## Getting Metadata from Cinemeta

There might be cases where you would need metadata based on IMDB ID. To do this, you will need both IMDB ID and the type of the item (either `movie` or `series`).

Because Cinemeta is also an add-on, you can request the metadata from it.

Here is an example using `needle` to do a HTTP request to Cinemeta for metadata:

```javascript
var needle = require('needle')

// we will get metadata for the movie: Big Buck Bunny

var itemType = 'movie'
var itemImdbId = 'tt1254207'

needle.get('https://v3-cinemeta.strem.io/meta/' + itemType + '/' + itemImdbId + '.json', function(err, resp, body) {

  if (body && body.meta) {

    // log Big Buck Bunny's metadata
    console.log(body.meta)

  }

})
```


## Resolving Movie / Series names to IMDB ID

What if you have a movie or series name, but you need it's IMDB ID?

We recommend using [name-to-imdb](https://github.com/Ivshti/name-to-imdb) in this case, and it's really easy to use:

```javascript
var nameToImdb = require("name-to-imdb");

nameToImdb({ name: "south park" }, function(err, res, inf) { 
  console.log(res); // prints "tt0121955"
  console.log(inf); // inf contains info on where we matched that name - e.g. metadata, or on imdb
})
```

Also setting the `type` and `year` in the request helps on ensuring that the IMDB ID that is returned is correct.


## Using User Data in Add-ons

This example does not use the Stremio Add-on SDK, it uses Node.js and Express to serve replies.

User data is passed in the Add-on Repository URL, so instead of users installing add-ons from the normal manifest url (for example: `https://www.mydomain.com/manifest.json`), users will also need to add the data they want to pass to the add-on in the URL (for example: `https://www.mydomain.com/c9y2kz0c26c3w4csaqne71eu4jqko7e1/manifest.json`, where `c9y2kz0c26c3w4csaqne71eu4jqko7e1` could be their API Authentication Token)

Simplistic Example:

```javascript
const express = require('express')
const addon = express()

addon.get('/:someParameter/manifest.json', function (req, res) {
  res.send({
    id: 'org.parameterized.'+req.params.someParameter,
    name: 'add-on for '+req.params.someParameter,
    resources: ['stream'],
    types: ['series'],
  })
})

addon.get('/:someParameter/stream/:type/:id.json', function(req, res) {
  // @TODO do something depending on req.params.someParameter
  res.send({ streams: [] })
})

addon.listen(7000, function() {
  console.log('http://127.0.0.1:7000/[someParameter]/manifest.json')
})
```

This is not a working example, it simply shows how data can be inserted by users in the Add-on Repository URL so add-ons can then make use of it.

For working examples, you can check these add-ons:
- [IMDB Lists](https://github.com/jaruba/stremio-imdb-list)
- [IMDB Watchlist](https://github.com/jaruba/stremio-imdb-watchlist)
- [Jackett Add-on for Stremio](https://github.com/BoredLama/stremio-jackett-addon) (community built)

Another use case for passing user data through the Add-on Repository URL is creating proxy add-ons. This case presumes that the id of a different add-on is sent in the Add-on Repository URL, then the proxy add-on connects to the add-on of which the id it got, requests streams, passes the stream url to some API (for example Real Debrid, Premiumize, etc) to get a different streaming url that it then responds with for Stremio.


## Using Internal Links in Add-ons

Stremio supports [internal links](./internal-links.md), such links can also be used in add-ons to link internally to Stremio.

First, set the `stream` resource in your [manifest](./api/responses/manifest.md):
- `resources: ["stream"]`

Here's an example:

```javascript
// this responds with one stream for the Big Buck Bunny
// movie, that if clicked, will redirect Stremio to the
// Board page
addon.defineStreamHandler(function(args) {
  if (args.type === 'movie' && args.id === 'tt1254207') {
    // serve one stream for big buck bunny
    const stream = { externalUrl: 'stremio://board' }
    return Promise.resolve({ streams: [stream] })
  } else {
    // otherwise return no streams
    return Promise.resolve({ streams: [] })
  }
})
```


## Proxying Other Add-ons

Stremio add-ons use a HTTP server to handle requests and responses, this means that other add-ons can also request their responses.

This can be useful for many reasons, a guide on how this can be done is included in the readme of the [IMDB Watchlist](https://github.com/jaruba/stremio-imdb-watchlist) Add-on which proxies the [IMDB Lists](https://github.com/jaruba/stremio-imdb-list) add-on to get the IMDB List for a particular IMDB user.

IMDB Watchlist only proxies the catalog from IMDB Lists, to proxy other resources you can use the same pattern as IMDB Watchlist does, and check the endpoints and patterns for other resources on the [Protocol Documentation](./protocol.md) page.



## Crawler / Scraping Add-ons

Scraping HTML pages presumes downloading the HTML source of a web page in order to get specific data from it.

A guide showing a simplistic version of doing this is in the readme of the [IMDB Watchlist Add-on](https://github.com/jaruba/stremio-imdb-watchlist). The add-on uses [needle](https://www.npmjs.com/package/needle) to request the HTML source and [cheerio](https://www.npmjs.com/package/cheerio) to start a jQuery instance in order to simplify getting the desired information.

