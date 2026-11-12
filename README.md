# Mantenimiento y optimizacion del almacenamiento de Docker Desktop en Windows

- **Objetivo:** diagnosticar, limpiar y recuperar espacio en disco ocupado por Docker Desktop utilizando WSL2, sin perder imagenes, contenedores ni volumenes persistentes
- **Plataforma:** Windows 10/11 + Docker Desktop + WSL2  
- **Nivel:** Intermedio  
- **Frecuencia recomendada:** Revision mensual o cuando el almacenamiento de Docker crezca considerablemente
- **Equipo:** CoreDataOps
---

## 1. Introduccion

Docker Desktop utiliza discos virtuales para almacenar imagenes, capas, contenedores, volumenes y caches de compilacion
En Windows con backend WSL2, uno de los archivos principales de almacenamiento es:

```text
%LOCALAPPDATA%\Docker\wsl\disk\docker_data.vhdx
```

Este archivo utiliza almacenamiento dinamico. Puede aumentar de tamaño a medida que Docker genera imagenes, descarga dependencias y realiza compilaciones
Un aspecto importante es que eliminar datos dentro de Docker no implica necesariamente que Windows recupere inmediatamente el espacio fisico ocupado por el archivo VHDX
Por ello, el mantenimiento se divide en dos operaciones:

1. Limpieza logica: eliminar recursos Docker que ya no son necesarios
2. Compactacion fisica: reducir el tamaño del disco virtual para devolver espacio a Windows
---

## 2. Diagnostico del almacenamiento

Antes de eliminar cualquier recurso, es necesario identificar que componentes consumen almacenamiento
### 2.1. Consultar el consumo general

Ejecutar en PowerShell:

```powershell
docker system df
```

El resultado presenta cuatro categorias:

| Categoria | Descripcion |
|---|---|
| Images | Imagenes Docker almacenadas |
| Containers | Espacio utilizado por contenedores |
| Local Volumes | Datos persistentes de los volumenes |
| Build Cache | Cache de compilaciones de BuildKit |

La columna `RECLAIMABLE` indica cuanto espacio Docker considera recuperable
### 2.2. Consultar el detalle

```powershell
docker system df -v
```

Muestra informacion detallada sobre imagenes, contenedores, volumenes y cache
Para revisar especificamente la cache de compilacion:

```powershell
docker buildx du
```

### 2.3. Identificar el tamaño del disco virtual

```powershell
$vhd = "$env:LOCALAPPDATA\Docker\wsl\disk\docker_data.vhdx"

Get-Item $vhd |
Select-Object FullName,
    @{Name="TamañoGB";Expression={[math]::Round($_.Length / 1GB, 2)}}
```

**Importante:** el tamaño del VHDX representa su tamaño fisico en Windows, no necesariamente el espacio logico que Docker esta utilizando actualmente
---

## 3. Limpieza de recursos Docker

La limpieza debe realizarse selectivamente para evitar perder informacion importante
### 3.1. Limpiar la cache de compilacion

La cache de BuildKit almacena capas y resultados intermedios de compilaciones anteriores para acelerar futuros builds
Consultar su consumo:

```powershell
docker buildx du
```

Eliminar toda la cache no utilizada:

```powershell
docker builder prune -a
```

Docker solicitara confirmacion
Esta operacion elimina cache de compilacion que no esta siendo utilizada. No elimina las imagenes, los contenedores ni los volumenes
**Consideracion:** las siguientes compilaciones pueden tardar mas porque Docker tendra que reconstruir las capas eliminadas
Tambien es posible eliminar unicamente la cache no utilizada durante mas de 30 dias:

```powershell
docker builder prune -a --filter "until=720h"
```

### 3.2. Limpiar imagenes no utilizadas

Consultar las imagenes:

```powershell
docker images
```

Eliminar imagenes no utilizadas por contenedores:

```powershell
docker image prune -a
```

Esta operacion puede obligar a descargar nuevamente imagenes que se necesiten en el futuro
No ejecutarla indiscriminadamente en entornos donde se necesite conservar imagenes locales especificas
### 3.3. Revisar contenedores

```powershell
docker ps -a --size
```

Eliminar contenedores detenidos:

```powershell
docker container prune
```

### 3.4. Revisar volumenes persistentes

```powershell
docker volume ls
```

Inspeccionar un volumen:

```powershell
docker volume inspect NOMBRE_DEL_VOLUMEN
```

**Precaucion:** los volumenes pueden almacenar bases de datos PostgreSQL, MongoDB, Redis y otros datos persistentes
No utilizar automaticamente:

```powershell
docker volume prune
```

Tampoco utilizar:

```powershell
docker system prune -a --volumes
```

sin verificar previamente que datos se perderian
---

## 4. Configurar la recoleccion automatica de BuildKit

Docker incorpora Garbage Collection (GC) para gestionar automaticamente la cache de compilacion
### 4.1. Acceder a la configuracion

1. Abrir Docker Desktop
2. Entrar en Settings
3. Seleccionar Docker Engine
4. Localizar el objeto JSON de configuracion
Agregar o integrar la siguiente propiedad:

```json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "8GB"
    }
  }
}
```

Si existen otras propiedades, conservarlas e integrar `builder` dentro del objeto JSON principal
5. Seleccionar Apply & Restart
### 4.2. Interpretacion de los parametros

| Parametro | Funcion |
|---|---|
| enabled | Habilita la recoleccion automatica |
| defaultKeepStorage | Define el umbral de almacenamiento utilizado por las politicas de GC |

El valor de 8 GB es una configuracion orientativa para entornos de desarrollo con almacenamiento limitado
**Importante:** `defaultKeepStorage` no constituye un limite rigido de todo Docker ni de todo el archivo VHDX. Influye en las politicas de recoleccion de cache de BuildKit
La configuracion predeterminada de Docker ya incluye politicas de limpieza automatica. Personalizarla es opcional
---

## 5. Compactacion del disco virtual VHDX

Despues de eliminar grandes cantidades de datos, puede ocurrir que el archivo `docker_data.vhdx` siga ocupando practicamente el mismo espacio en Windows
Para recuperar ese espacio fisico, es necesario compactarlo
### 5.1. Identificar el archivo VHDX

La ubicacion predeterminada para Docker Desktop con WSL2 es:

```powershell
$vhd = "$env:LOCALAPPDATA\Docker\wsl\disk\docker_data.vhdx"
```

Si Docker Desktop utiliza otra ubicacion de disco, consultar Settings → Resources → Advanced
### 5.2. Cerrar Docker Desktop

Antes de compactar:

1. Guardar los trabajos en ejecucion
2. Cerrar las terminales WSL
3. Cerrar VS Code si utiliza Dev Containers
4. En la bandeja del sistema, seleccionar Quit Docker Desktop
Ejecutar PowerShell como administrador:

```powershell
wsl --shutdown
```

Comprobar que no existan distribuciones ejecutandose:

```powershell
wsl --list --running
```

El disco no debe estar siendo utilizado por otro proceso
### 5.3. Compactar mediante DiskPart

Windows proporciona DiskPart para administrar discos virtuales
Crear y ejecutar el siguiente procedimiento desde PowerShell como administrador:

```powershell
$vhd = "$env:LOCALAPPDATA\Docker\wsl\disk\docker_data.vhdx"

if (!(Test-Path $vhd)) {
    Write-Error "No se encontro el archivo VHDX."
    return
}

$before = (Get-Item $vhd).Length / 1GB

Write-Host "Tamaño antes: $([math]::Round($before, 2)) GB"

$script = @"
select vdisk file="$vhd"
attach vdisk readonly
compact vdisk
detach vdisk
exit
"@

$scriptPath = "$env:TEMP\compact-docker.txt"

$script | Set-Content -Path $scriptPath -Encoding ASCII

diskpart /s $scriptPath

$after = (Get-Item $vhd).Length / 1GB

Write-Host "`nTamaño despues: $([math]::Round($after, 2)) GB"

Write-Host "Espacio recuperado: $([math]::Round($before - $after, 2)) GB"
```

El procedimiento:

1. Selecciona el disco virtual
2. Lo adjunta en modo de solo lectura
3. Ejecuta la compactacion
4. Desadjunta el disco
5. Compara el tamaño anterior y posterior
La operacion no pretende eliminar datos del disco virtual
### 5.4. Si aparece un error de archivo en uso

Un error frecuente es:

```text
El proceso no tiene acceso al archivo
porque esta siendo utilizado por otro proceso
```

Esto significa que algun proceso todavia mantiene abierto el archivo VHDX
Procedimiento recomendado:

1. Cerrar Docker Desktop completamente
2. Ejecutar `wsl --shutdown`
3. Verificar `wsl --list --running`
4. Cerrar aplicaciones que utilicen WSL o Docker
5. Si el bloqueo continua, reiniciar Windows
6. No abrir Docker Desktop despues del reinicio
7. Abrir PowerShell como administrador
8. Ejecutar nuevamente la compactacion
No terminar procesos arbitrariamente ni eliminar el archivo VHDX
---

## 6. Verificacion posterior

Una vez finalizada la compactacion:

1. Abrir Docker Desktop
2. Esperar a que el motor Docker este disponible
3. Ejecutar los siguientes comandos:

```powershell
docker system df
```

```powershell
docker ps -a
```

```powershell
docker volume ls
```

Comprobar el tamaño del archivo VHDX:

```powershell
$vhd = "$env:LOCALAPPDATA\Docker\wsl\disk\docker_data.vhdx"

Get-Item $vhd |
Select-Object FullName,
    @{Name="TamañoGB";Expression={[math]::Round($_.Length / 1GB, 2)}}
```

Verificar que las imagenes, contenedores y volumenes importantes continuen disponibles
---

## 7. Ejemplo de mantenimiento realizado

El siguiente caso ilustra los resultados obtenidos durante un mantenimiento real
### 7.1. Estado inicial

| Recurso | Espacio ocupado | Recuperable |
|---|---:|---:|
| Images | 8.197 GB | 3.35 GB |
| Containers | 274.4 KB | 274.4 KB |
| Local Volumes | 2.617 GB | 1.924 GB |
| Build Cache | 24.46 GB | 20.41 GB |

El principal consumidor era BuildKit, con 313 entradas de cache
### 7.2. Limpieza realizada

Se ejecuto:

```powershell
docker builder prune -a
```

Resultado posterior:

| Recurso | Espacio ocupado |
|---|---:|
| Images | 2.149 GB |
| Containers | 274.4 KB |
| Local Volumes | 2.617 GB |
| Build Cache | 0 B |

No se ejecuto una limpieza de volumenes
### 7.3. Compactacion realizada

| Estado | Tamaño fisico |
|---|---:|
| Antes | 78.31 GB |
| Despues | 11.91 GB |
| Espacio recuperado | 66.40 GB |

La compactacion permitio recuperar aproximadamente el 84.8 % del tamaño fisico inicial del disco virtual
---

## 8. Plan de mantenimiento preventivo

| Frecuencia | Actividad |
|---|---|
| Semanal | Consultar `docker system df` |
| Mensual | Revisar cache con `docker buildx du` |
| Mensual | Identificar imagenes y contenedores innecesarios |
| Mensual | Revisar volumenes persistentes |
| Cuando corresponda | Ejecutar limpieza selectiva |
| Despues de una limpieza importante | Evaluar compactacion del VHDX |

No es necesario compactar el disco virtual despues de cada compilacion o limpieza pequeña
---

## 9. Precauciones importantes

- No eliminar manualmente `docker_data.vhdx`
- No compactar mientras Docker Desktop este utilizando el archivo
- No ejecutar limpiezas de volumenes sin inspeccionar sus datos
- No confundir el tamaño fisico del VHDX con el consumo logico de Docker
- Realizar copias de seguridad de los datos importantes antes de operaciones de mantenimiento
- La ruta y el procedimiento descritos corresponden al backend WSL2; Hyper-V y Docker VMM pueden utilizar ubicaciones diferentes
---

## 10. Referencias oficiales

- [Docker Docs - Build garbage collection](https://docs.docker.com/build/cache/garbage-collection/)
- [Docker Docs - Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)
- [Docker Docs - Backup and restore](https://docs.docker.com/desktop/settings-and-maintenance/backup-and-restore/)
- [Microsoft Learn - DiskPart](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/diskpart)
