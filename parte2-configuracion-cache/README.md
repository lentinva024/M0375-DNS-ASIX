# Parte 2: Configuración y Caché [2p]

> Todo sistema operativo guarda las resoluciones DNS para no saturar la red.

---

## 1. Cambio de servidores DNS

### 1.1. ¿Cómo puedes ver mediante consola (CLI) qué servidores DNS tienes asignados actualmente?

**Windows:**
```powershell
ipconfig /all
```
> Busca la línea "Servidores DNS" bajo tu adaptador de red activo.

**Linux:**
```bash
resolvectl status
# o
cat /etc/resolv.conf
```

---

### 1.2. Cambia la configuración de red poniendo los DNS del Benchmark

> Instrucciones detalladas en la guía.

**Captura:** `capturas/parte2/cambio-dns.png`

---

### 1.3. ¿En qué menú de tu dispositivo móvil puedes forzar el uso de unos DNS específicos?

**Android:** Ajustes → Red e Internet → Wi-Fi → (red conectada) → Avanzado → IP settings → Estático → DNS 1 / DNS 2

**iOS:** Ajustes → Wi-Fi → (botón "i" junto a la red) → Configurar DNS → Manual → Añadir servidores

---

## 2. Gestión de la caché DNS

### 2.1. Mostrar direcciones almacenadas en caché

**Windows:**
```powershell
ipconfig /displaydns
```

**Linux:**
```bash
resolvectl statistics
# o
sudo killall -USR1 systemd-resolved
journalctl -u systemd-resolved | grep "CACHE"
```

**Captura:** `capturas/parte2/caché-dns.png`

---

### 2.2. Vaciar la caché DNS

**Windows:**
```powershell
ipconfig /flushdns
```

**Linux:**
```bash
sudo resolvectl flush-caches
```

> **¿Para qué es útil?** Un administrador de sistemas vacía la caché DNS cuando:
> - Ha cambiado la IP de un servidor y necesita que la resolución refleje el cambio inmediatamente
> - Sospecha de envenenamiento de caché (DNS poisoning)
> - Está haciendo troubleshooting de conectividad
> - Quiere forzar la resolución desde los servidores DNS configurados en lugar de usar valores cacheados

**Captura:** `capturas/parte2/flush-dns.png`

---

## Checklist Parte 2

- [ ] Comandos CLI para ver DNS (Windows y Linux) documentados
- [ ] DNS cambiados a los del Benchmark
- [ ] Captura del cambio de DNS
- [ ] Menú de DNS en móvil documentado
- [ ] Caché DNS mostrada (captura)
- [ ] Caché DNS vaciada (captura)
- [ ] Justificación del flush de caché
