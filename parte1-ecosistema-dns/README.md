# Parte 1: El ecosistema DNS (OSINT y Web) [2p]

> El sistema DNS es una estructura jerárquica a nivel mundial. Para administrar redes, primero debemos entender quién gestiona cada parte del pastel.

---

## 1. Investigación de Jerarquía

### 1.1. ¿Qué organismo internacional coordina y asigna los parámetros a nivel global del sistema de nombres de dominio e IPs?

**Respuesta:**

> ICANN (Internet Corporation for Assigned Names and Numbers) es la organización internacional sin ánimo de lucro que coordina a nivel global el sistema de nombres de dominio e IPs. Es responsable de la operación estable y segura de la red de identificadores únicos de Internet.
>
> Dentro de ICANN, la **IANA (Internet Assigned Numbers Authority)** es el departamento operativo encargado de gestionar directamente los recursos técnicos: raíz del DNS, espacios de direcciones IP, y números de protocolo. IANA funciona bajo el contrato con ICANN.
>
> **Jerarquía de autoridad:**
> - **ICANN**: coordinación global, política, supervisión
> - **IANA**: departamento operativo dentro de ICANN que gestiona la raíz del DNS
> - **Registros raíz (Root Servers)**: 13 servidores raíz repartidos mundialmente (a.root-servers.org a m.root-servers.org)

**Fuente:** https://www.icann.org/ | https://www.iana.org/

---

### 1.2. ¿Qué empresa u organismo gestiona (Registry) cada uno de los siguientes dominios de nivel superior (TLD)?

| TLD | Organismo Registry | Descripción |
|---|---|---|
| **.es** | **Red.es** (NIC.es / dominios.es) | Entidad pública empresarial del Ministerio de Asuntos Económicos y Transformación Digital de España. Gestiona el ccTLD de España. |
| **.cat** | **Fundació puntCAT** | Fundación privada sin ánimo de lucro con sede en Barcelona, creada para gestionar el dominio .cat, dedicado a la lengua y cultura catalana. |
| **.edu** | **EDUCAUSE** | Asociación sin ánimo de lucro estadounidense que gestiona el TLD .edu, reservado para instituciones educativas postsecundarias acreditadas. |
| **.ifp.es** | Subdominio gestionado por **Red.es** como registry del ccTLD .es, delegado a la organización propietaria (en este caso IFP - Institut de Formació Professional). | El dominio .ifp.es es un subdominio de segundo nivel bajo .es. El registry sigue siendo Red.es, pero la gestión del subdominio la realiza la entidad registrante. |

**Fuentes:**
- .es: https://www.dominios.es/ | https://www.red.es/
- .cat: https://www.puntcat.cat/
- .edu: https://www.educause.edu/

---

## 2. Herramientas OSINT (Whois y DNS Lookup)

> Utiliza herramientas online (como Dominios.es, whois.com, nslookup.io) para responder a lo siguiente.

### 2.1. ¿Qué información te brinda una consulta Whois sobre un dominio?

**Respuesta:**

> Una consulta Whois proporciona información pública sobre el registro de un dominio, incluyendo:
>
> - **Datos del registrante:** Nombre, organización, dirección, email, teléfono (pueden estar protegidos por privacidad/WHOIS Privacy)
> - **Datos del registrador:** Nombre del registrador (empresa con la que se compró el dominio, ej. GoDaddy, Namecheap)
> - **Fechas:** Fecha de creación (Creation Date), fecha de última actualización (Updated Date), fecha de expiración (Registry Expiry Date)
> - **Servidores de nombres (Name Servers):** Los servidores DNS autoritativos del dominio
> - **Estado del dominio (Domain Status):** Códigos EPP como clientHold, serverTransferProhibited, etc.
> - **DNSSEC:** Si el dominio tiene DNSSEC activado

**Herramientas recomendadas:**
- https://who.is/
- https://www.whois.com/
- https://www.dominios.es/

**Captura:** `capturas/parte1/whois-consulta.png`

---

### 2.2. Define brevemente la diferencia entre el Registry de la base de datos y el Registrar (Registrador) del dominio

**Respuesta:**

> | Concepto | Definición | Ejemplo |
> |---|---|---|
> | **Registry (Registro)** | Organización que gestiona y mantiene la base de datos central de un TLD concreto. Tiene el control autoritativo sobre todos los dominios de ese TLD. Opera el registro a nivel de la zona raíz. | Verisign para .com, Red.es para .es, Fundació puntCAT para .cat |
> | **Registrar (Registrador)** | Empresa acreditada por el Registry (o por ICANN) que vende dominios al público final. Actúa como intermediario entre el usuario y el Registry. | GoDaddy, Namecheap, Arsys, DonDominio |
>
> **Analogía:** El Registry es como el fabricante de matrículas de coche para todo un país (define quién tiene qué matrícula), y el Registrar es el concesionario donde tú vas a registrar tu coche y obtener la matrícula.

---

### 2.3. Investiga: ¿Qué es DNSSEC y qué problema de seguridad intenta resolver en las resoluciones DNS?

**Respuesta:**

> **DNSSEC (Domain Name System Security Extensions)** es un conjunto de extensiones de seguridad del protocolo DNS que proporciona **autenticación de origen** e **integridad de datos** en las respuestas DNS.
>
> **Problema que resuelve:** El DNS original no tiene mecanismos de seguridad. Un atacante puede realizar un ataque de **DNS Spoofing / Cache Poisoning**, que consiste en inyectar respuestas DNS falsas en la caché de un servidor DNS resolutor, de modo que cuando un usuario intenta acceder a un dominio legítimo, es redirigido a una IP maliciosa controlada por el atacante (por ejemplo, para phishing).
>
> **Cómo funciona DNSSEC:**
> - Utiliza **criptografía de clave pública** (firmas digitales)
> - Cada zona DNS genera un par de claves: **KSK (Key Signing Key)** y **ZSK (Zone Signing Key)**
> - Los registros DNS se firman digitalmente con la ZSK
> - La ZSK se firma con la KSK, que se publica en el registro padre (cadena de confianza o "chain of trust")
> - La clave pública KSK de la raíz se distribuye de forma segura y es el punto de partida de la cadena
> - El resolutor valida las firmas al recibir las respuestas; si la firma no coincide, descarta la respuesta
>
> **Tipo de registros DNSSEC:** DNSKEY, RRSIG, DS, NSEC, NSEC3

**Fuentes:** https://www.icann.org/dnssec | https://datatracker.ietf.org/doc/html/rfc4033

---

## 3. Rendimiento DNS: GRC DNS Benchmark

### 3.1. Descargar y ejecutar GRC DNS Benchmark

> 1. Ve a https://www.grc.com/dns/benchmark.htm
> 2. Descarga la aplicación (DNSBench.exe, ~160KB, no requiere instalación)
> 3. Ejecútala como administrador

### 3.2. Ejecutar el test

> 1. Abre DNSBench
> 2. Ve a la pestaña "Nameservers"
> 3. Pulsa "Run Benchmark" (botón superior izquierdo)
> 4. Espera a que complete el test (~2-3 minutos)
> 5. Ordena por "Speed" (velocidad de respuesta)

### 3.3. Selecciona los 3 servidores más rápidos

> **Argumentación de los servidores seleccionados:**
>
> | Posición | Servidor DNS | IP | Empresa/Proveedor | Argumentación |
> |---|---|---|---|---|
> | 1º | (anotar IP) | (IP) | (empresa) | (breve descripción de la empresa) |
> | 2º | (anotar IP) | (IP) | (empresa) | (breve descripción de la empresa) |
> | 3º | (anotar IP) | (IP) | (empresa) | (breve descripción de la empresa) |
>
> **Nota:** Los resultados dependerán de tu ubicación geográfica (L'Hospitalet de Llobregat, España). Es normal que servidores con presencia en Europa respondan más rápido.

**Captura:** `capturas/parte1/grc-dns-benchmark.png`

---

## Checklist Parte 1

- [x] Jerarquía DNS investigada (ICANN/IANA)
- [x] TLDs identificados (.es, .cat, .edu, .ifp.es)
- [ ] Consulta Whois realizada y capturada
- [ ] Diferencia Registry vs Registrar explicada
- [ ] DNSSEC investigado
- [ ] GRC DNS Benchmark ejecutado y capturado
- [ ] 3 servidores DNS más rápidos anotados
