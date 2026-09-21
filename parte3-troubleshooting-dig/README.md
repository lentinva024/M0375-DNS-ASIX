# Parte 3: Administración - Troubleshooting con DIG y CLI [3p]

> En un entorno profesional, especialmente servidores Linux, la herramienta nslookup se considera obsoleta, siendo dig - Domain Information Groper - el estándar de la industria.

Requisitos: Si estás en Windows, necesitas WSL o una máquina virtual Linux. En Linux, `dig` viene en el paquete `dnsutils` (Debian/Ubuntu).

```bash
sudo apt install dnsutils   # si no lo tienes instalado
```

---

## 1. Consultas específicas de registros (Usa dig en Linux o WSL)

### 1.1. Registro A

```bash
dig aliexpress.com
```

La salida de dig se divide en secciones:

- **HEADER:** Versión de dig, opcode (QUERY), status (NOERROR), flags (qr rd ra) y número de respuestas.
- **QUESTION SECTION:** La consulta original: `aliexpress.com. IN A`
- **ANSWER SECTION:** Los registros A resueltos. Cada línea muestra el nombre del dominio, TTL, clase (IN), tipo (A) y la dirección IP.
- **Footer:** Tiempo de consulta, servidor usado y tamaño del mensaje.

El registro A devuelve la dirección IPv4 asociada al dominio. Aliexpress.com puede devolver múltiples IPs (round-robin) para distribuir la carga entre servidores.

**Captura:** `capturas/parte3/dig-a.png`

---

### 1.2. Short

```bash
dig +short aliexpress.com
```

El formato `+short` devuelve únicamente la IP (o IPs) sin cabeceras ni metadatos. La salida es una IP por línea.

Esto es útil en scripts de Bash porque la salida se puede:
- Asignar a una variable: `IP=$(dig +short aliexpress.com)`
- Usar en pipes: `dig +short aliexpress.com | head -1`
- Usar en condiciones: `if ping -c 1 $(dig +short google.com); then ...`

Con `dig` normal habría que parsear la salida con grep/awk para extraer la IP. Con `+short`, la salida ya está limpia.

**Captura:** `capturas/parte3/dig-short.png`

---

### 1.3. MX

```bash
dig MX aliexpress.com
```

El registro MX (Mail Exchange) indica qué servidores de correo reciben los emails del dominio. La respuesta incluye:

- Nombre del dominio
- TTL
- Clase (IN)
- Tipo (MX)
- Prioridad/Preference: un número entero. Cuanto menor sea el número, mayor es la prioridad. El servidor con prioridad más baja se intenta primero.
- Servidor de correo (FQDN)

Ejemplo:
```
aliexpress.com.  3600  IN  MX  10  mx1.aliexpress.com.
aliexpress.com.  3600  IN  MX  20  mx2.aliexpress.com.
```
mx1 tiene prioridad 10 (mayor prioridad, se intenta primero) y mx2 tiene prioridad 20 (se intenta si mx1 no responde).

**Captura:** `capturas/parte3/dig-mx.png`

---

### 1.4. NS

```bash
dig NS aliexpress.com
```

El registro NS (Name Server) lista los servidores de nombres autoritativos para el dominio. Estos son los servidores que tienen la respuesta oficial y definitiva para las consultas DNS de aliexpress.com.

Ejemplo:
```
aliexpress.com.  86400  IN  NS  ns1.aliexpress.com.
aliexpress.com.  86400  IN  NS  ns2.aliexpress.com.
```

El TTL de los registros NS suele ser alto (86400 segundos = 24 horas) porque los servidores autoritativos raramente cambian.

**Captura:** `capturas/parte3/dig-ns.png`

---

## 2. Autoridad y Caché (TTL)

### 2.1. ¿Qué diferencia existe entre un registro SOA (Start of Authority) y un registro NS (Name Server)?

| Aspecto | Registro SOA | Registro NS |
|---|---|---|
| Función | Define el inicio de una zona DNS y contiene metadatos de gestión | Lista los servidores autoritativos para una zona |
| Información | Servidor primario, email del administrador, número de serie, tiempos de refresco/reintento/expiración y TTL mínimo | Solo nombres de los servidores autoritativos |
| Cantidad | Un registro SOA por zona | Múltiples registros NS por zona (mínimo 2 recomendado) |
| Uso | Transferencias de zona entre primario y secundarios | Delegación de autoridad; los resolutores consultan estos servidores |

El SOA define cómo se gestiona la zona (metadatos de sincronización). El NS define quién tiene autoridad sobre la zona (qué servidores responden).

**Captura del SOA:**
```bash
dig SOA aliexpress.com
```
**Captura:** `capturas/parte3/dig-soa.png`

---

### 2.2. Comportamiento del TTL

Ejecutar estos comandos con 5 segundos de diferencia:
```bash
dig aliexpress.com
# esperar 5 segundos
dig aliexpress.com
```

Al ejecutar la primera consulta, el TTL del registro A puede ser, por ejemplo, 300 segundos. Al ejecutar la segunda consulta 5 segundos después, el TTL debería haber disminuido a 295 segundos (300 - 5).

Esto demuestra que la segunda respuesta viene de la caché local, no del servidor autoritativo. El resolutor DNS local ha cacheado la respuesta de la primera consulta y, en la segunda, devuelve la versión cacheada con el TTL decrementado.

Si la segunda respuesta viniera del servidor autoritativo, el TTL volvería a ser 300 (el valor original) porque el servidor autoritativo siempre devuelve el TTL completo configurado en la zona.

La disminución del TTL demuestra que la respuesta proviene de la caché y no del servidor autoritativo. Esto es relevante porque las respuestas cacheadas son más rápidas pero pueden estar desactualizadas si el dominio cambió de IP recientemente.

**Captura:** `capturas/parte3/ttl-comparativa.png` (captura con ambas consultas mostrando el TTL diferente)

---

## 3. Trazabilidad Completa (Trace)

```bash
dig +trace aliexpress.com
```

El comando `dig +trace` simula el proceso completo de resolución DNS recursiva, mostrando cada nivel de la jerarquía DNS:

**Paso 1: Servidores Raíz (Root Servers)**
```
.           518400  IN  NS  a.root-servers.net.
.           518400  IN  NS  b.root-servers.net.
```
Se contacta con los 13 servidores raíz (a.root-servers.net a m.root-servers.net). Son el punto de partida de toda resolución DNS. Su función es redirigir la consulta al TLD correspondiente.

**Paso 2: Servidores del TLD (.com)**
```
com.        172800  IN  NS  a.gtld-servers.net.
com.        172800  IN  NS  b.gtld-servers.net.
```
Los servidores raíz responden con la lista de servidores autoritativos para el TLD .com (gestionados por Verisign). Estos servidores conocen todos los dominios registrados bajo .com.

**Paso 3: Servidores autoritativos del dominio**
```
aliexpress.com.  172800  IN  NS  ns1.aliexpress.com.
aliexpress.com.  172800  IN  NS  ns2.aliexpress.com.
```
Los servidores del TLD .com responden con los servidores autoritativos de aliexpress.com. Son los servidores que tienen la zona del dominio y las respuestas definitivas.

**Paso 4: Resolución final (Registro A)**
```
aliexpress.com.  300  IN  A  47.246.31.83
```
Se consulta a los servidores autoritativos de aliexpress.com, que responden con el registro A (la dirección IP del dominio).

Recorrido completo:
1. Mi PC → Root Servers (.) → "Pregúntale a los servidores de .com"
2. Mi PC → Servidores .com → "Pregúntale a los servidores autoritativos de aliexpress.com"
3. Mi PC → Servidores autoritativos de aliexpress.com → "La IP es 47.246.31.83"

**Captura:** `capturas/parte3/dig-trace.png`

---

## Resumen de comandos a ejecutar

| # | Comando | Captura |
|---|---|---|
| 1 | `dig aliexpress.com` | `capturas/parte3/dig-a.png` |
| 2 | `dig +short aliexpress.com` | `capturas/parte3/dig-short.png` |
| 3 | `dig MX aliexpress.com` | `capturas/parte3/dig-mx.png` |
| 4 | `dig NS aliexpress.com` | `capturas/parte3/dig-ns.png` |
| 5 | `dig SOA aliexpress.com` | `capturas/parte3/dig-soa.png` |
| 6 | `dig aliexpress.com` (x2, 5s de diferencia) | `capturas/parte3/ttl-comparativa.png` |
| 7 | `dig +trace aliexpress.com` | `capturas/parte3/dig-trace.png` |

---

## Checklist Parte 3

- [ ] dig A ejecutado y capturado
- [ ] dig +short ejecutado y explicado
- [ ] dig MX ejecutado con prioridades
- [ ] dig NS ejecutado con servidores autoritativos
- [ ] dig SOA ejecutado
- [ ] Diferencia SOA vs NS explicada
- [ ] Comportamiento TTL documentado (2 consultas)
- [ ] dig +trace ejecutado y explicado paso a paso
