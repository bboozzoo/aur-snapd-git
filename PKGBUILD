# Maintainer: Maciej Borzecki <maciek.borzecki@gmail.com>
# Contributor: Zygmunt Krynicki <me at zygoon dot pl>
# Contributor: Timothy Redaelli <timothy.redaelli@gmail.com>

_pkgbase=snapd
pkgname=snapd-git
pkgdesc="Service and tools for management of snap packages."
depends=('squashfs-tools' 'libseccomp' 'libsystemd')
optdepends=('bash-completion: bash completion support')
pkgver=2.34.2.r331.gc6ce33127
pkgrel=1
arch=('x86_64')
url="https://github.com/snapcore/snapd"
license=('GPL3')
makedepends=('git' 'go' 'go-tools' 'libseccomp' 'libcap' 'systemd' 'xfsprogs' 'python-docutils')
# the following checkdepends are only required for static checks and unit tests,
# both are currently disabled
# checkdepends=('python' 'squashfs-tools' 'indent' 'shellcheck')
# Community package is split into snapd and snap-confine, make sure we replace
# both bits
conflicts=($_pkgbase 'snap-confine')
options=('!strip' 'emptydirs')
install=snapd.install
source=("git+https://github.com/snapcore/$_pkgbase.git")
sha256sums=('SKIP')

provides=($_pkgbase)

_gourl=github.com/snapcore/snapd

pkgver() {
    cd "$srcdir/snapd"
    git describe --tag | sed -r 's/([^-]*-g)/r\1/; s/-/./g'
}

prepare() {
  cd "$_pkgbase"

  export GOPATH="$srcdir/go"
  mkdir -p "$GOPATH"

  # Have snapd checkout appear in a place suitable for subsequent GOPATH. This
  # way we don't have to go get it again and it is exactly what the tag/hash
  # above describes.
  mkdir -p "$(dirname "$GOPATH/src/${_gourl}")"
  ln --no-target-directory -fs "$srcdir/$_pkgbase" "$GOPATH/src/${_gourl}"
}

build() {
  cd "$_pkgbase"
  export GOPATH="$srcdir/go"

  export CGO_ENABLED="1"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export CGO_LDFLAGS="${LDFLAGS}"

  ./mkversion.sh $pkgver-$pkgrel

  # Use get-deps.sh provided by upstream to fetch go dependencies using the
  # godeps tool and dependencies.tsv (maintained upstream).
  cd "$GOPATH/src/${_gourl}"
  XDG_CONFIG_HOME="$srcdir" ./get-deps.sh

  gobuild="go build -x -v -buildmode=pie"
  gobuild_static="go build -x -v -buildmode=pie -ldflags=-extldflags=-static"
  # Build/install snap and snapd
  $gobuild -o $GOPATH/bin/snap "${_gourl}/cmd/snap"
  $gobuild -o $GOPATH/bin/snapctl "${_gourl}/cmd/snapctl"
  $gobuild -o $GOPATH/bin/snapd "${_gourl}/cmd/snapd"
  $gobuild -o $GOPATH/bin/snap-seccomp "${_gourl}/cmd/snap-seccomp"
  # build snap-exec and snap-update-ns completely static for base snaps
  $gobuild_static -o $GOPATH/bin/snap-update-ns "${_gourl}/cmd/snap-update-ns"
  $gobuild_static -o $GOPATH/bin/snap-exec "${_gourl}/cmd/snap-exec"

  # Generate data files such as real systemd units, dbus service, environment
  # setup helpers out of the available templates
  make -C data \
       BINDIR=/bin \
       LIBEXECDIR=/usr/lib \
       SYSTEMDSYSTEMUNITDIR=/usr/lib/systemd/system \
       SNAP_MOUNT_DIR=/var/lib/snapd/snap \
       SNAPD_ENVIRONMENT_FILE=/etc/default/snapd

  cd cmd
  autoreconf -i -f
  ./configure \
    --prefix=/usr \
    --libexecdir=/usr/lib/snapd \
    --with-snap-mount-dir=/var/lib/snapd/snap \
    --disable-apparmor \
    --enable-nvidia-biarch \
    --enable-merged-usr
  make $MAKEFLAGS
}

check() {
  export GOPATH="$srcdir/go"
  cd "$GOPATH/src/${_gourl}"

  # XXX: Those files are unknown to gitignore but are checked by run-checks
  # --static. Before gitignore is updated we just remove the junk one and move
  # the valuable one aside.
  rm -f cmd/snap-confine/snap-confine-debug
  mv data/info $srcdir/xxx-info

  # Don't run unit tests
  # ./run-checks --unit || /bin/true

  # XXX: Static checks choke on autotools generated cruft. Let's not run them
  # here as they are designed to pass on a clean tree, before anything else is
  # done, not after building the tree.
  # ./run-checks --static
  TMPDIR=/tmp make -C cmd -k check

  mv $srcdir/xxx-info data/info
}

package_snapd-git() {
  cd "$_pkgbase"
  export GOPATH="$srcdir/go"

  # Install bash completion
  install -Dm644 data/completion/snap \
    "$pkgdir/usr/share/bash-completion/completion/snap"
  install -Dm644 data/completion/complete.sh \
    "$pkgdir/usr/lib/snapd/complete.sh"
  install -Dm644 data/completion/etelpmoc.sh \
    "$pkgdir/usr/lib/snapd/etelpmoc.sh"

  # Install systemd units, dbus services and a script for environment variables
  make -C data/ install \
     DBUSSERVICESDIR=/usr/share/dbus-1/services \
     BINDIR=/usr/bin \
     SYSTEMDSYSTEMUNITDIR=/usr/lib/systemd/system \
     SNAP_MOUNT_DIR=/var/lib/snapd/snap \
     DESTDIR="$pkgdir"

  # Install polkit policy
  install -Dm644 data/polkit/io.snapcraft.snapd.policy \
    "$pkgdir/usr/share/polkit-1/actions/io.snapcraft.snapd.policy"

  # Install executables
  install -Dm755 "$GOPATH/bin/snap" "$pkgdir/usr/bin/snap"
  install -Dm755 "$GOPATH/bin/snapctl" "$pkgdir/usr/bin/snapctl"
  install -Dm755 "$GOPATH/bin/snapd" "$pkgdir/usr/lib/snapd/snapd"
  install -Dm755 "$GOPATH/bin/snap-seccomp" "$pkgdir/usr/lib/snapd/snap-seccomp"
  install -Dm755 "$GOPATH/bin/snap-update-ns" "$pkgdir/usr/lib/snapd/snap-update-ns"
  install -Dm755 "$GOPATH/bin/snap-exec" "$pkgdir/usr/lib/snapd/snap-exec"

  # pre-create directories
  install -dm755 "$pkgdir/var/lib/snapd/snap"
  install -dm755 "$pkgdir/var/cache/snapd"
  install -dm755 "$pkgdir/var/lib/snapd/assertions"
  install -dm755 "$pkgdir/var/lib/snapd/desktop/applications"
  install -dm755 "$pkgdir/var/lib/snapd/device"
  install -dm755 "$pkgdir/var/lib/snapd/hostfs"
  install -dm755 "$pkgdir/var/lib/snapd/mount"
  install -dm755 "$pkgdir/var/lib/snapd/seccomp/bpf"
  install -dm755 "$pkgdir/var/lib/snapd/snap/bin"
  install -dm755 "$pkgdir/var/lib/snapd/snaps"
  install -dm755 "$pkgdir/var/lib/snapd/lib/gl"
  install -dm755 "$pkgdir/var/lib/snapd/lib/gl32"
  install -dm755 "$pkgdir/var/lib/snapd/lib/vulkan"
  install -dm755 "$pkgdir/var/lib/snapd/lib/glvnd"
  # these dirs have special permissions
  install -dm000 "$pkgdir/var/lib/snapd/void"
  install -dm700 "$pkgdir/var/lib/snapd/cookie"
  install -dm700 "$pkgdir/var/lib/snapd/cache"

  make -C cmd install DESTDIR="$pkgdir/"
  # move snapd-generator to systemd generators
  install -dm755 "$pkgdir/usr/lib/systemd/system-generators"
  mv "$pkgdir/usr/lib/snapd/snapd-generator" "$pkgdir/usr/lib/systemd/system-generators/"

  # Install man file
  mkdir -p "$pkgdir/usr/share/man/man1"
  "$GOPATH/bin/snap" help --man > "$pkgdir/usr/share/man/man1/snap.1"

  # Install the "info" data file with snapd version
  install -m 644 -D "$GOPATH/src/${_gourl}/data/info" \
          "$pkgdir/usr/lib/snapd/info"

  # Remove snappy core specific units
  rm -fv "$pkgdir/usr/lib/systemd/system/snapd.system-shutdown.service"
  rm -fv "$pkgdir/usr/lib/systemd/system/snapd.autoimport.service"
  rm -fv "$pkgdir"/usr/lib/systemd/system/snapd.snap-repair.*
  rm -fv "$pkgdir"/usr/lib/systemd/system/snapd.core-fixup.*
  # and scripts
  rm -fv "$pkgdir/usr/lib/snapd/snapd.core-fixup.sh"
  rm -fv "$pkgdir/usr/bin/ubuntu-core-launcher"
  rm -fv "$pkgdir/usr/lib/snapd/system-shutdown"
}
