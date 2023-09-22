# OnlyOffice - 2023 September - No limits - Complete log

## Introduction
This is a complete log of how we managed to bring out the 'No limits' version of OnlyOffice for those of you that like technical articles.

It's 2023 September and we need to build OnlyOffice 7.4.x.
The main build_tools is still working on 7.4.x so if we assume the same thing happens with the other repos (master is kept to 7.4.x and not 7.3.x) I might be able to skip having to create a single github organisation to clone all of the onlyoffice repos.

You are also advised to check: [README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md](README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md) which are more straight-forward build and **use** instructions.

## Initial tasks TODO

- Internal build system is already there. No need to build it again.
  - Debian 11 Netinst was choosen (Any other Debian based distro which supports docker should also be fine).
  - Required RAM: 16 GB RAM (Minimum) or 8 GB RAM with 8 GB SWAP.
  - Recommended: 50 GB Hard disk space

- As we did before we should be able to build stuff thanks to `document-server-package` repo without almost no changes.

- Consider editing Mobile Edit to be turned on if we have some left time and we don't mind adding yet another repo. **Probably not.**

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
The most recent 7.4.x version is:

7.4.1-36

so that's the version that we will be using.

### server repo update

Old stuff that we already had:
- branch: 7.2.2.56-btactic2
- tag: v7.2.2.56-btactic2
- commit (connection limit): 53e8cdea481e8b96010449b6960591f56adf498f

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
git checkout tags/v7.4.1.36 -b 7.4.1.36-btactic
```
.

Cherry-pick what we already had:

```
git cherry-pick 53e8cdea481e8b96010449b6960591f56adf498f
```

Let's push and create appropiate tags:

```
git push origin 7.4.1.36-btactic
git tag -a 'v7.4.1.36-btactic' -m '7.4.1.36-btactic'
git push origin v7.4.1.36-btactic
```

### build_tools repo update

Old stuff that we already had:
- branch: 7.2.2.56-btactic2
- tag: v7.2.2.56-btactic2
- commit (use custom repo): fb9686f70154c4e802a5bedf3fca1b31a580701b

Let's fetch recent information from official OnlyOffice repo.

```
cd onlyoffice_repos/build_tools
git checkout master
git pull upstream-origin master
git fetch --all --tags
```

We create a new branch based on the recently fetched tag.

```
git checkout tags/v7.4.1.36 -b 7.4.1.36-btactic
```

.

Cherry-pick what we already had:

```
git cherry-pick fb9686f70154c4e802a5bedf3fca1b31a580701b
```

and now we have a **conflict** in base.py. We will have to deal with this manually. It was easy to fix.

Let's push and create appropiate tags:

```
git push origin 7.4.1.36-btactic
git tag -a 'v7.4.1.36-btactic' -m '7.4.1.36-btactic'
git push origin v7.4.1.36-btactic
```

### Build from build virtual machine

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.4.1.36-btactic \
    https://github.com/btactic/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.4.1' -e BUILD_NUMBER='36' -e NODE_ENV='production' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.4.1.36-btactic'
```

Notice how `PRODUCT_VERSION` and `BUILD_NUMBER` come from our tag version.

After many hours ( about 6 hours and 15 minutes in 2023 09) the build went well:

```
[fetch & build]: hunspell
[fetch & build]: harfbuzz
[fetch]: hyphen
[fetch]: googletest
------------------------------------------
BUILD_PLATFORM: linux_64
------------------------------------------
make file: makefiles/build.makefile_linux_64
copy warning [file not exist]: /build_tools/scripts/../../sdkjs/common/HtmlFileInternal/AllFonts.js
delete warning [folder not exist]: /build_tools/scripts/../../document-server-integration/web/documentserver-example/nodejs/node_modules
[git] update: onlyoffice.github.io
branch does not exist...
switching to master...
[git] update: onlyoffice.github.io
branch does not exist...
switching to master...
delete warning [file not exist]: /build_tools/scripts/../out/linux_64/onlyoffice/documentserver-snap/var/www/onlyoffice/documentserver/example/nodejs/example
install dependencies...
Node.js version cannot be less 14
Reinstall
install qt...
---------------------------------------------
build branch: tags/v7.4.1.36-btactic
---------------------------------------------
---------------------------------------------
build modules: desktop builder server
---------------------------------------------
```

### document-server-package repo update

Although we could just work with the latest onlyoffice document-server-package master branch we are going to create our own so that we know with which commit we are packaging all of this.

Let's fetch recent information from official OnlyOffice repo.

```
cd onlyoffice_repos/document-server-package
git checkout master
git pull upstream-origin master
git fetch --all --tags
```

We create a new branch based on the recently fetched tag.

```
git checkout tags/v7.4.1.36 -b 7.4.1.36-btactic
```

.

Let's push and create appropiate tags.
Yes, having a branch here for everyone of the releases is not needed at all.
We could just upload the tag.

```
git push origin 7.4.1.36-btactic
git tag -a 'v7.4.1.36-btactic' -m '7.4.1.36-btactic'
git push origin v7.4.1.36-btactic
```

### Package from build virtual machine

```
apt install build-essential m4 npm
npm install -g pkg

cd /root
git clone https://github.com/btactic/document-server-package.git -b v7.4.1.36-btactic
# Ignore DETACHED warnings
# Workaround for installing dependencies - BEGIN
cd /root/document-server-package

cat << EOF >> Makefile

deb_dependencies: \$(DEB_DEPS)

EOF

PRODUCT_VERSION='7.4.1' BUILD_NUMBER='36~btactic1' make deb_dependencies
cd /root/document-server-package/deb/build
apt build-dep ./
# Workaround for installing dependencies - END

cd /root/document-server-package
PRODUCT_VERSION='7.4.1' BUILD_NUMBER='36~btactic1' make deb
```

Well, package failed here there is what we have:
```
   dh_fixperms
        find debian/onlyoffice-documentserver -true -print0 2>/dev/null | xargs -0r chown --no-dereference 0:0
        find debian/onlyoffice-documentserver ! -type l -a -true -a -true -print0 2>/dev/null | xargs -0r chmod go=rX,u+rw,a-s
        find debian/onlyoffice-documentserver/usr/share/doc -type f -a -true -a ! -regex 'debian/onlyoffice-documentserver/usr/share/doc/[^/]*/examples/.*' -print0 2>/dev/null | xargs -0r chmod 0644
        find debian/onlyoffice-documentserver/usr/share/doc -type d -a -true -a -true -print0 2>/dev/null | xargs -0r chmod 0755
        find debian/onlyoffice-documentserver -type f \( -name '*.so.*' -o -name '*.so' -o -name '*.la' -o -name '*.a' -o -name '*.js' -o -name '*.css' -o -name '*.scss' -o -name '*.sass' -o -name '*.jpeg' -o -name '*.jpg' -o -name '*.png' -o -name '*.gif' -o -name '*.cmxs' -o -name '*.node' \) -a -true -a -true -print0 2>/dev/null | xargs -0r chmod 0644
        find debian/onlyoffice-documentserver/usr/bin -type f -a -true -a -true -print0 2>/dev/null | xargs -0r chmod a+x
        find debian/onlyoffice-documentserver/usr/lib -type f -name '*.ali' -a -true -a -true -print0 2>/dev/null | xargs -0r chmod uga-w
   debian/rules execute_after_dh_fixperms
make[2]: se entra en el directorio '/root/document-server-package/deb/build'
chmod o-rwx debian/{{package_sysname}}-documentserver/etc/{{package_sysname}}/documentserver/*.json
chmod: no se puede acceder a 'debian/{{package_sysname}}-documentserver/etc/{{package_sysname}}/documentserver/*.json': No existe el fichero o el directorio
make[2]: *** [debian/rules:15: execute_after_dh_fixperms] Error 1
make[2]: se sale del directorio '/root/document-server-package/deb/build'
make[1]: *** [debian/rules:7: binary] Error 2
make[1]: se sale del directorio '/root/document-server-package/deb/build'
dpkg-buildpackage: fallo: debian/rules binary subprocess returned exit status 2
make: *** [Makefile:496: deb/onlyoffice-documentserver_7.4.1-36~btactic1_amd64.deb] Error 
```
.

This makes more or less sense because when building the main program some json files were missing. Let me copy and paste the output.

```
copy warning [file not exist]: /build_tools/scripts/../../sdkjs/common/HtmlFileInternal/AllFonts.js                                                                                          
delete warning [folder not exist]: /build_tools/scripts/../../document-server-integration/web/documentserver-example/nodejs/node_modules                                                     
delete warning [file not exist]: /build_tools/scripts/../out/linux_64/onlyoffice/documentserver-snap/var/www/onlyoffice/documentserver/example/nodejs/example
```

Wait a moment... those files.. exists... I think that the commit `b41c3d0e4bf52a51c0dc89ce67c678ed8fea63d5` from document-server-package has the problem.

Let me fix with a new commit and we will rebuild with btactic2.

### Package from build virtual machine ( Second try )

```
apt install build-essential m4 npm
npm install -g pkg

cd /root
git clone https://github.com/btactic/document-server-package.git -b v7.4.1.36-btactic2
# Ignore DETACHED warnings
# Workaround for installing dependencies - BEGIN
cd /root/document-server-package

cat << EOF >> Makefile

deb_dependencies: \$(DEB_DEPS)

EOF

PRODUCT_VERSION='7.4.1' BUILD_NUMBER='36~btactic2' make deb_dependencies
cd /root/document-server-package/deb/build
apt build-dep ./
# Workaround for installing dependencies - END

cd /root/document-server-package
PRODUCT_VERSION='7.4.1' BUILD_NUMBER='36~btactic2' make deb
```

### Final package

The final `onlyoffice-documentserver_7.4.1-36~btactic2_amd64.deb` deb package can be found at: `/root/document-server-package/deb/` directory.



### Release

Get your `onlyoffice-documentserver_7.4.1-36~btactic2_amd64.deb` to your local machine, rename it to: `onlyoffice-documentserver_7.4.1-36-btactic2_amd64.deb`.

Visit: [https://github.com/btactic/document-server-package/releases/tag/v7.4.1.36-btactic](https://github.com/btactic/document-server-package/releases/tag/v7.4.1.36-btactic) and click **Create release from tag** button.

Fill in appropiate release title, description, and finally attach your renamed deb as a binary.

Publish release.

## Useful links

- [https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en](https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en)
- [https://github.com/btactic/document-server-package/releases/tag/v7.4.1.36-btactic](https://github.com/btactic/document-server-package/releases/tag/v7.4.1.36-btactic)
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
