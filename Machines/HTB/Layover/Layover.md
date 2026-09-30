# HTB — Web Application to Local Privilege Escalation

> **Contexto:** laboratorio de Hack The Box.
> Los detalles sensibles de explotación, credenciales, payloads y valores específicos de la máquina se han omitido debido a que la máquina sigue activa.

## Resumen

El objetivo de este laboratorio fue conseguir acceso inicial a una aplicación web, aprovechar una vulnerabilidad de ejecución remota de comandos en el CMS y utilizar la información obtenida durante la enumeración para avanzar hacia un usuario local con mayores privilegios.

Finalmente, se identificó un servicio local de impresión expuesto únicamente en `localhost` y se aprovechó una vulnerabilidad de escalada de privilegios local en CUPS para obtener privilegios de `root`.

La cadena conceptual fue:

```text
Aplicación web
      ↓
Acceso autenticado
      ↓
RCE en CMS
      ↓
Enumeración del sistema y configuración
      ↓
Obtención de credenciales/secretos reutilizables
      ↓
Acceso como usuario local
      ↓
Enumeración de servicios internos
      ↓
CUPS / IPP en localhost
      ↓
LPE mediante CVE-2026-34990
      ↓
root
```

---

## 1. Reconocimiento inicial

Durante el reconocimiento se identificaron dos puertos un 22 ssh y un RDP 3389. Debido a que la maquina otorga credenciales al principio las usamos para acceder al rdp.


Una vez dentro del RDP iniciamos el escaneo manual de la maquina.

Lo mas interesante fue ver que era un veth y habia dos WLAN que son wireless lans , creadas a proposito.

Debido a ese encuentro lo primero que se me ocurrio fue comprobar si la maquia tenia WIFI y efectivamente vemos que la maquina posee WIFI y nos conectamos.

---

## 2 WIFI
Una vez con el wifi conectado , decidi capturar el trafico de la red wifi para ver si captaba algo y voala. Salieron claves.

Con esas claves nos metes a la web y accedimos al panel de admin con dichas claves y nos encontramos una web basada en un **CMS Craft**

La versión instalada resultó ser relevante para la investigación, ya que existían vulnerabilidades conocidas que afectaban a determinadas versiones del CMS.

En particular, **CVE-2026-44011** afecta a versiones de Craft CMS anteriores a las versiones corregidas indicadas por el fabricante. La vulnerabilidad permite a un usuario autenticado manipular determinados datos de configuración de objetos y alcanzar ejecución de comandos en el servidor.

La vulnerabilidad requiere autenticación en el panel, por lo que el acceso inicial a la aplicación fue un requisito importante de la cadena.

---

## 3. Ejecución remota de comandos

Una vez identificado el vector correspondiente al CMS vulnerable, se consiguió ejecutar comandos en el contexto del usuario **www-data** que ejecutaba la aplicación web.

En lugar de intentar realizar inmediatamente una escalada de privilegios, se realizó una fase de enumeración para comprender:

* Usuario actual.
* Sistema operativo.
* Interfaces de red.
* Servicios ejecutándose.
* Archivos de configuración.
* Variables de entorno.
* Base de datos.
* Credenciales y secretos almacenados por la aplicación.

---

## 4. Enumeración de configuración y secretos

Durante la revisión del proyecto se encontró el fichero de configuración de entorno de la aplicación.

Este tipo de fichero puede contener información especialmente sensible, como:

* Claves criptográficas de la aplicación.
* Credenciales de base de datos.
* Configuración de servicios externos.
* Parámetros de autenticación.

También se identificaron valores almacenados de forma cifrada dentro de la configuración de la aplicación.

Dado que la configuracion no se podia romper de ninguna forma ya que no era un hash o un cifrado normal, se investigo en la aplicación como es que se almacenaron dichos datos en la BBDD a traves del panel
**modules/**

La clave conceptual fue:

```text
Configuración de aplicación
        +
Clave de seguridad del framework
        ↓
Mecanismo nativo de descifrado
        ↓
Secreto recuperado
```

Esto permitió recuperar credenciales utilizadas por otro servicio interno.

---

## 4. Movimiento lateral / cambio de contexto

Las credenciales recuperadas permitieron autenticarse como un usuario diferente del usuario que ejecutaba inicialmente la aplicación web.

Este cambio de contexto fue importante porque proporcionó una shell local con un conjunto diferente de permisos y acceso a recursos del sistema.

A partir de este punto se dejó de investigar principalmente la aplicación web y se comenzó una enumeración orientada a una **Escalada de privilegios**.

---

## 5. Enumeración local

Se revisaron diferentes mecanismos habituales de escalada:

* Identidad y grupos del usuario.
* Permisos mediante `sudo`.
* Binarios SUID/SGID.
* Linux capabilities.
* Servicios activos.
* Procesos.
* Puertos locales.
* Archivos modificables.
* Configuración de servicios.

La enumeración de sockets locales resultó especialmente interesante.

Además de los servicios accesibles externamente, existían servicios escuchando únicamente en `localhost` como es el caso de **IPP / CUPS**.

Esto es importante porque un servicio en:

```text
127.0.0.1:<puerto>
```

puede ser invisible desde el atacante externo pero perfectamente accesible desde una shell obtenida dentro de la máquina.

---

## 6. Identificación del servicio interno

Uno de los puertos locales correspondía al protocolo **IPP (Internet Printing Protocol)**.

La enumeración de servicios confirmó que el sistema ejecutaba **CUPS**, el sistema de impresión utilizado habitualmente en Linux.

La relación conceptual fue:

```text
Puerto local
    ↓
IPP
    ↓
CUPS
```

---

## 7. Escalada de privilegios mediante CUPS

La versión/configuración de CUPS era compatible con la investigación de **CVE-2026-34990**.

Esta vulnerabilidad afecta a determinadas versiones de OpenPrinting CUPS y permite a un usuario local sin privilegios abusar de mecanismos de autenticación local y de la gestión de colas de impresión para conseguir una escritura arbitraria de archivos con privilegios elevados. INCIBE describe el problema como una vulnerabilidad de autenticación incorrecta y documenta el impacto de escalada local de privilegios.

A nivel conceptual, el ataque funciona mediante:

1. Interacción con el servicio CUPS local.
2. Obtención/abuso de un mecanismo de autenticación local.
3. Operaciones administrativas sobre una cola de impresión.
4. Abuso de una cola configurada para escribir en un recurso controlado.
5. Escritura de un archivo con privilegios elevados.
6. Utilización de ese archivo para conseguir ejecución con privilegios de `root`.

La vulnerabilidad está documentada públicamente y afecta a CUPS hasta determinadas versiones anteriores a la corrección correspondiente.

---

### Cadena final

```text
Initial Enumeration
        ↓
WIFI CONNECTION
        ↓
Authenticated Web Access
        ↓
Craft CMS RCE
        ↓
Application Enumeration
        ↓
Configuration / Secret Discovery
        ↓
Credential Recovery
        ↓
Local User Access
        ↓
Local Service Enumeration
        ↓
CUPS / IPP
        ↓
CVE-2026-34990
        ↓
Local Privilege Escalation
        ↓
     **ROOT**
```

---

## Referencias

* Craft CMS — CVE-2026-44011 / GHSA-qrgm-p9w5-rrfw.
* INCIBE-CERT — CVE-2026-34990 (CUPS).
* Red Hat Bugzilla — análisis técnico de CVE-2026-34990.

> **Nota:** Los comandos de explotación, credenciales, IPs, payloads y otros elementos específicos de la máquina se han omitido para mantener el write-up como documentación técnica y no como una guía directa para comprometer el objetivo del laboratorio.

