_

Release Process
===============

* update translations, ping @bitbeaner in The Farm on:
    https://live.beancash.org (for now)

* update (commit) version in sources

::

  Beancash-qt.pro
  src/version.h
  share/setup.nsi
  doc/README*
  

* tag version in git

   git tag -a v1.1.0.0

* write release notes.  git shortlog helps a lot:

   git shortlog --no-merges v1.3.x

perform gitian builds
~~~~~~~~~~~~~~~~~~~~~

  From a directory containing the Bean Cash source, gitian-builder and gitian.sigs:
  ::
  
   export SIGNER=(your gitian key, ie bluematt, sipa, etc)
   export VERSION=1.1.0.0
   cd ./gitian-builder

  Fetch and build inputs:
  ::
  
   mkdir -p inputs; cd inputs/
   wget 'http://www.openssl.org/source/openssl-1.0.2t.tar.gz'
   wget 'http://download.oracle.com/berkeley-db/db-5.3.tar.gz'
   wget 'http://zlib.net/zlib-1.2.6.tar.gz'
   wget 'ftp://ftp.simplesystems.org/pub/libpng/png/src/libpng-1.5.9.tar.gz'
   wget 'http://fukuchi.org/works/qrencode/qrencode-4.0.2.tar.gz'
   wget 'http://downloads.sourceforge.net/project/boost/boost/1.59.0/boost_1_59_0.tar.bz2'
   wget 'http://download.qt.nokia.com/qt/source/qt-everywhere-opensource-src-4.7.4.tar.gz'
   cd ..
   ./bin/gbuild ../Beancash/contrib/gitian-descriptors/boost-win32.yml
   cp build/out/boost-win32-1.47.0-gitian.zip inputs/
   ./bin/gbuild ../Beancash/contrib/gitian-descriptors/qt-win32.yml
   cp build/out/qt-win32-4.7.4-gitian.zip inputs/
   ./bin/gbuild ../Beancash/contrib/gitian-descriptors/deps-win32.yml
   cp build/out/Beancash-deps-1.3.zip inputs/

Build Beancashd and Beancash-qt on Linux32, Linux64, and Win32:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./bin/gbuild --commit Beancash=v${VERSION} ../Beancash/contrib/gitian-descriptors/gitian.yml
   ./bin/gsign --signer $SIGNER --release ${VERSION} --destination ../gitian.sigs/ ../Beancash/contrib/gitian-descriptors/gitian.yml
   pushd build/out
   zip -r Beancash-${VERSION}-linux-gitian.zip *
   mv Beancash-${VERSION}-linux-gitian.zip ../../
   popd
   ./bin/gbuild --commit Beancash=v${VERSION} ../Beancash/contrib/gitian-descriptors/gitian-win32.yml
   ./bin/gsign --signer $SIGNER --release ${VERSION}-win32 --destination ../gitian.sigs/ ../Beancash/contrib/gitian-descriptors/gitian-win32.yml
   pushd build/out
   zip -r Beancash-${VERSION}-win32-gitian.zip *
   mv Beancash-${VERSION}-win32-gitian.zip ../../
   popd

*Build output expected:*

  1. linux 32-bit and 64-bit binaries + source (Beancash-${VERSION}-linux-gitian.zip)
  2. windows 32-bit binary, installer + source (Beancash-${VERSION}-win32-gitian.zip)
  3. Gitian signatures (in gitian.sigs/${VERSION}[-win32]/(your gitian key)/

Repackage gitian builds for release as stand-alone zip/tar/installer exe:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

  * Linux .tar.gz:
   unzip Beancash-${VERSION}-linux-gitian.zip -d Beancash-${VERSION}-linux
   tar czvf Beancash-${VERSION}-linux.tar.gz Beancash-${VERSION}-linux
   rm -rf Beancash-${VERSION}-linux

  * Windows .zip and setup.exe:
   unzip Beancash-${VERSION}-win32-gitian.zip -d Beancash-${VERSION}-win32
   mv Beancash-${VERSION}-win32/Beancash-*-setup.exe .
   zip -r Beancash-${VERSION}-win32.zip Beancash-${VERSION}-win32
   rm -rf Beancash-${VERSION}-win32

Perform Mac build:
~~~~~~~~~~~~~~~~~~

  See this blog post for how Gavin set up his build environment to build the OSX
  release; note that a patched version of macdeployqt is not needed anymore, as
  the required functionality and fixes are implemented directly in macdeployqtplus:
  
    http://gavintech.blogspot.com/2011/11/deploying-Beancash-qt-on-osx.html
    
  Gavin also had trouble with the macports py27-appscript package; he
  ended up installing a version that worked with: /usr/bin/easy_install-2.7 appscript.
  
::

  qmake RELEASE=1 USE_QRCODE=1 Beancash-qt.pro
  make
  export QTDIR=/opt/local/share/qt4  # needed to find translations/qt_*.qm files
  T=$(contrib/qt_translations.py $QTDIR/translations src/qt/locale)
  python2.7 contrib/macdeploy/macdeployqtplus Beancash-qt.app -add-qt-tr $T -dmg -fancy contrib/macdeploy/fancy.plist

Build output expected:

  *Beancash-qt.dmg*

Post Build Process:
~~~~~~~~~~~~~~~~~~~

* upload builds to SourceForge

* create SHA256SUMS for builds, and PGP-sign it

* update Beancash.org version

* update forum version

* update wiki download links

* update wiki changelog: https://wiki.beancash.org/changelog.html

* Commit your signature to gitian.sigs:
  pushd gitian.sigs
  git add ${VERSION}/${SIGNER}
  git add ${VERSION}-win32/${SIGNER}
  git commit -a
  git push  # Assuming you can push to the gitian.sigs tree
  popd

-------------------------------------------------------------------------

After 3 or more people have built using gitian, repackage gitian-signed zips and from a directory containing Bean Cash source, gitian.sigs and gitian zips:
::

   export VERSION=0.5.1
   mkdir Beancash-${VERSION}-linux-gitian
   pushd Beancash-${VERSION}-linux-gitian
   unzip ../Beancash-${VERSION}-linux-gitian.zip
   mkdir gitian
   cp ../Beancash/contrib/gitian-downloader/*.pgp ./gitian/
   for signer in $(ls ../gitian.sigs/${VERSION}/); do
     cp ../gitian.sigs/${VERSION}/${signer}/Beancash-build.assert ./gitian/${signer}-build.assert
     cp ../gitian.sigs/${VERSION}/${signer}/Beancash-build.assert.sig ./gitian/${signer}-build.assert.sig
   done
   zip -r Beancash-${VERSION}-linux-gitian.zip *
   cp Beancash-${VERSION}-linux-gitian.zip ../
   popd
   mkdir Beancash-${VERSION}-win32-gitian
   pushd Beancash-${VERSION}-win32-gitian
   unzip ../Beancash-${VERSION}-win32-gitian.zip
   mkdir gitian
   cp ../Beancash/contrib/gitian-downloader/*.pgp ./gitian/
   for signer in $(ls ../gitian.sigs/${VERSION}-win32/); do
     cp ../gitian.sigs/${VERSION}-win32/${signer}/Beancash-build.assert ./gitian/${signer}-build.assert
     cp ../gitian.sigs/${VERSION}-win32/${signer}/Beancash-build.assert.sig ./gitian/${signer}-build.assert.sig
   done
   zip -r Beancash-${VERSION}-win32-gitian.zip *
   cp Beancash-${VERSION}-win32-gitian.zip ../
   popd

Upload gitian zips to SourceForge
