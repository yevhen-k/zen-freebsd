# Zen Browser 1.24b on FreeBSD 15.1-RELEASE-p6 (FireFox 159.0) 

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
git clone https://github.com/freebsd/freebsd-ports.git ports
export PORTSDIR=$HOME/dev/ports
cd ports
git checkout 0e2b6a5 # for firefox 159.0

cd www/firefox/
make -V DISTVERSION
make patch
```

### Zen

Release [1.24b](https://github.com/zen-browser/desktop/releases/tag/1.24b)

```sh
git clone https://github.com/zen-browser/desktop.git zen-freebsd
cd zen-freebsd
git checkout f75390a # for zen 1.24b

python3 -m venv venv
source venv/bin/activate
pip install zstandard
npm ci --allow-git=all
npm approve-scripts sharp core-js-pure
npm install sharp

```

Copy patched FireFox to Zen:
```sh
cp -r ~/dev/ports/www/firefox/work/firefox-159.0 engine
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
# ^ this command might fail with:
# ModuleNotFoundError: No module named 'zstandard'
# in this case we need to install zstandard with this commands:
# 1. get current python interpreter path
cd engine
./mach python -c "import sys; print(sys.executable)"
# ~/.mozbuild/srcdirs/engine-7cf57152d407/_virtualenvs/common/bin/python
# 2. install zstandard with this interpreter
~/.mozbuild/srcdirs/engine-7cf57152d407/_virtualenvs/common/bin/python -m pip install zstandard==0.24.0
# 3. run `npm run init` again
cd ..
npm run init
```

This command will requrie root password to install some system-wide rependencies if necessary.

### Adjust the Build

For some reason on my machine I have to remove `--with-system-libvpx` from the `engine/.mozconfig`
```sh
sed -i '' '/--with-system-libvpx/d' engine/.mozconfig
```

Also `alsa`, `jack`, and `sndio` should be disabled in the `engine/.mozconfig` on my machine if any.

We should configure local mozconfig that will hold our local setting. On my machine I disable wasm sandbox because with enabled wasm sandbox zen won't compile. This config is convenient to store `MOZ_GOOGLE_LOCATION_SERVICE_API_KEY` and `MOZ_GOOGLE_SAFEBROWSING_API_KEY`. Also let's add support for pulseaudio.
```sh
mkdir -p configs/freebsd
echo -e "# FreeBSD development configuration.\n# disable wams temporary\nac_add_options --without-wasm-sandboxed-libraries" >> configs/freebsd/mozconfig
echo -e "\nac_add_options --enable-pulseaudio" >> configs/freebsd/mozconfig

cat engine/.mozconfig | grep MOZ_GOOGLE_LOCATION_SERVICE_API_KEY >> configs/freebsd/mozconfig
cat engine/.mozconfig | grep MOZ_GOOGLE_SAFEBROWSING_API_KEY >> configs/freebsd/mozconfig
```

## Branding, Locales and Patches

`npm run import` applies patches. Console output reports no error but in reality patches are not applyed.

```sh
npm run surfer -- ci --brand release --display-version 1.24b
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
# export ZEN_RELEASE=1
npm run build
# ^ this command will hang. to check what is happened use:
# cd engine &&./mach build
# Multiple default mozconfig files present. Remove all but one. desktop-1.24b/engine/.mozconfig, desktop-1.24b/engine/mozconfig
# `mozconfig` contains all our custom configs from `configs/freebsd/mozconfig`, so we delete `engine/.mozconfig`
rm engine/.mozconfig
# and restart the build
npm run build
```

## Packaging

Packaging has some issues. For instance, `#!/bin/bash` shebang is hardcoded and doesn't work on FreeBSD.

```sh
mkdir -p tools/update-packaging
ln -s ../../engine/tools/update-packaging/make_full_update.sh tools/update-packaging/make_full_update.sh
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
# bin.l10n-stage.lock
# output.mar
# package_name.txt
# zen-1.24b.en-US.freebsd15.1-x86_64.tar.xz
# zen-1.24b.en-US.freebsd15.1-x86_64.txt
# zen-1.24b.en-US.freebsd15.1-x86_64.xpt_artifacts.zip
```

## "Installing"

I'm extracting tar archive to my locad dir that is in my PATH:

```sh
mv ~/opt/zen ~/opt/zen-prev
rm -f ~/opt/bin/zen
tar -xf dist/zen-1.24b.en-US.freebsd15.1-x86_64.tar.xz -C ~/opt/
ln -s ~/opt/zen/zen ~/opt/bin/zen

which zen
# ~/opt/bin/zen

ldd $(which zen)
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
- https://github.com/zen-browser/desktop/blob/dev/.github/workflows/linux-release-build.yml
- https://github.com/zen-browser/desktop/blob/dev/.github/workflows/src/release-build.sh

## Limitations
1. Building with `export ZEN_RELEASE=1` is not implemented yet.
2. No AUS (automatic update system)
