# Parte 3: Administración - Troubleshooting con DIG y CLI [3p]

> En un entorno profesional, especialmente servidores Linux, la herramienta nslookup se considera obsoleta, siendo dig - Domain Information Groper - el estándar de la industria.

**Requisitos previos:**
> Si estás en Windows, necesitas tener WSL (Windows Subsystem for Linux) instalado, o usar una máquina virtual Linux. Si estás en Linux nativo, `dig` viene incluido en el paquete `dnsutils` (Debian/Ubuntu) o `bind-utils` (CentOS/RHEL).
>
> Instalación si no lo tienes:
> ```bash
> sudo apt install dnsutils   # Debian/Ubuntu
> ```

---

## 1. Consultas específicas de registros (Usa dig en Linux o WSL)

> Documenta con capturas de pantalla y explica el resultado de ejecutar las siguientes consultas sobre el dominio **aliexpress.com**.

### 1.1. Registro A

```bash
dig aliexpress.com
```

**Análisis de la ANSWER SECTION:**
> La salida de `dig` se divide en varias secciones:
>
> - **HEADER:** Muestra la versión de dig, el opcode (QUERY), el status (NOERROR), flags (qr rd ra) y el número de respuestas.
> - **QUESTION SECTION:** Contiene la consulta original: `aliexpress.com. IN A`
> - **ANSWER SECTION:** Aquí aparecen los registros A resueltos. Cada línea muestra:
>   - Nombre del dominio (aliexpress.com.)
>   - TTL (ej. 300)
>   - Clase (IN)
>   - Tipo (A)
>   - Dirección IP (ej. 47.246.31.83)
> - **AUTHORITY SECTION:** (puede no aparecer si la respuesta viene de caché)
> - **ADDITIONAL SECTION:** (puede no aparecer)
> - **Footer:** Tiempo de consulta, servidor usado, tamaño del mensaje.
>
> El registro A devuelve la dirección IPv4 asociada al dominio. Aliexpress.com puede devolver múltiples IPs (round-robin) para distribuir la carga entre diferentes servidores.

**Captura:** `capturas/parte3/dig-a.png`

---

### 1.2. Formato Short

```bash
dig +short aliexpress.com
```

**¿Por qué es útil este formato en scripts de Bash?**
> El formato `+short` devuelve únicamente la IP (o IPs) sin cabeceras, metadatos ni texto adicional. La salida es una IP por línea, sin nada más.
>
> Esto es ideal para scripts de Bash porque la salida se puede:
> - Asignar directamente a una variable: `IP=$(dig +short aliexpress.com)`
> - Usar en pipes: `dig +short aliexpress.com | head -1`
> - Usar en condiciones: `if ping -c 1 $(dig +short google.com); then ...`
> - Iterar con bucles: `for ip in $(dig +short aliexpress.com); do ...`
>
> Con `dig` normal, habría que parsear la salida con grep/awk para extraer la IP, lo cual es frágil y propenso a errores. Con `+short`, la salida ya está limpia y lista para usar.

**Captura:** `capturas/parte3/dig-short.png`

---

### 1.3. Registro MX

```bash
dig MX aliexpress.com
```

**Campo de Prioridad/Preference:**
> El registro MX (Mail Exchange) indica qué servidores de correo reciben los emails del dominio. La respuesta incluye:
>
> - **Nombre del dominio** (aliexpress.com.)
> - **TTL** (ej. 3600)
> - **Clase** (IN)
> - **Tipo** (MX)
> - **Prioridad/Preference:** Un número entero. Cuanto **menor** sea el número, **mayor** es la prioridad. El servidor de correo con prioridad más baja se intenta primero.
> - **Servidor de correo:** El FQDN del servidor (ej. mail.aliexpress.com.)
>
> Por ejemplo, si vemos:
> ```
> aliexpress.com.  3600  IN  MX  10  mx1.aliexpress.com.
> aliexpress.com.  3600  IN  MX  20  mx2.aliexpress.com.
> ```
> mx1 tiene prioridad 10 (mayor prioridad, se intenta primero) y mx2 tiene prioridad 20 (se intenta si mx1 no responde).

**Captura:** `capturas/parte3/dig-mx.png`

---

### 1.4. Registro NS

```bash
dig NS aliexpress.com
```

**Servidores que tienen la autoridad sobre las zonas de este dominio:**
> El registro NS (Name Server) lista los servidores de nombres autoritativos para el dominio. Estos son los servidores que tienen la respuesta "oficial" y definitiva para las consultas DNS de aliexpress.com.
>
> La respuesta mostrará algo como:
> ```
> aliexpress.com.  86400  IN  NS  ns1.aliexpress.com.
> aliexpress.com.  86400  IN  NS  ns2.aliexpress.com.
> ```
>
> Estos servidores son los que tienen la autoridad sobre la zona del dominio. Cualquier cambio en los registros DNS de aliexpress.com debe hacerse en estos servidores autoritativos. El TTL de los registros NS suele ser alto (86400 segundos = 24 horas) porque los servidores autoritativos raramente cambian.

**Captura:** `capturas/parte3/dig-ns.png`

---

## 2. Autoridad y Caché (TTL)

### 2.1. ¿Qué diferencia existe entre un registro SOA (Start of Authority) y un registro NS (Name Server)?

> | Aspecto | Registro SOA (Start of Authority) | Registro NS (Name Server) |
> |---|---|---|
> | **Función principal** | Define el inicio de una zona DNS y contiene metadatos de gestión de la zona | Lista los servidores de nombres autoritativos para una zona |
> | **Información que contiene** | Servidor primario, email del administrador, número de serie (serial), tiempos de refresco (refresh), reintento (retry), expiración (expire) y TTL mínimo (minimum) | Solo nombres de los servidores autoritativos |
> | **Cantidad por zona** | Exactamente un registro SOA por zona | Puede haber múltiples registros NS por zona (mínimo 2 recomendado) |
> | **Uso** | Transferencias de zona entre servidor primario y secundarios; sincronización | Delegación de autoridad; los resolutores consultan estos servidores para obtener respuestas autoritativas |
> | **Ejemplo de uso** | Un servidor secundario usa el SOA para saber si tiene que hacer una transferencia de zona (comparando el serial) | Un resolver pregunta a los NS para resolver un registro A del dominio |
>
> **En resumen:** El SOA define **cómo se gestiona** la zona (metadatos de sincronización), mientras que el NS define **quién tiene autoridad** sobre la zona (qué servidores responderán).

**Captura del SOA (comando adicional recomendado):**
```bash
dig SOA aliexpress.com
```
**Captura:** `capturas/parte3/dig-soa.png`

---

### 2.2. Comportamiento del TTL

**Ejecuta estos comandos con 5 segundos de diferencia:**
```bash
dig aliexpress.com
# esperar 5 segundos
dig aliexpress.com
```

> **¿Qué ha pasado con el valor numérico del TTL?**
>
> Al ejecutar la primera consulta, el TTL del registro A puede ser, por ejemplo, 300 segundos. Al ejecutar la segunda consulta 5 segundos después, el TTL debería haber disminuido a 295 segundos (300 - 5).
>
> Esto demuestra que **la segunda respuesta viene de la caché local**, no del servidor autoritativo. El resolutor DNS local (o el cliente) ha cacheado la respuesta de la primera consulta y, en la segunda, devuelve la versión cacheada con el TTL decrementado.
>
> Si la segunda respuesta viniera del servidor autoritativo, el TTL volvería a ser 300 (el valor original) porque el servidor autoritativo siempre devuelve el TTL completo configurado en la zona.
>
> **Conclusión:** La disminución del TTL demuestra que la respuesta proviene de la **caché** y no del servidor autoritativo. Esto es importante porque:
> - Las respuestas cacheadas son más rápidas (no hay que ir al servidor autoritativo)
> - Pero pueden estar desactualizadas si el dominio cambió de IP recientemente
> - El TTL es el mecanismo que asegura que las entradas cacheadas eventualmente expiren y se vuelvan a consultar

**Captura:** `capturas/parte3/ttl-comparativa.png` (captura con ambas consultas mostrando el TTL diferente)

---

## 3. Trazabilidad Completa (Trace)

```bash
dig +trace aliexpress.com
```

**Explicación paso a paso:**

> El comando `dig +trace` simula el proceso completo de resolución DNS recursiva, mostrando cada nivel de la jerarquía DNS:
>
> **Paso 1: Servidores Raíz (Root Servers)**
> ```
> .           518400  IN  NS  a.root-servers.net.
> .           518400  IN  NS  b.root-servers.net.
> ...
> ```
> El primer paso contacta con los **servidores raíz** (los 13 servidores raíz: a.root-servers.net a m.root-servers.net). Estos servidores son el punto de partida de toda resolución DNS. Su función es redirigir la consulta al TLD correspondiente.
>
> **Paso 2: Servidores del TLD (.com)**
> ```
> com.        172800  IN  NS  a.gtld-servers.net.
> com.        172800  IN  NS  b.gtld-servers.net.
> ...
> ```
> Los servidores raíz responden con la lista de servidores autoritativos para el TLD **.com** (gestionados por Verisign). Estos servidores conocen todos los dominios registrados bajo .com.
>
> **Paso 3: Servidores autoritativos del dominio**
> ```
> aliexpress.com.  172800  IN  NS  ns1.aliexpress.com.
> aliexpress.com.  172800  IN  NS  ns2.aliexpress.com.
> ...
> ```
> Los servidores del TLD .com responden con los servidores autoritativos de **aliexpress.com**. Estos son los servidores que tienen la zona del dominio y las respuestas definitivas.
>
> **Paso 4: Resolución final (Registro A)**
> ```
> aliexpress.com.  300  IN  A  47.246.31.83
> ```
> Finalmente, se consulta a los servidores autoritativos de aliexpress.com, que responden con el registro A (la dirección IP del dominio).
>
> **Resumen del recorrido:**
> 1. Mi PC → Root Servers (.) → "No sé la IP de aliexpress.com, pero pregúntale a los servidores de .com"
> 2. Mi PC → Servidores .com → "No sé la IP exacta, pero pregúntale a los servidores autoritativos de aliexpress.com"
> 3. Mi PC → Servidores autoritativos de aliexpress.com → "La IP de aliexpress.com es 47.246.31.83"

**Captura:** `capturas/parte3/dig-trace.png`

---

## Resumen de comandos a ejecutar

| # | Comando | Captura | Descripción |
|---|---|---|---|
| 1 | `dig aliexpress.com` | `capturas/parte3/dig-a.png` | Registro A |
| 2 | `dig +short aliexpress.com` | `capturas/parte3/dig-short.png` | Formato corto |
| 3 | `dig MX aliexpress.com` | `capturas/parte3/dig-mx.png` | Registros MX |
| 4 | `dig NS aliexpress.com` | `capturas/parte3/dig-ns.png` | Servidores autoritativos |
| 5 | `dig SOA aliexpress.com` | `capturas/parte3/dig-soa.png` | Registro SOA |
| 6 | `dig aliexpress.com` (x2, 5s) | `capturas/parte3/ttl-comparativa.png` | Comportamiento TTL |
| 7 | `dig +trace aliexpress.com` | `capturas/parte3/dig-trace.png` | Trazabilidad |

---

## Checklist Parte 3

- [ ] dig A ejecutado y capturado
- [ ] dig +short ejecutado y explicado
- [ ] dig MX ejecutado con prioridades
- [ ] dig NS ejecutado con servidores autoritativos
- [ ] dig SOA ejecutado (extra)
- [ ] Diferencia SOA vs NS explicada
- [ ] Comportamiento TTL documentado (2 consultas)
- [ ] dig +trace ejecutado y explicado paso a paso
