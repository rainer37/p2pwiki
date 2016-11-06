# P2P Wikipedia Specifications

People who install the app will be able to create, view, modify and delete Wikipedia-style
articles. The articles will be stored on other computers (peers) where the app is
installed but each user will be able to access the full collection of articles by
performing lookups.

To achieve this, the peers will be structured using Chord (or a similar) protocol.
To allow new peers to join the network, a traditional web server will be used to
provide IP addresses of (some) existing peers. It may also provide other services
such as monitoring the health of the peer-to-peer network and providing a (best-effort)
directory of articles. Note that the peer-to-peer component does NOT depend on the
web server to function -- it is an optimization (although it is needed for new nodes
to join).

Communication among the peers will be performed using TCP for Chord operations and
RPC for article operations. TCP will also be used by nodes to communicate with the
web server.

Limitations
- Chord will only allow for full-key searches. Therefore, the user must know the
  full title of the article they wish to view. Partly mitigated by having the article
  directory (but it could be out-of date, even if article creation is broadcast to
  all nodes -- some might be down during the broadcast).
- The number of article replication is static as proportional to the MAX_NUMBER_OF_NODES
  that the system can support. However, the optimal number should be based on the current
  number of living nodes, and based on the nodes' "liveness" (average uptime).
- All peers have equal permission to read/modify any articles. Malicious peers could be very
  distructive. May introduce the blacklist, or peer-reviews on large modification.

## _Questions_
- Q1: How will the web server get the IP addresses?
- A1: _It will know the address of the node sending the health report (and possibly the cached
   IP address in the status file)._
   
- Q2: How will the app get the IP addresses?
- A2: _Request it during `init` operation._

- Q3: Are IP addresses going to be sufficient identifiers (i.e. if we want to blacklist
   apps that make bad changes to articles then we will need to include some identifying
   information with the articles about who modified it -- IP addresses change)
- A3: Adding MAC address may relief the problem.

- Q4: Is the server smart enough to make sure it doesn't tell every new app to join
   to the same peer?
- A4: 

- Q5: Could the server provide some health information about the p2p network?
- A5: _Nodes can send their health report periodically._

- Q6: Could some nodes become blacklisted? (This information could be stored on the
   server and made available to apps that at as hosts when a new app joins)
- A6: If using IP as the only identifier, this blacklist is not so useful, and could be
     misleading. 
      
- Q7: The server is now a single point of failure (although operation of the existing
   peers should NOT be affected by its failure)
- A7: A more robust server can be used. 

- Q8: Choosing values for Chord parameters (how often to run stabilize, fix, etc.; 
     the length of successor list; number of finger table entries; etc.) (**all** params for 
     chord/the app need to be listed)
- A8: 




## Installation
People interested in using the app will download it from a traditional web server
external from peer-to-peer network. This server will also provide the IP addresses
of a subset of peers for which the newly installed app can connect to. This will
allow the app to join the network (it will also allow disconnected apps to rejoin
if none of their cached peers are online).

To install, run `p2pwiki --init`. This creates the following directory structure
under the parent directory of the executable
```
parent_dir
 |-- p2pwiki.exe
 |-- wikiNodeStats.json
 |-- logs
 |-- articles
     |-- unpublished  // draft articles
     |-- cached  // articles that we recently created or read (these are used solely by this node)
     |-- title1.json
     |-- title2.json
```

This will also create the special private file `wikiNodeStats.json` containing the
following
```
// some identifier that will be globally unique with high probability
// for example, the peer's MAC address concatenated with a GUID
peerId

// Some number of IP addresses (say up to 30) recording during successful lookup Operations
// These can be used to try re-joining the network in case of node failure, starting with the most recent first
// If no cached address works, contact the web server to get more recent list
peerIPAddresses

installDate

// The numbers are since install date
numberFailures
numberArticlesCreated
numberArticlesHosted
numberReplicasHosted
numberLookups
```
This file is periodically sent to the web server for monitoring the health of the
peer-to-peer network.

## Starting the app
To start the app, run `p2pwiki`.

What the app does during startup?
1. Attempt to join the network using the cached IP addresses in `wikiNodeStats.json`,
   starting with the most recent entry. If all of the cached IP addresses fail,
   request a list of addresses from the web server. Try joining to each of those
   addresses. If those fail, exit with error. The user can try again later since
   the web server will have a different list of IP addresses.
2. Check for article changes. For each article:
   1. Check the article hash against all other copies of the article
      (the original and replicas)
   2. Find and copy the latest version of the article, replacing the local copy
      (but maintaining the local title)


## Stopping the app
To stop the app, send the app a `SIGTERM`.

What does the app do during normal shutdown(voluntary leave)?
1. Replicate all the original articles AND replicated articles to successor.
3. Tell server about leaving, to remove its IP from server's list.
4. Log useful information.

If any operation cannot be complete due to network partition, retry the operation ??ONCE??.
If still not work, then just shutdown, we can handle this as a fail stop.

## Fail stop
What does the app do if it fails?
- Author Down:
- Q1: Who should be the new author?
- A1: 1. LookUp the title by the hash, and find the one that is alive, but need some way to get content of article.
      2. Nearest replica. Probably not.
- Q2: How do other peers/replicas know the failure of author?
- A2: They don't util they try to lookup the articles.
- Q3: Which version the peers should use if the author's down?
- A3: Use no one util the new author is up.

### May be we need an author election algo.

- Replica Down:
- Q1: Does author need to know that? and how?
- A1: Health report. Routine check.
- Q2: 

- Q: How does the server knows about the failure?

How does it recover during startup?
Try to read the logs if there is one, and broadcast for informaiton to SOME peers.
### May be we need a recover algo as well.(What info to request for?)
How is the p2p network affected?
Short period of delay on some articles look up.

Q: What happens if node with original article is down indefinitely (i.e. uninstalled)?
Does title_repl become considered the new article and get replicated as title_repl_repl??
A: 

## Chord Operations
_TODO Outline the Chord operations_
- JOIN: 
- STABLIZE:
- NOTIFY:
- GET_SUCCESSOR:
- GET_PREDECESSOR:
- FIX_FINGERS:
- LEAVE:

## Article Operations
Operations built on top of the Chord protocol will use RPCs. Chord operations (lookup,
stabilize, etc.) use TCP.

### Creation
How does the user actually get to create a document? (either the app has some kind
of shell or we use a webpage).


An _article_ is a JSON document with the fields:
```
title: string
estimatedReplication: int  // how many replicas will be attempted to be made
...
```
The title of the document must be specified before creation so that a lookup can
be performed to determine if it is unique. If not, the user will instead be shown
the existing article which they can choose to modify.

The article is stored as `<title>.json` in the articles directory on the peer with
key closest to `SHA(<title>.json)` that is larger or equal.

Part of the creation process will involve broadcasting the title and node that will
be storing the article to all nodes in the ring (on a best effort basis). This will
allow peers to keep some form of article directory and can be used a basic cache
mechanism to speedup lookups.

### Replication
Replication of an article is performed by appending an integer (starting at 1) to
the article title. This will cause the article's SHA value to be different from
the original article. The replication method will check that duplicate articles
(those with the same title up to the appended integer) are not stored on the same
peer (the duplicate will not be saved and the client should try to store an article
with a larger integer appended).

The peer-to-peer network health (provided by the server) will aid in determining
the number of replicas that should be created (this value will be stored as an
estimate with the article). As the health of the network fluctuates, a background
process on each node will create/retire replicas.

Q: What is the target number of replicas based on health?

### Retrieve
The `retrieve` command uses the Chord lookup operation to find the IP address of
the node hosting the specified article (or a replica). The command then makes an
RPC call to the specified host to fetch the article, storing it in the requestee's
_cached_ directory.

### Modification


### Deletion
