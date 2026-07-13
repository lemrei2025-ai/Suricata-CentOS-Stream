## Instalación de Suricata IDS (CentOS Stream 10)

**Curso:** Seguridad Operativa
**Docente:** Edgar Mauricio Lopez Rojas

Taller original de instalación de Suricata desde código fuente, revisado,
corregido y actualizado a la última versión estable para funcionar en
**CentOS Stream 10** (familia EL10: también válido para RHEL 10, AlmaLinux 10
y Rocky Linux 10).

## ⚠️ Antes de empezar

- Corre esto en una VM o laboratorio propio, no en un servidor de producción.
- Necesitás privilegios de `root` / `sudo` para todos los comandos.
- El taller pide documentar el proceso en video durante la clase — ver
  [Instrucciones del docente](#instrucciones-del-docente) al final.

## Qué cambió respecto al taller original

El taller original apuntaba a Suricata 7.0.11 y tenía algunos detalles que no
funcionan (o no son óptimos) en CentOS Stream 10. Este repositorio corrige:

| # | Original | Corregido | Por qué |
|---|---|---|---|
| 1 | Suricata **7.0.11** | Suricata **8.0.6** | La serie 7.x llegó a su **EOL** (7.0.17 fue la última versión); 8.0.6 es la versión estable actual (julio 2026) |
| 2 | `rust-toolset` en la lista de paquetes | `rust` (+ `cargo`, que ya estaba) | `rust-toolset` era un módulo de AppStream en EL8/EL9; **no existe en EL10**, donde Rust se instala como paquete individual |
| 3 | `--enable-geopip` (typo) | `--enable-geoip` | El flag real de `./configure` es `--enable-geoip`; con el typo, `configure` lo ignora silenciosamente y el soporte GeoIP queda desactivado |
| 4 | Sin `libmaxminddb-devel` | Se agrega `libmaxminddb-devel` | Las versiones actuales de Suricata implementan GeoIP sobre `libmaxminddb` (formato GeoIP2), no sobre la librería GeoIP legacy; sin esta `-devel`, `--enable-geoip` falla en el `configure` |
| 5 | Paquetes duplicados (`libyaml-devel`, `jansson-devel`, `lua-devel`, `libpcap-devel` repetidos) | Lista limpia, sin duplicados | Cosmético — no rompía nada, pero ensuciaba el comando |
| 6 | Sin `--disable-gccmarch-native` | Se agrega | Recomendado por la documentación oficial cuando se compila dentro de una VM o se busca portabilidad del binario |
| 7 | Pasos de EPEL/CRB (`dnf install epel-release`, `dnf config-manager --set-enabled crb`) | Sin cambios | Verificado: CentOS Stream 10 todavía usa DNF4 y estos comandos funcionan igual que en EL9 |

El resto del procedimiento (actualizar el sistema, `Development Tools`,
`ldconfig`, `suricata-update`, el test final) se mantiene igual porque ya
era correcto.

## Requisitos

- CentOS Stream 10 (o RHEL/AlmaLinux/Rocky 10) con acceso a internet.
- Usuario con privilegios `sudo` o acceso a `root`.

## Instalación automática

Este repo incluye [`install_suricata.sh`](./install_suricata.sh), que
encadena todos los pasos del taller:

```bash
chmod +x install_suricata.sh
sudo ./install_suricata.sh
```

## Instalación paso a paso (manual)

Si preferís correr cada paso a mano (recomendado la primera vez, para
entender qué hace cada comando):

1. Actualizar el sistema operativo:
   ```bash
   sudo dnf -y update
   ```
2. Habilitar EPEL y las herramientas de gestión de repos:
   ```bash
   sudo dnf -y install epel-release dnf-plugins-core
   ```
3. Habilitar el repositorio CRB (CodeReady Builder):
   ```bash
   sudo dnf config-manager --set-enabled crb
   ```
4. Instalar el grupo de herramientas de desarrollo:
   ```bash
   sudo dnf -y group install "Development Tools"
   ```
5. Instalar las dependencias de compilación:
   ```bash
   sudo dnf -y install diffutils file-devel gcc jansson-devel make nss-devel \
       libyaml-devel libcap-ng-devel libpcap-devel \
       python3 python3-pyyaml rust cargo zlib-devel curl wget tar \
       lua lua-devel lz4-devel pcre2-devel libmaxminddb-devel
   ```
6. Descargar Suricata (versión actual: **8.0.6**):
   ```bash
   wget https://www.openinfosecfoundation.org/download/suricata-8.0.6.tar.gz -P /tmp
   ```
7. Cambiar de directorio:
   ```bash
   cd /tmp
   ```
8. Descomprimir:
   ```bash
   tar xzf suricata-8.0.6.tar.gz
   ```
9. Entrar al directorio del código fuente:
   ```bash
   cd suricata-8.0.6
   ```
10. Configurar variables y rutas:
    ```bash
    ./configure --sysconfdir=/etc --localstatedir=/var --prefix=/usr/ \
        --disable-gccmarch-native --enable-lua --enable-geoip
    ```
11. Compilar:
    ```bash
    make -j"$(nproc)"
    ```
12. Instalar (configuración + reglas incluidas):
    ```bash
    sudo make install-full
    ```
13. Corregir el linker cache:
    ```bash
    sudo ldconfig
    ```
14. Actualizar el set de reglas:
    ```bash
    sudo suricata-update
    ```
15. Probar la configuración:
    ```bash
    sudo suricata -c /etc/suricata/suricata.yaml -T -v
    ```

Si el test del paso 15 termina con `Configuration provided was successfully loaded`
(o similar, sin errores), la instalación quedó funcional.

## Nota sobre GeoIP

`--enable-geoip` solo compila el soporte GeoIP dentro de Suricata; no incluye
ninguna base de datos. Para usar reglas con la keyword `geoip`, hace falta
además:

1. Crear una cuenta gratuita en [MaxMind](https://www.maxmind.com/en/geolite2/signup).
2. Generar una license key.
3. Descargar la base `GeoLite2-Country` (formato `.mmdb`) y configurar la ruta
   correspondiente en `suricata.yaml`.

## Solución de problemas

- **`crb` no se puede habilitar / no existe el repo:** confirmá que
  `dnf-plugins-core` quedó instalado (paso 2) antes de correr
  `config-manager`.
- **Error de dependencias no resueltas al instalar paquetes `-devel`:**
  confirmá que el paso 3 (habilitar `crb`) se ejecutó *antes* del paso 5;
  varias `-devel` (por ejemplo `jansson-devel`) vienen de EPEL/CRB, no del
  repositorio base.
- **`configure` no encuentra Rust/cargo:** verificá con `cargo --version`;
  si el paquete de la distro quedara desactualizado, se puede instalar Rust
  directamente desde [rustup](https://www.rust-lang.org/en-US/install.html)
  siguiendo la sección "Rust support" de la
  [documentación oficial](https://docs.suricata.io/en/suricata-8.0.6/install.html#rust-support).
- **DNF5 en vez de DNF4:** si en el futuro CentOS Stream pasa a usar DNF5 por
  defecto, el equivalente de `dnf config-manager --set-enabled crb` sería
  `dnf5 config-manager --enable crb` (o `dnf5 config-manager setopt crb.enabled=1`).
- **SELinux / firewalld:** este taller solo corre un test de configuración
  (`-T`), no levanta Suricata como servicio en modo IDS en vivo. Si más
  adelante lo corrés como `systemd` service o en modo captura en una
  interfaz real, revisá el estado de SELinux (`getenforce`) y las reglas de
  `firewalld` si algo falla al iniciar o capturar tráfico.

## Referencias

- [Descargas oficiales de Suricata](https://www.openinfosecfoundation.org/download/)
- [Documentación de instalación (Suricata 8.0.6)](https://docs.suricata.io/en/suricata-8.0.6/install.html)
- [Anuncio de la versión 8.0.6 / 7.0.17 (EOL de la serie 7)](https://suricata.io/2026/07/09/suricata-8-0-6-and-7-0-17-released/)
- [Cómo habilitar EPEL en RHEL 10 / CentOS Stream 10](https://computingforgeeks.com/how-to-install-epel-on-rhel-10-and-centos-stream-10/)

## Instrucciones del docente

> Documente muy bien el proceso en un video durante el ejercicio en clase.
> Si durante este se presenta algún inconveniente o error, antes de hablar
> con el docente por favor valide posibles soluciones a aplicar. Ojo con APA.
>
> Cualquier duda pregunte por favor.

## Licencia

MIT — ver [LICENSE](./LICENSE).
