# Parte 3: Administración - Troubleshooting con DIG y CLI [3p]

> En un entorno profesional, especialmente servidores Linux, la herramienta nslookup se considera obsoleta, siendo dig - Domain Information Groper - el estándar de la industria.

---

## 1. Consultas específicas de registros

> Documenta con capturas de pantalla y explica el resultado de ejecutar las siguientes consultas sobre el dominio **aliexpress.com**.

### 1.1. Registro A

```bash
dig aliexpress.com
```

**Análisis de la ANSWER SECTION:**
> (Documentar el resultado, explicar qué IP(s) devuelve y qué significa el registro A)

**Captura:** `capturas/parte3/dig-a.png`

---

### 1.2. Formato Short

```bash
dig +short aliexpress.com
```

**¿Por qué es útil este formato en scripts de Bash?**
> El formato `+short` devuelve únicamente la IP sin cabeceras ni metadatos. Esto es ideal para scripts de Bash porque la salida se puede asignar directamente a una variable o usar en pipes, sin necesidad de parsear texto:
> ```bash
> IP=$(dig +short aliexpress.com)
> ping -c 3 $IP
> ```

**Captura:** `capturas/parte3/dig-short.png`

---

### 1.3. Registro MX

```bash
dig MX aliexpress.com
```

**Campo de Prioridad/Preference:**
> (Documentar los servidores de correo y sus prioridades)

**Captura:** `capturas/parte3/dig-mx.png`

---

### 1.4. Registro NS

```bash
dig NS aliexpress.com
```

**Servidores autoritativos:**
> (Documentar los servidores de nombres con autoridad sobre la zona)

**Captura:** `capturas/parte3/dig-ns.png`

---

## 2. Autoridad y Caché (TTL)

### 2.1. Diferencia entre SOA y NS

> | Registro | Función |
> |---|---|
> | **SOA (Start of Authority)** | Define el inicio de una zona DNS. Contiene metadatos: servidor primario, email del administrador, número de serie, tiempos de refresco/retry/expiración y TTL mínimo. Indica quién tiene la autoridad sobre la zona. |
> | **NS (Name Server)** | Lista los servidores de nombres autoritativos para una zona. No contiene metadatos de gestión, solo indica qué servidores responden con autoridad para ese dominio. |

---

### 2.2. Comportamiento del TTL

```bash
dig aliexpress.com
# esperar 5 segundos
dig aliexpress.com
```

> (Documentar el valor TTL en ambas consultas y explicar si disminuyó o se mantuvo)

**Captura:** `capturas/parte3/ttl-comparativa.png`

---

## 3. Trazabilidad Completa (Trace)

```bash
dig +trace aliexpress.com
```

> Explicar paso a paso:
> 1. Contacto con Root Servers (.)
> 2. Root responde con los servidores del TLD (.com)
> 3. Servidores .com responden con los servidores autoritativos del dominio
> 4. Servidor autoritativo responde con el registro A

**Captura:** `capturas/parte3/dig-trace.png`

---

## Checklist Parte 3

- [ ] dig A documentado y analizado
- [ ] dig +short explicado (scripts Bash)
- [ ] dig MX con prioridades identificadas
- [ ] dig NS con servidores autoritativos
- [ ] Diferencia SOA vs NS explicada
- [ ] Comportamiento TTL documentado
- [ ] dig +trace explicado paso a paso
