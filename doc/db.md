# Database automation

In a Jepsen test, a `DB` encapsulates code for setting up and tearing down a
database, queue, or other distributed system we want to test. We could perform
the setup and teardown by hand, but letting Jepsen handle it lets us run tests
in a CI system, parameterize database configuration, run multiple tests
back-to-back with a clean slate, and so on.

In `src/jepsen/etcdemo.clj`, we'll require the `jepsen.db`, `jepsen.control`,
`jepsen.control.util`, and `jepsen.os.debian` namespaces, aliasing each to a
short name. `clojure.string` will help us build configuration strings for etcd.
We'll also pull in every function from `clojure.tools.logging`, giving us log
functions like `info`, `warn`, etc.

```clj
(ns jepsen.etcdemo
  (:gen-class)
  (:require [clojure.tools.logging :refer :all]
            [clojure.string :as str]
            [jepsen [cli :as cli]
                    [control :as c]
                    [db :as db]
                    [tests :as tests]]
            [jepsen.control.util :as cu]
            [jepsen.os.debian :as debian]))
```

Then, we'll write a function that constructs a Jepsen DB, given a particular
version string.

```clj
(defn db
  "Etcd DB for a particular version."
  [version]
  (reify db/DB
    (setup! [_ test node]
      (info node "installing etcd" version))

    (teardown! [_ test node]
      (info node "tearing down etcd"))))
```

The string after `(defn db ...` is a *docstring*, documenting the function's
behavior. When given a `version`, the `db` function uses `reify` to construct a
new object satisfying jepsen's `DB` protocol (from the `db` namespace). That
protocol specifies two functions all databases must support: `(setup db test
node)`, and `(teardown! db test node)`. We provide trivial implementations
here, which simply log an informational message.

Now, we'll extend the default `noop-test` by adding an `:os`, which tells
Jepsen how to handle operating system setup, and a `:db`, which we can
construct using the `db` function we just wrote. We'll test etcd version
`v3.1.5`.

```clj
(defn etcd-test
  "Given an options map from the command-line runner (e.g. :nodes, :ssh,
  :concurrency, ...), constructs a test map."
  [opts]
  (merge tests/noop-test
         {:name "etcd"
          :os debian/os
          :db (db "v3.1.5")}
         opts))
```

`noop-test`, like all Jepsen tests, is a map with keys like `:os`, `:name`,
`:db`, etc. See [jepsen.core](../../master/jepsen/src/jepsen/core.clj) for an
overview of test structure, and `jepsen.core/run` for the full definition of a
test.

Right now `noop-test` has stub implementations for those keys. But we can
use `merge` to build a copy of the `noop-test` map with *new* values for some
keys.

If we run this test, we'll see Jepsen using our code to set up debian,
pretend to tear down and install etcd, then start its workers.

```bash
$ lein run test
...
INFO [2017-03-30 10:08:30,852] jepsen node n2 - jepsen.os.debian :n2 setting up debian
INFO [2017-03-30 10:08:30,852] jepsen node n3 - jepsen.os.debian :n3 setting up debian
INFO [2017-03-30 10:08:30,852] jepsen node n4 - jepsen.os.debian :n4 setting up debian
INFO [2017-03-30 10:08:30,852] jepsen node n5 - jepsen.os.debian :n5 setting up debian
INFO [2017-03-30 10:08:30,852] jepsen node n1 - jepsen.os.debian :n1 setting up debian
INFO [2017-03-30 10:08:52,385] jepsen node n1 - jepsen.etcdemo :n1 tearing down etcd
INFO [2017-03-30 10:08:52,385] jepsen node n4 - jepsen.etcdemo :n4 tearing down etcd
INFO [2017-03-30 10:08:52,385] jepsen node n2 - jepsen.etcdemo :n2 tearing down etcd
INFO [2017-03-30 10:08:52,385] jepsen node n3 - jepsen.etcdemo :n3 tearing down etcd
INFO [2017-03-30 10:08:52,385] jepsen node n5 - jepsen.etcdemo :n5 tearing down etcd
INFO [2017-03-30 10:08:52,386] jepsen node n1 - jepsen.etcdemo :n1 installing etcd v3.1.5
INFO [2017-03-30 10:08:52,386] jepsen node n4 - jepsen.etcdemo :n4 installing etcd v3.1.5
INFO [2017-03-30 10:08:52,386] jepsen node n2 - jepsen.etcdemo :n2 installing etcd v3.1.5
INFO [2017-03-30 10:08:52,386] jepsen node n3 - jepsen.etcdemo :n3 installing etcd v3.1.5
INFO [2017-03-30 10:08:52,386] jepsen node n5 - jepsen.etcdemo :n5 installing etcd v3.1.5
...
```

See how the version string `"v3.1.5"` was passed from `etcd-test` to `db`,
where it was captured by the `reify` expression? This is how we *parameterize*
Jepsen tests, so the same code can test multiple versions or options. Note also
that the object `reify` returns closes over its lexical scope, *remembering*
the value of `version`.

## Installing the DB

With the skeleton of the DB in place, it's time to actually install something. Let's take a quick look at the [installation instructions for etcd](https://github.com/coreos/etcd/releases/tag/v3.1.5). It looks like we'll need to download a tarball, unpack it into a directory, set an environment variable for the API version, and run the `etcd` binary with it.

We'll have to be root to install packages, so we'll use `jepsen.control/su` to
assume root privileges. Note that `su` (and its companions `sudo`, `cd`, etc)
establish *dynamic*, not *lexical* scope--their effects apply not only to the
code they enclose, but down the call stack to any functions called. Then we'll
use `jepsen.control.util/install-tarball!` to download the etcd tarball and
install it to `/opt/etcd`.

```clj
(def dir "/opt/etcd")

(defn db
  "Etcd DB for a particular version."
  [version]
  (reify db/DB
    (setup! [_ test node]
      (info node "installing etcd" version)
      (c/su
       (let [url (str "https://storage.googleapis.com/etcd/" version
                  "/etcd-" version "-linux-amd64.tar.gz")]
        (cu/install-tarball! c/*host* url dir))))

    (teardown! [_ test node]
      (info node "tearing down etcd"))))
```
We're using `jepsen.control/su` to become root (via sudo) while we remove the
etcd directory. `jepsen.control` provides a comprehensive DSL for executing
arbitrary shell commands on remote nodes.

Now `lein run test` will go and install etcd itself. Note that Jepsen runs
`setup` and `teardown` concurrently across all nodes. This might take a little
while because each node has to download the tarball, but on future runs, Jepsen
will re-use the cached tarball on disk.

## Starting the DB


Per the [clustering instructions](https://coreos.com/etcd/docs/latest/v2/clustering.html), we'll need to generate a string like `"ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380"`, so that our nodes know which nodes are a part of the cluster. Let's write a few small functions to build those strings:

```clj
(defn node-url
  "An HTTP url for connecting to a node on a particular port."
  [node port]
  (str "http://" (name node) ":" port))

(defn peer-url
  "The HTTP url for other peers to talk to a node."
  [node]
  (node-url node 2380))

(defn client-url
  "The HTTP url clients use to talk to a node."
  [node]
  (node-url node 2379))

(defn initial-cluster
  "Constructs an initial cluster string for a test, like
  \"foo=foo:2380,bar=bar:2380,...\""
  [test]
  (->> (:nodes test)
       (map (fn [node]
              (str (name node) "=" (peer-url node))))
       (str/join ",")))
```

The `->>` threading macro takes each form and inserts it into the next form as
a final argument. So `(->> test :nodes)` becomes `(:nodes test)`, and `(->> test
:nodes (map-indexed (fn ...)))` becomes `(map-indexed (fn ...) (:nodes test))`,
and so on. Normal function calls often look "inside out", but the `->>` macro
lets us write a chain of operations "in order"--like an object-oriented
language's `foo.bar().baz()` notation.

So in `initial-cluster`, we take the nodes from a test map, and map each node
to a string: its name, the "=" string, and its peer url. Then we join all those
strings together with commas.

With this in place, we'll tell the DB how to start up etcd as a daemon. We
could use an init script or service to start and stop the program, but since
we're working with a plain binary here, we'll use Debian's `start-stop-daemon`
command to run `etcd` in the background.

We'll need a few more constants: the etcd binary, where to log output to, and
where to store a pidfile.

```clj
(def dir     "/opt/etcd")
(def binary "etcd")
(def logfile (str dir "/etcd.log"))
(def pidfile (str dir "/etcd.pid"))
```

Now we'll use `jepsen.control.util`'s functions for starting and stopping
daemons to spin up etcd. Per the
[documentation](https://coreos.com/etcd/docs/latest/v2/clustering.html), we'll
need to provide a node name, urls to listen on for clients and peers, and some
initial cluster state.

```clj
    (setup! [_ test node]
      (info node "installing etcd" version)
      (c/su
        (let [url (str "https://storage.googleapis.com/etcd/" version
                       "/etcd-" version "-linux-amd64.tar.gz")]
          (cu/install-tarball! c/*host* url dir))

        (cu/start-daemon!
          {:logfile logfile
           :pidfile pidfile
           :chdir   dir}
          binary
          :--log-output                   :stderr
          :--name                         (name node)
          :--listen-peer-urls             (peer-url   node)
          :--listen-client-urls           (client-url node)
          :--advertise-client-urls        (client-url node)
          :--initial-cluster-state        :new
          :--initial-advertise-peer-urls  (peer-url node)
          :--initial-cluster              (initial-cluster test))

          (Thread/sleep 10000)))
```

We'll sleep for a little bit after starting the cluster, so it has a chance to boot up and perform its initial handshakes.

## Tearing down

To make sure that every run starts fresh, even if a previous run crashed,
Jepsen performs a DB teardown at the start of the test, before setup. Then it
tears the DB down again at the conclusion of the test. To tear down, we'll use
`stop-daemon!`, and then delete the etcd directory, so that future runs won't
accidentally read data from our current run.

```clj
    (teardown! [_ test node]
      (info node "tearing down etcd")
      (cu/stop-daemon! binary pidfile)
      (c/su
        (c/exec :rm :-rf dir)))))
```

We use `jepsen.control/exec` to run a shell command: `rm -rf`. 
Jepsen automatically binds `exec` to operate on the `node` being set up
during `db/setup!`, but we can connect to arbitrary nodes if we need to. Note
that `exec` can take any mixture of strings, numbers, keywords--it'll convert
them to strings and perform appropriate shell escaping. You can use
`jepsen.control/lit` for an unescaped literal string, if need be. `:>` and
`:>>` perform shell redirection as you'd expect. That's an easy way to write
out configuration files to disk, for databases that need config.

Let's try it out.

```bash
$ lein run test
NFO [2017-03-30 12:08:19,755] jepsen node n5 - jepsen.etcdemo :n5 installing etcd v3.1.5
INFO [2017-03-30 12:08:19,755] jepsen node n1 - jepsen.etcdemo :n1 installing etcd v3.1.5
INFO [2017-03-30 12:08:19,755] jepsen node n2 - jepsen.etcdemo :n2 installing etcd v3.1.5
INFO [2017-03-30 12:08:19,755] jepsen node n4 - jepsen.etcdemo :n4 installing etcd v3.1.5
INFO [2017-03-30 12:08:19,855] jepsen node n3 - jepsen.etcdemo :n3 installing etcd v3.1.5
INFO [2017-03-30 12:08:20,866] jepsen node n4 - jepsen.control.util starting etcd
INFO [2017-03-30 12:08:20,866] jepsen node n1 - jepsen.control.util starting etcd
INFO [2017-03-30 12:08:20,866] jepsen node n5 - jepsen.control.util starting etcd
INFO [2017-03-30 12:08:20,866] jepsen node n2 - jepsen.control.util starting etcd
INFO [2017-03-30 12:08:20,963] jepsen node n3 - jepsen.control.util starting etcd
...
```

Looks good. We can confirm that `teardown` did its job by checking that the etcd directory is empty after the test.

```bash
$ ssh n1 ls /opt/etcd
ls: cannot access /opt/etcd: No such file or directory
```

## Log files

Hang on--if we delete etcd's files after every run, how do we figure out
what the database did? It'd be nice if we could download a copies of the
database's logs before cleaning up. For that, we'll use the `db/LogFiles`
protocol, and return a list of log file paths to download.

```clj
(defn db
  "Etcd DB for a particular version."
  [version]
  (reify db/DB
    (setup! [_ test node]
      ...)

    (teardown! [_ test node]
      ...)

    db/LogFiles
    (log-files [_ test node]
      [logfile])))
```

Now, when we run a test, we'll find a copy of the log for each node, stored in the local directory `store/latest/<node-name>/`. If we get stuck setting up the DB, we can check those logs to see what's gone wrong.

```bash
$ less store/latest/n1/etcd.log
...
2017-03-30 10:23:45.199100 I | etcdserver: published {Name:n1 ClientURLs:[http://n1:2379]} to cluster 96b564e4558d8dc1
2017-03-30 10:23:45.199122 I | embed: ready to serve client requests
2017-03-30 10:23:45.199427 N | embed: serving insecure client requests on 192.168.122.11:2379, this is strongly discouraged!
2017-03-30 10:23:45.245218 N | etcdserver/membership: set the initial cluster version to 3.1
2017-03-30 10:23:45.245268 I | etcdserver/api: enabled capabilities for version 3.1
...
```

With the database ready, it's time to [write a client](client.md).
