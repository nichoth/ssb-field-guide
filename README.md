# ssb survival guide

A field guide to developing with ssb

Further reading:
* [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/)

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




