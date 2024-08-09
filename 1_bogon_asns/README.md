# Catching bogon ASNs on the Internet

- [Publish Date](#publish-date)
- [Installation](#installation)
  * [BGP Scanner](#bgp-scanner)
    + [Building from source using Podman](#building-from-source-using-podman)
  * [FlashText](#flashtext)
  * [Python script](#python-script)
  * [RIS Raw Data](#ris-raw-data)
- [Usage](#usage)
- [References](#references)

Today we will try to find bogon ASNs without any network equipment using only RIPE RIS wild Internet routing information and our modest coding skills in Python 3. Probably, you may want to use this PoC to monitor your own network.

## Publish Date
* 2019-06-14
## Update History
* 2024-08-09: Fix broken and outdated links, update Dockerfile/Containerfile and replace docker by podman

## Installation

### BGP Scanner
First you need to install [bgpscanner](https://gitlab.com/Isolario/bgpscanner) <sup id="a1">[1](#f1)</sup> utility from [Isolario](https://conference.apnic.net/45/assets/files/APNT806/Isolario-project-The-real-time-Internet-routing-observatory.pdf) <sup id="a2">[2](#f2)</sup> project.

#### Building from source using Podman
To build from source we will use Podman and temporary build image without any packaging to DEB or RPM packages.

Create two directories with names `bgpscanner` and `build`:
```bash
$ mkdir bgpscanner build
```

Create a [Dockerfile or Containerfile](https://github.com/freefd/bogon_ASNs/blob/master/Dockerfile) <sup id="a10">[10](#f10)</sup> for `bgpscanner` build process in the directory `build`. It is based on the Debian image:
```bash
$ cat <<$'EOF' > build/Containerfile
FROM debian:latest
RUN apt update \
    && apt install -y git cmake ninja-build pkg-config \
    python3-pip zlib1g-dev libbz2-dev liblzma-dev liblz4-dev \
    && pip3 install meson --break-system-packages \
    && git clone https://gitlab.com/Isolario/bgpscanner.git /root/bgpscanner \
    && mkdir /root/bgpscanner/build
WORKDIR /root/bgpscanner/build
RUN /usr/local/bin/meson --buildtype=release .. \
    && export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/bgpscanner/build/./subprojects/isocore \
    && ldconfig \
    && ninja \
    && cp -f /root/bgpscanner/build/bgpscanner /opt/ \
    && cp -f /root/bgpscanner/build/./subprojects/isocore/libisocore.so /opt/
EOF
```
Run `podman build` command to build a temporary image that will hold compiled `bgpscanner` and [libisocore.so](https://gitlab.com/Isolario/isocore) <sup id="a3">[3](#f3)</sup> binaries:
```
$ podman build --force-rm -t bgpscnr build/
STEP 1/4: FROM debian:latest
STEP 2/4: RUN apt update     && apt install -y git cmake ninja-build pkg-config     python3-pip zlib1g-dev libbz2-dev liblzma-dev liblz4-dev     && pip3 install meson --break-system-packages     && git clone https://gitlab.com/Isolario/bgpscanner.git /root/bgpscanner     && mkdir /root/bgpscanner/build

... output omitted for brevity ...

[32/32] Linking target bgpscanner
COMMIT bgpscnr
--> c8d67e872dd3
Successfully tagged localhost/bgpscnr:latest
c8d67e872dd3aa0e869bc26a321f721cd2f93e30c2cd013375daf4f174724e46
```

After completing a process, check that the image exists:

```
$ podman image ls -f reference=bgpscnr
REPOSITORY                TAG         IMAGE ID      CREATED        SIZE
localhost/bgpscnr         latest      c8d67e872dd3  3 minutes ago  807 MB
```

Now you must be able to run the container to get the compiled binary files to directory `bgpscanner`:

`$ podman run -ti --rm -v $(pwd)/bgpscanner:/mnt bgpscnr:latest sh -c "cp /opt/* /mnt/"`

The container will copy two files, exit and to be terminated immediately. Please check the contents of `bgpscanner` directory after that all:

```
$ ls -l bgpscanner
total 284
-rwxr-xr-x 1 root root  46992 Apr 15 18:55 bgpscanner
-rwxr-xr-x 1 root root 238392 Apr 15 18:55 libisocore.so
```

If so, you can also remove the temporary image:

```
$ podman image rm bgpscnr:latest
```

Due to the fact that `bgpscanner` was built using a shared library, we have to change LD_LIBRARY_PATH at runtime and check that all libs are accessible:

```
$ cd bgpscanner
$ LD_LIBRARY_PATH=. ldd ./bgpscanner 
	linux-vdso.so.1 (0x00007ffcbd9f3000)
	libisocore.so => ./libisocore.so (0x00007f8d7f3a1000) <<<< HERE IS THE MAGIC
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f8d7f364000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8d7f17a000)
	libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007f8d7ef5d000)
	libbz2.so.1.0 => /lib/x86_64-linux-gnu/libbz2.so.1.0 (0x00007f8d7ef4a000)
	liblz4.so.1 => /usr/lib/x86_64-linux-gnu/liblz4.so.1 (0x00007f8d7ed13000)
	liblzma.so.5 => /lib/x86_64-linux-gnu/liblzma.so.5 (0x00007f8d7eaeb000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f8d7f7e4000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f8d7eae5000)
```

If there are no "not found" libs, we can now invoke `bgpscanner` as follows:

```
$ LD_LIBRARY_PATH=. ./bgpscanner [options]
```

For example, simulate `help` output, however, there is no `-help` argument:
```bash
$ LD_LIBRARY_PATH=. ./bgpscanner -h
./bgpscanner: invalid option -- 'h' # <<< Throws an error and it is okay
bgpscanner: The Isolario MRT data reader utility
Usage:
	bgpscanner [-cdlL] [-mM COMMSTRING] [-pP PATHEXPR] [-i ADDR] [-I FILE] [-a AS] [-A FILE] [-e PREFIX] [-E FILE] [-t ATTR_CODE] [-T FILE] [-o FILE] [FILE...]
	bgpscanner [-cdlL] [-mM COMMSTRING] [-pP PATHEXPR] [-i ADDR] [-I FILE] [-a AS] [-A FILE] [-s PREFIX] [-S FILE] [-t ATTR_CODE] [-T FILE] [-o FILE] [FILE...]
	bgpscanner [-cdlL] [-mM COMMSTRING] [-pP PATHEXPR] [-i ADDR] [-I FILE] [-a AS] [-A FILE] [-u PREFIX] [-U FILE] [-t ATTR_CODE] [-T FILE] [-o FILE] [FILE...]
	bgpscanner [-cdlL] [-mM COMMSTRING] [-pP PATHEXPR] [-i ADDR] [-I FILE] [-a AS] [-A FILE] [-r PREFIX] [-R FILE] [-t ATTR_CODE] [-T FILE] [-o FILE] [FILE...]

...
```

### FlashText
For a fast and efficient loop through the records of a huge data file, we will use the flashtext<sup id="a4">[4](#f4)</sup> python's module implementation<sup id="a5">[5](#f5)</sup>:
```bash
$ pip3 install --user flashtext --break-system-packages
```

### Python script
Clone [bogons_ASNS](https://github.com/freefd/bogon_ASNs) <sup id="a6">[6](#f6)</sup> repository:
```bash
$ git clone https://github.com/freefd/bogon_ASNs
Cloning into 'bogon_ASNs'...
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 14 (delta 2), reused 12 (delta 0), pack-reused 0
Unpacking objects: 100% (14/14), done.
```
And change working directory to it

```bash
$ cd bogon_ASNs/
```

### RIS Raw Data
Actually, RIS files are contain data in MRT format what is described in [RFC6396](https://tools.ietf.org/html/rfc6396) <sup id="a7">[7](#f7)</sup>. 
Download a latest [RIS Raw Data](http://data.ris.ripe.net/rrc00/) <sup id="a8">[8](#f8)</sup> snapshot:
```bash
$ wget http://data.ris.ripe.net/rrc00/latest-bview.gz
```

## Usage
Run command to extract and store RIS Raw Data with a text file named "bogons":
```bash
$ LD_LIBRARY_PATH=.. ../bgpscanner latest-bview.gz | awk -F'|' '{print $3"|"$2}' 2>/dev/null | sort -V | uniq > bogons
```
The output should be similar as follows:
```
396503 32097 33387 42615|185.186.8.0/24
396503 32097 33387 44421|185.234.214.0/24
396503 32097 33387 64236 17019|66.85.92.0/22
```

Run python script to find out whether any bogon ASNs in the file "bogons" that was previously saved:
```bash
$ python3 scripts/bogon_asns.py bogons
```
First argument must be a path to the file we created on the previous step.

The example output:
```
65225|3333 1103 9498 55410 55410 38266 65225|2402:3a80:ca0::/43
65542, 65542|3333 1103 9498 132717 132717 65542 65542 134279|103.197.121.0/24
64646, 65001, 65083|3333 1103 30844 327693 37611 {16637,64646,65001,65083}|169.0.0.0/15
64555|3333 1103 58453 45899 45899 45899 45899 {16625,64555}|2001:ee0:3200::/40
65817|3333 1136 9498 703 65817|202.75.201.0/24
```

Where the format is:
```
bogon ASN(s)|as-path|prefix(es)
```

However, you can define filter(s) to extract only records you need:
```bash
$ LD_LIBRARY_PATH=.. ../bgpscanner -p ' 55410 ' latest-bview.gz 2>/dev/null | awk -F'|' '{print $3"|"$2}' | sort -V | uniq > bogons
```
The example above will return only records contained AS55410 in AS path. More information about filtering you can find at the References below<sup id="a9">[9](#f9)</sup>.

## References
<b id="f1">1</b>. [BGP Scanner](https://gitlab.com/Isolario/bgpscanner) [↩](#a1)<br/>
<b id="f2">2</b>. [Isolario Project](https://conference.apnic.net/45/assets/files/APNT806/Isolario-project-The-real-time-Internet-routing-observatory.pdf) [↩](#a2)<br/>
<b id="f3">3</b>. [Isocore](https://gitlab.com/Isolario/isocore) [↩](#a3)<br/>
<b id="f4">4</b>. [FlashText algorithm](https://arxiv.org/pdf/1711.00046.pdf) [↩](#a4)<br/>
<b id="f5">5</b>. [FlashText python module documentation](https://flashtext.readthedocs.io/en/latest/) [↩](#a5)<br/>
<b id="f6">6</b>. [Bogons ASNs Repository](https://github.com/freefd/bogon_ASNs) [↩](#a6)<br/>
<b id="f7">7</b>. [RFC6396](https://tools.ietf.org/html/rfc6396) [↩](#a7)<br/>
<b id="f8">8</b>. [RIS Raw Data](https://www.ripe.net/analyse/internet-measurements/routing-information-service-ris/ris-raw-data) [↩](#a8)<br/>
<b id="f9">9</b>. About bgpscanner filtering options: [↩](#a9)
* [https://gitlab.com/Isolario/bgpscanner/wikis/Home#filtering](https://gitlab.com/Isolario/bgpscanner/wikis/Home#filtering)
* [https://ripe77.ripe.net/presentations/12-pres.pdf#page=20](https://ripe77.ripe.net/presentations/12-pres.pdf#page=20) (page 20)

<br/>
<b id="f10">10</b>. [Dockerfile](https://github.com/freefd/bogon_ASNs/blob/master/Dockerfile) [↩](#a10)<br/>