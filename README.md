# REVERSING-ANALYSIS-.NET-APP

> **Autor:** Randy Nin **Laboratorio CyberOps | Análisis de Aplicación .NET**

Análisis técnico completo de una aplicación WinForms en C# con .NET 8.0 que oculta su lógica de validación mediante encodings encadenados, ofuscación de strings y obtención remota de la contraseña desde GitHub. La resolución requiere análisis estático con dnSpy, enumeración de puertos con Process Explorer, interacción con un servicio TCP local y decodificación manual de seis técnicas de encoding distintas.

---

## Contenido del repositorio

```
REVERSING-ANALYSIS-.NET-APP/
├── Documentación Técnica Profesional Reversing y Análisis - Reto CyberOps (Randy Nin -- 2025-0660).pdf
└── README.md
```

---

## Documentación técnica

La documentación técnica completa de este laboratorio está disponible en:

**[Documentación Técnica Profesional Reversing y Análisis - Reto CyberOps (Randy Nin -- 2025-0660)](Documentación%20Técnica%20Profesional%20Reversing%20y%20Análisis%20-%20Reto%20CyberOps%20(Randy%20Nin%20--%202025-0660).pdf)**

Incluye análisis estructural del binario .NET, análisis estático de todos los métodos de la clase Form1, flujo completo del programa, análisis de tráfico de red, descripción de todos los encodings encontrados, resolución paso a paso del reto y respuestas a las preguntas del reporte.

---

## Herramientas utilizadas

|Herramienta|Uso|Obtención|
|:--|:--|:--|
|`dnSpy v6.6.0-rc1`|Decompilación y análisis estático del ensamblado .NET|github.com/dnSpyEx/dnSpy|
|`Process Explorer`|Enumeración de puertos TCP activos del proceso|Sysinternals (Microsoft)|
|`CyberChef`|Decodificación de HEX, Base64, Decimal, ROT13, URL|gchq.github.io/CyberChef|
|`ncat`|Conexión al servicio TCP del hint server|nmap.org|

**Requisito del sistema:** .NET 8.0 Runtime instalado en Windows para ejecutar la aplicación.

---

## Comportamiento de la aplicación

Al ejecutarse, `WinFormsApp1.exe` presenta un formulario de contraseña y realiza tres acciones en segundo plano sin notificar al usuario:

1. Inicia un timer de 1 segundo que envía un ping ICMP a `8.8.8.8` para verificar conectividad. Si el ping falla, la aplicación muestra un error y se cierra.
2. Abre un servidor TCP en `127.0.0.1` en un puerto ephemeral asignado dinámicamente por el OS (puerto `0`), que entrega pistas del reto a quien se conecte.
3. Al presionar Enter, realiza una petición HTTP GET a un archivo en GitHub con User-Agent `CyberOps-Lab` para obtener la contraseña en formato hexadecimal y compararla con el input del usuario, ambos en Base64.

---

## Cómo funciona la validación

El programa nunca compara la contraseña en texto claro. El flujo completo es:

```
ProcessUserInput():
  1. input_b64 = Base64(textBox1.Text)
  2. remote_hex = GET https://raw.githubusercontent.com/.../rons_sr.txt
  3. remote_text = HexToString(remote_hex)
  4. remote_b64 = Base64(remote_text)
  5. input_b64 == remote_b64 ? "Contraseña correcta" : "Contraseña incorrecta"
```

La URL de GitHub se construye dinámicamente en `ConstructString()` ensamblando el hash del commit desde 10 fragmentos, cada uno codificado con un método diferente, sin que la URL aparezca completa en ningún punto del binario.

---

## Encodings identificados

|Encoding|Dónde aparece|
|:--|:--|
|Decimal a UTF-8|Todas las strings del programa (`ByteArrayToString`): mensajes, URLs, User-Agent|
|HEX|Contraseña en `rons_sr.txt` (GitHub) y segmentos 1, 6 y 10 de `ConstructString`|
|Base64|Texto del hint message en `PrintHintMessage` y segmentos 2 y 7 de `ConstructString`|
|ROT13|Texto del hint message (aplicado sobre el Base64 decodificado)|
|Reverse String|Segmentos 4 y 9 de `ConstructString`|
|URL Encoding (%XX)|Segmento 5 de `ConstructString`|

---

## Construcción de la URL (ConstructString)

El hash del commit `0c46000d54d963148956df879eb518d384c5cf64` se ensambla en 10 partes:

|Segmento|Método|Input|Output|
|:-:|:--|:--|:--|
|1|HexToString|`"30633436"`|`0c46`|
|2|B64ByteToString|`"MDAwZA=="`|`000d`|
|3|DecimalToString|`"53-52-100-57"`|`54d9`|
|4|ReverseString|`"4136"`|`6314`|
|5|UrlDecode|`"%38%39%35%36"`|`8956`|
|6|HexToString|`"64663837"`|`df87`|
|7|B64ByteToString|`"OWViNQ=="`|`9eb5`|
|8|DecimalToString|`"49-56-100-51"`|`18d3`|
|9|ReverseString|`"5c48"`|`84c5`|
|10|HexToString|`"63663634"`|`cf64`|

URL final: `https://raw.githubusercontent.com/JonathanERC/ScriptsPOC/0c46000d54d963148956df879eb518d384c5cf64/rons_sr.txt`

---

## Respuestas al reporte

**¿Qué puertos utiliza el programa?** Un único puerto TCP en `127.0.0.1` asignado dinámicamente por el OS (puerto `0` al crear el `TcpListener`). En el laboratorio se asignó el puerto `50052`. Varía en cada ejecución.

**¿Cómo realiza la prueba de conectividad?** Timer de 1000 ms que ejecuta `new Ping().Send("8.8.8.8")` cada segundo. Si falla, muestra el error y cierra la aplicación.

**¿Qué tráfico genera?** Paquetes ICMP Echo Request a `8.8.8.8` cada segundo, y una petición HTTP GET a GitHub con User-Agent `CyberOps-Lab` al presionar Enter.

**¿Cómo se conectó al servicio de pistas?** Se identificó el puerto en Process Explorer (TCP/IP tab del proceso), luego se ejecutó `.\ncat.exe 127.0.0.1 50052` desde PowerShell.

---

## Video demostrativo

**Enlace:** https://www.youtube.com/watch?v=qKkxR0cooWc

---

## Disclaimer

Este análisis fue realizado con fines exclusivamente académicos sobre un binario entregado para análisis en el contexto del curso CyberOps. Las técnicas de reversing documentadas aplican únicamente a software propio o entregado explícitamente para análisis en entornos controlados.

---

_Randy Nin / Matrícula 2025-0660_

---
