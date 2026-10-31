# Zen Browser 1.23.4b on FreeBSD 15.1-RELEASE-p6 (FireFox 158.0.1) 

## Prerequisites

### Prepare Dependencies

Assume, rustup, rust, cargo, python, node, npm are installed.

```sh
sudo pkg install wasi-libc19 wasi-libcxx19 
```

## Get the source code

### FireFox

We need posts for firefox source code and freebsd patches
```sh
git clone https://git.freebsd.org/ports.git
export PORTSDIR=$HOME/dev/ports
cd posts
git checkout 4fb33f1da0537e23d7b86c7ac998cdc3a257cfe7 # for firefix 158.0.1

cd www/firefox/
make -V DISTVERSION
make patch
```

### Zen

Release [1.23.4b](https://github.com/zen-browser/desktop/releases#release-1.23.4b)

```sh

curl -sL https://github.com/zen-browser/desktop/archive/refs/tags/1.23.4b.zip -o 1.23.4b.zip
unzip 1.23.4b.zip
cd desktop-1.23.4b/

python3 -m venv venv
source venv/bin/activate
pip install zstandard
npm ci --allow-git=all
npm approve-scripts sharp core-js-pure
npm install sharp

```

Copy patched FireFox to Zen:
```sh
cp -r $HOME/dev/ports/www/firefox/work/firefox-158.0.1 engine
```

## Adjust node libs for FreeBSD

Add `freebsd` next to `linux`in the `node_modules/@zen-browser/surfer/dist/constants/index.js`
```sh
sed -i '' "s/exports.BUILD_TARGETS = \['linux', 'windows', 'macos'\]/exports.BUILD_TARGETS = ['linux', 'freebsd', 'windows', 'macos']/" node_modules/@zen-browser/surfer/dist/constants/index.js
```

Add `freebsd` next to `linux` in the `node_modules/@zen-browser/surfer/dist/commands/build.js`
```sh
sed -i '' "/^    linux: 'linux',$/a\\
    freebsd: 'freebsd',
" node_modules/@zen-browser/surfer/dist/commands/build.js
```

## Initialize the Build
```sh
npm run init
```

This command will requrie root password to install some system-wide rependencies if necessary.

### Adjust the Build

For some reason on my machine I have to remove `--with-system-libvpx` from the `engine/.mozconfig`
# remove --with-system-libvpx from engine/.mozconfig
```sh
sed -i '' '/--with-system-libvpx/d' engine/.mozconfig
```

Also `alsa`, `jack`, and `sndio` should be disabled in the `engine/.mozconfig` on my machine if any.

We should configure local mozconfig that will hold our local setting. On my machine I disable wasm sandbox because with enabled wasm sandbox zen won't compile. This config is convenient to store `MOZ_GOOGLE_LOCATION_SERVICE_API_KEY` and `MOZ_GOOGLE_SAFEBROWSING_API_KEY`.
```sh
mkdir -p configs/freebsd
echo -e "# FreeBSD development configuration.\n# disable wams temporary\nac_add_options --without-wasm-sandboxed-libraries" >> configs/freebsd/mozconfig

cat engine/.mozconfig | grep MOZ_GOOGLE_LOCATION_SERVICE_API_KEY >> configs/freebsd/mozconfig
cat engine/.mozconfig | grep MOZ_GOOGLE_SAFEBROWSING_API_KEY >> configs/freebsd/mozconfig
```

## Branding, Locales and Patches

`npm run import` applies patches. Console output reports no error but in reality patches are not applyed.

```sh
npm run surfer -- ci --brand release --display-version 1.23.4b
npm run download
source $HOME/.cargo/env
npm run import
sh scripts/download-language-packs.sh
```

So, we apply patches manually:
```sh
cd engine
for p in $(find ../src -name '*.patch')
  echo "===== $p ====="
  patch -N -p1 < $p
end
```

## Bootstrap Application

Bootstrapping will require root permission to install necessary dependencies if any.

```sh
cd engine
export PATH=/usr/local/llvm19/bin:$PATH
export CC=/usr/local/llvm19/bin/clang
export CXX=/usr/local/llvm19/bin/clang++
export LLVM_CONFIG=/usr/local/llvm19/bin/llvm-config
export SURFER_PLATFORM="freebsd"

./mach --no-interactive bootstrap --application-choice browser
# ^ it might fail complaining `ModuleNotFoundError: No module named 'zstandard'`.
# in this case we need to install zstandard with this commands:
# 1. get current python interpreter path
./mach python -c "import sys; print(sys.executable)"
# ~/.mozbuild/srcdirs/engine-7b01386296f8/_virtualenvs/common/bin/python
# 2. install zstandard with this interpreter
~/.mozbuild/srcdirs/engine-7b01386296f8/_virtualenvs/common/bin/python -m pip install zstandard==0.24.0
# and then run ./mach <...> again
./mach --no-interactive bootstrap --application-choice browser
```


## Build

Once bootstrap is completed, we have to make the final adjustments to the configs:

```sh
cd ..
cat configs/freebsd/mozconfig >> engine/.mozconfig 

sed -i -e 's/^#ifdef XP_LINUX$/#if defined(XP_LINUX) || defined(XP_FREEBSD)/' $(readlink -f engine/browser/themes/shared/zen-icons/jar.inc.mn)
```

Now we're ready to build:

```sh
git add .
git commit -m "zen of freebsd"
# ^ this is required by `npm build`

# export ZEN_RELEASE=1
npm run build
# ^ this command will hang. to check what is happened use:
# cd engine &&./mach build
# Multiple default mozconfig files present. Remove all but one. desktop-1.23.4b/engine/.mozconfig, desktop-1.23.4b/engine/mozconfig
# `mozconfig` contains all our custom configs from `configs/freebsd/mozconfig`, so we delete `engine/.mozconfig`
rm engine/.mozconfig
# and restart the build
npm run build
```

## Packaging

Packaging has some issues. For instance, `#!/bin/bash` shebang is hardcoded and doesn't work on FreeBSD.

```sh
mkdir -p tools/update-packaging
ln -s ../../engine/tools/update-packaging/make_full_update.sh \
                                                       tools/update-packaging/make_full_update.sh
sed -i '' '1s|^#!/bin/bash$|#!/usr/local/bin/bash|' engine/tools/update-packaging/make_full_update.sh

sed -i '' "/await (0, browser_1\.generateBrowserUpdateFiles)();/c\\
if (process.surferPlatform !== 'freebsd') {\\
    await (0, browser_1.generateBrowserUpdateFiles)();\\
} else {\\
    log_1.log.warning('Skipping browser AUS update files: FreeBSD is not supported by Surfer');\\
}" node_modules/@zen-browser/surfer/dist/commands/package.js

npm run package
```

Check packaged Zen browser:
```sh
ls -l dist
# total 208971
# -rw-r--r--  1 user user         0 Oct 31 16:11 bin.l10n-stage.lock
# -rw-r--r--  1 user user 106518101 Oct 31 16:14 output.mar
# -rw-r--r--  1 user user        44 Oct 31 16:12 package_name.txt
# -rw-r--r--  1 user user 107034760 Oct 31 16:12 zen-1.23.4b.en-US.freebsd15.1-x86_64.tar.xz
# -rw-r--r--  1 user user       102 Oct 31 16:12 zen-1.23.4b.en-US.freebsd15.1-x86_64.txt
# -rw-r--r--  1 user user    265405 Oct 31 16:12 zen-1.23.4b.en-US.freebsd15.1-x86_64.xpt_artifacts.zip

du -hs .
# 13G	.
du -hs ../ports
# 6.7G	../ports
du -hs ~/.mozbuild
# 70M	~/.mozbuild
```

## "Installing"

I'm extracting tar archive to my locad dir that is in my PATH:

```sh
tar -xf dist/zen-1.23.4b.en-US.freebsd15.1-x86_64.tar.xz -C ~/opt/
ln -s ~/opt/zen/zen ~/opt/bin/zen

which zen
# ~/opt/bin/zen

dd $(which zen)
# ~/opt/bin/zen:
#	libc++.so.1 => /lib/libc++.so.1 (0x1541f8c2f000)
#	libcxxrt.so.1 => /lib/libcxxrt.so.1 (0x1541f957b000)
#	libm.so.5 => /lib/libm.so.5 (0x1541f9f45000)
#	libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x1541fb3c4000)
#	libthr.so.3 => /lib/libthr.so.3 (0x1541fc379000)
#	libc.so.7 => /lib/libc.so.7 (0x1541fc7f7000)
#	libsys.so.7 => /lib/libsys.so.7 (0x1541fa7a1000)
#	[vdso] (0x1541f765f000)
```

## References:
- https://docs.zen-browser.app/contribute/desktop/building
- https://github.com/zen-browser/desktop/blob/dev/.github/workflows/linux-release-build.yml
- https://github.com/zen-browser/desktop/blob/dev/.github/workflows/src/release-build.sh

## Limitations
1. Building with `export ZEN_RELEASE=1` is not implemented yet.
2. Proper packaging
