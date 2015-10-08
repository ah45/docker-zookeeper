Dynamic Dockerized ZooKeeper
============================

A ZooKeeper Docker container that runs the 3.5.1-alpha for
[dynamically reconfiguring the cluster][dyn-conf].

[dyn-conf]: https://zookeeper.apache.org/doc/trunk/zookeeperReconfig.html

Main features:

* Can be run as a standalone server
* Can grow a cluster from 1 server to many just by starting new
  instances
* Exposes a volume for data storage
* Exposes JMX for monitoring/statistics gathering

Currently the dynamic reconfiguration is limited to seamless addition
of new nodes to the cluster and _simplified_—but still
manual—removal of nodes from the cluster.

Note that this image can be used to run a standalone ZooKeeper server,
there is no expectation of it forming part of a cluster or a
requirement to bootstrap a cluster with a certain number of nodes. You
can just run a single instance of this image and start using ZooKeeper
straight away (and then later add additional nodes as required.)

## Build

    docker build -t ah45/zookeeper .

## Run

As a standalone instance:

    docker run -d --name zk1 -h zk1 --env ID=1 -p 2181:2181 ah45/zookeeper

To join an existing cluster:

    docker run -d --name zk2 -h zk2 --env ID=2 --env QUORUM=zk1:2181 -p 2182:2181 ah45/zookeeper

If you run those two commands on the same Docker host you'll have a 2
node cluster, as simple as that.

(Note: don't ever run a two node cluster.)

### `ENV` Variables

* `ID` the ZooKeeper server ID of this instance.
* `QUORUM` the, optional, address of an existing ZooKeeper server
  whose cluster this server should join.

There are two additional variables for controlling JMX connectivity:

* `JMX_HOST` the hostname/IP to advertise as. This should be the
  _external_ IP clients will be connecting to. Defaults to the
  container IP.
* `JMX_PORT` the port to expose for JMX connections, see the JMX
  section below for more details. Defaults to `7000`.

## Cluster Management

As noted in the previous section adding a new node to a cluster is as
simple as specifying a cluster quorum address for it to connect to
(`--env QUORUM=zk1:2181`.)

To remove a node from a cluster you need to manually tell the cluster
that it is going/has gone away. This can be done in two ways:

1. From the node itself.

   The image contains a `leave-cluster` command that you can `exec`
   which will cause the node to leave the cluster it is currently
   joined to:

        docker exec <container> leave-cluster

2. From any node in the cluster.

   The image also contains a `remove-node` command which takes the
   IDs of one or more servers as its arguments and removes those nodes
   from the cluster:

        # remove a single node
        docker exec <container> remove-node 4
        # remove multiple nodes
        docker exec <container> remove-node 4 5 6

(This is necessary as there is no sensible way to distinguish between
a node being unavailable and it having being removed permanently
without manual intervention.)

Having removed the node from the cluster you can safely stop the
container and delete it.

## Data Storage

All ZooKeeper data is written to `/data` which is created as a Docker
volume. You'll probably want to mount it somewhere permanent and safe.

## JMX

[Monitoring and statistics gathering of ZooKeeper][monitor] can be
performed via JMX. Unfortunately remote JMX access to processes
running inside Docker containers can be a little finicky to setup.

[monitor]: https://zookeeper.apache.org/doc/trunk/zookeeperJMX.html

This image hopefully takes away most of the pain and confusion.

There are really only two things to remember:

* The external port needs to match the internal port.

  Don't map the internal port to an unknown external port (`-p <jmx
  port>`) or to a different external port (`-p <some port>:<jmx
  port>`) _always_ keep them the same (`-p <jmx port>:<jmx port>`.)
* JMX needs to know the _external_ IP clients will connect to.

  By default the containers IP address is used as the JMX hostname, if
  you need to specify a different value then do so as a `JMX_HOST` env
  variable (`--env JMX_HOST=<jmx host IP>`.)

So, you should have something like:

    --env JMX_HOST=192.168.99.100 --env JMX_PORT=7000 -p 7000:7000

… in your Docker `run` command and not anything like:

    -p 7000
    -p 32790:7000
    --env JMX_PORT=10000 -p 10000

Providing you adhere to those two maxims everything should just work.

### Security

Due to a desire not to overly complicate the configuration JMX is
running _un_secured: with no authentication and with SSL
disabled. Don't expose it to the outside world.

The "best" approach is would be to _not_ expose JMX on the Docker host
and instead run a metrics collector in another container linked to
this one. If you were to do that you don't need to set the `JMX_PORT`
(the default of 7000 should be fine) and should set `JMX_HOST` to the
link name you'll use (e.g. `--env JMX_HOST=zk1` if you'll link it
to the collector container as `--link <zk container>:zk1`.)

## License

Copyright © 2015 Adam Harper.

This project is licensed under the terms of the MIT license.
