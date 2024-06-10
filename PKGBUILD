#!/bin/bash

major=6
minor=1
subpatch=.90

pkgver=${major}.${minor}${subpatch}
PKGbase=linux
CONF=config-lite
cc=

pkgbase=kernel
pkgrel=1
arch=("$CARCH")
license=('GPL2')
url='https://www.kernel.org/'
options=('!lfs' !addep)
makedepends=('libelf' xz 'zstd' openssl 'kmod')
if [ "$cc" = 'clang' ]; then makedepends+=(clang); fi

source=(
	"${CONF}" kernel_signing_key.pub
	"https://kernel.org/pub/linux/kernel/v${major}.x/linux-${pkgver}.tar".{xz,sign})
sha256sums=(
	'e5da4c649249cd4b726f884d1ff9e8578cb40ca4ea0917de982c30698a1e6019'
	'71b5dddd64b9dbfb9abf6286adf17d137a6c3d260ee014e59aecbc0904f662e0'
	"$(curl -sL kernel.org/pub/linux/kernel/v${major}.x/sha256sums.asc|grep linux-${pkgver}.tar.xz|awk '{print $1}')"
	'SKIP')
validpgpkeys=(
	'647F28654894E3BD457199BE38DBBDC86092693E'
	'ABAF11C65A2970B130ABE3C479BE3E4300411886')

prepare(){
	[ -d "linux" ] || ln -s linux{-${pkgver},}
	cd linux
	make mrproper
	make clean

	local src
	for src in "${source[@]}"; do
		src="${src%%::*}"
		src="${src##*/}"
		src="${src%.xz}"
		[[ $src = *.patch ]] || continue
		msg2 "Applying patch '$src'..."
		patch -Np1 < "../$src"
	done

	cp ../$CONF .config
	if [ "$cc" = clang ]; then
		scripts/config --disable LTO_CLANG_FULL
		scripts/config --enable LTO_CLANG_THIN
		make CC=clang olddefconfig
	else
		make olddefconfig
	fi

	cp .config ${PKGDEST}/config
	diff -u ../$CONF .config > ${PKGDEST}/config-${pkgver}.diff || true
	scripts/diffconfig .config.old .config > ${PKGDEST}/config-change.diff

	make -s kernelrelease > version
	msg2 "Prepared linux version $(<version)"
}

build(){
	msg2 "pkgbase = $PKGbase"

	if [ "$cc" = clang ]; then
		make CC=clang V=2 -C linux
	else
		make V=2 -C linux
	fi
}

_pack(){
	pkgdesc="The Linux kernel and modules"
	depends=('kmod' 'initramfs' 'coreutils')
	optdepends=('linux-firmware: firmware images needed for some devices')
	provides=("linux=${pkgver}")
	options+=('!strip')

	cd linux
	kerver="$(<version)"
	modulesdir="$pkgdir/usr/lib/modules/${kerver}"

	msg2 "Installing boot image..."
	install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

	msg2 "Installing modules..."
	make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 DEPMOD=/doesnt/exist modules_install  # Suppress depmod

	echo "$PKGbase"  | install -Dm644 /dev/stdin "$modulesdir/pkgbase"
	rm -f $modulesdir/{source,build}

	touch "$modulesdir/no_initrd"  # 防止 mkinitfs hook 生成 initramfs
	install -d "$modulesdir/kernel/drivers/video"  # 防止 更新/卸载 时 dkms 删除 nv 驱动后，pacman 不删除旧内核中的此目录
}

_pack-headers(){
	pkgdesc="Headers and scripts for building modules for the Linux kernel"
	provides=("linux-headers=$pkgver")
	if [ "$cc" = clang ]; then depends+=(clang); fi

	is_enabled() {
		grep -q "^$1=y" include/config/auto.conf
	}
	cd linux

	local kerver="$(<version)"
	local builddir="$pkgdir/usr/src/linux-${kerver}"
	local modulesdir="$pkgdir/usr/lib/modules/${kerver}"

	msg2 "Installing build files..."
	install -Dt "$builddir" -m644 .config Makefile Module.symvers version
	install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
	local file
	while read -rd '' file; do
		install -Dt "$builddir/$(dirname $file)" $file
	done < <(find scripts -type f -print0)

	if is_enabled CONFIG_OBJTOOL; then
		msg2 "Installing objtool..."
		install -Dt "$builddir/tools/objtool" -m755 tools/objtool/objtool
	fi

	msg2 "Installing headers..."
	cp -t "$builddir" -a include
	cp -t "$builddir/arch/x86" -a arch/x86/include

	msg2 "Removing loose objects..."
	find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

	msg2 "Removing .gitignore file..."
	find "$builddir" -name '.gitignore' -delete

	msg2 "Adding symlink..."
	install -d ${modulesdir}
	ln -s /usr/src/linux-${kerver} "${modulesdir}/build"
	ln -s build "${modulesdir}/source"
}

pkgname=(${pkgbase}{,-headers})
for _p in "${pkgname[@]}"; do
	eval "package_$_p() {
		$(declare -f "_pack${_p#${pkgbase}}")
		_pack${_p#${pkgbase}}
	}"
done
