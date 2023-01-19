# OnlyOffice 2023 No limits - Complete log ( 2023 02 )

## Problem with app edit

Spreadsheets cannot be previewed from Nextcloud app.

The problem relies on:
https://onlyoffice.example.net/web-apps/apps/spreadsheeteditor/mobile/index.html
redirecting to a 404.

This is because: `/var/www/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html` is not in the final package.

We are going to attempt a vanilla build so that we can submit a bug to upstream if that's the case.

# Attempt vanilla build

### Build from build virtual machine

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.2.2.56 \
    https://github.com/ONLYOFFICE/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.2.2' -e BUILD_NUMBER='56' -e NODE_ENV='production' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.2.2.56'
```

After a long time building we get:

```
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
build branch: tags/v7.2.2.56
---------------------------------------------
---------------------------------------------
build modules: desktop builder server
---------------------------------------------
```

And we can check if the file exists or not.
```
ls -l out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html
ls: no se puede acceder a 'out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html': No existe el fichero o el directorio
```

The file does not exist.

We report this as a bug here: https://github.com/ONLYOFFICE/build_tools/issues/605 .

Now let's do the same thing but without the production Node, I guess it will default to the development Node.

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.2.2.56 \
    https://github.com/ONLYOFFICE/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.2.2' -e BUILD_NUMBER='56' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.2.2.56'
```

And we can check if the file exists or not.
```
ls -l out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html
ls: no se puede acceder a 'out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html': No existe el fichero o el directorio
```

The file does not exist.

# Node.js version bump workaround

Apparently a workaround so that the build outputs these files is bumping the Node.js version.

So we will make some new commits on the `7.2.2.56-btactic2` branch and `v7.2.2.56-btactic2` tag (in all of the repos)

That way we will have a completely different tag because the old one was `7.2.2.56-btactic`.

And we will try to build this again.

## server repo

Server repo does not have any meaningful change

```
git checkout 7.2.2.56-btactic
git checkout -b 7.2.2.56-btactic2
git tag -a 'v7.2.2.56-btactic2' -m '7.2.2.56-btactic2'
git push origin 7.2.2.56-btactic2
git push origin v7.2.2.56-btactic2
```

## document-server-package repo

document-server-package repo does not have any meaningful change

```
git checkout 7.2.2.56-btactic
git checkout -b 7.2.2.56-btactic2
git tag -a 'v7.2.2.56-btactic2' -m '7.2.2.56-btactic2'
git push origin 7.2.2.56-btactic2
git push origin v7.2.2.56-btactic2
```

## build_tools repo

build_tools repo needs an additional commit.

So first of all let's create the new branch and work on it.

```
git checkout 7.2.2.56-btactic
git checkout -b 7.2.2.56-btactic2
```
.

We just replace 10.x to 12.x.

```
sed -i -e 's/_10\.x/_12.x/g' tools/linux/deps.py
git add tools/linux/deps.py
git commit -m 'Update node.js to 12.x.'
```

Finally we create a new tag and push everything.

```
git tag -a 'v7.2.2.56-btactic2' -m '7.2.2.56-btactic2'
git push origin 7.2.2.56-btactic2
git push origin v7.2.2.56-btactic2
```

## New virtual machine based build ( btactic2 tag)

```
sudo -i
cd /root
git clone \
    --depth=1 \
    --recursive \
    --branch v7.2.2.56-btactic2 \
    https://github.com/btactic/build_tools.git \
    /root/build_tools
# Ignore detached head warning
cd /root/build_tools
mkdir out
docker build --tag onlyoffice-document-editors-builder .
docker run -e PRODUCT_VERSION='7.2.2' -e BUILD_NUMBER='56' -e NODE_ENV='production' -v $(pwd)/out:/build_tools/out onlyoffice-document-editors-builder /bin/bash -c 'cd tools/linux && python3 ./automate.py --branch=tags/v7.2.2.56-btactic2'
```

Notice how `PRODUCT_VERSION` and `BUILD_NUMBER` come from our tag version.


After many hours the build ends and our expected file is there:
```
ls -l out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html
-rw-r--r-- 1 root root 6458 ene 18 22:46 out/linux_64/onlyoffice/documentserver/web-apps/apps/spreadsheeteditor/mobile/index.html
```

### Package from build virtual machine

```
apt install build-essential m4 npm
npm install -g pkg

cd /root

git clone \
    --depth=1 \
    --recursive \
    --branch v7.2.2.56-btactic2 \
    https://github.com/btactic/document-server-package.git \
    /root/document-server-package
# Ignore detached head warning
cd /root/document-server-package

cat << EOF >> Makefile

deb_dependencies: \$(DEB_DEPS)

EOF

PRODUCT_VERSION='7.2.2' BUILD_NUMBER='56+btactic2' make deb_dependencies
cd /root/document-server-package/deb/build
apt build-dep ./
# Workaround for installing dependencies - END

cd /root/document-server-package
PRODUCT_VERSION='7.2.2' BUILD_NUMBER='56+btactic2' make deb
```

Packaging everything takes quite a long ( 1 hour in 2023), for what it's usual in a Debian package. I think it's related to how the packaging uses strip to remove debug data so that the final package is not so big.

### Final package

The final `onlyoffice-documentserver_7.2.2-56+btactic2_amd64.deb` deb package can be found at: `/root/document-server-package/deb/` directory.

### Release

Get your `onlyoffice-documentserver_7.2.2-56+btactic2_amd64.deb` to your local machine.

Visit: [https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic2](https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic2) and click **Create release from tag** button.

Fill in appropiate release title, description, and finally attach your renamed deb as a binary.

Publish release.

## Useful links

- [https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en](https://www.btactic.com/build-onlyoffice-from-source-code-2023/?lang=en)
- [https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic2](https://github.com/btactic/document-server-package/releases/tag/v7.2.2.56-btactic2)
- [https://github.com/btactic/document-server-package/blob/btactic-documentation/README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md](https://github.com/btactic/document-server-package/blob/btactic-documentation/README-BUILD-DEBIAN-PACKAGE-NO-LIMITS.md)
- [https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-01.md](https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-01.md)
- [https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-02.md](https://github.com/btactic/document-server-package/blob/btactic-documentation/onlyoffice-no-limits-2023-02.md)

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


## Warning

This is not an official onlyoffice build. Do not seek for help on OnlyOffice issues/forums unless you replicate it on original source code or original binaries from them.
