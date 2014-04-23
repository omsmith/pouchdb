---
layout: post

title: Secondary indexes have landed in PouchDB

author: Nolan Lawson

---

With the release of PouchDB 2.2.0, we're happy to introduce a feature that's been cooking on the slow simmer for some time: secondary indexes, a.k.a. persistent map/reduce.

This is a powerful new tool for developers, since it allows you to index anything in your JSON documents &ndash; not just the doc IDs. Your data is sortable and searchable in ways that just weren't feasible before, thanks to a new cross-platform indexing engine we've built to work with every supported backend - IndexedDB, WebSQL, and LevelDB (and soon: LocalStorage!).

And did I mention?  It's fast.  Our [performance tests](https://gist.github.com/nolanlawson/11100235) show that the new persistent map/reduce API could give you orders of magnitude improvements over the old on-the-fly `query()` API.  In a database containing 1,000 documents, it's anywhere between 20 and 100 times faster.


Ew, you put map/reduce in my JavaScript?
------------

First, let's define what map/reduce is, so you can understand why you might want it.

#### Indexes in SQL databases

Here's a quick refresher on how indexes work.  In classic relational databases like MySQL and PostgreSQL, you can typically query whatever field you want:

    SELECT * FROM pokemon WHERE name = 'Pikachu';
    
But if you don't want your performance to be terrible, you first add an index:

    ALTER TABLE pokemon ADD INDEX myIndex ON (name);
    
The job of the index is to ensure the the field is stored in a B-tree within the database, so your queries run in _O(log(n))_ time instead of _O(n)_ time.

#### Indexes in NoSQL databases

All of the above is also true in document stores like CouchDB and MongoDB, but conceptually it's a little different. By default, documents are assumed to be schemaless blobs with one primary key (called `_id` in both Mongo and Couch), and any other keys need to be specified separately.  The concepts are largely the same; it's mostly just the vocabulary that's different.
 
In CouchDB, queries are called _map/reduce functions_.  This is because, as in many NoSQL databases, CouchDB is designed to scale well across multiple nodes, and to perform efficient query operations in parallel.  Basically, the idea is that you divide your index into a _map_ function and a _reduce_ function, each of which may be executed in parallel in a multi-node cluster.

#### Map functions

It may sound daunting at first, but in the simplest (and most common) case, you only need the _map_ function.  A basic map function might look like this:

```js
function (doc) {
  emit(doc.name);
}
```  

This is functionally equivalent to the SQL index given above.  What it basically says is: "for a given document, emit its name as a key."

And since it's just JavaScript, you're allowed to get as fancy as you want here:

```js
function (doc) {
  if (doc.type === 'pokemon') {
    if (doc.name === 'Pikachu') {
      emit('Pika pi!');
    } else {
      emit(name);
    }
  }
}
```

#### Reduce functions

As for _reduce_ functions, there are basically a few handy built-ins that do aggregate operations (e.g. `'_sum'` and `'_count'`), and you can typically steer clear of writing your own. But if you're adventurous, you can check out the [CouchDB documentation](http://couchdb.readthedocs.org/en/latest/couchapp/views/intro.html) for details.

What map/reduce is capable of
------------

To see how map/reduce performs in practice, let's walk through a simple example.  Say we have a database full of aging rock stars, which looks like this:

```js
[{
      "_id"  : "bowie",
      "name" : "David Bowie",
      "age"  : 67
    }, {
      "_id"  : "dylan",
      "name" : "Bob Dylan",
            "age"  : 72
    }, {
      "_id"  : "younger_dylan",
      "name" : "Jakob Dylan",
            "age"  : 44
    }, {
      "_id"  : "hank_the_third",
      "name" : "Hank Williams",
            "age"  : 41
    }, {
      "_id"  : "hank",
      "name" : "Hank Williams",
            "age"  : 91
    }]
```

If you call `db.allDocs()`, you'll just get every document sorted by its `_id`:

```js
    {
      "id": "bowie",
      "key": "bowie",
      "value": {
        "rev": "1-e018c5bbceba92e52bf4e46b503828f5"
      }
    }, {
      "id": "dylan",
      "key": "dylan",
      "value": {
        "rev": "1-1ce006d980cc690d89787bbf2241dc01"
      }
    }, {
      "id": "hank",
      "key": "hank",
      "value": {
        "rev": "1-38801b1e21b177f5479625a6a3899659"
      }
    }, {
      "id": "hank_the_third",
      "key": "hank_the_third",
      "value": {
        "rev": "1-cfda51b9e3966d9c36d2c8fcc99cae3b"
      }
    }, {
      "id": "younger_dylan",
      "key": "younger_dylan",
      "value": {
        "rev": "1-d884b08a9b759274bf49c7d994ca8542"
      }
    }
```

{% include alert_start.html variant="info"%}

If you don't specify a doc's <code>_id</code>, it will be auto-generated, e.g. <code>'36483B8A-DF4A-4AEB-B946-096BA0FA8813'</code>.

{% include alert_end.html %}

However, with the `query()` API, you can sort these rockers (somewhat unflatteringly) by their age:

```js
function (rocker) {
  emit(rocker.age);
}

```

This returns:

```js
[ { id: 'hank_the_third', key: 41, value: null },
     { id: 'younger_dylan', key: 44, value: null },
     { id: 'bowie', key: 67, value: null },
     { id: 'dylan', key: 72, value: null },
     { id: 'hank', key: 91, value: null } ] 
```

Or by their last name:

```js
function (rocker) {
  emit(rocker.name.split(' ')[1]);
}

```

This returns:

```js
   [ { id: 'bowie', key: 'Bowie', value: null },
     { id: 'dylan', key: 'Dylan', value: null },
     { id: 'younger_dylan', key: 'Dylan', value: null },
     { id: 'hank', key: 'Williams', value: null },
     { id: 'hank_the_third', key: 'Williams', value: null } ] }
```

A `reduce` function runs aggregate operations against whatever's emitted in the `map` function.  So for instance, we can also use that to count up the number of rockers by first name:

```js
{
  map: function (rocker) {
    emit(rocker.name.split(' ')[0]);
  },
  reduce: '_count'
}
```

This returns:

```js
{ rows: 
   [ { key: 'Bob', value: 1 },
     { key: 'David', value: 1 },
     { key: 'Hank', value: 2 },
     { key: 'Jakob', value: 1 } ] }
```

There are also custom reduce functions, complex keys, `group_level`, closures, and other neat tricks that are beyond the scope of this article, so check out [the PouchDB documentation](http://pouchdb.com/api.html#query_database) for details.

#### Paginating views

Just like `allDocs()`, you can paginate over your saved views. **TODO write more stuff here.**


Map/reduce, reuse, recycle
----------

PouchDB actually already had map/reduce since way before version 2.2.0, and all the code above would have worked. It would have had a big flaw, though: before 2.2.0, all queries were performed in-memory, reading every document the entire database just to perform a single operation.

This is fine for small data sets, but it could be murder for large databases. On mobile devices especially, memory is a precious commodity, so you want to be as frugal as possible with the resources given to you by the OS.

The new persistent `query()` method is much more memory-efficient, and it won't read in the entire database unless you tell it to.  (Read up on pagination with `startkey` and `skip` to understand how to avoid this.)  It has two modes:

* Temporary views
* Persistent views (new)

Both of these concepts exist in CouchDB, and they're faithfully emulated in PouchDB.

#### A room with a view

First off, some vocabulary: CouchDB technically calls indexes _views_, and these views are stored in a special document called a _design document_. Basically, a design document describes a view, and a view describes a map/reduce query, which tells the database that you plan to use that query later, so it better start indexing it now. In other words, creating a view is the same as saying `CREATE INDEX` in a relational database.

**Temporary views** are just that &ndash; temporary. They create a view, write an index to the database, query the view, and then poof, they're deleted.  That's why in older version of PouchDB, we could get away by doing it all in-memory.

Crucially, though, temporary views have to do a full table scan every time you execute them (along with all the reading and writing to disk that that entails), just to throw it away at the end.  That may be fine for quick testing during development, but in production it can be a big drag on performance.

**Persistent views**, however, are a much better solution if you want your queries to be fast.  Persistent views require you to first save a design document containing your view, so that the emitted fields are already indexed by the time you need to look them up.  Subsequent lookups don't need to do any additional writes on the database, unless documents have been added or modified, in which case only the diff needs to be applied.

{% include alert_start.html variant="warning"%}

Technically, the view will not be built up on disk until after the first time you <code>query()</code> it.  A good pattern is to always call <code>query({stale: 'update after'})</code> after creating a view, to ensure that it starts building in the background.

{% include alert_end.html %}

#### Tips for writing views

It's helpful to remember that, whatever you write for your map function, it will have to be executed for each document in your database.  Thus you want to minimize the number of times you create an index.

For example, here's a pattern I've seen in a lot of PouchDB code, which you should avoid from now on:

```js
function myMapFunction(doc) {
  if (doc.name === 'foobar') {
    emit(doc.name);
  }
}
db.query(myMapFunction).then(function (res) {
  console.log(res.rows); // prints the "foobar" doc
});
```

A better pattern is this:

```js
function myMapFunction(doc) {
  emit(doc.name);
}
// save the view...
db.query('my_view', {key: 'foobar'}).then(function (res) {
  console.log(res.rows); // prints the "foobar" doc
});
```

Both of these examples will do the same thing (find all docs with the name `'foobar'`) but the second one will only need to build the index once. Plus, from then on you can query for `'foobar'`, or `'foobaz'`, or whatever you want, by using the `key` option.

So don't forget: the same pagination options that work for `allDocs()` also work for `query()`!

For those who get nostalgic
----

Of course, some of our loyal Pouchinistas may have been perfectly satisfied with the old way of doing things. The fact that the `query()` function read the entire database into memory might actually have served them just fine, thank you.

To be sure, there are some legitimate use cases for this. If you don't store a lot of data, or if all your users are on desktop computers, or if you require closures (not supported by persistent views), then doing map/reduce in memory may have been fine for you. In fact, an upgrade to 2.2.0 might actually make your app perform more _slowly_, because now it has to read and write to disk.

Luckily we have a great solution for you: instead of using the `query()` API, you can use the much more straightforward `allDocs()` API.  This will read all documents into memory, just like before, but best of all, you can apply whatever functions you want to the data, without having to bother with learning a new API or even what the heck "map/reduce" is.

When *not* to use map/reduce
----

Now that I've sold you on how awesome map/reduce is, let's talk about the situations where you might want to avoid it.

First off, you may have noticed that the CouchDB map/reduce API is pretty daunting.  As a newbie Couch user who very recently struggled with the API, I can empathize: the vocabulary is new, the concepts are (probably) new, and there's no easy way to learn it, except to clack at your keyboard for awhile and try it out.

Second off, views take quite some time to build up, both in CouchDB and PouchDB. Each document has to be read into memory in order for the map function to be applied to it, and this can take awhile for larger databases. If your database is constantly changing, you will also incur the penalty at `query()` time of running the map function over the changed documents.

Luckily, it turns out that the primary indexes provided by both CouchDB and PouchDB are often good enough for a variety of querying and sorting operations. If you're clever about how you name your document `_id`s, you can avoid using map/reduce altogether.

For instace, say your documents are emails that you want to sort by recency: boom, set the `_id` to `new Date().toJSON()`.  Or say they're web pages that you want to look up by URL: use the URL itself as the `_id`! Neither CouchDB nor PouchDB have a limit on how long your `_id`s can be, so you can get as fancy as you want with this.

For a more elaborate example, let's imagine your database looks like this:

```js
[ { _id: 'artist_bowie',
    type: 'artist',
    name: 'David Bowie',
    age: 67 },
  { _id: 'artist_dylan',
    type: 'artist',
    name: 'Bob Dylan',
    age: 72 },
  { _id: 'album_bowie_ziggy_stardust',
    artist: 'artist_bowie',
    title: 'The Rise and Fall of Ziggy Stardust and the Spiders from Mars',
    type: 'album',
    year: 1972 },
  { _id: 'album_bowie_hunky_dory',
    artist: 'artist_bowie',
    title: 'Hunky Dory',
    type: 'album',
    year: 1971 },
  { _id: 'album_dylan_highway_61',
    artist: 'artist_dylan',
    title: 'Highway 61 Revisited',
    type: 'album',
    year: 1965 } ]
```

Notice what we've done here: artist-type documents are prefixed with `artist_`, and album-type documents are prefixed with `album`.  This naming scheme is clever enough that we can already do lots of complex queries using `allDocs()`, even though we're storing two different types of documents.

Want to find all artists?  It's just `allDocs({startkey: 'artist_', endkey: 'artist__'})`.  

Want to find all albums?  Try `allDocs({startkey: 'albums_', endkey: 'albums__'})`.

Want to find all albums by David Bowie?  Wham bam, thank you ma'am: `allDocs({startkey: 'album_bowie_', endkey: 'album_bowie__'})`.

In this example, you're getting all those "indexes" for free, each time a document is added to the database.  It doesn't take up any additional space on disk compared to UUIDs, and you neither have to wait for a view to get built up, nor do you have to understand the map/reduce API at all.

Of course, this system starts to get shaky when you need to search by a variety of criteria: e.g. albums sortedy by year, artists sorted by age, etc. But for a lot of simple apps, you should be able to get by without using the `query()` API at all.

{% include alert_start.html variant="info"%}

<strong>Performance tip</strong>: if you're just using the randomly-generated doc `_id`s, then you're not only missing out on an opportunity to get a free index &ndash; you're also incurring the overhead of a building an index you're never going to use!

{% include alert_end.html %}


For the nerds: implementation details
-------

If you want to know how map/reduce works in PouchDB, it's actually exceedingly easy: just open up your dev console, and see where we've created a bonus database to store the index.

One of the neat things about how we implemented map/reduce is that we actually did it entirely with PouchDB itself.  Yes folks, that's right: your map/reduce view is just a regular old PouchDB database, and map/reduce itself is just a plugin with no special privileges, creating databases in exactly the same way a regular user would.  The map/reduce code is exactly the same for all three backends: LevelDB, IndexedDB, and WebSQL.

And just to be clear: it's not like we set out to make PouchDB some kind of map/reduce engine.  In fact, as we were debating how to implement this feature, we originally ran with the assumption that we'd need to build additional functionality on top of PouchDB, and it was only after a few months of work that we realized the whole thing could be done in PouchDB proper.  In the end, the only additional functionality that had to be added was a hook to tell PouchDB to destroy the map/reduce database when its parent database was destroyed.  That's it.

This fact also has a nice parallel in CouchDB: it turns out that they, too, reused most of the core view code itself to implement the database.  As they say in the docs, `_all_Docs` is just a special kind of view.

As for the index itself, what it basically boils down to is clever serialization of arbitrary JSON values into CouchDB-style collation ordering (e.g. booleans before ints, ints before strings, strings before arrays, arrays before objects, and so on recursively).

Then, we take this value and concatenate it with whatever else needs to be indexed (usually just the doc `_id`), meaning the entire lookup key is a single string.  We decided on this implementation because:

1. LevelDB does not support complex indexes.
2. IndexedDB does support complex indexes, but not in IE.
3. WebSQL supports multi-field indexes, but it's also a dead spec, so we're not going to prioritize it above the others.

Since we're storing many types of documents in a single database that must be synced with one another, and since by design CouchDB/PouchDB do not have multi-document transaction semantics, we implemented a task queue on top of the database to serialize writes.  Interestingly, this is the same approach Google took when they implemented IndexedDB on top of LevelDB, and it's similar to what we're currently doing for our own LevelDB adapter (although since we don't need full transactions, we avoid a full task queue for most operations, and just use optimistic locks instead).

Aside from that, the only other neat trick is a liberal use of `_local` documents, which are a special class of documents that aren't counted in the `total_rows` count, don't show up in `allDocs`, but may be accessed using the normal `put()` and `get()` methods.  This is all it takes to fully reimplement CouchDB map/reduce in a tiny JavaScript database!

Anyone interested in more implementation details can read the months-long discussions in mapreduce/12, 1549, and 1658.  The code itself is also pretty succinct.