# ssb field guide
A field guide to developing with ssb

Further reading:
* [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/)
* [gossip](https://github.com/nichoth/ssb-field-guide/blob/master/gossip.md)

## ssb
The database layer is [secure-scuttlebutt](https://github.com/ssbc/secure-scuttlebutt). 

> A database of unforgeable append-only feeds, optimized for efficient replication for peer to peer protocols

The database is implemented with [flumedb](https://github.com/flumedb/flumedb). Your data is probably stored in a file at `~/.ssb/flume/log.offset`. This is an append only log that has your messages and probably your friends and foafs messages as well.

Also in `~/.ssb/flume` you will see some JSON files. These are materialized views of the database log (flume views). `friends.json` is a view of people within two hops in your foaf graph.

### feeds

A feed is a signed append-only sequence of messages. Each identity has exactly one feed. Your ssb log contains many feeds (one for each identity that you are gossiping), all mixed together. Each message in a feed has a field `sequence` that is an increasing number. That way you can tell peers the latest sequence that you have, and then request or send newer messages.


## sbot

[scuttlebot](https://github.com/ssbc/scuttlebot) (or sbot) provides network services on top of secure-scuttlebutt (the database).

> The gossip and replication server for Secure Scuttlebutt - a distributed social network

See also:

* [Epidemic Broadcast Trees (explained like i'm five)](gossip.md)


## client

A client is an application that uses sbot with various plugins as a backend. Each peer on the network corresponds to one database and one sbot instance, and the same sbot may be shared by several clients (see [ssb-party](https://www.npmjs.com/package/ssb-party)). All of these processes are probably running on one machine.

### ssb-client

[ssb-client](https://github.com/ssbc/ssb-client) can be used to connect to an sbot running in a separate process. There are some important configuration bits that your application needs, `caps.shs` and `caps.sign`. Your client needs `shs`, and sbot needs `sign`. These determine which network your app connects to. By setting these you can do tests and development on a separate network. SHS stands for [secret handshake](https://github.com/auditdrivencrypto/secret-handshake). There is a [whitepaper](http://dominictarr.github.io/secret-handshake-paper/shs.pdf) about it too.

Setting `caps.shs` makes gossip connections not occur with peers that have a different shs key.

Setting `caps.sign` makes messages to be considered invalid that were created with a different sign key.

If you only set the shs key, messages could leak out of your network if someone in the network changes their shs key, or adds messages manually. Setting the sign key ensures that the messages will not be able to be published (or validated) on the main ssb network, only on a network where that same sign key is used.

`shs` and `sign` should be base64 encoded random strings
```js
var crypto = require('crypto')
crypto.randomBytes(32).toString('base64')
```

```js
var Client = require('ssb-client')
var path = require('path')
var home = require('os-homedir')
var getKeys = require('ssb-keys').loadOrCreateSync

var SSB_PATH = path.join(home(), '.ssb')
var keys = getKeys(path.join(SSB_PATH, 'secret'))

Client(keys, {
    path: SSB_PATH,
    caps: {
        shs: 'abc'
    }
}, (err, sbot, config) => {
    // `sbot` here is an rpc client for a local sbot server 
})
```

### start an sbot
The client depends on an sbot using the same `caps.shs` as the client. You can pass it in as an argument

    $ sbot server -- --caps.shs="abc" --caps.sign="123"


See also [ssb-minimal](https://github.com/av8ta/ssb-minimal)


## sbot plugins 
Plugins expose methods via rpc to interact with an sbot. A plugin will typically read data from the ssb log, then create a view of that data that is saved somewhere, so it doesn't need to be recalculated from the beginning. 

An example is [ssb-contacts](https://github.com/ssbc/ssb-contacts). This uses flumeview-reduce to persist it's state, but you could use any persistence.

Plugins have an interface used by [secret-stack](https://github.com/ssbc/secret-stack). For plugin authors this means you need a few fields in your module. `exports.manifest` is familiar if you have used [muxrpc](https://github.com/ssbc/muxrpc). The manifest is used to tell remote clients what functions your service is exposing via rpc.

```js
exports.name = 'contacts'
exports.version = require('./package.json').version
exports.manifest = {
    stream: 'source',
    get: 'async'
}
exports.init = function (ssb, config) { /* ... */ }
```


## replication
https://github.com/ssbc/ssb-invite/blob/master/index.js#L195
The remote server info is embedded in the invite code.

The addresses of peers are  stored in `~/.ssb/gossip.json`.

[The line in ssb-invite that writes the address](https://github.com/ssbc/ssb-invite/blob/master/index.js#L263)

[The part in ssb-gossip that writes the file](https://github.com/ssbc/ssb-gossip/blob/04b17c781b983980b318d9a0701060d0f831f7a7/index.js#L420)


see @cel's [msg in ssb](https://viewer.scuttlebot.io/%250KAk8CvE7hNeV4GAFyzYdW8Qy%2Bb47tH%2F5O3RdH4znu0%3D.sha256)

---------------------------------------

## make a plugin
Need to pass an object with a certain API that gets called by `sbot.use`

```js
var myPlugin = {
    name: 'aaaaa',
    version:  0,
    manifest: {
    },
    init: init
}

var sbot = Sbot.use(myPlugin)(config)
console.log('**sbot aaaaa**', sbot.aaaaa)
// logs the return value of `init`

function init (sbot) {
    // this return value is accessible at `sbot.aaaaa`
    return { foo: 'foo' }
}
```

## make a databse view
Here we create a materialized database view (a flumeView)

* [flumeDB](https://github.com/flumedb/flumedb)
* [ssb-links](https://github.com/ssbc/ssb-links/blob/master/index.js)
* [flumeview-reduce](https://github.com/flumedb/flumeview-reduce)
* [flumeUse](https://github.com/ssbc/ssb-db#db_flumeuse-view)
* [flumeview-links](https://github.com/flumedb/flumeview-links)
-------------------
* [flumeview-hashtable](https://github.com/flumedb/flumeview-hashtable)
-----------------------------

```js
var codec = require('flumecodec')
var createReduce = require('flumeview-reduce/inject')
// this way our view state is persisted to disk
var Store = require('flumeview-reduce/store/fs')

var myPlugin = {
    name: 'aaaaa',
    version:  0,
    manifest: {
    },
    init: init
}

var sbot = Sbot.use(myPlugin)(config)
console.log('**sbot aaaaa**', sbot.aaaaa)

sbot.publish({
    type: 'post',
    text: 'Hello, world!'
}, function (err, msg) {
    // the API we returned from `init`
    // which here is the api return by flumeview-reduce 
    sbot.aaaaa.get(function (err, data) {
        console.log('**get**', err, msg)
    })
})

function init (sbot) {
    function reducer (acc, val) {
        acc.foo = acc.foo + 1
        return acc
    }
    function mapper (msg) {
        return msg
    }

    var initState = { foo: 0 }

    // need to use this API if we want to store data
    var Reduce = createReduce(Store)

    // view state is saved at ~/app-name/ok.json
    // b/c that's what we named it in `_flumeUse` here
    // the path is based on the path to the flume log
    // https://github.com/flumedb/flumeview-reduce/blob/master/inject.js#L90
    var view = sbot._flumeUse('ok',
        Reduce(1, reducer, mapper, codec.json, initState))

    // the thing returned from here is at `sbot.aaaaa`
    return view
}
```

## ssb-db-2, JITDB
`ssb-db2` is just a plugin like any other. I’m already using it in Manyverse.

These projects are primarily for better performance. Secondary goal is to reset some of the legacy choices that wear us down. Third goal is to have a nicer API for querying.

### JITDB
JITDB is just a lower level component. The name is called JIT because the indexes are created just-in-time, automatically inferred from the query. These are only bitvector indexes and prefix indexes.

----------------------------------------

## ssb browser
See [ssb-browser-core](https://github.com/arj03/ssb-browser-core)

### storage
`ssb-browser-core` uses partial replication so the storage requirement is not as big as a normal client.

The private key is saved in localstorage.

When you run out of space, new writes will fail.

[this article](https://web.dev/storage-for-the-web/) states that chrome can use up to 80% of available disk

All of this is built on top of async-append-only-log, this in turn uses polyraf, which is a wrapper around [random-access-storage](https://github.com/random-access-storage/) family of libraries so that it works both in the browser and in node with the same api. In the browser it will use random-access-web which for firefox will use indexeddb but for chrome will use a special faster backend.

#### blobs storage
blobs are different than the log, as it is in a normal ssb client as well. There is a [small module](https://github.com/arj03/ssb-browser-core/blob/master/simple-blobs.js) that uses the same blobs protocol but stores the data using the `polyraf` library. The core library has a parameter where you can say that you only want to download and store blobs under a certain size. It will then stream larger blobs so they don’t take up space. Another approach would be use something like the [blobs purge](https://github.com/arj03/ssb-browser-demo/issues/8) library.

-------------------------------------------------

## secret handshake
The handshake is mostly about authentication / verification; can the public keys of both peers be verified and are they using the same network key? If the answer to both of those questions is yes, then a **shared secret** is created - which I think of as a proof of verification or proof of identity. The shared secret can then be used to encrypt and decrypt further communications.

you can use the same stream for the handshake and the box stream. The handshake will return a set of keys to each peer and those keys are used to create a pair of boxstreams (one for reading and one for writing).


## ssb-browser-core

### tests
The way I have been testing this in browser demo is to have 2 browsers, could be 2 incognito modes or a chrome and a firefox. Then you basically create a follow messages on one of the browsers for the other identity. Do the same with the other browser. After that if you connect to the same room. You can use the one in browser demo, then the two browsers should start exchanging messages.

-----------------------------------

> a room is basically a server that multiple peers can connect to. The room will faciliate e2e encrypted tunnels between the peers. This means that it won't have any trouble with hole punching and all of that stuff. This is an address to a room that works in the browser: https://github.com/arj03/ssb-browser-demo/blob/e935636feaddd6d13868a104a2ea0fdb3c176bff/defaultprefs.json#L12

So you basically connect both clients to that using something like: https://github.com/arj03/ssb-browser-demo/blob/master/ui/connections.js

