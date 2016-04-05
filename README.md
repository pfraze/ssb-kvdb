# SSB Key-Value DB

A key-value database plugin for [Scuttlebot applications](https://github.com/ssbc/scuttlebot).

```bash
sbot plugin.install ssb-kvdb
```

## About

This interface lets you focus on writing applications, instead of working with the SSB log.
It is similar to other Eventually-Consistent databases (CouchDB, Riak).
The KV documents are translated into update messages, and then replicated on the mesh network.

Overview:

```js
sbot.kv.get(namespace, key)
// get a single value for a namespace+key
// (gives the main user's value)

sbot.kv.get(namespace, key, { author: feedId })
// get a specific user's value

sbot.kv.getAll(namespace, key)
// get all users' values for namespace+key, as an array

sbot.kv.getAll(namespace, key, { authors: feedIds })
// only give values from specified authors

sbot.kv.put(namespace, key, value)
// update the value at namespace+key

sbot.kv.post(namespace, value)
// set the value at a generated key
// (gives the key in the response cb)

sbot.kv.post(namespace, value, { shared: true, authors: feedIds })
// creates a shared value, owned by `authors`
// (this is the only way to create shared values)
```

#### Siloed values

By default, kvdb "siloes" the dataset for each author.
That means, at a given key, each user has their own value, which only they can update.

Siloing is convenient when users have their own datasets.
The Patchwork "user profiles" system, for instance, uses siloing so that each user has their own contacts.

**Example usage for siloed datasets:**

```js
// user profiles are in the 'about' namespace
// lets read/write some profile data

// set bob's profile
sbot.kv.put('about', bobsId, { name: 'Bob', desc: 'My friend bob' }, function (err) {

  // get the profile I created
  sbot.kv.get('about', bobsId, function (err, profile) {
    console.log(profile) /* => {
      key: bobsId,
      author: myId,
      value: {
        name: 'Bob',
        desc: 'My friend bob'
      }
    } */
  })

  // update his name
  sbot.kv.put('about', bobsId, { name: 'Robert' }, function (err) {

    // check result
    sbot.kv.get('about', bobsId, function (err, profile) {

      // notice that only `name` was changed:
      console.log(profile) /* => {
        key: bobsId,
        author: myId,
        value: {
          name: 'Robert',
          desc: 'My friend bob'
        }
      } */
    })
  })
})

// get everybody's profiles for bob
sbot.kv.getAll('about', bobsId, function (err, profiles) {
  console.log(profiles) /* => [
    { key: bobsId, author: myId, value: { name: 'Robert', desc: 'My friend bob' } },
    { key: bobsId, author: alicesId, value: { name: 'Bobby', desc: 'My dear husband' } },
    { key: bobsId, author: bobsId, value: { name: 'Bob', desc: 'Only the coolset guy ever' } },
    ...
  ] */
})

// get alice's and my profiles for bob
sbot.kv.getAll('about', bobsId, { authors: [myId, alicesId] }, function (err, profiles) {
  console.log(profiles) /* => [
    { key: bobsId, author: myId, value: { name: 'Robert', desc: 'My friend bob' } },
    { key: bobsId, author: alicesId, value: { name: 'Bobby', desc: 'My dear husband' } },
  ] */
})
```

#### Shared values

Kvdb can create "shared values," which allow multiple users to make updates to a single key.
Shared values use the [multi-value register CRDT](https://github.com/pfraze/crdt_notes#multi-value-register-mv-register).
They're a little more complex to work with, so avoid shared values unless you know you need them.

Shared values must be explicitly created with a `post()` call, which specifies the authors.

**Example usage for shared values:**

```js
// create a new todo list
var initValue = { title: 'Our Todo List', items: {} }
sbot.kv.post('todos', initValue, { shared: true, authors: [myId, alicesId] }, function (err, todoList) {
  console.log(todoList) /* => {
    key: generatedId,
    shared: true,
    authors: [myId, alicesId],
    value: {
      title: 'Our Todo List',
      items: {},
      completed: {}
    }
  } */

  // add a todo item
  todoList.value.items[genId()] = {
    label: 'Write a kvdb app',
    ts: Date.now()
  }
  sbot.kv.put('todos', todoList.key, { items: todoList.value.items }, function () { 

    // fetch new state
    sbot.kv.get('todos', todoList.key, function (err, todoList) {
      console.log(todoList) /* => {
        key: generatedId,
        shared: true,
        authors: [myId, alicesId],
        value: {
          title: 'Our Todo List',
          items: {
            '2ac4f': { label: 'Write a kvdb app', ts: 1459892651421 }
          },
          completed: {}
        }
      } */
  })
})
```

#### Conflicts

In shared values, if two users update a value at the same time, then both values are kept.
This is called a "conflict".
You can resolve the conflict by reading the conflicting values, choosing what to keep, and writing a new value.

**Example conflict-resolution for shared values:**

```js
// continuing from the above "shared value" example

// fetch current state
sbot.kv.get('todos', todoList.key, function (err, todoList) {

  // let's assume alice made a conflicting update:
  console.log(todoList) /* => {
    key: generatedId,
    shared: true,
    authors: [myId, alicesId],
    value: {
      title: 'Our Todo List',
      items: {
        '2ac4f': { label: 'Write a kvdb app', ts: 1459892651421 }
      },
      completed: {}
    },
    conflicts: {
      items: [
        { '2ac4f': { label: 'Write a kvdb app', ts: 1459892651421 } },
        { 'cc30a': { label: 'Create a todo item', ts: 1459892671830 } }
      ]
    }
  } */
  // it appears `items` is in conflict

  // handle the conflicts:
  if (todoList.conflicts) {

    if (conflicts.title) {
      // there's no good way to merge title, just take the first
      current.title = conflicts.title[0]
    }

    if (conflicts.items) {
      // this is easy, since we give unique IDs to each item 
      // -- just merge the objects
      conflicts.items.forEach(function (items) {
        Object.assign(current.items, items)
      })
    }

    if (conflicts.completed) {
      // same as with items
      conflicts.completed.forEach(function (completed) {
        Object.assign(current.completed, completed)
      })
    }

    delete todoList.conflicts
  }

  // new state:
  console.log(todoList) /* => {
    key: generatedId,
    shared: true,
    authors: [myId, alicesId],
    value: {
      title: 'Our Todo List',
      items: {
        '2ac4f': { label: 'Write a kvdb app', ts: 1459892651421 },
        'cc30a': { label: 'Create a todo item', ts: 1459892671830 },
      },
      completed: {}
    }
  } */

  // NOTE: do not automatically publish the new, corrected version
  // doing so could cause the computers to get into a ping-pong match of updates
  // instead, just have the peers run the same `merge` function when there is a conflict...
  // ...and use the merged version when the next publish occurs.
})
```

#### Namespaces

The namespace sets the `type` attribute on the ssb messages, and sets the attribute for each document's "key."
For instance, in a kvdb with a namespace of `about`:

```js
sbot.kv.put('about', bobsId, { name: 'Bob' })
```

The resulting message from that `put` would look like this:

```js
{
  type: 'about',
  about: bobsId,
  name: 'Bob'
}
```

If the namespace were, eg, `bobs-app-about`, the message would look like this:

```js
{
  type: 'bobs-app-about',
  'bobs-app-about': bobsId,
  name: 'Bob'
}
```

This means that `type`, and namespace value, are reserved keys that can't be used in the value object.

**Why is the namespace value used as an attribute?**

SSB messages are consumed by some applications directly.
It's useful to maintain certain conventions, so that the message content is pleasant to work with.
Messages which use the `type` value as a key are a common pattern in SSB.
The key tends to indicate the "subject" of the message.

#### Private datasets

Encrypted datasets are not yet supported.
All kvdbs are public.

## API

# Everything below here is still todo and somewhat wrong. Go that way ^

 - `sbot.kv.getAll()`
 - `sbot.kv.get()`
 - `sbot.kv.put()`
 - `sbot.kv.post()`
 - `sbot.kv.del()`
 - `sbot.kv.batch()`
 - `sbot.kv.list()`
 - `sbot.kv.listen()`

---

### sbot.kv.put(namespace, key, value, [options], cb)

Write a value at the given key.

#### Reserved keys

The following keys can not be used in the value object:

 - `type`: this key is special in ssb messages.
 - `updates`: this key is used to link to past messages, indicating the old values which are being overwritten.
 - `deleted`: this key is used to indicate that the value has been deleted.
 - The namespace value. For instance, if `namespace` is "foo", then the "foo" key is reserved.

If `value` uses these keys, `put` will error.

---

### sbot.kv.get(key, [options], cb)

Get the value at the given key.

If the key is in conflict, then this method will get the "first" according to a deterministic comparison.
If you wish to get all current values by all authors, use `getMV()`.

`options.noConflict` will cause the get to fail if the item is in conflict.

---

### sbot.kv.getMV(key, [options], cb)

Get the values at the given key, in an array.

If the key is in conflict, then this method will retrieve all of the current values.

A key that is not in conflict will respond with a values array of length `0` or `1`.

---

### sbot.kv.del(key, [options], cb)

Remove the value at the given key.

---

### sbot.kv.batch(array, [options], cb)

Complete a sequence of put/del operations.

---

### sbot.kv.createReadStream([options])

Read sequentially from the database.

---

### sbot.kv.on("change")

Emitted when any of the values is updated or deleted.

---

### sbot.kv.on("change:{key}")

Emitted when the value at `{key}` is updated or deleted.
