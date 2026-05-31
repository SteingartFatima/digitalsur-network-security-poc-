# Integración de BunkerWeb con GNS3

## Objetivo

El objetivo de esta implementación fue integrar la topología simulada en GNS3 con el entorno Docker donde se ejecuta BunkerWeb, permitiendo que los dispositivos de la red virtual pudieran acceder al WAF y a los servicios protegidos.

---

## Topología



<img width="501" height="511" alt="image" src="https://github.com/user-attachments/assets/518c6311-34e5-489e-bfa2-53ccd84f98f8" />


---

## 1. Incorporación de Ubuntu Docker Guest

Dentro de GNS3 se agregó un nodo **Ubuntu Docker Guest** desde:

```text
End Devices
    ↓
Ubuntu Docker Guest
```

Este nodo se utilizó para validar la conectividad entre la topología simulada y el entorno Docker.

<img width="815" height="383" alt="image" src="https://github.com/user-attachments/assets/8420a6e6-54d9-4586-8c4d-e171dbb8bbf5" />


---

## 2. Conexión del Cloud

Se agregó un nodo **Cloud** para conectar la topología GNS3 con la red Docker del host Ubuntu.

La conexión se realizó entre:

```text
Cloud1
    ↓
Router-firewall-master
    ↓
ether2
```

<img width="734" height="219" alt="image" src="https://github.com/user-attachments/assets/0d4cf561-aaa0-4f61-b482-26bc59df433e" />


---

## 3. Configuración de Ubuntu Docker Guest

Se configuró una dirección IP manual para las pruebas.

```bash
ip addr add 10.10.10.50/24 dev eth0
ip link set eth0 up
ip route add default via 10.10.10.1
```

Verificación:

```bash
ping 10.10.10.1
```

Resultado esperado:

```text
64 bytes from 10.10.10.1
```

<img width="823" height="419" alt="image" src="https://github.com/user-attachments/assets/4a3540de-663c-4da2-9435-302e1366ad69" />


---

## 4. Identificación de la red Docker

Desde el host Ubuntu se identificaron las IPs asignadas a los contenedores.

```bash
docker inspect backend-cliente1
docker inspect backend-cliente2
docker inspect tpn4-styr-main-bunkerweb-1
```

IPs obtenidas:

| Servicio | Dirección IP |
|-----------|-----------|
| backend-cliente1 | 172.18.0.2 |
| backend-cliente2 | 172.18.0.3 |
| bunkerweb | 172.18.0.4 |

---

## 5. Configuración de MikroTik

Se agregó una dirección IP sobre la interfaz conectada al Cloud para permitir el acceso a la red Docker.

```routeros
/ip address add address=172.18.0.250/16 interface=ether2
```

Verificación:

```routeros
/ip address print
```

---

## 6. Validación de conectividad

### Acceso al bridge Docker

```routeros
ping 172.18.0.1
```

Resultado esperado:

```text
packet-loss=0%
```

<img width="895" height="230" alt="image" src="https://github.com/user-attachments/assets/7016850e-216f-4e63-9c0b-92cee43d46b4" />


---

### Acceso al contenedor BunkerWeb

```routeros
ping 172.18.0.4
```

Resultado esperado:

```text
packet-loss=0%
```

<img width="895" height="241" alt="image" src="https://github.com/user-attachments/assets/e7cb4ba8-dd7d-4c17-858c-7e6a89c87843" />


---

## 7. Validación HTTP

Se verificó que MikroTik pudiera acceder al servicio HTTP publicado por BunkerWeb.

```routeros
/tool fetch url="http://172.18.0.4"
```

Resultado esperado:

```text
status: finished
code: 200
```

<img width="882" height="165" alt="image" src="https://github.com/user-attachments/assets/a66eba4d-2750-44c0-a403-99d0a25e645b" />


---

---

# 8. Validación del WAF

Una vez verificada la conectividad entre GNS3, Docker y BunkerWeb, se realizaron pruebas de seguridad para comprobar que el WAF inspeccionaba y bloqueaba solicitudes maliciosas antes de que alcanzaran el servidor protegido.

---

## Prueba de Cross-Site Scripting (XSS)

Se realizó una solicitud HTTP incluyendo código JavaScript malicioso dentro de un parámetro de la URL.

```bash
curl -i "http://cliente1.localhost/?nombre=%3Cscript%3Ealert(1)%3C/script%3E" | head -10
```

Resultado:

- Solicitud bloqueada por BunkerWeb.
- Código HTTP 403 Forbidden.

<img width="821" height="432" alt="image" src="https://github.com/user-attachments/assets/86473dbd-1d0a-4ee1-9839-4328333a7671" />


---

## Prueba de SQL Injection

Se realizó una prueba simulando una inyección SQL mediante una consulta manipulada.

```bash
curl -i "http://cliente1.localhost/?id=UNION%20SELECT%201,2,3" | head -10
```

Resultado:

- Solicitud bloqueada por BunkerWeb.
- Código HTTP 403 Forbidden.

<img width="831" height="426" alt="image" src="https://github.com/user-attachments/assets/9412891f-d4b8-4b4a-936a-674ecbd6e9be" />


---

## Registro de Eventos

Se verificó que los intentos de ataque quedaran registrados por ModSecurity y las reglas OWASP CRS integradas en BunkerWeb.

Los registros mostraron la detección de las solicitudes maliciosas y la generación automática de respuestas HTTP 403.

<img width="820" height="519" alt="image" src="https://github.com/user-attachments/assets/0ffad685-6847-45c2-8c3d-93d82dc0c111" />

