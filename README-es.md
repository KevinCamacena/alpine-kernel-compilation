---
Type: Procedure
---

# Playbook: Compilación de un Kernel de Linux Personalizado en Alpine

**Entorno de Destino:** Máquina Virtual (VM) con Alpine Linux (musl libc, syslinux/extlinux)
**Versión del Kernel:** 7.0.0-rc6

Esta guía documenta el proceso de extremo a extremo para compilar un kernel de Linux mainline en un host de Alpine Linux. Incluye mitigaciones específicas para compilar dentro de un entorno de libc `musl` y eludir las firmas criptográficas de los constructores oficiales de Alpine.

---

## Requisitos Previos

Asegúrate de que el repositorio `community` esté habilitado en la configuración de tu gestor de paquetes.

```bash
# Habilitar el repositorio community si no está presente
doas vi /etc/apk/repositories

# Actualizar el índice de paquetes
doas apk update
```

Instala todas las herramientas de compilación requeridas, reemplazando los applets de BusyBox predeterminados de Alpine con las utilidades de GNU necesarias para los scripts de compilación del kernel, junto con las bibliotecas matemáticas para los plugins de GCC.

```bash
doas apk add \
  build-base linux-headers bash \
  flex bison bc perl \
  openssl-dev elfutils-dev ncurses-dev zstd-dev xz tar wget \
  coreutils findutils diffutils gawk grep sed \
  pkgconf python3 argp-standalone pahole \
  gmp-dev mpfr-dev mpc1-dev \
  fastfetch
```

## Paso 1: Preparación y Configuración

Crea un directorio de compilación, extrae el código fuente del kernel y clona la configuración del kernel actual en ejecución.

```bash
# Revisa las versiones del kernel aquí: [https://github.com/torvalds/linux/tags](https://github.com/torvalds/linux/tags)
wget -qO- [https://github.com/torvalds/linux/archive/refs/tags/v7.0-rc6.tar.gz](https://github.com/torvalds/linux/archive/refs/tags/v7.0-rc6.tar.gz)
# Configurar directorios y extraer el código fuente
mkdir -p ~/.build-rc
tar -xf v7.0-rc6.tar.gz -C ~/.build-rc/
cd ~/.build-rc/linux-7.0-rc6/

# Copiar la configuración del kernel existente en el host
cp /boot/config-$(uname -r) .config
```

## Paso 2: Parcheo de Configuraciones para Alpine

Al clonar una configuración de kernel oficial de Alpine, las claves del constructor codificadas (hardcoded) y las incompatibilidades con musl libc deben ser parcheadas antes de compilar.

1. Borrar Claves Criptográficas Codificadas
Borra las rutas absolutas a las claves de firma del mantenedor oficial de Alpine para evitar que la compilación se detenga, y configura la clave de firma del módulo para autogenerar una clave efímera.

```bash
./scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
./scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
./scripts/config --set-str MODULE_SIG_KEY "certs/signing_key.pem"

# Aplicar los cambios de configuración
make olddefconfig
```

2. Solucionar Falta de Definición attribute_const en Musl Libc
Parchea la herramienta del host insn_sanity.c para incluir explícitamente las definiciones del compilador de Linux, eludiendo el cumplimiento estricto de musl libc que carece de esta macro de GNU.

```bash
sed -i '1i #include <linux/compiler.h>' arch/x86/tools/insn_sanity.c
```

## Paso 3: Compilación

Compila el kernel utilizando todos los núcleos de CPU disponibles.

```bash
make -j$(nproc)
```

## Paso 4: Instalación

Instala los módulos del kernel compilados en /lib/modules/ y copia manualmente los archivos principales del kernel al directorio de arranque (/boot).

```bash
# Instalar módulos
doas make modules_install

# Copiar la imagen del kernel, el mapa del sistema y la configuración
doas cp arch/x86/boot/bzImage /boot/vmlinuz-7.0.0-rc6
doas cp System.map /boot/System.map-7.0.0-rc6
doas cp .config /boot/config-7.0.0-rc6
```

## Paso 5: Configuración del Cargador de Arranque (Syslinux/Extlinux)

Alpine requiere generar un disco RAM inicial (initramfs) para los nuevos módulos y agregar manualmente la entrada de arranque a extlinux.conf.

1. Generar el Initramfs

```bash
doas mkinitfs -o /boot/initramfs-7.0.0-rc6 7.0.0-rc6
```

2. Agregar Entrada al Cargador de Arranque

```bash
doas vi /boot/extlinux.conf
```

_Acción_: Copia tu bloque de entrada de arranque existente (por ejemplo, LABEL lts) y pégalo debajo. Actualiza las variables label, menu label, kernel e initrd para que apunten a los nuevos archivos -7.0.0-rc6. Crucial: Preserva la línea APPEND existente exactamente como está para asegurar que el UUID del sistema de archivos raíz siga siendo correcto.

Opcionalmente, cambia el parámetro DEFAULT en la parte superior del archivo para que apunte a tu nueva etiqueta y así arrancar automáticamente.

Debería verse así:

```.conf
DEFAULT custom
PROMPT 0
MENU TITLE Alpine/Linux Boot Menu
MENU HIDDEN
MENU AUTOBOOT Alpine will be booted automatically in # seconds.
TIMEOUT 10
LABEL lts
  MENU DEFAULT
  MENU LABEL Linux lts
  LINUX vmlinuz-lts
  INITRD initramfs-lts
  APPEND root=UUID=xxx modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4

MENU SEPARATOR

LABEL custom
  MENU LABEL Linux 7.0.0-rc6
  LINUX vmlinuz-7.0.0-rc6
  INITRD initramfs-7.0.0-rc6
  APPEND root=UUID=xxx modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4

MENU SEPARATOR
```

## Paso 6: Verificación

Reinicia el sistema y verifica que el nuevo kernel se esté ejecutando.

```bash
doas reboot

# Después de reiniciar, confirmar la versión en ejecución
fastfetch
```

---

## Solución de Problemas

Errores de Compilación Anómalos (Inversión de Bits / Bit Flips)
Al descargar, extraer o compilar grandes bases de código en la memoria, los rayos cósmicos o fluctuaciones menores de hardware pueden causar ocasionalmente la inversión de un solo bit (bit flip). Esto se manifiesta como errores de "identificador no declarado" altamente específicos en la salida del compilador debido a un error tipográfico repentino en el código fuente.

### Ejemplo de Error

```./include/linux/gfp.h:133:44: error: 'GFP_ZONMS_SHIFT' undeclared (first use in this function); did you mean 'GFP_ZONES_SHIFT'?
```

### Resolución

No descartes el directorio de compilación ni comiences de nuevo. Es probable que el proceso central de compilación esté intacto.

1. Identifica el archivo específico y el error tipográfico en la salida de error del compilador.

2. Usa sed para parchear el bit invertido en el archivo fuente afectado:

```bash
# Ejemplo de solución para el error específico de arriba
sed -i 's/GFP_ZONMS_SHIFT/GFP_ZONES_SHIFT/g' include/linux/gfp.h
```

3. Reanuda la compilación. El comando make continuará automáticamente desde donde se quedó y solo reconstruirá las dependencias para el archivo parcheado:

```bash
make -j$(nproc)
```

Una vez que la compilación se complete con éxito, procede directamente al Paso 4: Instalación.
