# Parte 4: Análisis de Tráfico de Red (Wireshark) [3p]

> **Esta parte la realiza un compañero de equipo.**
> Aquí se documentan los pasos para que tu compañero pueda completarla.

> Vamos a comprobar qué viaja realmente por el cable físico cuando resolvemos un nombre.

---

## 1. Captura de tráfico DNS

### Pasos:

1. Abre Wireshark en tu equipo
2. Selecciona tu tarjeta de red principal (la que tenga tráfico)
3. Inicia la captura
4. Aplica el filtro: `dns` o `udp.port == 53`
5. Abre una terminal y ejecuta:
   ```bash
   nslookup -type=mx google.com
   ```
6. Detén la captura
7. Localiza el intercambio Query → Response

**Captura general:** `capturas/parte4/wireshark-captura-general.png`

---

## 2. Preguntas a responder

### 2.1. Capa de Transporte

> **¿Qué protocolo se utiliza (TCP o UDP)?**
>
> (Documentar respuesta)
>
> **¿Por qué DNS utiliza este protocolo por defecto en lugar del otro?**
>
> (Explicar: DNS usa UDP por defecto porque las consultas son pequeñas (<512 bytes normalmente), UDP es más rápido (sin handshake), y no requiere conexión. TCP se usa solo para respuestas grandes (transferencias de zona, respuestas >512 bytes con EDNS0 deshabilitado) o para DNS over TCP/TLS.)

**Captura:** `capturas/parte4/wireshark-protocolo.png`

---

### 2.2. Puertos

> **Puerto de origen (cliente):** (dinámico, >1024, ej. 51234)
> **Puerto de destino (servidor):** 53 (well-known)

**Captura:** `capturas/parte4/wireshark-puertos.png`

---

### 2.3. Identificador de transacción (Transaction ID)

> (Expandir sección Domain Name System en Wireshark y documentar el Transaction ID que vincula Query con Response)

**Captura:** `capturas/parte4/wireshark-transaction-id.png`

---

### 2.4. Flags

> (En el paquete de Response, desplegar sección Flags)
> - **Authoritative Answer (AA):** ¿Está a 0 o a 1?
> - **Significado:** (explicar)

**Captura:** `capturas/parte4/wireshark-flags.png`

---

### 2.5. Respuestas (Answers)

> (Desplegar bloque de respuestas del registro MX de google.com)
> **Servidor de correo con mayor prioridad (preference más bajo):** (documentar)

**Captura:** `capturas/parte4/wireshark-answers.png`

---

## Checklist Parte 4

- [ ] Captura de tráfico DNS realizada
- [ ] Filtro dns aplicado
- [ ] Protocolo de transporte identificado (UDP)
- [ ] Puertos origen y destino documentados
- [ ] Transaction ID identificado
- [ ] Flags analizadas (Authoritative Answer)
- [ ] Answers del MX analizadas
- [ ] Todas las capturas con recortes del panel de detalles
