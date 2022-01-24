# System Base

- **Debian 11** Netinst was choosen (Any other Debian based distro which supports docker should also be fine).
- **Required RAM**: 16 GB RAM (Recommended) or 8 GB RAM with 8 GB SWAP.
- **Hard disk** (Recommended): 50 GB

# Phase 1 - Build DocumentServer Binaries

## System preparation

```
sudo -i
# Enter user password

sudo apt-get update
 sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io

systemctl status docker
# docker.io start/running, process 14394
```


## Fetch special branch

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.0.0.132-btactic \
    https://github.com/btactic/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.0.0' -e BUILD_NUMBER='132' -e NODE_ENV='production' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.0.0.132-btactic'
```

# Phase 2 - Package DocumentServer Binaries


```
apt install build-essential m4 npm
npm install -g pkg

cd /root
git clone https://github.com/ONLYOFFICE/document-server-package.git
cd document-server-package/deb
apt build-dep ./
cd /root/document-server-package
# Workaround for my system where uname -p says: unknown
sed -i 's/TARGET := $(PLATFORM)_$(ARCHITECTURE)/TARGET := $(PLATFORM)_64/g' Makefile
# Workaround to avoid ending with an error
sed -i 's/--changes-option=-u./--changes-option=-u. --buildinfo-option=-u./g' Makefile

PRODUCT_VERSION='7.0.0' BUILD_NUMBER='132~btactic1' make deb
```

## Package is built

Package `onlyoffice-documentserver_7.0.0-132~btactic1_amd64.deb` should be found at `/root/document-server-package/deb/` directory.

# Usage

Use documentation on how to install OnlyOffice package on Ubuntu such as [https://helpcenter.onlyoffice.com/installation/docs-community-install-ubuntu.aspx](https://helpcenter.onlyoffice.com/installation/docs-community-install-ubuntu.aspx). Do not add OnlyOffice repo.

When asked to install onlyoffice-documentserver package do instead:

```
sudo apt-get install /path/to/onlyoffice-documentserver_7.0.0-132~btactic1_amd64.deb
```

# Developer notes

- Use tags to make sure to use your exact version
- This build not only depends on btactic organisation repos but on ONLYOFFICE ones. TODO: Create btactic-onlyoffice org and clone all the needed repos there.
- Even then the build depends on other software downloads so you could not perform it offline (accessing offline local repos).
- Hopefully they manage to remove the need of building everything instead of only server because building everything makes us to build qt which we don't need and it takes too long.
- It would be nice for them to also open their Debian package system as they say in https://github.com/ONLYOFFICE/build_tools/pull/338#pullrequestreview-691809988 .
- Find new upstream packages at [https://download.onlyoffice.com/](https://download.onlyoffice.com/)
- Compare files found at md5sums file (sorted) from control.tar.gz from new upstream packages to learn if changes are needed to our alternate package.

## Work on built container

If you have just built an container and you want to do Debian package manually (in order to avoid waiting for 9 hours for it to build again) you can do like this:

Once:
```
docker run -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder
```
has finished we should have:

```
CONTAINER ID        IMAGE                                        COMMAND                CREATED             STATUS              PORTS               NAMES
035367486ecd        onlyoffice-document-editors-builder:latest   "/bin/sh -c 'cd tool   39 minutes ago      Exited (0) 3 minutes ago                           loving_pare
```

Now we can do a commit to freeze that status with:
```
docker commit 035367486ecd onlyoffice-just-built
```

and then we can enter that image without triggering the default build process:


```
cd /build_tools
docker run -it --entrypoint /bin/bash -v $(pwd)/out:/build_tools/out onlyoffice-just-built
```
.

Now we can visit: `/build_tools/out/package/onlyoffice` inside the docker image to build manually our package thanks to the `dpkg-buildpackage -b -us -uc` command.


## Branches

- 7.0.0.132-btactic : Use special btactic repos.
- v7.0.0.132-btactic : Tag for the first release
- https://github.com/btactic/build_tools/commits/7.0.0.132-btactic
- https://github.com/btactic/server/commits/7.0.0.132-btactic

## Check also:

TODO: btactic server repos with branch and tags


# Warning

This is not an official onlyoffice build. Do not seek for help on OnlyOffice issues/forums unless you replicate it on original source code or original binaries from them.