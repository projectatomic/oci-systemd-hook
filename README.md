# OCI systemd hooks

OCI systemd hook enables users to run systemd in docker and [OCI][] compatible runtimes such as runc without requiring `--privileged` flag.

This project produces a C binary that can be used with runc and Docker (with minor code changes).
If you clone this branch and build/install `oci-systemd-hook`, a binary should be placed in
`/usr/libexec/oci/hooks.d` named `oci-systemd-hook`.

Running Docker or OCI runc containers with this executable, oci-systemd-hook is called just before a container is started and after it is provisioned.  If the CMD to run inside of the container is `init` or `systemd`, this hook will configure the container image to run a systemd environment.  For all other CMD's, this hook will just exit.

When oci-systemd-hook detects systemd inside of the container it does the following:

* Mounts a tmpfs on /run and /tmp
-  If there is content in the container image's /run and /tmp that content will be copied onto the tmpfs.
* Creates a /etc/machine-id based on the the container's UUID
* Mounts the hosts /sys/fs/cgroups file systemd read-only into the container
- /sys/fs/cgroup/systemd will be mounted read/write into the container.

When the container stops, these file systems will be umounted.

systemd is expected to be able to run within the container without requiring
the `--privileged` option.  However you will still need to specify a special `--stop-signal`.  Standard docker containers sends SIGTERM to pid 1, but systemd
does not shut down properly when it recieves a SIGTERM.  systemd specified that it needs to receive a RTMIN+3 signal to shutdown properly.


**Usage**

If you created a container image based on a Dockerfile like the following:
```
cat Dockerfile
FROM fedora:latest
ENV container docker
RUN yum -y update && yum -y install httpd && yum clean all
RUN systemctl mask dnf-makecache.timer && systemctl enable httpd
CMD [ "/sbin/init" ]
```

(The `systemctl mask dnf-makecache.timer` is a workaround for a container base image bug)

You should then be able to execute the following commands:

```
docker build -t httpd .
docker run -ti --stop-signal=RTMIN+3 httpd
```

If you run this hook along with oci-register-machine oci hook, you will be able
to show the container's journal information on the host, using journalctl.

```
journalctl -M CONTAINER_UUID
```

To use this directly with runc, modify or add the following to config.json.

```
    "hooks": {
        "prestart": [
            {
                "path": "/usr/libexec/oci/hooks.d/oci-systemd-hook"
            }
        ]
    },
...
    "process": {
        "capabilities": {
...
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE",
                "CAP_MKNOD",
                "CAP_CHOWN",
                "CAP_DAC_OVERRIDE",
                "CAP_FSETID",
                "CAP_FOWNER",
                "CAP_NET_RAW",
                "CAP_SETGID",
                "CAP_SETUID",
                "CAP_SETFCAP",
                "CAP_SETPCAP",
                "CAP_SYS_CHROOT"
...
        "env": [
            "container=oci",
```

**Disabling oci-systemd-hook**

To disable oci-systemd-hook for a particular run, which is primarily useful in an Atomic Host environment, the environment variable 'oci-systemd-hook' can be set to 'disabled'.  This prevents oci-systemd-hook from being run for that invocation.  A sample usage is:

```
docker run --env oci-systemd-hook=disabled -it --rm  fedora /bin/bash
```

**To build and install**

Prior to installing oci-systemd-hook, install the following packages on your linux distro:

* autoconf
* automake
* gcc
* git 
* go-md2man
* libmount-devel
* libselinux-devel
* yajl-devel 

In Fedora, you can use this command:

```
 yum -y install \
    autoconf \
    automake \
    gcc \
    git \
    go-md2man \
    libmount-devel \
    libselinux-devel \
    yajl-devel
```

Then **clone** this branch and follow these steps:

```
git clone https://github.com/projectatomic/oci-systemd-hook
cd oci-systemd-hook
autoreconf -i
./configure --libexecdir=/usr/libexec/oci/hooks.d
make
make install
```

[OCI]: https://github.com/opencontainers/runtime-spec
