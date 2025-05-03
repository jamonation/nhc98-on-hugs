# hugs + nhc98 bootstrapping

This repo contains sources of `hugs98-plus-Sep2006.tar.gz` and `nhc98src-1.22.tar.gz`
along with patches to create a fairly reliable environment to carry on the work that
_rekado_ started in 2017 on [Bootstrapping Haskell](https://elephly.net/posts/2017-01-09-bootstrapping-haskell-part-1.html).

We'll use a Debian <strike>Potato</strike> Woody chroot, because the version of GCC it
shipped with was 2.95 and is plenty old enough to work around the segfaults that _rekado_
encountered with 4.x GCC.

Woody also has the added benefit of being signed by a known GPG key from Debian, whereas
Potato wasn't, or the public key has been removed from default keyrings.

## Prerequisites
1. `debootstrap`
2. `git` (or download a copy of this repo)
3. `chroot`
4. dogged determination

### Sources

If you want to get tarballs on your own (you probably should), these are the sources that are unpacked in this repo:
```
curl -O https://www.haskell.org/hugs/downloads/2006-09/hugs98-plus-Sep2006.tar.gz
curl -OL http://www.haskell.org/nhc98/nhc98src-1.22.tar.gz
```

## Instructions

Broadly speaking, the steps involved are to clone this repo, setup a Debian <strike>Potato</strike> 
(or maybe Woody?) chroot, and then run the compilation steps to produce some of the
binaries that make up an nhc98 toolchain.

This should probably be a bash script and `-euo pipefail` to prevent any transcription errors.
But copypasta, go step by step, and have fun!

```bash
# clone this repo
git clone https://github.com/jamonation/nhc98-on-hugs
cd nhc98-on-hugs
export CHROOT="${PWD}"
export NHCDIR=/usr/src/nhc98-1.22

# make chroot, get old gcc, profit
sudo debootstrap --arch=i386 woody .
sudo chroot "${CHROOT}" /bin/bash -s -c 'apt-get update && apt-get -y install build-essential'

# hush git messages about permissions
sudo chown -R $USER:$USER "${CHROOT}"

# make and install hugs
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
    cd /usr/src/hugs98-plus-Sep2006
    ./configure --prefix=/usr && make -j$(nproc) && make install
EOF

# quiet the annoying -m32 cc1 warnings
cd "${CHROOT}/${NHCDIR}"
sed 's#-m32##g' -i configure

# configure and make nhc98 runtime
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
    cd "${NHCDIR}"
    ./configure --prefix=usr --builddir=$PWD/build
    cd src/runtime
    make
EOF

# add hugs-nhc script
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
cat > "${NHCDIR}/hugs-nhc" <<'SCRIPT'
#!/bin/bash

# Root directory of Hugs installation
HUGSDIR="$(dirname $(readlink -f $(which runhugs)))/../"

# TODO: "libraries" alone may be sufficient
SEARCH_HUGS=$(printf "${NHCDIR}/src/%s/*:" compiler prelude libraries)

# Filter everything from "+RTS" to "-RTS" from $@ because MainNhc98.hs
# does not know what to do with these flags.
ARGS=""
SKIP="false"
for arg in "$@"; do
    if [[ $arg == "+RTS" ]]; then
        SKIP="true"
    elif [[ $arg == "-RTS" ]]; then
        SKIP="false"
    elif [[ $SKIP == "false" ]]; then
        ARGS="${ARGS} $arg"
    fi
done

runhugs -98 \
        -P${HUGSDIR}/lib/hugs/packages/*:${SEARCH_HUGS} \
        ${NHCDIR}/src/compiler98/MainNhc98.hs \
        $ARGS
SCRIPT

chmod +x "${NHCDIR}/hugs-nhc"
EOF

# add hugs-cpphs script
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
cat > "${NHCDIR}/hugs-cpphs" <<'SCRIPT'
#!/bin/bash

runhugs ${NHCDIR}/src/cpphs/cpphs.hs --noline -D__HASKELL98__ "$@"
SCRIPT

chmod +x "${NHCDIR}/hugs-cpphs"
EOF

# add hugs-greencard script
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
cat > "${NHCDIR}/hugs-greencard" <<'SCRIPT'
#!/bin/bash

HUGSDIR="$(dirname $(readlink -f $(which runhugs)))/../"
SEARCH_HUGS=$(printf "${NHCDIR}/src/%s/*:" compiler prelude libraries)

runhugs -98 \
        -P${HUGSDIR}/lib/hugs/packages/*:${NHCDIR}/include/*:${SEARCH_HUGS} \
        ${NHCDIR}/src/greencard/GreenCard.lhs \
        $@
SCRIPT

chmod +x "${NHCDIR}/hugs-greencard"
EOF

# apply patches to match rekado's setup
cd "${CHROOT}/${NHCDIR}"
patch -p1 < ../patches/*

# pre-process all the things
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
cd ${NHCDIR}/src/greencard

CPPPRE="${NHCDIR}/hugs-cpphs -D__NHC__"

FILES="DIS.lhs \
 HandLex.hs \
 ParseLib.hs \
 HandParse.hs \
 FillIn.lhs \
 Proc.lhs \
 NHCBackend.hs \
 NameSupply.lhs \
 Process.lhs"

for file in $FILES; do
    cp $file $file.original && $CPPPRE $file.original > $file && rm $file.original
done
EOF

# pre-process more of the things
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
cd ${NHCDIR}
CPPPRE="${NHCDIR}/hugs-cpphs -D__HUGS__"

FILES="src/compiler98/GcodeLowC.hs \
 src/libraries/filepath/System/FilePath.hs \
 src/libraries/filepath/System/FilePath/Posix.hs"

for file in $FILES; do
    cp $file $file.original && $CPPPRE $file.original > $file && rm $file.original
done
EOF

# sed fun outside chroot
cd "${CHROOT}/${NHCDIR}"
sed -i \
  -e '0,/^GREENCARD=.*$/s||GREENCARD="$NHC98BINDIR/../hugs-greencard"|' \
  -e '0,/^CPPHS=.*$/s||CPPHS="$NHC98BINDIR/../hugs-cpphs -D__NHC__"|' \
  -e '0,/^PRAGMA=.*$/s||PRAGMA=cat|' \
  script/nhc98

# YOLO it!
sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
    cd ${NHCDIR}/src/prelude
    time make -j1 NHC98COMP=${NHCDIR}/hugs-nhc # -j1 importantísimo!
EOF

# almost there, just hmake left
cd "${CHROOT}/${NHCDIR}"/src/hmake
mv FileName.hs{,.broken}
tr '\366' 'o' < FileName.hs.broken > FileName.hs
rm FileName.hs.broken

sudo -E chroot "${CHROOT}" /bin/bash <<'EOF'
    cd "${NHCDIR}"/src/hmake
    NHC98COMP=${NHCDIR}/hugs-nhc make HC=${NHCDIR}/script/nhc98
EOF
```

## Results
The good stuff will reside in `${CHROOT}/${NHCDIR}/lib/x86_64/`:
```bash
ls lib/x86_64-Linux
config  main.o  MkConfig  MkProg  mutator.o  mutlib.o  nhc98heap  Older  Prelude.a  runhs  Runtime.a
```

So we've arrived where _rekado_ left off. What's next? I haven't a clue. `MkConfig` wants `MACHINE`
defined, and seems to use it as a prefix of some sort. Once that's setup, `MkProg` will shout a bit
more, so define whatever variable it complains is unset. 

Then what??

### Package as an OCI container

If/when you get something working at any stage and want to make it reusable, Git is great.

You can also package your chroot as a container image and use Docker or Podman to iterate, without 
borking your build tree. Ask me how I know.

```bash
sudo tar --numeric-owner -cf /tmp/nhc98-and-hugs.tar -C $CHROOT .
docker import -m "nhc98 import" -c 'CMD ["/bin/bash"]' /tmp/nhc98-and-hugs.tar nhc98-and-hugs:1.22
```

Podman should be something like:
```bash
podman import /tmp/nhc98-and-hugs.tar nhc98-and-hugs:1.22
```

Run it:
```bash
docker run -it nhc98-and-hugs:1.22
1029bd37ce7f:/#
```

I've added a copy here, but I wouldn't recommend using it for anything but experimenting. After all, you
wouldn't run any random container image that you found in a parking lot would you?

Regardless, here it is:

```bash
docker pull ghcr.io/jamonation/nhc98-and-hugs:1.22
```
