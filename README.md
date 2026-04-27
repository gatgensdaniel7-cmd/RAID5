# 🛡️ Implementación y Puesta en Producción: RAID 5

> **¿Qué es RAID 5?**
> RAID 5 distribuye los datos y la información de paridad entre **tres o más discos**. La paridad es un mecanismo matemático que permite reconstruir los datos si uno de los discos falla. A diferencia de RAID 1, **no desperdicia la mitad del espacio**: con 3 discos de 20 GB, se obtienen ~40 GB útiles (se "pierde" solo el equivalente a un disco para paridad). Tolera la falla de **un único disco** a la vez.

---

## 📋 Tabla de Contenidos

1. [Creación de la Máquina Virtual](#1-creación-de-la-máquina-virtual)
2. [Configuración de Discos](#2-configuración-de-discos)
3. [Instalación del Sistema Operativo](#3-instalación-del-sistema-operativo)
4. [Configuración del Arreglo RAID](#4-configuración-del-arreglo-raid)
5. [Monitoreo del Arreglo RAID](#5-monitoreo-del-arreglo-raid)
6. [Monitoreo de Logs del Sistema](#6-monitoreo-de-logs-del-sistema)
7. [Simulación de Falla de Disco](#7-simulación-de-falla-de-disco)
8. [Reconstrucción del Arreglo RAID](#8-reconstrucción-del-arreglo-raid)
9. [Análisis del Proceso](#9-análisis-del-proceso)

---

## 🗺️ Mapa General: ¿Cuándo se hace cada cosa?

| Etapa | ¿Dónde? | ¿Qué se hace con los discos? |
|-------|---------|------------------------------|
| **1. Crear VM** | VirtualBox | Se agregan los 4 discos como archivos vacíos. Sin nombres, sin particiones. |
| **2. Instalar SO** | Instalador del SO | Se particiona **solo `sda`**. Los discos `sdb`, `sdc` y `sdd` se dejan en blanco. |
| **3. Post-instalación** | Terminal del servidor | Se crea el arreglo RAID con `sdb`, `sdc` y `sdd`, se particiona el arreglo y se monta. |

> 💡 **Los nombres `sda`, `sdb`, `sdc`, `sdd` los asigna Linux automáticamente** según el orden en que detecta los discos. Tú no los nombras; simplemente los agregas en orden en VirtualBox y Linux los numera.

### Árbol completo de discos y particiones (resultado final)

```
/dev/sda  (10 GB) → Sistema Operativo
├── /dev/sda1  →  swap      (1 GB)    ← particionado durante instalación del SO
├── /dev/sda2  →  /         (4 GB)    ← particionado durante instalación del SO
└── /dev/sda3  →  /home     (~5 GB)   ← particionado durante instalación del SO

/dev/sdb  (20 GB) → sin particiones propias, entregado al RAID
/dev/sdc  (20 GB) → sin particiones propias, entregado al RAID
/dev/sdd  (20 GB) → sin particiones propias, entregado al RAID
          ↑ los 3 discos juntos forman el arreglo (este es el último que se simula fallar)

/dev/md/RAID5  (arreglo virtual ~40 GB) → creado post-instalación con mdadm
├── /dev/md/RAID5p1  →  /produccion  (20 GB)  ← particionado post-instalación
└── /dev/md/RAID5p2  →  /desarrollo  (~20 GB) ← particionado post-instalación
```

> 💡 **¿Por qué el arreglo tiene ~40 GB si los discos suman 60 GB?**
> En RAID 5 con 3 discos, el espacio de **un disco completo se destina a paridad** (distribuida entre todos los discos). Por eso: `(3 - 1) × 20 GB = 40 GB` útiles.

---

## 1. Creación de la Máquina Virtual

Crear una nueva máquina virtual con las siguientes características:

### Configuración General Recomendada

| Parámetro | Valor sugerido |
|-----------|---------------|
| Sistema Operativo | Ubuntu Server 22.04 LTS (o Debian) |
| RAM | 2 GB mínimo |
| CPU | 2 núcleos |

### Tarjeta de Red

A diferencia de la práctica de RAID 1, esta VM usa **una sola** tarjeta de red:

| Adaptador | Tipo | Propósito |
|-----------|------|-----------|
| Adaptador 1 | **Solo Anfitrión** *(Host-Only)* | Conexión desde terminal (MobaXterm/Putty) |

> ⚠️ **Sin adaptador NAT**, esta VM no tendrá acceso a internet. Si necesitas instalar paquetes durante la práctica (como `mdadm`), puedes agregar temporalmente un adaptador NAT y retirarlo después, o instalarlo durante la configuración inicial.

---

## 2. Configuración de Discos

Agregar **cuatro discos duros** a la máquina virtual desde `Configuración → Almacenamiento → Agregar disco duro`.

> ⚠️ **En esta etapa solo se crean los discos como archivos vacíos.** No se nombran ni se particionan aquí.

Para **cada disco**, marcar las siguientes opciones antes de confirmar:

- ✅ **Unidad de Estado Sólido (SSD)**
- ✅ **Conectable en Caliente** *(Hot-plug)*

> 💡 **¿Por qué marcar "Conectable en Caliente"?**
> Permite desconectar y reconectar un disco mientras la VM está encendida, sin apagarla. Es fundamental para el ejercicio de simulación de falla más adelante.

### Tabla de Discos

Agregar los discos en este orden. Linux los detectará y nombrará automáticamente:

| Nombre (asignado por Linux) | Tamaño a crear en VirtualBox | Uso posterior |
|-----------------------------|------------------------------|---------------|
| `sda` *(primer disco)* | **10 GB** | Sistema Operativo |
| `sdb` *(segundo disco)* | **20 GB** | Miembro del arreglo RAID 5 |
| `sdc` *(tercer disco)* | **20 GB** | Miembro del arreglo RAID 5 |
| `sdd` *(cuarto disco)* | **20 GB** | Miembro del arreglo RAID 5 *(el que se simula fallar)* |

> 💡 **A diferencia de RAID 1**, aquí los tres discos del arreglo son del **mismo tamaño** (20 GB cada uno), por lo que no hay desperdicio de espacio por diferencia de capacidad.

---

## 3. Instalación del Sistema Operativo

Al arrancar el instalador, este detectará los cuatro discos. Aquí se particiona **únicamente `sda`** (el de 10 GB). Los discos `sdb`, `sdc` y `sdd` deben quedar completamente intactos.

### ¿Cómo identificar cuál es `sda` en el instalador?

El instalador generalmente lista los discos por tamaño. Seleccionar el de **10 GB** para instalar el SO. Los tres de 20 GB deben quedar sin tocar.

### Particionamiento del disco `sda` (Sistema Operativo)

En la pantalla de particionamiento del instalador, crear las siguientes particiones **en `sda`**:

| Partición | Tamaño | Tipo | Punto de montaje |
|-----------|--------|------|-----------------|
| `sda1` | **1 GB** | Linux swap | `swap` |
| `sda2` | **4 GB** | ext4 | `/` *(raíz del sistema)* |
| `sda3` | **~5 GB** *(resto)* | ext4 | `/home` |

> 💡 **¿Qué es cada partición?**
> - `swap`: Memoria RAM virtual en disco. Se usa cuando la RAM física se llena.
> - `/`: Partición raíz donde se instala el sistema operativo.
> - `/home`: Carpeta personal de los usuarios del sistema.

> ⚠️ **Los discos `sdb`, `sdc` y `sdd` NO se tocan en este paso.** Si el instalador pregunta qué hacer con ellos, seleccionar "dejar sin usar" o simplemente no asignarles ninguna partición ni punto de montaje.

---

## 4. Configuración del Arreglo RAID

Una vez instalado el SO, iniciar sesión en el servidor e instalar la herramienta de gestión de RAID:

```bash
sudo apt update
sudo apt install mdadm -y
```

### Verificar que los discos estén disponibles

Antes de crear el arreglo, confirmar que `sdb`, `sdc` y `sdd` estén limpios y sin particiones:

```bash
lsblk
```

La salida debe mostrar los tres discos de 20 GB sin ninguna partición listada debajo de ellos.

### Crear el arreglo RAID 5

El arreglo debe llamarse `RAID5` y se construye con los tres discos de 20 GB:

```bash
sudo mdadm --create /dev/md/RAID5 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
```

> 💡 **Desglose del comando:**
> | Parte | Descripción |
> |-------|-------------|
> | `--create /dev/md/RAID5` | Crea el arreglo con ese nombre |
> | `--level=5` | Define el tipo de RAID (en este caso RAID 5) |
> | `--raid-devices=3` | Indica que el arreglo usará 3 discos |
> | `/dev/sdb /dev/sdc /dev/sdd` | Los discos que forman el arreglo |

El sistema comenzará la sincronización inicial automáticamente. Verificar el estado con:

```bash
sudo mdadm -D /dev/md/RAID5
```

> ⚠️ **La sincronización inicial puede tardar varios minutos.** Puedes continuar con los pasos de particionamiento mientras el arreglo termina de sincronizarse en segundo plano.

### Particionamiento del arreglo RAID (`/dev/md/RAID5`)

Con el arreglo creado, particionarlo:

```bash
sudo parted /dev/md/RAID5
```

Dentro de `parted`:
```
mklabel gpt
mkpart produccion ext4 0% 20GB
mkpart desarrollo ext4 20GB 100%
quit
```

| Partición | Tamaño | Punto de montaje |
|-----------|--------|-----------------|
| `/dev/md/RAID5p1` | **20 GB** | `/produccion` |
| `/dev/md/RAID5p2` | **~20 GB** *(resto)* | `/desarrollo` |

### Formatear las particiones

```bash
sudo mkfs.ext4 /dev/md/RAID5p1
sudo mkfs.ext4 /dev/md/RAID5p2
```

### Crear los puntos de montaje y montar

```bash
sudo mkdir -p /produccion /desarrollo
sudo mount /dev/md/RAID5p1 /produccion
sudo mount /dev/md/RAID5p2 /desarrollo
```

### Hacer el montaje permanente (fstab)

Para que las particiones se monten automáticamente al reiniciar:

```bash
echo '/dev/md/RAID5p1 /produccion ext4 defaults 0 0' | sudo tee -a /etc/fstab
echo '/dev/md/RAID5p2 /desarrollo ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

### Guardar la configuración del RAID

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

---

## 5. Monitoreo del Arreglo RAID

Conectarse al servidor mediante **MobaXterm** (usando el adaptador Solo Anfitrión) y ejecutar el siguiente comando para monitorear el RAID en tiempo real:

```bash
while true; do mdadm -D /dev/md/RAID5; sleep 3; clear; done;
```

### Desglose del comando

| Parte | Descripción |
|-------|-------------|
| `while true; do ... done` | Bucle infinito que repite el comando |
| `mdadm -D /dev/md/RAID5` | Muestra el detalle completo del arreglo RAID |
| `sleep 3` | Espera 3 segundos entre cada actualización |
| `clear` | Limpia la pantalla para ver siempre la info más reciente |

### Estructura base del comando de consulta

```bash
mdadm -D /dev/md/NombreArreglo
```

### ¿Qué observar en la salida?

Al ejecutar `mdadm -D`, buscar estos campos clave:

| Campo | Estado normal | Estado de alerta | Durante reconstrucción |
|-------|--------------|-----------------|----------------------|
| `State` | `clean` | `degraded` | `clean, degraded, recovering` |
| `Active Devices` | `3` | `2` | `2` |
| `Failed Devices` | `0` | `1` | `0` |
| `Rebuild Status` | *(no aparece)* | *(no aparece)* | `X% complete` |

> 💡 **Diferencia clave con RAID 1:** RAID 5 sigue funcionando con 2 de 3 discos activos (en estado `degraded`), pero **no tolera una segunda falla** mientras está degradado. Si cae otro disco en ese estado, se pierde todo el arreglo.

---

## 6. Monitoreo de Logs del Sistema

Abrir **una segunda consola** en MobaXterm usando el modo **Split Horizontal** (`View → Split Horizontal`), y ejecutar:

```bash
tail -f /var/log/syslog
```

> 💡 **¿Qué hace este comando?**
> `tail -f` muestra las últimas líneas del archivo de log del sistema y se mantiene actualizado en tiempo real. Aquí el kernel registra eventos críticos como:
> - Desconexión de un disco
> - Detección de fallo en el RAID
> - Inicio del proceso de reconstrucción
> - Finalización de la sincronización

### Vista recomendada en MobaXterm

```
┌─────────────────────────────────┐
│   Terminal 1: Estado del RAID   │
│   (mdadm -D en bucle)           │
├─────────────────────────────────┤
│   Terminal 2: Logs del sistema  │
│   (tail -f /var/log/syslog)     │
└─────────────────────────────────┘
```

---

## 7. Simulación de Falla de Disco

Con ambas terminales activas y visibles, simular la falla del **último disco** del arreglo (`sdd`):

### Pasos

1. Ir a las **propiedades de la máquina virtual** en VirtualBox
2. Localizar el disco de **20 GB** correspondiente a `sdd` *(el cuarto disco, el último agregado)*
3. **Desconectarlo** (sin apagar la VM, gracias a la opción *Hot-plug* habilitada anteriormente)

> 💡 **¿Por qué se elimina `sdd` y no `sdb` o `sdc`?**
> La guía especifica "el último disco del arreglo", que corresponde a `sdd`. En RAID 5 la falla de cualquier disco produce el mismo comportamiento, pero se elige el último para poder identificarlo fácilmente en VirtualBox.

### ¿Qué debe observarse?

**En la Terminal 1 (mdadm):**
- El campo `State` cambia de `clean` → `degraded`
- `Active Devices` baja de `3` → `2`
- `Failed Devices` sube a `1`

**En la Terminal 2 (syslog):**
- Aparecen mensajes como:
  ```
  md/raid5:md127: Disk failure on sdd, disabling device
  md/raid5:md127: Operation continuing on 2 devices
  ```

> ✅ **Punto clave:** el RAID sigue funcionando gracias a la paridad. Los datos en `/produccion` y `/desarrollo` siguen accesibles. Sin embargo, en este estado **no hay redundancia**: si fallara otro disco, se perdería todo el arreglo.

---

## 8. Reconstrucción del Arreglo RAID

### Paso 1: Reconectar el disco

Desde las propiedades de la VM en VirtualBox, volver a conectar el disco de 20 GB (`sdd`).

### Paso 2: Agregar el disco al arreglo

Desde una terminal (MobaXterm o Putty), ejecutar:

```bash
sudo mdadm /dev/md/RAID5 --add /dev/sdd
```

Este comando reintegra el disco al arreglo e inicia automáticamente la **reconstrucción**, usando la paridad almacenada en `sdb` y `sdc` para recalcular y restaurar los datos en `sdd`.

### ¿Qué debe observarse durante la reconstrucción?

**En la Terminal 1 (mdadm):**
- El campo `State` cambia a `clean, degraded, recovering`
- Aparece el campo `Rebuild Status` con el porcentaje de avance:
  ```
  Rebuild Status : 34% complete
  ```

**En la Terminal 2 (syslog):**
- Mensajes de inicio y progreso de la reconstrucción:
  ```
  md: recovery of RAID array md127
  md: minimum _guaranteed_  speed: 1000 KB/sec/disk
  ```

**En la consola de VirtualBox:**
- Se puede observar actividad de disco mientras se realiza la sincronización.

> ⏱️ **Tiempo estimado:** La reconstrucción puede tardar desde segundos hasta varios minutos dependiendo del tamaño del arreglo y el rendimiento del sistema. En RAID 5 el proceso es más exigente que en RAID 1, ya que el sistema debe recalcular la paridad.

---

## 9. Análisis del Proceso

### Resumen del flujo completo

```
Crear VM → Instalar SO → Crear RAID 5 → Particionar RAID
    → Monitorear → Simular falla (sdd) → Observar degradación
    → Reconectar disco → Reconstruir → Observar recuperación
```

### Comparación RAID 1 vs RAID 5

| Característica | RAID 1 | RAID 5 |
|---------------|--------|--------|
| Discos mínimos | 2 | 3 |
| Discos en esta práctica | 2 (20G + 30G) | 3 (20G c/u) |
| Espacio útil | Igual al disco más pequeño | `(n-1) × tamaño disco` |
| Espacio útil en la práctica | ~20 GB | ~40 GB |
| Tolerancia a fallos | 1 disco | 1 disco |
| Método de protección | Espejo completo | Paridad distribuida |
| Disco que se simula fallar | `sdc` | `sdd` |
| Comando de reconstrucción | `--add /dev/sdc` | `--add /dev/sdd` |

### Conclusiones esperadas

| Concepto | Observación práctica |
|----------|---------------------|
| **Tolerancia a fallos** | El sistema siguió funcionando con 2 de 3 discos activos |
| **Eficiencia de espacio** | Se aprovechan ~40 GB de 60 GB totales (vs ~20 GB de 50 GB en RAID 1) |
| **Reconstrucción por paridad** | Al agregar el disco, mdadm recalcula los datos usando paridad |
| **Riesgo en estado degradado** | Con un disco fallado, una segunda falla destruiría el arreglo |
| **Visibilidad en tiempo real** | Los logs y `mdadm -D` reflejaron cada evento al instante |

### Comandos de referencia rápida

```bash
# Ver estado del RAID
sudo mdadm -D /dev/md/RAID5

# Monitoreo en tiempo real
while true; do mdadm -D /dev/md/RAID5; sleep 3; clear; done

# Ver todos los arreglos RAID activos y su estado
cat /proc/mdstat

# Ver logs del sistema en vivo
tail -f /var/log/syslog

# Agregar disco al arreglo (reconstrucción)
sudo mdadm /dev/md/RAID5 --add /dev/sdd

# Verificar discos disponibles y sus particiones
lsblk
```

---

> 📝 **Nota sobre la tarjeta de red:**
> Esta práctica usa únicamente el adaptador **Solo Anfitrión**, a diferencia de la práctica de RAID 1 que usaba dos adaptadores. Esto significa que la VM no tiene acceso a internet. Si al momento de instalar `mdadm` no está disponible, será necesario agregar temporalmente un adaptador NAT desde las propiedades de la VM para poder instalarlo, y luego retirarlo.
