Running StashCache Cache in a Container
=======================================

The OSG operates the [StashCache data federation](/data/stashcache/overview), which
provides organizations with a method to distribute their data in a scalable manner to thousands of jobs without needing
to pre-stage data across sites or operate their own scalable infrastructure.

[Stash Caches](/data/stashcache/install-cache) transfer data to clients such as jobs or users. A set of caches are operated across the OSG for the benefit of nearby sites;
in addition, each site may run its own cache in order to reduce the amount of data transferred over the WAN.
This document outlines how to run StashCache in a Docker container.

Before Starting
---------------

Before starting the installation process, consider the following points:

1. **Docker:** For the purpose of this guide, the host must have a running docker service and you must have the ability to start containers (i.e., belong to the `docker` Unix group).
1. **Network ports:** Stash Cache listens for incoming HTTP/S connections on port 8000 (by default)
1. **File Systems:** Stash Cache needs a partition on the host to store cached data

Configuring Stashcache
----------------------

In addition to the required configuration above (ports and file systems), you may also configure the behavior of your cache with the following variables using an enviroment variable file:

Where the environment file on the docker host, `/opt/xcache/.env`, has (at least)the following contents:
```file
XC_RESOURCENAME=ProductionCache
XC_ROOTDIR=/cache
```

### Optional Configuration ###

Further behaviour of the stashcache can be cofigured by setting the followin in the enviroment variable file

- `XC_SPACE_HIGH_WM`: High watermark for disk usage;
      when usage goes above the high watermark, the cache deletes until it hits the low watermark
- `XC_SPACE_LOW_WM`: Low watermark for disk usage;
      when usage goes above the high watermark, the cache deletes until it hits the low watermark
- `XC_PORT`: TCP port that XCache listens on
- `XC_RAMSIZE`: Amount of memory to use for storing blocks before writting them to disk. (Use higher for slower disks).
- `XC_BLOCKSIZE`: The size of the blocks in the cache
- `XC_PREFETCH`: Number of blocks to prefetch from a file at once.
       This is a value of how aggressive the cache is to request portions of a file. Set to `0` to disable

### Disabling OSG monitoring ###

By default, XCache reports to the OSG so that OSG staff can monitor the health of data federations.
If you would like to report monitoring information to another destination, you can disable the OSG monitoring by setting
the following in your environment variable configuration:

```file
DISABLE_OSG_MONITORING = true
```

!!! warning
    Do not disable OSG monitoring in any service that is to be used for any other than testing

Running Stashcache
------------------

To run the container, use docker run with the following options, replacing the text within angle brackets with your own values:

```console
user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```

It is recommended to use a container orchestration service such as [docker-compose](https://docs.docker.com/compose/) or [kubernetes](https://kubernetes.io/), or start the StashCache container with systemd.

### Running Stashcache on container with systemd

An example systemd service file for StashCache.
This will require creating the environment file in the directory `/opt/xcache/.env`. 

!!! note
    This example systemd file assumes `<HOST PORT>` is `8000` and  `<HOST PARTITION>` is `/srv/cache`.

```file
[Unit]
Description=StashCache Container
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker stop %n
ExecStartPre=-/usr/bin/docker rm %n
ExecStartPre=/usr/bin/docker pull opensciencegrid/stash-cache:stable
ExecStart=/usr/bin/docker run --rm --name %n --publish 8000:8000 --volume /srv/cache:/cache --env-file /opt/xcache/.env opensciencegrid/stash-cache:stable

[Install]
WantedBy=multi-user.target
```

This systemd file can be saved to `/etc/systemd/system/docker.stash-cache.service` and started with:

```console
root@host $ systemctl enable docker.stash-cache
root@host $ systemctl start docker.stash-cache
```

!!! warning
    You must [register](/data/stashcache/install-cache/#registering-the-cache) the cache before considering it a production service.



### Network Optimization ###

For caches that are connected to NIC's over `40Gbps` we recommend to disable the virtualized network and "bind" the container to the host network:

```console
user@host $ docker run --rm  \
             --network="host" \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```

### Memory Optimization ###

Stashcache uses the hosts memory in two ways:

1. Uses the own linux kernel as a way to cache files read from disk
1. As a buffer for writting blocks of files first in memory and then in disk (to account for slow disks).

An easy way to increase the performance of stashcache is to assign it more memory. You can use the [docker option](https://docs.docker.com/config/containers/resource_constraints/#limit-a-containers-access-to-memory) `--memory` and set it up to at least twice of `XC_RAMSIZE`.

```console
user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --memory=64g \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```


### Multiple Disks ###

For caches that store over `10 TB` or that have assigned space for storing the cached files over multiple partitions (`/partition1, /partition2, ...`) we recommend the following.

1. Create a config file `90-my-stash-cache-disks.cfg` with the following contents

        :::file
        pfc.spaces data
        oss.space data /data1
        oss.space data /data2
        .
        .
        .

1. Run the container with the following options:

        :::console
        user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --volume <HOST PARTITION>:/cache \
             --volume /partition1:/data1 \
             --volume /partition2:/data2 \
             --volume /opt/xcache/90-my-stash-cache-disks.cfg:/etc/xrootd/config.d/90-stash-cache-disks.cfg \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable

!!! note
    Under this configuration the `<HOST PARTITION>` is not used to store the files rather to store symlinks to the files in `/partition1` and `/partition2`

!!! warning
    For over 100 TB of assigned space we highly encourage to use this setup and mount `<HOST PARTITION>` in solid state disks or NVME.


Validating StashCache
---------------------

For example, if you've chosen `8212` as your host port, you can verify that it worked with the command:

```console
user@host $ curl http://localhost:8212/user/dweitzel/public/blast/queries/query1
```

Which should output:

```
>Derek's first query!
MPVSDSGFDNSSKTMKDDTIPTEDYEEITKESEMGDATKITSKIDANVIEKKDTDSENNITIAQDDEKVSWLQRVVEFFE
```

Getting Help
------------

To get assistance, please use the [this page](/common/help) or contact <help@opensciencegrid.org> directly.

