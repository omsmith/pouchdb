---
layout: post

title: Secondary indexes have landed in PouchDB

author: Nolan Lawson

---

With the release of PouchDB 2.2.0, we're happy to introduce a feature that's been cooking on the slow simmer for some time: secondary indexes, a.k.a. persistent map/reduce.

This is a powerful new tool for developers, since it allows you to index anything in your JSON documents &ndash; not just the doc IDs. Starting in 2.2.0, your data is sortable and searchable in ways that just weren't feasible before, thanks to a new cross-platform indexing engine we've built to work with every supported backend - IndexedDB, WebSQL, and LevelDB (and soon: LocalStorage!).

And did I mention?  It's fast.  Our [performance tests](https://gist.github.com/nolanlawson/11100235) show that the new persistent map/reduce API could give you orders of magnitude improvements over the old on-the-fly `query()` method.  We tested a database containing 1,000 documents, and PouchDB 2.2.0 was between 20 and 100 times faster than PouchDB 2.1.2.

The old way you might query:

```js
var pouch = new PouchDB('mydb');

// query in-memory (slow)
pouch.query(function (doc) {
  emit(doc.name);
});
```

The new way you should query:

```js
var pouch = new PouchDB('mydb');

// create a view (i.e. an index)
pouch.putView('my_index', function (doc) {
  emit(doc.name);
}).catch(function (err) {
  // view already exists, ignore
}).then(function () {
  // query the index (much faster)
  pouch.query('my_index');
})

```


Ew, you put map/reduce in my JavaScript?
------------

First, let's define what map/reduce is, so you can understand why you might want it.

#### Indexes in SQL databases

Here's a quick refresher on how indexes work.  In classic relational databases like MySQL and PostgreSQL, you can query whatever field you want:

```sql
SELECT * FROM pokemon WHERE name = 'Pikachu';
```
    
But if you don't want your performance to be terrible, you first add an index:

```sql
ALTER TABLE pokemon ADD INDEX myIndex ON (name);
```
    
The job of the index is to ensure the the field is stored in a B-tree within the database, so your queries run in _O(log(n))_ time instead of _O(n)_ time.

#### Indexes in NoSQL databases

All of the above is also true in document stores like CouchDB and MongoDB, but conceptually it's a little different. By default, documents are assumed to be schemaless blobs with one primary key (called `_id` in both Mongo and Couch), and any other keys need to be specified separately.  The concepts are largely the same; it's mostly just the vocabulary that's different.
 
In CouchDB, queries are called _map/reduce functions_.  This is because, like most NoSQL databases, CouchDB is designed to scale well across multiple nodes, and to perform efficient query operations in parallel. Basically, the idea is that you divide your query into a _map_ function and a _reduce_ function, each of which may be executed in parallel in a multi-node cluster.

#### Map functions

It may sound daunting at first, but in the simplest (and most common) case, you only need the _map_ function.  A basic map function might look like this:

```js
function (doc) {
  emit(doc.name);
}
```  

This is functionally equivalent to the SQL index given above.  What it essentially says is: "for each document in the database, emit its name as a key."

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

Then you can query it:

```js
// find docs with name === 'Pika pi!'
pouch.query('my_index', {key: 'Pika pi!', include_docs: true});

// find the first 5 docs whose name starts with 'P'
pouch.query('my_index', {startkey: 'P', limit: 5, include_docs: true});
```

{% include alert_start.html variant="info"%}

The pagination options for <code>query()</code> &ndash; i.e., <code>startkey</code>/<code>endkey</code>/<code>key</code>/<code>keys</code>/<code>skip</code>/<code>limit</code>/<code>descending</code> &ndash; are exactly the same as with <code>allDocs()</code>. For a beginner's guide to pagination, read <a href='http://pouchdb.com/2014/04/14/pagination-strategies-with-pouchdb.html'>Pagination strategies with PouchDB</a>.

{% include alert_end.html %}

#### Reduce functions

As for _reduce_ functions, there are a few handy built-ins that do aggregate operations (e.g. `'_sum'` and `'_count'`), and you should typically steer clear of trying to write your own. But if you're adventurous, you should check out the [CouchDB documentation](http://couchdb.readthedocs.org/en/latest/couchapp/views/intro.html) or the [the PouchDB documentation](http://pouchdb.com/api.html#query_database) for details.

Map/reduce, reuse, recycle
----------

PouchDB has actually had map/reduce queries since way before version 2.2.0, via the `query()` API. It had a big flaw, though: all queries were performed in-memory, reading every document the entire database just to perform a single operation.

This is fine when your data set is small, but it could be murder on large databasess. On mobile devices especially, memory is a precious commodity, so you want to be as frugal as possible with the resources given to you by the OS.

The new persistent `query()` method is much more memory-efficient, and it won't read in the entire database unless you tell it to.  It has two modes:

* Temporary views (like the old system)
* Persistent views (new system)

Both of these concepts exist in CouchDB, and they're faithfully emulated in PouchDB.

#### A room with a view

First off, some vocabulary: CouchDB calls indexes _views_, and these views are stored in a special document called a _design document_. Basically, a design document describes a view, and a view describes a map/reduce query, which tells the database that you plan to use that query later, so it better start indexing it now. In other words, creating a view is the same as saying `CREATE INDEX` in a SQL database.

**Temporary views** are just that &ndash; temporary. They create a view, write it to disk, query it, and then poof, it's deleted.  That's why, in older version of PouchDB, we could get away with doing it all in-memory.

Crucially, though, temporary views have to do a full table scan every time you execute them, along with all the reading and writing to disk that that entails, only to throw it away at the end.  That may be fine for quick testing during development, but in production it can be a big drag on performance.

**Persistent views**, however, are a much better solution if you want your queries to be fast.  Persistent views require you to first save a design document containing your view, so that the emitted fields are already indexed by the time you need to look them up.  Subsequent lookups don't need to do any additional writes on the database, unless documents have been added or modified, in which case only the updated documents need to be processed.

#### Writing your first view

A design document with a view looks just like a regular document.  The simplest one might look like this:

```js
{
  _id: '_design/my_index',
  views: {
    'my_index': {
      map: function(doc) {
        emit(doc.name);
      }.toString()
    }
  }
}
```

Normally you would `put()` this document, and then it would be available for querying using the name `my_index`.

In PouchDB 2.2.0, though, we tried to make view creation easy by taking away the boilerplate.  Instead of explicitly creating a design document like the above, you can just write:

```js
pouch.putView('my_index', function (doc) {
  emit(doc.name);
});
```

And it will automatically create the above design document and put it in your database (unless there's a conflict, i.e. it already exists).

{% include alert_start.html variant="info"%}

<strong>Why design docs?</strong> Design docs are synced, stored, updated, and deleted exactly like normal documents.  They also show up in <code>allDocs()</code> queries, but you can filter them out by using <code>{startkey: 'design0'}</code>.  A design doc can contain multiple views, but there's really no advantage to this, and it can even be a performance issue in CouchDB. So by default, PouchDB only creates one view per design doc, and we use the same name for both.

{% include alert_end.html %}

#### Tips for writing views

It's helpful to remember that map functions will have to be executed for each document in your database.  Thus you want to minimize the number of times you create the view, and try to move as much logic as possible into your `query()` params instead.

For example, here's an anti-pattern I've occasionally seen, which you should avoid:

```js
// Anti-pattern - do not use!
function myMapFunction(doc) {
  if (doc.name === 'foobar') {
    emit(doc.name);
  }
}
pouch.query(myMapFunction).then(function (result) {
  /* got the 'foobar' doc */
});
```

A better pattern is this:

```js
function myMapFunction(doc) {
  emit(doc.name);
}
pouch.putView('my_view', myMapFunction).catch(function (err) {
 /* 'my_view' already exists, ignore */
}).then(function () {
  pouch.query('my_view', {key: 'foobar'}).then(function (result) {
    /* got the 'foobar' doc */
  });
});
```

Both of these examples will do the same thing, but the second one will only need to build the index once. Plus, from now on, you can query for `'foobar'` or `'foobaz'` or whatever you want, by using the `key` option.

{% include alert_start.html variant="warning"%}

<strong>Performance tip</strong>: Technically, the view will not be built up on disk until the first time you <code>query()</code> it.  So a good trick is to always call <code>query({stale: 'update after'})</code> after creating a view, to ensure that it starts building in the background.  You can also use <code>{stale: 'ok'}</code> to avoid waiting for the most up-to-date results. In the future, we may add a way for views to auto-update in the background.

{% include alert_end.html %}

For those who get nostalgic
----

Of course, some of our loyal Pouchinistas may have been perfectly satisfied with the old way of doing things. The fact that the `query()` function read the entire database into memory might actually have served them just fine, thank you.

To be sure, there are some legitimate use cases for this. If you don't store a lot of data, or if all your users are on desktop computers, or if you require closures (not supported by persistent views), then doing map/reduce in memory may have been fine for you. In fact, an upgrade to 2.2.0 might actually make your app perform _more slowly_, because now it has to read and write to disk.

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

Of course, this system starts to get shaky when you need to search by a variety of criteria: e.g. albums sortedy by year, artists sorted by age, etc. And you can only sort strings &ndash; not numbers, booleans, arrays, or arbitrary JSON objects.  But for a lot of simple apps, you should be able to get by without using the `query()` API at all.

{% include alert_start.html variant="info"%}

<strong>Performance tip</strong>: if you're just using the randomly-generated doc <code>_id</code>s, then you're not only missing out on an opportunity to get a free index; you're also incurring the overhead of building an index you're never going to use!

{% include alert_end.html %}


For the database geeks: implementation details
-------

If you want to know how map/reduce works in PouchDB, it's actually exceedingly easy: just open up your developer tools, head over to "Databases," and see where we've created a bonus database to store the index.

One of the neat things about how we implemented map/reduce is that we managed to do it entirely with PouchDB itself.  Yes folks, that's right: your map/reduce view is just a regular old PouchDB database, and map/reduce is just a plugin with no special privileges, creating databases in exactly the same way a regular user would.  Since the implementation sits on top of the PouchDB API, it's exactly the same for all three backends: LevelDB, IndexedDB, and WebSQL.

How did we pull off this trick?  To be clear: it's not like we set out to make PouchDB some kind of map/reduce engine.  In fact, as we were debating how to implement this feature, we originally ran on the assumption that we'd need to build additional functionality on top of PouchDB. It was only after a few months of discussions and experiments that we realized the whole thing could be done in PouchDB proper.

In the end, the only additional functionality we had to add to the main PouchDB codebase was a hook to tell PouchDB to destroy the map/reduce database when its parent database was destroyed.  That's it.

This fact also has a nice parallel in CouchDB: it turns out that they, too, reused the core `_view` code in order to write `_all_docs`.  In CouchDB, `_all_Docs` is just a special kind of `_view`, whereas in PouchDB, we built map/reduce views on top of `allDocs()`.  We did it backwards, but in spirit the idea was the same.

As for the index itself, what it basically boils down to is clever serialization of arbitrary JSON values into [CouchDB collation ordering](http://couchdb.readthedocs.org/en/latest/couchapp/views/collation.html) &ndash; i.e. null before booleans, booleans before numbers, numbers before strings, strings before arrays, arrays before objects, and so on recursively.  Every JSON object maps to an indexable string that guarantees the same ordering in every database backend, and we only deviate from CouchDB by using ASCII string ordering instead of ICU ordering (since that's what our backends use).

Then, we take this value and concatenate it with whatever else needs to be indexed (usually just the doc `_id`), meaning the entire lookup key is a single string.  We decided on this implementation because:

1. LevelDB does not support complex indexes.
2. IndexedDB _does_ support complex indexes, but not in IE.
3. WebSQL supports multi-field indexes, but it's also a dead spec, so we're not going to prioritize it above the others.

The next part of the design was to divide the data we needed to store into two kinds of documents: 1) emitted key/values with doc IDs, which are used for querying and pagination, and 2) mappings from doc IDs to those key/values, so that we can delete or update them when their parent documents are deleted or updated). Since, by design, PouchDB databases do not have multi-document transaction semantics, we implemented a task queue on top of the database to serialize writes.

Aside from that, the only other neat trick was a liberal use of `_local` documents, which are a special class of documents that aren't counted in the `total_rows` count, don't show up in `allDocs`, but may be accessed using the normal `put()` and `get()` methods.  This is all it takes to fully reimplement CouchDB map/reduce in a tiny JavaScript database!

Anyone interested in more implementation details can read the months-long discussions in [mapreduce/12][mapreduce-12], [1549][], and [1658][].  [The map/reduce code][mapreduce] itself is also pretty succinct.

[mapreduce]: https://github.com/pouchdb/mapreduce
[mapreduce-12]: https://github.com/pouchdb/mapreduce/issues/12
[1549]: https://github.com/pouchdb/pouchdb/issues/1549
[1658]: https://github.com/pouchdb/pouchdb/issues/1658