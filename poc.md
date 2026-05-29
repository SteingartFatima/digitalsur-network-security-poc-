## Topología

La topología implementada está compuesta por:

- Router-firewall-master (Prioridad 200)
- Router-backup (Prioridad 100)
- Switch Ethernet de GNS3
- Cliente1 (VLAN 10)
- Cliente2 (VLAN 20)
- Cliente3 (VLAN 30)

<img width="467" height="418" alt="image" src="https://github.com/user-attachments/assets/b286cd26-1faf-46c2-9fca-7095cd8cc7b2" />

---

## Configuración del Switch

| Puerto | Dispositivo | VLAN | Tipo |
|----------|----------|----------|----------|
| 0 | Router-firewall-master | 1 | dot1q |
| 1 | Router-backup | 1 | dot1q |
| 2 | Cliente1 | 10 | access |
| 3 | Cliente2 | 20 | access |
| 4 | Cliente3 | 30 | access |

### Configuración aplicada

<img width="691" height="488" alt="image" src="https://github.com/user-attachments/assets/24527a27-9d79-4993-b070-fa161babce75" />


---

## Configuración del Router Principal

### Bridge

```routeros
/interface bridge add name=bridge1 vlan-filtering=no
/interface bridge port add bridge=bridge1 interface=ether1
```

### VLANs

```routeros
/interface bridge vlan
add bridge=bridge1 tagged=bridge1,ether1 vlan-ids=10,20,30
```

```routeros
/interface vlan
add interface=bridge1 name=vlan10 vlan-id=10
add interface=bridge1 name=vlan20 vlan-id=20
add interface=bridge1 name=vlan30 vlan-id=30
```

### VRRP

```routeros
/interface vrrp
add interface=vlan10 name=vrrp10 priority=200 vrid=10
add interface=vlan20 name=vrrp20 priority=200 vrid=20
add interface=vlan30 name=vrrp30 priority=200 vrid=30
```

### Dirección IP virtual

Se configuró una dirección IP virtual para cada VLAN utilizando VRRP. Estas direcciones actúan como gateway para los clientes y permanecen disponibles incluso ante la falla del router principal.

| VLAN | Gateway Virtual |
|--------|--------|
| VLAN 10 | 10.10.10.1 |
| VLAN 20 | 10.10.20.1 |
| VLAN 30 | 10.10.30.1 |


### Direccionamiento IP

```routeros
/ip address
add address=10.10.10.2/24 interface=vlan10
add address=10.10.20.2/24 interface=vlan20
add address=10.10.30.2/24 interface=vlan30

add address=10.10.10.1/24 interface=vrrp10
add address=10.10.20.1/24 interface=vrrp20
add address=10.10.30.1/24 interface=vrrp30
```

### DHCP

```routeros
/ip pool
add name=pool10 ranges=10.10.10.100-10.10.10.200
add name=pool20 ranges=10.10.20.100-10.10.20.200
add name=pool30 ranges=10.10.30.100-10.10.30.200
```

```routeros
/ip dhcp-server
add address-pool=pool10 interface=vlan10 name=dhcp10 disabled=no
add address-pool=pool20 interface=vlan20 name=dhcp20 disabled=no
add address-pool=pool30 interface=vlan30 name=dhcp30 disabled=no
```

```routeros
/ip dhcp-server network
add address=10.10.10.0/24 gateway=10.10.10.1
add address=10.10.20.0/24 gateway=10.10.20.1
add address=10.10.30.0/24 gateway=10.10.30.1
```

### Activación del filtrado VLAN

```routeros
/interface bridge set bridge1 vlan-filtering=yes
```

---

## Evidencia de configuración

### Interfaces creadas

<img width="733" height="489" alt="image" src="https://github.com/user-attachments/assets/d0008457-a21d-492f-892c-4ac0f7c42f54" />


### Direccionamiento IP

<img width="483" height="237" alt="image" src="https://github.com/user-attachments/assets/e4a2bffe-b65b-4d5b-ad24-4ba41e60a6a4" />


### DHCP

<img width="537" height="165" alt="image" src="https://github.com/user-attachments/assets/173dcdcf-1cfa-402d-8e43-22a48398ff1e" />


---

## Router Backup

Se replicó la configuración del router principal modificando:

- Prioridad VRRP = 100
- IP VLAN10 = 10.10.10.3
- IP VLAN20 = 10.10.20.3
- IP VLAN30 = 10.10.30.3

### Estado VRRP

Se observa el indicador **B (Backup)** en las tres interfaces VRRP.

<img width="809" height="209" alt="image" src="https://github.com/user-attachments/assets/92fad5d2-4562-4e62-b620-ac13c20b73fb" />


---

## Validación DHCP

### Cliente1

<img width="462" height="123" alt="image" src="https://github.com/user-attachments/assets/243cb403-c347-4e4e-9931-d304415ea616" />


### Cliente2

<img width="408" height="105" alt="image" src="https://github.com/user-attachments/assets/86721c05-10bf-4718-9dd9-0c7b665b9154" />


### Cliente3

<img width="420" height="140" alt="image" src="https://github.com/user-attachments/assets/afce401d-7a9c-4f44-a209-3ed08c55cbba" />


---

## Prueba de Failover

Se ejecutó un ping continuo hacia la IP virtual VRRP:

```bash
ping 10.10.10.1 -t
```

Posteriormente se apagó el router principal.

### Resultado

- Se perdieron únicamente 2 paquetes ICMP.
- El router backup asumió automáticamente el rol de master.
- La IP virtual continuó respondiendo.
- Los clientes mantuvieron conectividad sin reconfiguración.

### Evidencia

<img width="825" height="541" alt="image" src="https://github.com/user-attachments/assets/9e113403-3a8d-4f49-8bc5-ed1230a855e2" />

---

### Backup asumiendo el rol de Master

Durante la prueba de failover se ejecutó un ping continuo hacia la dirección virtual VRRP (`10.10.10.1`) desde Cliente1 y posteriormente se apagó el router principal.

Tras la caída del router principal, se verificó el estado de las interfaces VRRP en el router de respaldo mediante el siguiente comando:

```routeros
/interface vrrp print
```

Se observa el indicador **RM (Running Master)** en las tres interfaces VRRP, confirmando que el router backup asumió correctamente el rol de master y continuó prestando el servicio de gateway utilizando las direcciones IP virtuales configuradas.

<img width="823" height="233" alt="image" src="https://github.com/user-attachments/assets/d527b8f7-aa52-4426-a992-6bef1557f86a" />


Este resultado demuestra el correcto funcionamiento del protocolo VRRP, permitiendo mantener la disponibilidad del servicio con una interrupción mínima durante la conmutación.
