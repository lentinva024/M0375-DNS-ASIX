# Laboratorio: DNS maestro/esclavo + DHCP + DNSSEC con BIND9

> Montaje práctico en VirtualBox: un servidor DNS maestro que además hace de servidor DHCP, un servidor DNS esclavo y un cliente, todos en una red interna. La zona `lake.local` está firmada con DNSSEC y se ha probado que el esclavo sigue respondiendo con el maestro apagado.

Fecha de finalización y pruebas: 26 de octubre de 2026.

---

## 1. Resumen

| Elemento | Valor |
|---|---|
| Dominio | `lake.local` |
| Red interna | `192.168.56.0/24` (red interna de VirtualBox `redlab`) |
| Software | Ubuntu 24.04.4 LTS, BIND 9.18.39, ISC DHCP Server |
| Virtualizador | Oracle VirtualBox |
| Servicios | DNS maestro, DNS esclavo (AXFR), DHCP, DNSSEC |
| Usuario en todas las VMs | `lake` |

### Estado final

| Componente | Estado |
|---|---|
| DNS maestro (zona directa e inversa) | Funcionando |
| DHCP en la red interna | Funcionando (el cliente recibe IP y los dos DNS) |
| DNS esclavo con transferencia de zona | Funcionando (sincronizado) |
| DNSSEC en `lake.local` | Funcionando (`secure: yes` en maestro y esclavo) |
| Zona inversa firmada | No (decisión de diseño) |
| DHCP entrega el DNS secundario | Funcionando (`192.168.56.10, 192.168.56.11`) |
| Netplan del maestro (`dhcp4: false` en la red interna) | Corregido y verificado |
| Prueba de alta disponibilidad | Superada (el esclavo responde con firmas con el maestro apagado) |
| Migración del DHCP a Kea | Pendiente (ver observación) |

> **Observación sobre el servidor DHCP:** el profesor pidió que el DHCP se hiciera con **Kea** (`kea-dhcp4-server`), pero cuando nos lo indicó ya habíamos empezado la actividad con el DHCP de siempre, `isc-dhcp-server`. Por eso todo el laboratorio y las pruebas de este documento están hechos con `isc-dhcp-server`. Tenemos previsto pasarlo a Kea manteniendo la misma red, el mismo rango y los mismos DNS, y actualizaremos esta documentación cuando esté hecho.

### Cambios respecto a la versión anterior

1. **Netplan del maestro (`ServidorDNS`):** se puso `dhcp4: false` en la interfaz de la red interna `enp0s8`. Antes el servidor pedía IP por DHCP a su propio servicio.
2. **DHCP con los dos DNS:** `/etc/dhcp/dhcpd.conf` ahora reparte `192.168.56.10` y `192.168.56.11`, así el cliente tiene un DNS de respaldo.
3. **Esclavo (`SlaveDnsServer`):** `/etc/bind/named.conf.options` se dejó con las mismas opciones de seguridad, recursión y reenviadores que el maestro.
4. **Cliente (`cliente2`):** comprobado que recibe por DHCP los dos DNS y que resuelve el registro A `www.lake.local` y el CNAME `web.lake.local`.
5. **Alta disponibilidad:** con `ServidorDNS` apagado, el esclavo (`192.168.56.11`) responde `NOERROR`, con el flag `aa` y la firma `RRSIG`.

---

## 2. Topología

```
                         Internet
                            |
                   (NAT de VirtualBox)
                            |
           +----------------+----------------+
           |                                 |
   enp0s3 (NAT) 10.0.2.15           enp0s8 (NAT) 10.0.3.15
           |                                 |
 +---------+-----------+          +----------+-----------+
 |     ServidorDNS     |          |    SlaveDnsServer    |
 |  DNS maestro + DHCP |          |      DNS esclavo     |
 |  enp0s8 .56.10      |          |  enp0s3 .56.11       |
 +---------+-----------+          +----------+-----------+
           |                                 |
           +------ Red interna "redlab" -----+
                     192.168.56.0/24
                            |
                   +--------+--------+
                   |    cliente2     |
                   |  enp0s3 (DHCP)  |
                   |  192.168.56.101 |
                   +-----------------+
```

---

## 3. Máquinas virtuales

### 3.1. Datos de VirtualBox

| VM | Hostname | RAM | CPU | Disco | Adaptador 1 | Adaptador 2 |
|---|---|---|---|---|---|---|
| `ServidorDNS` | `ServidorDNS` | 5737 MB | 2 | 25 GB (VDI) | NAT | Red interna `redlab` |
| `SLAVE SERVER DNS` | `SlaveDnsServer` | 3840 MB | 2 | 25 GB (VDI) | Red interna `redlab` | NAT |
| Cliente | `cliente2` | 2048 MB | 1 | 25 GB (VDI) | Red interna `redlab` | - |

- Servidores: `ubuntu-24.04.4-live-server-amd64.iso`
- Cliente: `ubuntu-24.04.4-desktop-amd64.iso`

### 3.2. Direccionamiento

| VM | Interfaz | Función | Dirección | MAC |
|---|---|---|---|---|
| ServidorDNS | `enp0s3` | NAT | `10.0.2.15/24` (DHCP de VirtualBox) | `08:00:27:f5:63:a1` |
| ServidorDNS | `enp0s8` | Red interna | `192.168.56.10/24` (fija) | `08:00:27:6a:11:a4` |
| SlaveDnsServer | `enp0s3` | Red interna | `192.168.56.11/24` (fija) | `08:00:27:97:9c:40` |
| SlaveDnsServer | `enp0s8` | NAT | `10.0.3.15/24` (DHCP de VirtualBox) | `08:00:27:e7:9b:53` |
| cliente2 | `enp0s3` | Red interna | `192.168.56.101/24` (DHCP) | `08:00:27:d2:18:a6` |

---

## 4. Configuración de los servidores

### 4.1. Maestro `ServidorDNS` (192.168.56.10)

**Netplan** (`/etc/netplan/50-cloud-init.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.10/24]
```

**Opciones de BIND9** (`/etc/bind/named.conf.options`):

```
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.56.10; };
    allow-query { localhost; 192.168.56.0/24; };
    allow-transfer { 192.168.56.11; };
    recursion yes;
    allow-recursion { localhost; 192.168.56.0/24; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
};
```

**Zonas** (`/etc/bind/named.conf.local`):

```
zone "lake.local" {
    type master;
    file "/var/lib/bind/db.lake.local";
    dnssec-policy default;
    inline-signing yes;
    key-directory "/var/cache/bind/keys";
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.56";
};
```

**DHCP** (`/etc/dhcp/dhcpd.conf`, servicio `isc-dhcp-server`; ver la observación del apartado 1 sobre el paso a Kea):

```
authoritative;
default-lease-time 600;
max-lease-time 7200;
option domain-name "lake.local";
option domain-name-servers 192.168.56.10, 192.168.56.11;

subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.100 192.168.56.200;
}
```

### 4.2. Esclavo `SlaveDnsServer` (192.168.56.11)

**Opciones de BIND9** (`/etc/bind/named.conf.options`):

```
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.56.11; };
    allow-query { localhost; 192.168.56.0/24; };
    recursion yes;
    allow-recursion { localhost; 192.168.56.0/24; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
};
```

**Zonas** (`/etc/bind/named.conf.local`):

```
zone "lake.local" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.lake.local";
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.192.168.56";
};
```

---

## 5. Pruebas

### 5.1. Verificación en el cliente (`cliente2`)

- IP asignada: `192.168.56.101/24`
- DNS recibidos por DHCP: `192.168.56.10` y `192.168.56.11` (comprobado con `resolvectl status`)
- Resoluciones correctas: `www.lake.local` → `192.168.56.20` y el CNAME `web.lake.local`

### 5.2. Tolerancia a fallos (maestro apagado)

Con la VM `ServidorDNS` apagada del todo, se lanzó la consulta directamente al esclavo desde el cliente:

```bash
dig @192.168.56.11 www.lake.local +dnssec
```

Resultado:

- Estado: `NOERROR`
- Flags: `qr aa rd ra` (`aa` indica que la respuesta es autoritativa)
- Respuesta: `192.168.56.20` junto con su firma `RRSIG` (algoritmo 13)

El esclavo tiene la zona completa con sus firmas DNSSEC, así que el servicio sigue funcionando aunque caiga el maestro.

---

## 6. Demostración en directo

Comandos para enseñar el laboratorio funcionando, en este orden. Arrancar primero `ServidorDNS`, después `SlaveDnsServer` y por último `cliente2`.

### 6.1. En `ServidorDNS` (maestro + DHCP)

```bash
ip -br a                                         # enp0s8 con 192.168.56.10
cat /etc/netplan/50-cloud-init.yaml              # dhcp4: false en enp0s8
systemctl status named isc-dhcp-server --no-pager # los dos servicios active (running)
sudo named-checkconf && echo "BIND OK"           # sin errores de configuración
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf           # sintaxis del DHCP correcta
sudo rndc zonestatus lake.local                  # type: primary, secure: yes
cat /var/lib/dhcp/dhcpd.leases                   # concesión entregada a cliente2
```

### 6.2. En `SlaveDnsServer` (esclavo)

```bash
ip -br a                                         # enp0s3 con 192.168.56.11
sudo rndc zonestatus lake.local                  # type: secondary, secure: yes
ls -l /var/cache/bind/                           # ficheros de zona recibidos del maestro
sudo journalctl -u named --no-pager | grep -i transfer | tail -5   # transferencia completada
```

### 6.3. En `cliente2`

```bash
sudo nmcli device disconnect enp0s3 && sudo nmcli device connect enp0s3   # pide IP de nuevo por DHCP
ip -br a show enp0s3                             # IP dentro del rango .100-.200
resolvectl dns enp0s3                            # 192.168.56.10 y 192.168.56.11
dig www.lake.local +short                        # 192.168.56.20
dig web.lake.local                               # CNAME hacia www.lake.local
dig www.lake.local +dnssec                       # aparece el RRSIG
dig @192.168.56.10 lake.local AXFR               # Transfer failed: el cliente no puede copiar la zona
```

### 6.4. Prueba de alta disponibilidad

Apagar el maestro (`sudo poweroff` en `ServidorDNS`) y desde `cliente2`:

```bash
dig @192.168.56.11 www.lake.local +dnssec        # NOERROR, flag aa y RRSIG
```

---

## Checklist del laboratorio

- [x] DNS maestro con zona directa e inversa
- [x] DHCP en la red interna repartiendo los dos DNS
- [x] DNS esclavo sincronizado por transferencia de zona
- [x] Zona `lake.local` firmada con DNSSEC
- [x] Netplan del maestro corregido (`dhcp4: false`)
- [x] Resolución comprobada desde el cliente
- [x] Prueba con el maestro apagado superada
- [ ] Migración del DHCP de `isc-dhcp-server` a Kea
