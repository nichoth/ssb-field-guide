# gossip

[This was written by dominic](https://viewer.scuttlebot.io/%25dRIUnLTQ4t8fBVwcPlH%2FCGbyCd0e1Da7wjCsc7PQWpg%3D.sha256)

## Epidemic Broadcast Trees (explain i like i'm 5)

I want to broadcast a message to my friends around the world and I have a couple of requirements

* make sure everyone definitely gets it
* make sure everyone gets it soon
* send as few messages as possible
* it should work for everyone

How might I approach this?

## send it to everyone

the simplest way would be to make a connection to each friend and send it to them directly. If I have 100 friends, that is 100 messages - that is actually good! but the problem is if everyone wants to distribute messages this way everyone needs 100 connections! to connect everyone to everyone means thousands of connections! This is a lot of work for each computer.

## send it to a few people who send it on

lets say, everyone has just 5 connections to some other random friends, so you send a message to 5 people, they each send it to another 5 (thats 1+25 now) who then send it to another 5 (that is 1+25+125) now everyone has the message and some people have probarly received (since 151 messages went to 100 people) that isn't efficient. but it was less work for my computer! I only needed to send it to 5 people! I'm okay with that!

## send it to few people and send it on

okay so lets try the above, can we avoid sending it to anyone _again_? We could do that if we had a thing called a "spanning tree" a spanning tree is a network that connects everything but doesn't have any extra connections. That means there is no loops - there is always a shortest path between any two friends (that may go through other closer friends).

If you think of a ordinary tree it's like this - because all the branches come off the trunk, and then the leaves come off those branches. If you where an ant and wanted to travel from one leaf to another, you'd start walking towards the trunk, then go onto the branch the other leaf came off etc.

An ordinary tree grows in this shape, so it's easy to see what the paths are, but the problem we have is all the leaves are in a jumble and we don't actually know which ones are close to each other!

## centrally organize into a tree

if we had some sorts of top down view of the network, we could look and see which people where closest to each other, and tell them to connect. The people in far flung places would connect to people in more populous places who would connect to each other and messages would travel along those lines.

But this has a big important job that someone has to do, and who is gonna pay them to do it? We'd rather have a system where everyone can just do their thing without any one having special responsibility (because that isn't fun)

## decentrally organize into a tree

what if there was a way that people could just figure out on their own who is the closest? well, actually there is! When you send someone a message, you can measure how long it takes them to send a message back! this is equivalent to how close you are together! So, lets say we connect to random people like in the 2nd method - but then if we receive another message that we already know, we deactivate that connection. (since if we already know that message, we must have a faster path to the source along another path)

So, the first time we build the network, it will have lots of extra connections, but the ones we don't need will get switched off.
If we sometimes add new connections to other random people We'll eventually end up with the best network, the same as the top down network, but without having to have anyone at the top!

So, once the network has "warmed up" it only needs to send 100 messages to make sure everyone gets everything, but no one person needs to send more than a couple of messages.

---

okay, this story really needs pictures. also glosses over some aspects, but it's the basic gist of it.


--------------------------------------------


But I glossed over one thing, that I'm gonna explain now.
There is a problem with tree shape networks - we wanted to make sure that each message travels by the best route, but if we have only one way to get anywhere, what happens when something goes wrong? for example - one computer that is on the main branch crashes, runs out of batteries, enters a tunnel, or just the network is disrupted because too many people are using it. In real networks all of these things happen at least occasionally so when designing a network system we need to design it to still work when something goes wrong.

What does that mean? If our network is a pure tree - it's also very fragile, because if a branch breaks, half the tree falls off.
That means you suddenly can't share cat photos or debate about feminism because there is no path for your message to travel.

How can we fix this? well, what we need is "redundency", or more simply, _backups_. We need the network tree to have extra connections that are still there, but we don't use unless we need them. So in EBT when we "deactivate" a connection we do not disconnect it, we just put it into _backup mode_. When you receive a message, you broadcast the whole message on the _active connections_ but on your _backup connections_ you just send a very short _note_ saying that you have the new message.

Normally, you expect to receive the whole message along an active connection, and then receive notes about it on the backup connections. The message should arrive first, because it travels on the shortest path, and the notes take longer paths so they should arrive afterwards.

What happens if you see a note about a message you havn't seen yet? Well, that means the fastest path isn't the fastest anymore! when that happens you send a short message along the backup connection, reactivating it. Now you have a new tree, and the network is still connected.


--------------------------------


## save loads of bandwidth this weird old trick

There is also actually one more thing we do that saves a bunch of replication bandwidth when replicating - credit to [@cel](@f/6sQ6d2CMxRUhLpspgGIulDxDCwYD7DzFzPNr7u5AU=.ed25519) for having this idea.

So, in gossip, two friends meet, then they exchange the latest news. But to save time (and bandwidth) we don't want to tell someone something they already know.

So, when Alice and Bob connect, they'll first send a message saying what is the latest they've heard about each of their friends. so Bob might say "{Alice: yesterday, Bob: today, Carol: tuesday}" and if Alice says "{Alice: today, bob: yesterday, Carol: tuesday}" (note, alice and bob last talked yesterday) so Alice then knows to tell Bob her news from today - and Bob tells Alice her news, but neither of them have new updates about Carol.

When also talks to Bob again tomorrow, Alice remembers that Bob had heard from Carol on tuesday. Alice hasn't heard anything new about Carol since then, so she doesn't need to mention Carol at first. If Bob has heard new news from Carol, He'll say something, and then Alice will request anything newer than tuesday. If Bob does the same, then they won't waste time talking about people who arn't doing anything.

In most applications - you'll have some users who are very active, and then many others who don't interact very frequently. If there are thousands of users, some users might post many times a day, some every day, more every few days, some once a week and others might post then come back months later.

If you connect and then reconnect then even quite an active user can be one that hasn't updated since you last connected. If the network is interrupted you'll need to reconnect, and sometimes the network is really bad and this happens lots!

Because of this request skipping trick, connecting and starting the gossip only needs to mention the number of ids that have new news, but without this method, we have to send the total number of ids we want to replicate. If you have thousands of friends, that can add up to a lot of data!

> (hmm, this could be a bit more explain-like-im-5, but working on it)


