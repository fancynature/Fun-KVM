# Contributor: Maxim Devaev <mdevaev@gmail.com>
# Author: Maxim Devaev <mdevaev@gmail.com>


_variants=(
	v0-hdmi:rpi
	v0-hdmi:rpi2
	v0-hdmi:rpi3

	v0-hdmiusb:zerow
	v0-hdmiusb:rpi
	v0-hdmiusb:rpi2
	v0-hdmiusb:rpi3

	v2-hdmi:zerow
	v2-hdmi:rpi3
	v2-hdmi:rpi4

	v2-hdmiusb:rpi4
	v2-hdmiusb:generic
)


pkgname=(kvmd)
for _variant in "${_variants[@]}"; do
	_platform=${_variant%:*}
	_board=${_variant#*:}
	pkgname+=(kvmd-platform-$_platform-$_board)
done
pkgbase=kvmd
pkgver=2.7
pkgrel=1
pkgdesc="The main Pi-KVM daemon"
url="https://github.com/pikvm/kvmd"
license=(GPL)
arch=(any)
depends=(
	"python>=3.9"
	"python<3.10"
	python-yaml
	python-aiohttp
	python-aiofiles
	python-passlib
	python-pyserial
	python-spidev
	python-setproctitle
	python-psutil
	python-systemd
	python-dbus
	python-pygments
	python-pyghmi
	python-pam
	python-pillow
	python-xlib
	python-hidapi
	libgpiod
	freetype2
	v4l-utils
	nginx-mainline
	openssl
	platformio
	avrdude-svn
	make
	patch
	sudo
	iptables
	iproute2
	dnsmasq
	"raspberrypi-io-access>=0.5"
	"ustreamer>=1.19"

	# Avoid dhcpcd stack trace
	dhclient
	netctl
)
makedepends=(python-setuptools)
source=("$url/archive/v$pkgver.tar.gz")
md5sums=(SKIP)
backup=(
	etc/kvmd/{override,logging,auth,meta}.yaml
	etc/kvmd/{ht,ipmi,vnc}passwd
	etc/kvmd/nginx/{kvmd.ctx-{http,server},loc-{login,nocache,proxy,websocket},mime-types,ssl,nginx}.conf
)


build() {
	cd "$srcdir"
	rm -rf $pkgname-build
	cp -r kvmd-$pkgver $pkgname-build
	cd $pkgname-build
	python setup.py build
}


package_kvmd() {
	install=$pkgname.install

	cd "$srcdir/$pkgname-build"
	python setup.py install --root="$pkgdir"

	install -Dm644 -t "$pkgdir/usr/lib/systemd/system" configs/os/services/*.service
	install -DTm644 configs/os/sysusers.conf "$pkgdir/usr/lib/sysusers.d/kvmd.conf"
	install -DTm644 configs/os/tmpfiles.conf "$pkgdir/usr/lib/tmpfiles.d/kvmd.conf"

	mkdir -p "$pkgdir/usr/share/kvmd"
	cp -r {hid,web,extras,contrib/keymaps} "$pkgdir/usr/share/kvmd"
	find "$pkgdir/usr/share/kvmd/web" -name '*.pug' -exec rm -f '{}' \;

	local _cfg_default="$pkgdir/usr/share/kvmd/configs.default"
	mkdir -p "$_cfg_default"
	cp -r configs/* "$_cfg_default"

	find "$pkgdir" -name ".gitignore" -delete
	sed -i -e "s/^#PROD//g" "$_cfg_default/nginx/nginx.conf"
	find "$_cfg_default" -type f -exec chmod 444 '{}' \;
	chmod 400 "$_cfg_default/kvmd"/*passwd
	chmod 750 "$_cfg_default/os/sudoers"
	chmod 400 "$_cfg_default/os/sudoers"/*

	mkdir -p "$pkgdir/etc/kvmd/nginx/ssl"
	chmod 750 "$pkgdir/etc/kvmd/nginx/ssl"
	install -Dm444 -t "$pkgdir/etc/kvmd/nginx" "$_cfg_default/nginx"/*.conf
	chmod 644 "$pkgdir/etc/kvmd/nginx/nginx.conf"

	install -Dm644 -t "$pkgdir/etc/kvmd" "$_cfg_default/kvmd"/*.yaml
	install -Dm600 -t "$pkgdir/etc/kvmd" "$_cfg_default/kvmd"/*passwd

	mkdir -p "$pkgdir/var/lib/kvmd/msd"

	# Avoid dhcp problems
	install -DTm755 configs/os/netctl-dhcp "$pkgdir/etc/netctl/hooks/pikvm-dhcp"
}


for _variant in "${_variants[@]}"; do
	_platform=${_variant%:*}
	_board=${_variant#*:}
	eval "package_kvmd-platform-$_platform-$_board() {
		cd \"kvmd-\$pkgver\"

		pkgdesc=\"Pi-KVM platform configs - $_platform for $_board\"
		depends=(kvmd=$pkgver-$pkgrel raspberrypi-bootloader>=20201214-3 raspberrypi-bootloader-x>=20201214-3 raspberrypi-firmware>=20201214-3)

		if [[ $_platform =~ ^.*-hdmi$ ]]; then
			depends=(\"\${depends[@]}\")
			if [ $_board == rpi4 ]; then
				depends=(\"\${depends[@]}\" \"linux-raspberrypi4>=5.4.69\" \"linux-raspberrypi4-headers>=5.4.69\")
			else
				depends=(\"\${depends[@]}\" \"linux-raspberrypi>=5.4.69\" \"linux-raspberrypi-headers>=5.4.69\")
			fi
		fi

		backup=(
			etc/sysctl.d/99-kvmd.conf
			etc/udev/rules.d/99-kvmd.rules
			etc/kvmd/main.yaml
		)

		install -DTm644 configs/os/sysctl.conf \"\$pkgdir/etc/sysctl.d/99-kvmd.conf\"
		install -DTm644 configs/os/udev/$_platform-$_board.rules \"\$pkgdir/etc/udev/rules.d/99-kvmd.rules\"
		install -DTm444 configs/kvmd/main/$_platform-$_board.yaml \"\$pkgdir/etc/kvmd/main.yaml\"

		if [ -f configs/os/modules-load/$_platform.conf ]; then
			backup=(\"\${backup[@]}\" etc/modules-load.d/kvmd.conf)
			install -DTm644 configs/os/modules-load/$_platform.conf \"\$pkgdir/etc/modules-load.d/kvmd.conf\"
		fi

		if [ -f configs/os/sudoers/$_platform ]; then
			backup=(\"\${backup[@]}\" etc/sudoers.d/99_kvmd)
			install -DTm440 configs/os/sudoers/$_platform \"\$pkgdir/etc/sudoers.d/99_kvmd\"
			chmod 750 \"\$pkgdir/etc/sudoers.d\"
		fi

		if [[ $_platform =~ ^.*-hdmi$ ]]; then
			backup=(\"\${backup[@]}\" etc/kvmd/tc358743-edid.hex)
			install -DTm444 configs/kvmd/tc358743-edid.hex \"\$pkgdir/etc/kvmd/tc358743-edid.hex\"
		fi
	}"
done
