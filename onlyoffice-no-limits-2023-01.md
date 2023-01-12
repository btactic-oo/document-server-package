# OnlyOffice 2023 No limits - Complete log

## Introduction
This is a complete log of how we managed to bring out the 'No limits' version of OnlyOffice for those of you that like technical articles.

It's 2023 January and we need to build OnlyOffice 7.2.x.
The main build_tools is still working on 7.2.x so if we assume the same thing happens with the other repos (master is kept to 7.2.x and not 7.3.x) I might be able to skip having to create a single github organisation to clone all of the onlyoffice repos.

You are also advised to check: `[README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md](README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md)` which are more straight-forward build and **use** instructions.

## Initial tasks TODO

- Internal build system is already there. No need to build it again.
  - Debian 11 Netinst was choosen (Any other Debian based distro which supports docker should also be fine).
  - Required RAM: 16 GB RAM (Minimum) or 8 GB RAM with 8 GB SWAP.
  - Recommended: 50 GB Hard disk space

- As we did before we should be able to build stuff thanks to `document-server-package` repo without almost no changes.

- Consider editing Mobile Edit to be turned on if we have some left time and we don't mind adding yet another repo.

- Create custom btactic `build_tools` branch so that it uses our `server` repo with our patch.

- Create custom `server` branch with our patches (connection limits increase and perhaps mobile edit).

- Check what packages is available in the official Debian/Ubuntu repository so that we use that exact version.

- Update github documentation.

- Update Nextcloud forum post.

- Update www.btactic.com post or create a new one.

- Update Bibliography if needed.

## Additional work after the build

- Fill a bug on `document-server-package` repo so that it's easier to package for Debian.

- Update new deb binary package.

## Development log

### VM Snapshot

Snapshot our build virtual machine so that we can revert to initial state when needed.

### Official OnlyOffice package version

Our build virtual machine already has a repo.
OnlyOffice guys never update the default one so no bother to update it or any of the directories it's using.

```
sudo apt update
sudo apt-cache show onlyoffice-documentserver | less
```
The most recent 7.2.x version is:

7.2.2-56

so that's the version that we will be using.

### server repo update

Old stuff that we already had:
- branch: 7.1.0.215-btactic
- tag: v7.1.0.215-btactic
- commit (connection limit): d54efff2c09f45d3f8ba2706777b2d1333939370

Let's fetch recent information from official OnlyOffice repo.

```
cd onlyoffice_repos/server
git checkout master
git pull upstream-origin master
git fetch --all --tags
```
.

We create a new branch based on the recently fetched tag.

```
git checkout tags/v7.2.2.56 -b 7.2.2.56-btactic
```
.

Cherry-pick what we already had:

```
git cherry-pick d54efff2c09f45d3f8ba2706777b2d1333939370
```

Let's push and create appropiate tags:

```
git push origin 7.2.2.56-btactic
git tag -a 'v7.2.2.56-btactic' -m '7.2.2.56-btactic'
git push origin v7.2.2.56-btactic
```

### build_tools repo update

Old stuff that we already had:
- branch: 7.1.0.215-btactic
- tag: v7.1.0.215-btactic
- commit (use custom repo): 9199bfccdf2f6151e348a24630800d75f331650f

Let's fetch recent information from official OnlyOffice repo.

```
cd onlyoffice_repos/build_tools
git checkout master
git pull upstream-origin master
git fetch --all --tags
```

We create a new branch based on the recently fetched tag.

```
git checkout tags/v7.2.2.56 -b 7.2.2.56-btactic
```

.

Cherry-pick what we already had:

```
git cherry-pick 9199bfccdf2f6151e348a24630800d75f331650f
```

Let's push and create appropiate tags:

```
git push origin 7.2.2.56-btactic
git tag -a 'v7.2.2.56-btactic' -m '7.2.2.56-btactic'
git push origin v7.2.2.56-btactic
```

### Build from build virtual machine

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.2.2.56-btactic \
    https://github.com/btactic/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.2.2' -e BUILD_NUMBER='56' -e NODE_ENV='production' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.2.2.56-btactic'
```

Notice how `PRODUCT_VERSION` and `BUILD_NUMBER` come from our tag version.

After many hours ( about 10 hours in 2023) the build went well:

```
switching to master...
[git] update: desktop-apps
branch does not exist...
switching to master...
[fetch & build]: boost
delete warning [file not exist]: ./boost.data
[fetch & build]: cef
delete warning [file not exist]: ./cef_binary.7z.data
[fetch & build]: icu
[fetch & build]: openssl
delete warning [file not exist]: ./openssl.data
gn gen out.gn/linux_64 --args="v8_static_library=true is_component_build=false v8_monolithic=true v8_use_external_startup_data=false use_custom_libcxx=false treat_warnings_as_errors=false target_cpu=\"x64\" v8_target_cpu=\"x64\" is_debug=false is_clang=true use_sysroot=false"
[fetch & build]: hunspell
[fetch & build]: harfbuzz
------------------------------------------
BUILD_PLATFORM: linux_64
------------------------------------------
make file: makefiles/build.makefile_linux_64
copy warning [file not exist]: /build_tools/scripts/../../sdkjs/common/HtmlFileInternal/AllFonts.js
delete warning [folder not exist]: /build_tools/scripts/../../document-server-integration/web/documentserver-example/nodejs/node_modules
[git] update: onlyoffice.github.io
branch does not exist...
switching to master...
delete warning [file not exist]: /build_tools/scripts/../out/linux_64/onlyoffice/documentserver-snap/var/www/onlyoffice/documentserver/example/nodejs/example
install dependencies...
Node.js version cannot be less 10.20
Reinstall
install qt...
---------------------------------------------
build branch: tags/v7.2.2.56-btactic
---------------------------------------------
---------------------------------------------
build modules: desktop builder server
---------------------------------------------
```

### Package from build virtual machine

- **TODO for the next time I update this howto** : Make sure to create and upload a custom tag for this release for this repo way before than here and then use that specific tag for packaging. It should be quite similar to the `server` and `build_tools` repo update sections.

```
apt install build-essential m4 npm
npm install -g pkg

cd /root
git clone https://github.com/ONLYOFFICE/document-server-package.git
# Workaround for installing dependencies - BEGIN
cd /root/document-server-package

cat << EOF >> Makefile

deb_dependencies: \$(DEB_DEPS)

EOF

PRODUCT_VERSION='7.2.2' BUILD_NUMBER='56~btactic1' make deb_dependencies
cd /root/document-server-package/deb/build
apt build-dep ./
# Workaround for installing dependencies - END

cd /root/document-server-package
PRODUCT_VERSION='7.2.2' BUILD_NUMBER='56~btactic1' make deb
```

Packaging everything takes quite a long ( 1 hour in 2023), for what it's usual in a Debian package. I think it's related to how the packaging uses strip to remove debug data so that the final package is not so big.

### Final package

The final `onlyoffice-documentserver_7.2.2-56~btactic1_amd64.deb` deb package can be found at: `/root/document-server-package/deb/` directory.

### New release tag on document-server-package repo

- **TODO for the next time I update this howto**: Same TODO as described above in: `Package from build virtual machine` section .

Let's fetch recent information from official OnlyOffice repo.

```
cd onlyoffice_repos/document-server-package
git checkout master
git pull upstream-origin master
git fetch --all --tags
```

We create a new branch based on the recently fetched tag.

```
git checkout tags/v7.2.2.56 -b 7.2.2.56-btactic
```

.

Let's push and create appropiate tags.
Yes, having a branch here for everyone of the releases is not needed at all.
We could just upload the tag.

```
git push origin 7.2.2.56-btactic
git tag -a 'v7.2.2.56-btactic' -m '7.2.2.56-btactic'
git push origin v7.2.2.56-btactic
```

### Release

Get your `onlyoffice-documentserver_7.2.2-56~btactic1_amd64.deb` to your local machine, rename it to: `onlyoffice-documentserver_7.2.2-56-btactic1_amd64.deb`.

Visit: [https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic](https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic) and click **Create release from tag** button.

Fill in appropiate release title, description, and finally attach your renamed deb as a binary.

Publish release.

## Useful links

- [https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en](https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en)
- [https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic](https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic)
- [https://github.com/btactic/document-server-package/blob/btactic-documentation/README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md](https://github.com/btactic/document-server-package/blob/btactic-documentation/README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md)
- [https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-01.md](https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-01.md)

## Additional developer notes

- Fork OnlyOffice repos from Github UI

- Then clone the repos from your own username/organisation

- Onlyoffice remote for server repo:
```
cd onlyoffice_repos/server
git remote add upstream-origin git@github.com:ONLYOFFICE/server.git
```

- Onlyoffice remote for build_tools repo:
```
cd onlyoffice_repos/build_tools
git remote add upstream-origin git@github.com:ONLYOFFICE/build_tools.git
```

- Onlyoffice remote for document-server-package repo:
```
cd onlyoffice_repos/document-server-package
git remote add upstream-origin git@github.com:ONLYOFFICE/document-server-package.git
```

- Use screen, byobu or tmux when building in your virtual machine so that build is not lost because of a network disconnection

## Warning

This is not an official onlyoffice build. Do not seek for help on OnlyOffice issues/forums unless you replicate it on original source code or original binaries from them.
