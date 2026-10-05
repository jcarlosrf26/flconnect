# FLConnect 1.0-4 — Gestor de VPN para Tiny Core Linux (32 bits)

Interfaz gráfica en C++ con FLTK, al estilo de NetworkManager: importa perfiles
WireGuard (`.conf`) y OpenVPN (`.ovpn`), y conecta/desconecta cada perfil con
un interruptor ON/OFF. La importación de `.ovpn` es estilo NetworkManager: los
archivos referenciados (ca, cert, key, tls-auth, tls-crypt…) se copian junto al
perfil para que no se rompan las rutas, y si el perfil pide usuario/contraseña
se solicitan al conectar de forma segura.

![Licencia GPL v3](https://img.shields.io/badge/licencia-GPL--3.0-blue)

## Descarga (Release v1.0-4)

Tres archivos, los tres van juntos:

- `flconnect.tcz` — la extensión autocontenida (10 MB).
- `flconnect.tcz.dep` — lista de dependencias: **vacía a propósito** (0 bytes),
  el paquete no depende de ninguna otra extensión del repositorio.
- `flconnect.tcz.md5.txt` — suma MD5 del `.tcz` para comprobar la descarga.

## Instalación en Tiny Core 17.1 x86 (32 bits)

1. Copia los tres archivos a tu directorio de extensiones, p. ej.
   `/mnt/sda1/tce/optional/`.
2. Instala sin conexión (no descarga nada más):

   ```
   tce-load -i flconnect.tcz
   ```

3. Ejecuta:

   ```
   flconnect
   ```

   El ejecutable va con setuid root (4755): trabaja con privilegios sin pedir
   contraseña. Aparece también en el menú de aplicaciones del escritorio.

## Qué trae dentro el paquete (por eso el `.dep` va vacío)

- El binario `flconnect` 1.0-4 (i686, setuid root).
- Herramientas: `wg` y `wg-quick` (wireguard-tools), `wireguard-go` (respaldo
  en espacio de usuario), `ip` (iproute2), `openvpn` 2.6.x, `resolvconf`
  (openresolv) y `bash`, instalados bajo `/usr/local`.
- Todas las librerías necesarias (FLTK 1.3, X11, OpenSSL, etc.).
- Firmware Broadcom b43 (incluido el LP-PHY `ucode15.fw`) en `/lib/firmware`
  y `/usr/local/lib/firmware`.
- Entrada de menú `flconnect.desktop` e icono.

## Dependencias del sistema (para que WireGuard conecte sin errores)

El paquete es autocontenido, pero WireGuard usa el módulo del kernel, y en el
kernel `6.18.35-tinycore` eso tiene tres requisitos del sistema:

1. **Módulos `ipv6` + `wireguard`.** El kernel va sin IPv6 y `wireguard.ko`
   necesita los símbolos `ipv6_mod_enabled` / `ipv6_chk_addr`; sin el módulo
   `ipv6` cargado, `modprobe wireguard` falla con «unknown symbol in module»
   y FLConnect cae al respaldo `wireguard-go`. Solución (repo FLinux):

   ```
   tce-load -wi ipv6-netfilter-6.18.35-tinycore.tcz
   sudo modprobe ipv6
   sudo modprobe wireguard
   ```

   Para que quede automático en cada arranque, añade esas dos líneas de
   `modprobe` a `/opt/bootsync.sh`.

2. **`coreutils.tcz`** — aporta el comando `stat`, que `wg-quick` llama al
   revisar los permisos del `.conf`. Sin él verás «stat: command not found»
   en el registro (la conexión seguiría, pero con ruido y un aviso).

   ```
   tce-load -wi coreutils.tcz
   ```

3. **`nftables.tcz`** — los perfiles de túnel completo (`AllowedIPs =
   0.0.0.0/0`) fijan reglas con `nft`. Los módulos clásicos de iptables
   (`x_tables`, `ip_tables`) no existen en este kernel, así que hace falta nft:

   ```
   tce-load -wi nftables.tcz
   sudo modprobe nf_tables
   ```

   (FLConnect ya hace `modprobe wireguard` antes de conectar, pero el módulo
   solo cargará si `ipv6` está presente.)

Para OpenVPN basta el dispositivo `/dev/net/tun`, incluido en el kernel.

> En la ISO **flinux-jc** (v1.4+) todo esto ya viene instalado y cargado desde
> el arranque; estos pasos son para quien instale el `.tcz` suelto en un
> Tiny Core 17.1.

## Compilar desde el código fuente

Necesitas FLTK 1.3 y compilador con soporte 32 bits:

```
g++ -m32 -march=i686 $(fltk-config --cxxflags) -O2 -o flconnect flconnect.cpp $(fltk-config --ldflags)
```

## Uso

- **Importar WireGuard**: elige un archivo `.conf`.
- **Importar OpenVPN**: elige un archivo `.ovpn`.
- Selecciona un perfil y usa el interruptor para conectar/desconectar.
- Los perfiles se guardan en `~/.flconnect/profiles/` (o en
  `/root/.flconnect/profiles/` al ejecutarse como root).
- Cerrar la ventana no baja una conexión activa: desconecta con el interruptor.

## Licencia

GPL-3.0. Software libre: úsalo, cópialo y modifícalo bajo los términos de la
licencia (ver el archivo `LICENSE`).
