The latest versions of Docker for Windows and Docker for Mac can build non-native architecture containers. The Author (computermouth) reached out to Anil Madhavapeddy, an engineer at Docker, asking about why it wasn't the case on Linux, turns out [Linux can too](https://twitter.com/avsm/status/885028269397090306), there's just an issue with some distributions of qemu-user-static that's disallowing the functionality. These files enable (likely among other things) the ability to build `arm32v7/*` Docker images on a Linux host of a different architecture without putting qemu-*-static binaries in the image itself. (Only tested on Ubuntu 18.04 LTS x86_64)

### How do I know if I need them?

Run the following command:

`systemctl status systemd-binfmt`

If you see output like the following [note the `start condition failed` part]:

```
user@ubuntu:~# systemctl status systemd-binfmt
● systemd-binfmt.service - Set Up Additional Binary Formats
   Loaded: loaded (/lib/systemd/system/systemd-binfmt.service; static; vendor preset: enabled)
   Active: inactive (dead)
Condition: start condition failed at Tue 2019-12-24 17:09:25 CET; 30s ago
           ├─ ConditionDirectoryNotEmpty=|/lib/binfmt.d was not met
           ├─ ConditionDirectoryNotEmpty=|/usr/lib/binfmt.d was not met
           ├─ ConditionDirectoryNotEmpty=|/usr/local/lib/binfmt.d was not met
           ├─ ConditionDirectoryNotEmpty=|/etc/binfmt.d was not met
           └─ ConditionDirectoryNotEmpty=|/run/binfmt.d was not met
     Docs: man:systemd-binfmt.service(8)
           man:binfmt.d(5)
           https://www.kernel.org/doc/Documentation/binfmt_misc.txt

```

...you will need the files here in one of the above directories. Personally, I've been using them in `/lib/binfmt.d`.

### How do I install?

```
git clone https://github.com/eagle-1/qemu-static-conf.git
sudo mkdir -p /lib/binfmt.d
sudo cp qemu-static-conf/*.conf /lib/binfmt.d/
sudo systemctl restart systemd-binfmt.service
```

### How do I test if it's working

NOTE: You will need to `sudo docker` below if you haven't set up the appropriate [permissions](https://docs.docker.com/engine/installation/linux/linux-postinstall/). Your might have to restart docker service for [buildx](https://docs.docker.com/buildx/working-with-buildx/) to pick up the platforms.

```
user@ubuntu:~# docker buildx ls
NAME/NODE      DRIVER/ENDPOINT             STATUS  PLATFORMS
testbuilder *  docker-container
  testbuilder0 unix:///var/run/docker.sock running linux/amd64, linux/arm64, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
default        docker
  default      default                     running linux/amd64, linux/arm64, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6

user@ubuntu:~# docker run --rm -t arm32v6/alpine uname -m
armv7l

user@ubuntu:~# docker run --rm -t arm64v8/alpine uname -m
aarch64
```

If the container outputs its arch without error, you're good to go!

### Why isn't this built into Debian/Ubuntu (but it is in Fedora)?

Not sure, that systemd service is distributed with them out of the box, but Fedora supplies these files with their qemu-user-static package, while Debian and Ubuntu don't. Bug report is here: https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=868217
