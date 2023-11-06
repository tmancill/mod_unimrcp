# FreeSWITCH + mod_unimrcp in Docker

These are notes I took while setting up
[FreeSWITCH](https://github.com/signalwire/freeswitch/) version 1.10.10
once I realized that I also needed to build `mod_unimrcp` from sources.
It assumes you're running on Linux, in this case Debian bookworm, and that
Docker is already installed.  Your mileage may vary.

Also note that this process could be rolled up into a single build.
The reason they are split out is so the base image Dockerfile could
be pushed upstream
(which occurred in https://github.com/signalwire/freeswitch/pull/2234).

## Build a base image with `mod_unimrcp`

### Build a "base" runtime image

Create a [SignalWire](https://signalwire.com) account and then a
[personal access token](https://id.signalwire.com/personal_access_tokens)
that is used to retrieve SignalWire's packaged builds of FreeSWITCH for Debian.
Export the personal access token to your environment as `SIGNALWIRE_PAT` and then build a
the image.

```
git clone https://github.com/signalwire/freeswitch.git
cd freeswitch/docker/master
export FS_BASE_TAG=fs:vanilla

docker build --build-arg="TOKEN=${SIGNALWIRE_PAT}" \
             --build-arg="FS_META_PACKAGE=freeswitch-meta-vanilla" \
             --file Dockerfile . --tag ${FS_BASE_TAG}
```

When this completes, you will have a functioning runtime image that can also
be used as part of a multi-stage build to compile `mod_unimrcp`
(see [mod_unimrcp](#mod_unimrcp) below).

To run the image at this stage directly, adjust the [start-fs-docker.sh](https://gist.github.com/tmancill/99fae225568d06fcb26e75a831f1fa3c#file-start-fs-docker-sh) script
to match the image tag set by `FS_BASE_TAG` and for
the desired mount points, etc.

### mod_unimrcp

[mod_unimrcp](https://github.com/freeswitch/mod_unimrcp) is no longer part of
the official FreeSWITCH build.  If you use it to integrate to your
ASR/TTS resources, you can use the [Dockerfile-unimrcp](https://gist.github.com/tmancill/99fae225568d06fcb26e75a831f1fa3c#file-dockerfile-unimrcp)
to retrieve the sources, compile the module, and install it in the FS "base"
image built previously.  Or to use the module in a different base image, you
can build this image and then copy out the binaries as part of another
multi-stage build.

```
docker build . --build-arg="FS_BASE_TAG=${FS_BASE_TAG}" \
               --file Dockerfile-unimrcp --tag fs:unimrcp
```

### Add any additional packaged modules

If there any other modules you need as part of your runtime image, they can
be added to one of the previous docker builds, or in a final layer as in the
example [Dockerfile-fsgcp](https://gist.github.com/tmancill/99fae225568d06fcb26e75a831f1fa3c#file-dockerfile-fsgcp).
See [fs-packages-bookworm.txt](https://gist.github.com/tmancill/99fae225568d06fcb26e75a831f1fa3c#file-fs-packages-bookworm-txt) for a list of candidate packages.

```sh
docker build -f Dockerfile-fsgcp . -t fs:gcp
```

## Prepare the host system

The FS configuration, logs, etc. will be bind-mounted outside
of the container.  Since FS will run as the user `freeswitch`,
it is convenient for the container host to use the same UID and GID.

This assumes that UID and GID 999 are available.  If not, the UID and GID
should also be adjusted in `signalwire/freeswitch/docker/master/Dockerfile` **before** building the base image.

```sh
sudo groupadd -r freeswitch --gid=999 && useradd -r -g freeswitch --uid=999 freeswitch
sudo mkdir -p /etc/freeswitch /var/lib/freeswitch /var/log/freeswitch /usr/share/freeswitch
sudo chown freeswitch:freeswitch /etc/freeswitch /var/lib/freeswitch /var/log/freeswitch /usr/share/freeswitch
```

Note that the directories needed will vary based on your use case.

## Running FS in a container

Adjust the [start-fs-docker.sh](https://gist.github.com/tmancill/99fae225568d06fcb26e75a831f1fa3c#file-start-fs-docker-sh) script as needed and start
the image.  Having `/etc/freeswitch/` outside of the Docker container makes it
easy to modify the configs.  (During development and experiementation, I run a
`git init` on the configuration directory so it's easy to view diffs and track
changes to configuration.)

FreeSWITCH (FS) runs well in a container.  However, it (all but) requires host networking,
and thus only one instance of FS can be run per host.
If this instance of FS is going to communicate directly with the outside world - i.e., without a SBC - ensure that the VPC firewill is updated to permit the port ranges needed for SIP[S] and [S]RTP.
