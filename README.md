# SSB Key-Value DB

A key-value database for [Scuttlebot applications](https://github.com/ssbc/scuttlebot).
In development.


## About

KV documents are stored in [Secure Scuttlebutt](https://github.com/ssbc/secure-scuttlebutt) logs and replicated on the mesh network.

This interface mimics [leveldb](https://github.com/level/levelup), but uses a [multi-value register CRDT](https://github.com/pfraze/crdt_notes#multi-value-register-mv-register) to compute the database state.
Multiple users may update the database without locking.
In the case of conflict, the application must resolve them.


## API

 - `kvdb()`
 - `db.put()`
 - `db.get()`
 - `db.del()`
 - `db.batch()`
 - `db.createReadStream()`
 - `db.on("change")`
 - `db.on("change:<key>")`

### kvdb(namespace, [options])

Creates a new database instance.

#### `namespace`

The `namespace` string is required.

Reads and writes will be scoped to the namespace, making it easier to avoid accidental key collisions.

SSB Feed IDs act as a special namespace which can only be modified by the feed-owner.
Writes to a feed-id namespace which is different that the author's id will be rejected.

#### `options`

The `options` object is optional, in some cases.

`options.ssb` sets the secure-scuttlebutt instance which backs the `kvdb`.
If `window.ssb` is not available, then `options.ssb` must be provided.

`options.db` sets the backend for storing DB state.
If not provided, it will fallback to [memdown](https://github.com/level/memdown).

`options.sourceFeed` specifies the function from which messages are read.
It's values may be:

 - `'log'`: all items in the ssb log (string) (default)
 - `'self'`: all items in the feed of the current user (string)
 - `feed-id`: all items in the feed of the given user (string)
 - `[feed-ids]`: all items in the feeds of teh given users (array of strings)

### db.put(key, value, [options], cb)

Write a value at the given key.

### db.get(key, [options], cb)

Get the value at the given key.

### db.del(key, [options], cb)

Remove the value at the given key.

### db.batch(array, [options], cb)

Complete a sequence of put/del operations.

### db.createReadStream([options])

Read sequentially from the database.

### db.on("change")

Emitted when any of the values is updated or deleted.

### db.on("conflict")

Emitted when any of the values comes into conflict.

### db.on("change:<key>")

Emitted when the value at `<key>` is updated or deleted.

### db.on("conflict:<key>")

Emitted when the value at `<key>` comes into conflict.
