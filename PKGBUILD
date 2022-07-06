# Maintainer: Luke Arms <luke@arms.to>
# Contributor: Pavan Rikhi <pavan.rikhi@gmail.com>
# Contributor: BrLi <brli at chakralinux dot org>

_electronver=15
_nodever=14
pkgname=pencil
pkgver=3.1.0
epoch=1
pkgrel=1
pkgdesc="An open-source GUI prototyping tool"
arch=('i686' 'x86_64')
license=('GPL2')
url="http://pencil.evolus.vn/"
depends=("electron$_electronver")
makedepends=('git' 'nvm' 'jq' 'libxcrypt-compat')
provides=()
conflicts=()
source=("https://github.com/evolus/pencil/archive/v$pkgver.tar.gz"
    'upstream.patch'
    'fix-package-json.patch')
sha256sums=('e14eddd0aad28919cfdf8d47b726f9c75a3a0d2042605e8da96309c23a995f44'
    '3cca728d06df6bc3f5bcc00d9c6f1f70a28c1375ae12a557dd38e222e926726c'
    'c10b84d73e8f37344b14669f0291aa0acf948b82ee2786475d66ab55bcdb692b')

_ensure_local_nvm() {
    if type nvm &>/dev/null; then
        nvm deactivate
        nvm unload
    fi
    unset npm_config_prefix
    export NVM_DIR=${srcdir}/.nvm
    . /usr/share/nvm/init-nvm.sh
}

prepare() {
    cd "${srcdir}/${pkgname}-${pkgver}"

    # To generate `upstream.patch`:
    # - git diff --binary f1309bc..47b2389
    git apply --verbose --whitespace=nowarn "${srcdir}/upstream.patch"

    # Fix invalid electron-builder settings
    patch -p1 <"${srcdir}/fix-package-json.patch"

    _ensure_local_nvm
    # ` || false` is a workaround until this upstream fix is released:
    # https://github.com/nvm-sh/nvm/pull/2698
    nvm ls "$_nodever" &>/dev/null ||
        nvm install "$_nodever" || false
}

build() {
    cd "${srcdir}/${pkgname}-${pkgver}"
    _ensure_local_nvm
    nvm use "$_nodever"
    npm install --global yarn
    yarn install --frozen-lockfile --no-default-rc \
        --cache-folder "${srcdir}/.yarn" --no-progress
    # electron-builder only generates /usr/share/* assets for target package
    # types 'apk', 'deb', 'freebsd', 'p5p', 'pacman', 'rpm' and 'sh', so build a
    # pacman package and unpack it
    local _appname _appver _outfile _unpackdir=${srcdir}/${pkgname}-${pkgver}.unpacked
    _appname=$(jq -r '.name' app/package.json)
    _appver=$(jq -r '.version' app/package.json)
    _outfile=dist/${_appname}-${_appver}.pacman
    rm -rf "${_unpackdir}"
    mkdir -p "${_unpackdir}"

    local i686=ia32 x86_64=x64
    # Add -c.asar=false to suppress creation of an app archive
    ./node_modules/.bin/electron-builder build \
        --linux pacman \
        --"${!CARCH}" \
        -c.electronDist="/usr/lib/electron$_electronver" \
        -c.electronVersion="$(<"/usr/lib/electron$_electronver/version")"
    tar -C "${_unpackdir}" -xf "${_outfile}"

    install -Dm644 /dev/stdin "${_unpackdir}/usr/share/applications/pencil.desktop" <<END
[Desktop Entry]
Encoding=UTF-8
Name=Pencil
Comment=Sketching and GUI prototyping tool
Comment[cs]=Nástroj na kreslení a prototypování GUI
Comment[el]=Εργαλείο σχεδιασμού και κατασκευής πρωτοτύπων γραφικού περιβάλλοντος διεπαφής χρήστη
Comment[es]=Herramienta de esbozado y prototipado de interfaces gráficas de usuario
Comment[vi_VN]=Công cụ phát thảo giao diện Pencil
Exec=/usr/bin/pencil %u
Terminal=false
Type=Application
StartupNotify=true
Icon=pencil
Categories=Graphics;2DGraphics;Development;
MimeType=application/x-evolus-pencil;
END
    install -Dm644 /dev/stdin "${_unpackdir}/usr/share/mime/packages/pencil.xml" <<END
<?xml version="1.0" encoding="UTF-8"?>
<mime-info xmlns="http://www.freedesktop.org/standards/shared-mime-info">
    <mime-type type="application/x-evolus-pencil">
        <comment>Evolus Pencil Document</comment>
        <generic-icon name="image-x-generic"/>
        <glob pattern="*.ep"/>
        <glob pattern="*.epz"/>
        <glob pattern="*.epgz"/>
        <sub-class-of type="text/xml"/>
    </mime-type>
</mime-info>
END
}

package() {
    local _appname
    _appname=$(jq -r '.name' "${srcdir}/${pkgname}-${pkgver}/app/package.json")
    _appdir=opt/${_appname}/resources/app
    cd "${srcdir}/${pkgname}-${pkgver}.unpacked"
    mv "usr" "${pkgdir}/"
    install -d "${pkgdir}/usr/lib"
    [[ -d $_appdir ]] || {
        _appdir=${_appdir%/*}
        _appfile=/app.asar
    }
    mv "$_appdir" "${pkgdir}/usr/lib/${pkgname}"
    install -Dm644 "${srcdir}/${pkgname}-${pkgver}/LICENSE.md" "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE.md"
    install -Dm755 /dev/stdin "${pkgdir}/usr/bin/pencil" <<END
#!/bin/sh
exec /usr/lib/electron${_electronver}/electron /usr/lib/$pkgname$_appfile "\$@"
END
}
