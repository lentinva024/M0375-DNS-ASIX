# Parte 2: Configuración y Caché [2p]

> Todo sistema operativo guarda las resoluciones DNS para no saturar la red.

---

## 1. Cambio de servidores DNS

### 1.1. ¿Cómo puedes ver mediante consola (CLI) qué servidores DNS tienes asignados actualmente en Windows y en Linux?

**Windows (PowerShell o CMD):**
```powershell
ipconfig /all
```
> Este comando muestra toda la configuración de red. Busca la sección de tu adaptador de red activo y localiza la línea "Servidores DNS" (o "DNS servers" si está en inglés). Ahí aparecen las IPs de los DNS primario y secundario que tienes configurados actualmente.

**Linux:**
```bash
resolvectl status
```
> En distribuciones modernas con systemd-resolved, este comando muestra el estado de los resolutores DNS. La sección "Current DNS Server" indica el servidor DNS en uso. También puedes ver todos los configurados en "DNS Servers".
>
> Alternativa en Linux:
> ```bash
> cat /etc/resolv.conf
> ```
> Este archivo contiene líneas `nameserver` con las IPs de los servidores DNS configurados.

**Captura:** `capturas/parte2/ipconfig-all.png`

---

### 1.2. Cambia la configuración de red de tu equipo principal poniendo como DNS primario y secundario los que obtuviste en el Benchmark de la Fase 1

> **DNS a configurar (obtenidos del benchmark con DNS Jumper):**
> - DNS primario: **9.9.9.9** (Quad9 Security)
> - DNS secundario: **149.112.112.112** (Quad9 Security)

**Pasos en Windows:**
1. Abre **Configuración** → **Red e Internet** → **Centro de redes y recursos compartidos**
2. Clic en **"Cambiar configuración del adaptador"** (panel izquierdo)
3. Clic derecho sobre tu adaptador de red principal (Wi-Fi o Ethernet) → **Propiedades**
4. Selecciona **"Protocolo de Internet versión 4 (TCP/IPv4)"** → **Propiedades**
5. Selecciona **"Usar las siguientes direcciones de servidor DNS:"**
6. DNS preferido: `9.9.9.9`
7. DNS alternativo: `149.112.112.112`
8. Aceptar → Aceptar → Cerrar

**Verificación del cambio:**
```powershell
ipconfig /all
```
> Comprueba que ahora aparecen 9.9.9.9 y 149.112.112.112 como servidores DNS.

**Captura:** `capturas/parte2/cambio-dns.png` (captura de la ventana de propiedades de IPv4 con los DNS cambiados)
**Captura:** `capturas/parte2/ipconfig-verificacion.png` (captura del ipconfig /all mostrando los nuevos DNS)

---

### 1.3. ¿En qué menú de tu dispositivo móvil (Android/iOS) podrías forzar el uso de unos DNS específicos para tu conexión Wi-Fi?

**Android:**
> 1. Ve a **Ajustes** → **Red e Internet** (o **Conexiones**)
> 2. Toca **Wi-Fi**
> 3. Toca la red Wi-Fi conectada (o el icono de engranaje)
> 4. Ve a **Avanzado** → **Ajustes de IP**
> 5. Cambia de "DHCP" a **"Estático"**
> 6. Rellena los campos DNS 1 y DNS 2 con las IPs deseadas (9.9.9.9 y 149.112.112.112)
> 7. Guarda

> **Nota:** En Android 9+ también puedes configurar DNS privados de forma global en: Ajustes → Red e Internet → DNS privado. Puedes escribir `dns.quad9.net` para usar Quad9.

**iOS (iPhone/iPad):**
> 1. Ve a **Ajustes** → **Wi-Fi**
> 2. Toca el botón **"i"** azul junto a tu red conectada
> 3. Desplázate hasta **"Configurar DNS"**
> 4. Cambia de "Automática" a **"Manual"**
> 5. Elimina los servidores existentes (si los hay)
> 6. Toca **"Añadir servidor"** e introduce `9.9.9.9`
> 7. Toca **"Añadir servidor"** de nuevo e introduce `149.112.112.112`
> 8. Toca **"Guardar"**

**Captura:** `capturas/parte2/dns-movil.png` (captura del móvil mostrando la configuración DNS)

---

## 2. Gestión de la caché DNS (ipconfig / resolvectl)

### 2.1. Muestra una captura de pantalla de algunas direcciones almacenadas en la caché de tu equipo

**Windows:**
```powershell
ipconfig /displaydns
```
> Este comando muestra el contenido de la caché DNS del cliente de Windows. Verás entradas con:
> - **Nombre de registro:** el dominio resuelto (ej. google.com)
> - **Tipo:** A, AAAA, CNAME, etc.
> - **Tiempo de vida (TTL):** los segundos restantes antes de que la entrada expire
> - **Datos:** la IP asociada
>
> Navega por la lista con la barra de desplazamiento hasta encontrar varias entradas. La caché contendrá resoluciones recientes de tu navegación.

**Captura:** `capturas/parte2/cache-dns.png`

---

### 2.2. Vacía la caché de tu equipo y explica para qué es útil esta acción

**Windows:**
```powershell
ipconfig /flushdns
```
> Este comando vacía y reinicia la caché DNS. La respuesta esperada es:
> ```
> Configuración IP de Windows
> Se vació correctamente la caché de resolución DNS.
> ```

**Captura:** `capturas/parte2/flush-dns.png`

> **¿Para qué es útil esta acción en el día a día de un administrador de sistemas?**
>
> Vaciar la caché DNS es una herramienta esencial para un administrador de sistemas por las siguientes razones:
>
> 1. **Aplicar cambios de IP inmediatamente:** Cuando se migra un servicio a un nuevo servidor (cambio de IP), los equipos que ya consultaron el dominio tendrán la IP antigua en caché hasta que expire el TTL. Vaciar la caché fuerza la resolución desde el servidor DNS, obteniendo la IP nueva al instante.
>
> 2. **Troubleshooting de conectividad:** Si un usuario reporta que no puede acceder a un sitio web pero otros sí pueden, puede deberse a una entrada DNS corrupta o desactualizada en la caché local. Vaciar la caché es el primer paso de diagnóstico.
>
> 3. **Prevenir ataques de DNS poisoning/spoofing:** Si se detecta que un equipo está resolviendo dominios legítimos a IPs maliciosas (por un ataque de envenenamiento de caché), vaciar la caché elimina las entradas comprometidas inmediatamente.
>
> 4. **Forzar uso de servidores DNS nuevos:** Tras cambiar los servidores DNS (como hemos hecho en esta práctica), vaciar la caché garantiza que todas las nuevas resoluciones pasen por los nuevos servidores configurados en lugar de usar valores cacheados de los antiguos.
>
> 5. **Eliminación de entradas NXDOMAIN cacheadas:** Cuando un dominio no existe, la respuesta negativa (NXDOMAIN) también se cachea. Si el dominio se registra posteriormente, el equipo no podrá resolverlo hasta que expire el TTL de la entrada negativa. Vaciar la caché soluciona esto.

---

## Checklist Parte 2

- [ ] Comandos CLI para ver DNS (Windows y Linux) ejecutados y capturados
- [ ] DNS cambiados a 9.9.9.9 y 149.112.112.112 (Quad9)
- [ ] Captura del cambio de DNS en Propiedades de IPv4
- [ ] Captura de verificación con ipconfig /all
- [ ] Menú de DNS en móvil documentado (Android o iOS)
- [ ] ipconfig /displaydns ejecutado y capturado
- [ ] ipconfig /flushdns ejecutado y capturado
- [ ] Justificación del flush de caché redactada
