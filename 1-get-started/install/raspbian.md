---
layout: documentation
title: Install RethinkDB on Raspbian
title_image: /assets/images/docs/install-platforms/raspbian.png
active: docs
docs_active: install
permalink: docs/install/raspbian/
---
{% include install-docs-header.md %}
{% include install-community-platform-warning.md %}

These instructions were updated after the 1.16 release, but have not been tested.

# Compile from source #

## Get the build dependencies ##

Install the main dependencies:

```
sudo apt-get install g++ protobuf-compiler libprotobuf-dev \
                     libboost-dev curl m4 wget
```

## Prepare the raspberrypi ##

Building RethinkDB require more memory than what is available on a Raspberry Pi.
Make sure you have a SWAP partition of at least 1GB.

Make also sure that you have at least 1GB available on your SD card.


## Get the source code ##

Download and extract the archive:

```bash
wget http://download.rethinkdb.com/dist/rethinkdb-{{site.version.full}}.tgz
tar xf rethinkdb-{{site.version.full}}.tgz
```

## Build RethinkDB ##

Kick off the build process:

```bash
cd rethinkdb-{{site.version.full}}
./configure --with-system-malloc --allow-fetch
make
sudo make install
```

{% include install-next-step.md %}
