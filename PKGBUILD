# Maintainer: Felix Yan <felixonmars@archlinux.org>
# Contributor: morwel
# Contributor: Angel Velasquez <angvp@archlinux.org>
# Contributor: Stéphane Gaudreault <stephane@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>
# Contributor: Jason Chu <jason@archlinux.org>

shopt -s extglob

pkgbase=python
pkgname=(python python-tests)
pkgver=3.13.6
pkgrel=1
_pybasever=${pkgver%.*}
pkgdesc="The Python programming language (3.13)"
arch=('x86_64')
license=('PSF-2.0')
url="https://www.python.org/"
depends=(
  'bzip2'
  'expat'
  'gdbm'
  'libffi'
  'libnsl'
  'libxcrypt'
  'openssl'
  'zlib'
  'tzdata'
  'mpdecimal'
)
makedepends=(
  'bluez-libs'
  'cosign'
  'gdb'
  'llvm'
  'llvm-bolt'
  'mpdecimal'
  'sqlite'
  'tk'
)
source=(
  "https://www.python.org/ftp/python/${pkgver%rc*}/Python-${pkgver}.tar.xz"{,.sigstore}
  EXTERNALLY-MANAGED)
md5sums=('4170b57e642c15a1dfed17313ec57cc2'
         'a792650ead0be002cc633dc3cc4c774b'
         '7d2680a8ab9c9fa233deb71378d5a654')
provides=('python' 'python3' 'python-externally-managed')

verify() {
  cosign verify-blob \
    --new-bundle-format \
    --certificate-oidc-issuer 'https://accounts.google.com' \
    --certificate-identity 'thomas@python.org' \
    --bundle ./Python-${pkgver}.tar.xz.sigstore \
    ./Python-${pkgver}.tar.xz
}

prepare() {
  cd "${srcdir}/Python-${pkgver}" || exit 1

  # Ensure that we are using the system copy of various libraries (expat, libffi, and libmpdec),
  # rather than copies shipped in the tarball
  rm -rf Modules/expat
  rm -rf Modules/_decimal/libmpdec
}

build() {
  cd "${srcdir}/Python-${pkgver}" || exit 1

  # PGO should be done with -O3
  CFLAGS="${CFLAGS/-O2/-O3} -ffat-lto-objects"

  export CFLAGS+=" -fno-semantic-interposition"
  export CXXLAGS+=" -fno-semantic-interposition"

  # Disable bundled pip & setuptools
  # BOLT is disabled due LLVM or upstream issue
  # https://github.com/python/cpython/issues/124948
  ./configure \
    --prefix=/usr \
    --enable-ipv6 \
    --enable-loadable-sqlite-extensions \
    --enable-optimizations \
    --enable-shared \
    --with-computed-gotos \
    --with-dbmliborder=gdbm:ndbm \
    --with-lto \
    --with-system-expat \
    --with-system-libmpdec \
    --with-tzpath=/usr/share/zoneinfo \
    --without-ensurepip

  make EXTRA_CFLAGS="$CFLAGS"
}

check() {
  # test_tk: test_askcolor tkinter.test.test_tkinter.test_colorchooser.DefaultRootTest hangs
  # test_pyexpat: our `debug` implementation rewrites source location, which breaks the build-time
  #               only test test.test_pyexpat.HandlerExceptionTest as it cannot find source file in
  #               the to-be-installed debug package
  # test_socket: https://github.com/python/cpython/issues/79428
  # test_unittest: https://github.com/python/cpython/issues/108927
  # test_tkk: AssertionError: Tuples differ: (0,) != ('0',)
  # test_ssl: flaky tests issues

  cd "Python-${pkgver}" || exit 1

  LD_LIBRARY_PATH="${srcdir}/Python-${pkgver}":${LD_LIBRARY_PATH} \
    "${srcdir}/Python-${pkgver}/python" \
    -m test.regrtest \
    -v -uall \
    -x test_tk \
    -x test_ttk \
    -x test_ttk.test_widgets \
    -x test_tkinter \
    -x test_pyexpat \
    -x test_socket \
    -x test_unittest \
    -x test_ssl
}

package_python() {
  optdepends=(
    'python-setuptools: for building Python packages using tooling that is usually bundled with Python'
    'python-pip: for installing Python packages using tooling that is usually bundled with Python'
    'python-pipx: for installing Python software not packaged on Arch Linux'
    'sqlite: for a default database integration'
    'xz: for lzma'
    'tk: for tkinter'
  )
  provides=('python3' 'python-externally-managed')
  replaces=('python3' 'python-externally-managed')

  cd "${srcdir}/Python-${pkgver}" || exit 1

  # Hack to avoid building again
  sed -i 's/^all:.*$/all: build_all/' Makefile

  # PGO should be done with -O3
  CFLAGS="${CFLAGS/-O2/-O3}"

  make DESTDIR="${pkgdir}" EXTRA_CFLAGS="$CFLAGS" install

  # Why are these not done by default...
  ln -s "python3" "${pkgdir}/usr/bin/python"
  ln -s "python3-config" "${pkgdir}/usr/bin/python-config"
  ln -s "idle3" "${pkgdir}/usr/bin/idle"
  ln -s "pydoc3" "${pkgdir}/usr/bin/pydoc"
  ln -s "python${_pybasever}.1" "${pkgdir}/usr/share/man/man1/python.1"

  # some useful "stuff" FS#46146
  install -dm755 "${pkgdir}/usr/lib/python${_pybasever}/Tools/"{i18n,scripts}
  install -m755 "Tools/i18n/"{msgfmt,pygettext}".py" "${pkgdir}/usr/lib/python${_pybasever}/Tools/i18n/"
  install -m755 "Tools/scripts/"{README,*py} "${pkgdir}/usr/lib/python${_pybasever}/Tools/scripts/"

  # PEP668
  install -Dm644 "${srcdir}/EXTERNALLY-MANAGED" -t "${pkgdir}/usr/lib/python${_pybasever}/"

  # Split tests
  cd "${pkgdir}/usr/lib/python${_pybasever}/" || exit 1
  rm -r {test,idlelib/idle_test}
}

package_python-tests() {
  pkgdesc="Regression tests packages for Python"
  depends=('python')

  cd "${srcdir}/Python-${pkgver}" || exit 1

  make DESTDIR="${pkgdir}" EXTRA_CFLAGS="$CFLAGS" libinstall
  cd "${pkgdir}/usr/lib/python${_pybasever}/" || exit 1
  rm -r !(test)
}
