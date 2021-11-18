---
layout: post
title: Fix a PGO replica in a k8s cluster that fails to start
tags:
  - how-to
---

## The logs

```
2021-11-18 16:43:42,067 INFO: Lock owner: db-blahblah-blah; I am db-blah-blahblah-blah
2021-11-18 16:43:42,067 INFO: Still starting up as a standby.
2021-11-18 16:43:42,069 INFO: Lock owner: db-blahblah-blah; I am db-blah-blahblah-blah
2021-11-18 16:43:42,069 INFO: does not have lock
2021-11-18 16:43:42,069 INFO: establishing a new patroni connection to the postgres cluster
2021-11-18 16:43:42,254 INFO: establishing a new patroni connection to the postgres cluster
2021-11-18 16:43:42,256 WARNING: Retry got exception: 'connection problems'
2021-11-18 16:43:42,256 INFO: Error communicating with PostgreSQL. Will try again later
2021-11-18 16:43:48,719 WARNING: Retry got exception: 'connection problems'
```

## A fix

Get a shell in the the replica pod:

```sh
kubectl -n sweet-namespace exec -it deployment/db-blah -- bash

# inside the shell

$ patronictl list

+ Cluster: db (6923656205005025428) --------+---------+----------+----+-----------+
| Member                   | Host           | Role    | State    | TL | Lag in MB |
+--------------------------+----------------+---------+----------+----+-----------+
| db-blahblah-blah         | 100.120.102.15 | Leader  | running  | 50 |           |
| db-blah-blahblah-blah    | 100.121.174.24 | Replica | starting |    |   unknown |
+--------------------------+----------------+---------+----------+----+-----------+

$ patronictl reinit db db-blah-blahblah-blah

+ Cluster: db (6923656205005025428) --------+---------+----------+----+-----------+
| Member                   | Host           | Role    | State    | TL | Lag in MB |
+--------------------------+----------------+---------+----------+----+-----------+
| db-blahblah-blah         | 100.120.102.15 | Leader  | running  | 50 |           |
| db-blah-blahblah-blah    | 100.121.174.24 | Replica | starting |    |   unknown |
+--------------------------+----------------+---------+----------+----+-----------+
Are you sure you want to reinitialize members db-blah-blahblah-blah   ? [y/N]: y
Success: reinitialize for member db-blah-blahblah-blah

# some time later...

$ patronictl list

+ Cluster: db (6923656205005025428) --------+---------+---------+----+-----------+
| Member                   | Host           | Role    | State   | TL | Lag in MB |
+--------------------------+----------------+---------+---------+----+-----------+
| db-blahblah-blah         | 100.120.102.15 | Leader  | running | 50 |           |
| db-blah-blahblah-blah    | 100.121.174.24 | Replica | running | 49 |    12840  |
+--------------------------+----------------+---------+---------+----+-----------+
```

Looks okay, pod is in ready state. Lag seems too big, but whatever, it's recovering.

## Handy Stuff

```sh
# I always forget how I did it last time
kubectl -n ns exec -it deployment/db-blah -c database -- patronictl list
```
