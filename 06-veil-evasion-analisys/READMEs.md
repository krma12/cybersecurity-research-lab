
Evasión de Antivirus con Veil-Framework y Bat-To-Exe

Plataforma: Lab propio (Entorno Virtualizado)
Dificultad: Media
Fecha: 2026-11-01
Categoría: Red / Evasión de Defensas / Generación de Payloads

# Objetivo

El objetivo de este laboratorio fue comprender las metodologías utilizadas para la evasión de firmas estáticas mediante la creación de un payload ofuscado en PowerShell utilizando Veil-Framework, realizando posteriormente su conversión a un binario ejecutable (.exe) para analizar su tasa de detección en soluciones antivirus.

I. Entorno y herramientas

• Atacante: Kali Linux (root@****) | IP: 192.168.**.***
• Víctima: Windows 10 v1909 (Build 18363.418) | IP: 192.168.**.***
• Herramientas: Veil-Framework (v3.1.14), Bat To Exe Converter (v3.0.8), Wine, Python HTTP Server, Metasploit Framework (msfconsole), VirusTotal.

# Procedimiento


1. Reconocimiento / Enumeración

Se instaló e inició Veil-Framework en el equipo atacante. Al listar las herramientas disponibles en el menú principal, se seleccionó el módulo de evasión (Evasion) para desplegar la lista de los 41 módulos de payloads preconfigurados disponibles para su modificación.

2. Explotación

Para la generación del artefacto base, se siguieron los siguientes pasos técnicos:
1. Selección del Payload: Se cargó el módulo número 22 correspondiente a powershell/meterpreter/rev_tcp (Pure PowerShell Reverse TCP Stager).
2. Configuración de Variables: Se definió el parámetro LHOST apuntando a la IP de escucha (192.168.**.***) y se aplicó un retraso de evasión mediante set SLEEP 20 para dificultar el análisis dinámico en entornos sandbox rudimentarios.
3. Generación del Script: Se ejecutó el comando generate, nombrando al archivo de salida como powerepayload. Veil exportó el código fuente en formato de procesamiento por lotes (.bat) y generó automáticamente un archivo de recursos .rc para automatizar el despliegue del manejador en Metasploit.
4. Compilación Exe (B2E): Con el fin de empaquetar el script, se clonó el repositorio tokyoneon/B2E, se descomprimió la herramienta Bat_To_Exe_Converter.zip y se ejecutó su instalador sobre la capa de compatibilidad Wine. A través de la interfaz gráfica del compilador, se procesó el archivo .bat transformándolo en el ejecutable final pspayload.exe.
5. Transferencia del Archivo: Se levantó un servicio HTTP temporal con Python en el puerto 8000 dentro del directorio de trabajo. Desde la máquina víctima con Windows 10, se accedió al servidor web y se procedió a descargar el binario.

3. Post-explotación / Resultado

En el sistema atacante, se ejecutó Metasploit invocando el script de recursos generado por Veil (resource /var/lib/veil/output/handlers/powerepayload2.rc), lo cual inicializó de manera automatizada el módulo exploit/multi/handler.
Al ejecutarse el binario pspayload.exe en la máquina Windows, el payload interactuó de regreso de manera exitosa hacia el puerto 4444, evadiendo la restricción perimetral estática y otorgando dos sesiones activas de Meterpreter (x86/windows) correspondientes al usuario DESKTOP-IJIDQ95\pink2. Se interactuó con la sesión número 2 y se desplegó una shell interactiva del sistema (cmd.exe).

# Hallazgo

Al subir el archivo compilado pspayload.exe a la plataforma multiescaner VirusTotal, se obtuvo un ratio de detección de 38/71 proveedores de seguridad.
Aunque el mecanismo de ofuscación nativo de Veil y el empaquetado mediante Bat To Exe lograron cegar a una cantidad considerable de motores comerciales (demostrando las limitaciones del análisis puramente estático basado en hashes tradicionales), soluciones con firmas heurísticas o telemetría avanzada (como CrowdStrike Falcon o BitDefender) identificaron la firma subyacente del stager de PowerShell o el comportamiento del wrapper ejecutable como malicioso (etiquetado común: Trojan.Win/BZC).

🛡️ Mitigación / Remediación

Para proteger un entorno corporativo o real frente a técnicas de ejecución y evasión basadas en scripts ofuscados de PowerShell empaquetados, se deben implementar los siguientes controles de seguridad:

• Restricción de Ejecución y Constrained Language Mode: Configurar directivas de grupo (GPO) para forzar el uso de Constrained Language Mode en PowerShell, limitando drásticamente los objetos de .NET y las llamadas a la API de Windows que un script de usuario puede invocar de forma interactiva.
• Implementación de AMSI (Antimalware Scan Interface): Asegurar que las soluciones de Endpoint/EDR hagan uso activo de AMSI. Dado que este mecanismo inspecciona el script de PowerShell directamente en memoria justo antes de su ejecución, tiene la capacidad de analizar el código una vez que ha sido desofuscado por el propio intérprete, neutralizando el empaquetado del archivo executable.
• AppLocker / Windows Defender Application Control (WDAC): Configurar políticas estrictas de control de aplicaciones para permitir únicamente la ejecución de binarios que cuenten con firmas digitales confiables o que provengan de rutas del sistema explícitamente autorizadas, bloqueando por completo la ejecución de ejecutables desconocidos o no firmados dentro de los perfiles de usuario (%USERPROFILE%\Downloads, %TEMP%).

📚 Lecciones aprendidas

• Los atacantes recurren a la combinación de múltiples lenguajes o envoltorios (wrappers) como herramientas de conversión de scripts a ejecutables para enmascarar la lógica original de un código malicioso ante inspecciones estáticas básicas de archivos.
• El análisis basado únicamente en firmas de archivos tradicionales es insuficiente en sistemas modernos; el monitoreo del comportamiento dinámico en los endpoints y la visibilidad sobre los procesos secundarios de PowerShell son críticos para la detección temprana.
• La configuración defensiva por defecto en los sistemas operativos debe ser endurecida de manera proactiva, ya que herramientas de generación automática pueden hallar vectores ciegos si no existen políticas restrictivas de ejecución de software.



