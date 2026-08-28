# Troubleshooting

Registro de los problemas reales encontrados durante el montaje del
lab y cómo se resolvieron. La idea de este documento es mostrar el
proceso real (con tropiezos incluidos), no una versión idealizada
donde todo funcionó a la primera.

## 1. El OVA oficial de Wazuh fallaba al iniciar

**Problema:** se intentó primero desplegar Wazuh usando la OVA oficial
(appliance preconfigurada). El servicio `wazuh-indexer` fallaba
consistentemente al iniciar, con errores de timeout / señal `TERM`,
sin una causa clara en los logs, a pesar de que la VM tenía recursos
(RAM y disco) suficientes según los requisitos oficiales.

**Solución:** se abandonó la OVA y se optó por instalar Wazuh
mediante **Docker sobre Ubuntu Server**, usando el repo oficial
[`wazuh-docker`](https://github.com/wazuh/wazuh-docker) en su rama
`v4.14.7` (despliegue single-node). Este enfoque resultó mucho más
estable y reproducible.

## 2. Espacio en disco insuficiente al descargar las imágenes de Docker

**Problema:** la instalación inicial de Ubuntu Server dejó el disco
raíz en solo ~8.6 GB. Al ejecutar `docker compose up -d` para levantar
los contenedores de Wazuh, la descarga de imágenes falló con
`no space left on device`.

**Solución:**

1. Expandir el disco virtual desde VMware Workstation
   (*Settings → Hard Disk → Expand* a 55 GB)
2. Expandir la partición LVM dentro de Ubuntu:

   ```bash
   sudo apt install cloud-guest-utils -y
   sudo growpart /dev/sda 3
   sudo pvresize /dev/sda3
   sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
   sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
   ```

Resultado: 53 GB totales, 43 GB disponibles.

**Lección aprendida:** para una próxima instalación desde cero,
asignar el disco completo (50-60 GB) desde el inicio evita este
problema por completo.

## 3. Windows 10 Enterprise ISO no disponible por canales oficiales

**Problema:** el Evaluation Center de Microsoft dejó de ofrecer
directamente el ISO de Windows 10 Enterprise, redirigiendo a Windows
11.

**Decisión:** en vez de recurrir a fuentes no oficiales (por ejemplo,
Internet Archive), lo cual representa un riesgo de integridad del
archivo — especialmente relevante en un proyecto de ciberseguridad —
se optó por usar **Windows 11 IoT Enterprise LTSC Evaluation**
(licencia de evaluación de 90 días). Como beneficio adicional, Windows
11 es más representativo de un endpoint actual en un entorno real.

## 4. Idioma del sistema y compatibilidad con reglas de Wazuh

**Contexto:** las reglas nativas de detección de Wazuh para eventos de
Windows están escritas esperando el texto del Security Event Log en
**inglés**. Si el sistema operativo está en español (u otro idioma),
el texto de los eventos cambia y las reglas nativas no coinciden.

**Solución:** durante la instalación de la VM Windows se configuró
explícitamente el idioma/formato/región en **English (United
States)**, garantizando que el Security Event Log quede en inglés y
sea compatible con las reglas de detección out-of-the-box de Wazuh.

> Nota: el idioma del **Ubuntu Server** (Manager) sí se dejó en
> español sin problema, ya que el Manager corre en contenedores Docker
> con su propia configuración interna, independiente del idioma del
> sistema operativo host.

## 5. Error de permisos al activar "Remember server address"

**Problema:** al desplegar el agente desde el Dashboard y activar la
opción **"Remember server address"**, aparece el error:

```
3021 - Error setting configuration: EACCES: permission denied,
open '/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml'
```

**Causa:** el volumen montado por el contenedor del Dashboard no tiene
los permisos correctos para que el proceso escriba en `wazuh.yml`. Es
un problema conocido del despliegue Docker de Wazuh.

**Workaround aplicado:** no es bloqueante para el registro del agente
— simplemente se ingresó la IP del Manager (`10.0.1.130`) de forma
manual en el campo de server address, sin activar el checkbox, y el
despliegue del agente funcionó con normalidad.

**Fix de raíz (pendiente de aplicar / opcional):** ajustar el dueño
del archivo dentro del contenedor:

```bash
sudo docker exec -u 0 single-node-wazuh.dashboard-1 \
  chown wazuh-dashboard:wazuh-dashboard \
  /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

## 6. Copy-paste entre host y VM Windows no funcionaba

**Problema:** al intentar pegar el comando de instalación del agente
(generado por el Dashboard) dentro del PowerShell de la VM Windows,
`Ctrl+V` no funcionaba.

**Solución aplicada:** en vez de depender del portapapeles
host↔VM, se abrió el navegador **dentro de la propia VM Windows**
(`https://10.0.1.130`) y se generó/copió el comando ahí mismo,
evitando el problema por completo.

**Alternativas para el futuro:** habilitar *Copy and Paste* en
*VM → Settings → Options → Guest Isolation*, y/o confirmar que
VMware Tools esté instalado y corriendo (`Get-Service VMTools`) en la
VM.