# ssb survival guide

A guide to developing with ssb


## ssb

The database layer is [secure-scuttlebutt](https://github.com/ssbc/secure-scuttlebutt). 

> A database of unforgeable append-only feeds, optimized for efficient replication for peer to peer protocols

The database is implemented with [flumedb](https://github.com/flumedb/flumedb). Your data is probably stored in a file at `~/.ssb/flume/log.offset`. This is an append only log that has your messages and probably your friends and foafs messages as well.

Also in `~/.ssb/flume` you will see some JSON files. These are materialized views of the database log (flume views). `friends.json` is a view of people within two hops in your foaf graph.


## sbot

[scuttlebot](https://github.com/ssbc/scuttlebot) provides network services on top of ssb (the database).

> The gossip and replication server for Secure Scuttlebutt - a distributed social network

[Gossip](gossip.md)

## client

A client is an application that uses sbot with various plugins as a backend. Each peer on the network corresponds to one database and one sbot instance, and the same sbot may be shared by several clients (see [ssb-party](https://www.npmjs.com/package/ssb-party)). All of these processes are probably running on one machine.

### ssb-client

[ssb-client](https://github.com/ssbc/ssb-client) can be used to connect to an sbot running in a separate process. There are some important configuration bits that your client needs, `caps.shs` and `caps.sign`. These determine which network your client connects to. By setting these you can do tests and development on a separate network. SHS stands for [secret handshake](https://github.com/auditdrivencrypto/secret-handshake). There is a [whitepaper](http://dominictarr.github.io/secret-handshake-paper/shs.pdf) about it too.

Setting `caps.shs` makes gossip connections not occur with peers that have a different shs key.

Setting `caps.sign` makes messages to be considered invalid that were created with a different sign key.

If you only set the shs key, messages could leak out of your network if someone in the network changes their shs key, or adds messages manually. Setting the sign key ensures that the messages will not be able to be published (or validated) on the main ssb network, only on a network where that same sign key is used.

shs and sign should be base64 encoded random strings

```js
crypto.randomBytes(32).toString('base64')
```

**@TODO does the client need `caps.sign` or only the sbot? I don't see `sign` used in the ssb-client code**


```js
var Client = require('ssb-client')
var path = require('path')
var home = require('os-homedir')
var getKeys = require('ssb-keys').loadOrCreateSync

var PATH = path.join(home(), '.ssb')
var keys = getKeys(path.join(PATH, 'secret'))

Client(keys, {
    path: '/Users/me/.ssb',
    caps: {
        shs: 'abc'
    }
}, (err, sbot, config) => {
    // now you have an sbot, which is a separate process with rpc methods
})
```

### start an sbot

The client depends on an sbot using the same `caps.shs` as the client. You can pass it in as an argument

    $ sbot server -- --caps.shs="abc" --caps.sign="123"


See also [ssb-minimal](https://github.com/av8ta/ssb-minimal)



## plugins 

Plugins are a separate process that communicate with an sbot via rpc. A plugin will typically read data from the ssb log, then create a view of that data that is saved somewhere, so it doesn't need to be recalculated from the beginning. 

An example is [ssb-contacts](https://github.com/ssbc/ssb-contacts). This uses flumeview-reduce to persist it's state, but you could use any persistence.

Plugins have an interface used by [depject](https://github.com/depject/depject). For plugin authors this means you need a few fields in your module. `exports.manifest` is familiar if you have used [muxrpc](https://github.com/ssbc/muxrpc). The manifest is used to tell remote clients what functions your service is exposing via rpc.

```js
exports.name = 'contacts'
exports.version = require('./package.json').version
exports.manifest = {
    stream: 'source',
    get: 'async'
}
exports.init = function (ssb, config) { /* ... */ }
```




