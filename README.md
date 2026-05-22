**UNIVERSIDAD AUTÓNOMA DE ENTRE RÍOS**

FACULTAD DE CIENCIA Y TECNOLOGÍA

*Ingeniería en Telecomunicaciones*

## TRABAJO PRÁCTICO INTEGRADOR 4
### Administración, Seguridad y Automatización de Redes

| Materia | Servicios de Telecomunicaciones y Redes |
|--------|------------------------------------------|
| Año | 2026 |
| Profesor | Joaquin Salvarredy |
| Alumnos | Fatima Nis Steingart |


## Fase 1: Problema disparador
DigitalSur es un proveedor regional de hosting y servicios web con base en Paraná. Tiene 80 clientes con sitios web variados: WordPress, e-commerce, portales corporativos de municipios y un par de plataformas de gestión. Toda la infraestructura corre en un rack con 4 servidores físicos y un solo router MikroTik de borde.
En el último trimestre tuvieron 3 incidentes de seguridad graves: 
1. Una inyección SQL en el e-commerce de un cliente que expuso datos de tarjetas de crédito.
2. Un defacement del portal web de un municipio. 
3. Un ataque de fuerza bruta masivo contra los paneles de administración de WordPress.
   
El problema de fondo: los servidores web, las bases de datos y la red de gestión están todos en la misma subred, sin segmentación ni protección a nivel de capa 7.
Para empeorar las cosas, el router MikroTik de borde no tiene redundancia (si se cae, se caen los 80 sitios) y las configuraciones se guardan en un Google Drive compartido sin versionado ni automatización. Un técnico nuevo borró sin querer la config del firewall del Drive y nadie se dio cuenta hasta que hubo otro incidente.
Los clientes están furiosos y amenazan con irse a otro proveedor. La gerencia los contrata para resolver esto: (1) segmentar la red, (2) proteger los sitios web con algo que pueda frenar ataques a nivel HTTP, (3) evitar que un cliente acceda a los datos de otro, y (4) redundar el router de borde.

## Preguntas del cliente
El responsable técnico de la empresa les plantea estas preguntas concretas. No hace falta responderlas todas de entrada; sirven para orientar la investigación:

**1. Tenemos todo en la misma red: servidores web, bases de datos, gestión. ¿Cómo separo esto sin cambiar hardware?**

**2. Nos hackearon 3 veces en 3 meses. ¿Cómo protejo los 80 sitios web sin poner algo individual en cada uno?**

**3. Un cliente accedió a la base de datos de otro cliente porque estaban en la misma red. ¿Cómo evito que esto vuelva a pasar?**

**4. Si se cae el router de borde, perdemos todo. ¿Cómo evito eso?**

---

## Fase 2: Mapa de aprendizaje
Antes de arrancar a implementar, sienten con el equipo y completen esta tabla.  
La idea es que sean honestos sobre lo que saben y lo que no.  
No está mal no saber algo; está mal no darse cuenta.

| Ya sabemos | Necesitamos aprender | Cómo lo vamos a aprender |
|------------|----------------------|--------------------------|
| Conceptos de VLAN, routing inter-VLAN y VPN básicos | Reforzar conceptos de routeo, reglas de acceso para las VLAN | Tutoriales de youtube, artículos de internet |
| Conceptos básicos de firewall | Como implementar uno en un router, reglas de seguridad | Youtube, Internet |
| Conceptos de ciberseguridad | WAF en su totalidad | Youtube, Internet |
|  | VRRP, ¿Qué es?, ¿Cómo funciona?, ¿Cómo se implementa? | Youtube, Internet, Laboratorios prácticos |

Este mapa se entrega junto con el TPI. Vamos a mirar cómo evolucionó entre el inicio y el final del trabajo.

---

## Fase 3: Investigación dirigida
Acá van los temas que necesitan investigar para resolver el problema. No es obligatorio cubrir todo al mismo nivel de profundidad: enfóquense en lo que sea más relevante para su solución.

### VLANs: segmentación de red
Investiguen: qué es una VLAN, cómo funciona 802.1Q, diferencia entre puertos tagged y untagged. En MikroTik las VLANs se configuran con bridge VLAN filtering. Investiguen cómo crear un bridge, agregar puertos y asignar VLANs.
Diseñen el esquema de VLANs para DigitalSur completando esta tabla:

Una **VLAN (Virtual Local Area Network)** es una técnica de segmentación lógica que divide una red física en múltiples dominios de broadcast independientes sin necesidad de hardware separado. Esto mejora la seguridad, reduce el tráfico de broadcast y permite aislar grupos de dispositivos por función. En el caso de DigitalSur, la ausencia de segmentación permitió que un cliente accediera a la base de datos de otro [Kurose & Ross, *Computer Networking: A Top-Down Approach*, 8th ed., 2021, p. 488].

El estándar **IEEE 802.1Q** define el mecanismo de *VLAN tagging*: inserta un campo de 4 bytes en la trama Ethernet con el **VID (VLAN Identifier)**, un valor de 12 bits que permite identificar hasta 4094 VLANs distintas sobre un mismo enlace físico (trunk). En cuanto a los tipos de puertos, los **puertos untagged (access)** conectan el switch a dispositivos finales que no manejan 802.1Q — el switch agrega y elimina el tag de forma transparente —, mientras que los **puertos tagged (trunk)** transportan tramas con el tag 802.1Q y conectan switches entre sí o con un router [Tanenbaum & Wetherall, *Computer Networks*, 5th ed., 2011, p. 313].

En **MikroTik**, las VLANs se implementan mediante **bridge VLAN filtering**: se crea un bridge con `vlan-filtering=yes`, se agregan los puertos físicos, se define cuáles son `tagged` y cuáles `untagged` por VLAN, se crean interfaces VLAN lógicas con IPs de gateway y se configura un servidor DHCP por segmento [MikroTik, *RouterOS Documentation — Bridge VLAN Filtering*, wiki.mikrotik.com].

| VLAN ID | Nombre  | Rango IP       | Propósito                            |
|---------|---------|----------------|--------------------------------------|
| 10      | Web     | 10.10.10.0/24  | Servidores web (WordPress, portales) |
| 20      | DB      | 10.10.20.0/24  | Base de datos                        |
| 30      | Gestión | 10.10.30.0/24  | Red interna, administración, SSH     |

Los rangos siguen el esquema de direccionamiento privado RFC 1918 [IETF RFC 1918, 1996], con prefijo `/24` (254 hosts por segmento) y gateway en la dirección `.1` de cada subred.


### Firewall en MikroTik
El firewall de MikroTik usa chains: input (tráfico hacia el router), forward (tráfico que pasa a través del router), output (tráfico que sale del router). Investiguen cómo crear reglas para: permitir tráfico entre VLANs según política, bloquear acceso de VLAN Web a VLAN Gestión, y hacer NAT para salida a internet.
Connection tracking: MikroTik rastrea el estado de cada conexión. Las reglas más eficientes aceptan tráfico established y related al principio de la cadena, y después filtran solo las conexiones new. Investiguen por qué este orden importa para el rendimiento del router.

El firewall de MikroTik procesa los paquetes según tres **chains**: `input` (tráfico destinado al router), `forward` (tráfico que atraviesa el router entre interfaces, donde se controla la comunicación inter-VLAN) y `output` (tráfico originado por el router) [Cheswick, Bellovin & Rubin, *Firewalls and Internet Security*, 2nd ed., 2003, p. 47].

El **connection tracking** permite actuar como firewall stateful: en lugar de evaluar cada paquete de forma aislada, el router mantiene una tabla de conexiones y clasifica cada paquete como `new`, `established`, `related` o `invalid` [Stallings, *Network Security Essentials*, 6th ed., 2016, p. 389]. El orden de las reglas impacta directamente en el rendimiento: colocar primero las reglas que aceptan tráfico `established` y `related` es crítico, dado que en una red activa la gran mayoría de los paquetes pertenece a conexiones ya establecidas y no necesita recorrer toda la tabla de filtrado [Doyle & Carroll, *Routing TCP/IP*, Vol. I, 2nd ed., 2005, p. 612]:

```bash
/ip firewall filter
add chain=forward connection-state=established action=accept
add chain=forward connection-state=related    action=accept
add chain=forward connection-state=invalid    action=drop
```

La política inter-VLAN para DigitalSur bloquea el acceso de VLAN Web hacia VLAN Gestión, permite VLAN Web hacia VLAN DB solo en puertos de base de datos, bloquea conexiones salientes desde VLAN DB y da acceso total a VLAN Gestión. Para la salida a internet se utiliza **NAT masquerade** en la chain `srcnat` [Kurose & Ross, *Computer Networking: A Top-Down Approach*, 8th ed., 2021, p. 488].


### WAF / Web Application Gateway: investigación de alternativas
Un WAF (Web Application Firewall) opera en capa 7 (HTTP/HTTPS). Mientras que el firewall de MikroTik filtra por IP, puerto y protocolo, un WAF inspecciona el contenido de las peticiones web y bloquea ataques como inyección SQL, cross-site scripting (XSS) y los demás del OWASP Top 10.
En la red se ubica delante de los servidores web, como proxy reverso:

```text
[Internet]-->[Firewall L3/L4]-->[WAF/WAG]-->[Servidores Web]
El WAF inspecciona cada petición HTTP/HTTPS antes de que llegue
al servidor. Si detecta un ataque, lo bloquea.
```

Un firewall de capa 3/4 es ciego al contenido HTTP: un atacante puede enviar una petición SQL maliciosa al puerto 443 y el paquete pasará sin ser detectado porque es técnicamente válido a nivel de red. Un **WAF (Web Application Firewall)** resuelve esto operando en la **capa 7 del modelo OSI**, ubicándose como proxy reverso entre los clientes y los servidores [OWASP, *Web Application Firewall Evaluation Criteria*, 2006, p. 4]:

```
[Internet] → [Firewall L3/L4] → [WAF] → [Servidores Web]
```

El WAF inspecciona cada petición HTTP/HTTPS contra reglas basadas en firmas, análisis de comportamiento y listas blancas/negras, bloqueando ataques del **OWASP Top 10** [Stuttard & Pinto, *The Web Application Hacker's Handbook*, 2nd ed., 2011, p. 32]. Los tres incidentes de DigitalSur corresponden directamente a esta lista:

| Incidente en DigitalSur         | Categoría OWASP Top 10 (2021)             |
|---------------------------------|-------------------------------------------|
| Inyección SQL en e-commerce     | A03:2021 – Injection                      |
| Defacement del portal municipal | A05:2021 – Security Misconfiguration      |
| Brute force contra WordPress    | A07:2021 – Identification & Auth Failures |

[OWASP, *OWASP Top 10:2021*, 2021]

Investiguen mínimo 3 alternativas open-source y comparen:


| WAF                   | Despliegue     | OWASP CRS   | Rate Limit   | Comunidad  | Dificultad |
|-----------------------|----------------|-------------|--------------|------------|------------|
| BunkerWeb             | Docker nativo  | Incluido    | Nativo       | Media-alta | Baja       |
| ModSecurity + Coraza  | Módulo/Docker  | Incluido    | Configurable | Muy alta   | Media      |
| Caddy Security        | Plugin Caddy   | Limitado    | Básico       | Media      | Baja       |
| OpenAppSec            | Docker/K8s     | Incluido + ML | Sí         | Media      | Media-alta |

Elijan una y justifiquen la elección. No hay respuesta única correcta: lo importante es que la justificación tenga sentido.


**BunkerWeb** integra NGINX con WAF preconfigurado con OWASP CRS y protección contra fuerza bruta [docs.bunkerweb.io]. 

**ModSecurity + Coraza** es el motor WAF open-source más maduro; Coraza es su reimplementación moderna en Go, compatible con el mismo lenguaje de reglas [ModSecurity, *Reference Manual v3*, 2023]. 

**Caddy Security** es más adecuado para autenticación centralizada que para inspección profunda de ataques [github.com/greenpau/caddy-security]. 

**OpenAppSec** utiliza machine learning para detectar ataques sin depender exclusivamente de firmas, reduciendo falsos positivos en entornos con tráfico diverso como un hosting compartido [openappsec.io].

---

### VRRP: alta disponibilidad
¿Qué es VRRP (Virtual Router Redundancy Protocol)? Cómo funciona: router master, backup, prioridad, preemption, virtual IP. En MikroTik se configura con /interface vrrp. Investiguen cómo crear una interfaz VRRP, asignar prioridad y definir la virtual IP que será el gateway de las VLANs.

## VRRP: Alta Disponibilidad del Router de Borde

**VRRP (Virtual Router Redundancy Protocol)** es un protocolo estándar definido en el RFC 5798 que permite agrupar varios routers físicos bajo una única **dirección IP virtual (VIP)**, que los hosts de la red usan como gateway. De esta forma, si el router principal falla, otro toma su lugar de forma automática y transparente, eliminando el punto único de fallo (SPOF) del router de borde [IETF RFC 5798, 2010, p. 3].

Su funcionamiento se basa en los siguientes conceptos:

- **Master:** router activo que responde el tráfico destinado a la VIP y envía anuncios VRRP periódicos por multicast al grupo `224.0.0.18` cada 1 segundo (por defecto).
- **Backup:** router(es) que escuchan los anuncios del Master. Si dejan de recibirlos durante el **Master Down Interval**, inician una elección y el de mayor prioridad asume el rol de Master.
- **Prioridad:** valor entre 1 y 254 que determina qué router gana la elección. A mayor valor, mayor prioridad. En DigitalSur: MikroTik A con prioridad 200 (Master) y MikroTik B con prioridad 100 (Backup).
- **Preemption:** si está habilitado, un Backup con prioridad más alta recupera automáticamente el rol de Master cuando el Master original vuelve a estar disponible.
- **Virtual IP (VIP):** dirección IP compartida configurada como gateway en todos los hosts de las VLANs. Cuando cambia el Master, el nuevo router anuncia la VIP con su MAC mediante un **Gratuitous ARP**, redirigiendo el tráfico sin necesidad de reconfiguración en los hosts [Comer, *Internetworking with TCP/IP*, Vol. 1, 6th ed., 2013, p. 341].

El tiempo de convergencia con la configuración por defecto es de 3 a 4 segundos [Stevens & Wright, *TCP/IP Illustrated*, Vol. 1, 2nd ed., 2011, p. 564].

En **MikroTik** se configura con `/interface vrrp`. Se crea una interfaz VRRP sobre la interfaz física o bridge, se asigna el mismo `vrid` en ambos routers, se define la prioridad de cada uno y se asigna la VIP que funcionará como gateway de las VLANs:

```bash
# En MikroTik A (Master)
/interface vrrp add name=vrrp1 interface=bridge1 vrid=1 priority=200 interval=1

# En MikroTik B (Backup)
/interface vrrp add name=vrrp1 interface=bridge1 vrid=1 priority=100 interval=1

# Asignar la VIP en ambos routers (misma dirección)
/ip address add address=10.10.10.254/24 interface=vrrp1
```

La VIP `10.10.10.254` se configura como gateway en los servidores de cada VLAN (o se distribuye vía DHCP), de modo que todo el tráfico saliente use esa dirección independientemente de cuál router esté activo en ese momento [MikroTik, *RouterOS Documentation — VRRP*, wiki.mikrotik.com].

---

## Fase 4: Implementación
Ahora sí, manos a la obra. Monten la solución, prueben que funciona, documenten lo que hicieron. Es normal que las cosas no funcionen de primera; lo importante es registrar el proceso, incluyendo los errores y cómo los resolvieron.

### Topología en GNS3 con MikroTik CHR
Armen la siguiente topología en GNS3 usando MikroTik CHR como routers (descargar la imagen OVA de mikrotik.com, importar en GNS3):
```text
                     [Internet]
                          |
             +------------+------------+
             |                         |
        [MikroTik A]              [MikroTik B]
        VRRP Master               VRRP Backup
        (Prior. 200)              (Prior. 100)
             |                         |
             +------------+------------+
                          |
                   [Switch Core L2]
                    /       |       \
               VLAN 10   VLAN 20   VLAN 30
                 Web        DB     Gestión
                  |         |         |
            [Serv. Web] [Serv. DB] [Admin/SSH]

                    [Docker Host]
                [WAF PoC en Docker]
         (entre Internet y VLAN Web)
```
MikroTik CHR free tier tiene límite de 1 Mbps, suficiente para los labs. Para el switch pueden usar un switch ethernet de GNS3 o un MikroTik adicional en modo bridge.

### Parte 1: VLANs en MikroTik
Ejemplo de comandos MikroTik para crear VLANs con bridge VLAN filtering:
```text
# Crear bridge y habilitar VLAN filtering
/interface bridge add name=bridge1 vlan-filtering=yes

# Agregar puertos al bridge
/interface bridge port add bridge=bridge1 interface=ether2
/interface bridge port add bridge=bridge1 interface=ether3
/interface bridge port add bridge=bridge1 interface=ether4

# Definir VLANs en el bridge
/interface bridge vlan
add bridge=bridge1 tagged=bridge1 untagged=ether2 vlan-ids=10
add bridge=bridge1 tagged=bridge1 untagged=ether3 vlan-ids=20
add bridge=bridge1 tagged=bridge1 untagged=ether4 vlan-ids=30

# Crear interfaces VLAN y asignar IPs
/interface vlan add interface=bridge1 name=vlan10 vlan-id=10
/ip address add address=10.10.10.1/24 interface=vlan10
```
5. Crear bridge con VLAN filtering habilitado.
6. Agregar puertos al bridge, definir 3 VLANs (10=Web, 20=DB, 30=Gestión) con puertos tagged/untagged.
7. Crear interfaces VLAN y asignar IPs de gateway.
8. Configurar DHCP server por VLAN (/ip dhcp-server) con un pool por segmento.
9. Verificar: un servidor en VLAN 10 obtiene IP del rango correcto y puede hacer ping al gateway.

### Parte 2: Firewall + connection tracking en MikroTik
10. Configurar firewall filter (/ip firewall filter): permitir tráfico entre VLANs según política. VLAN Web solo puede comunicarse con VLAN DB (puertos específicos de base de datos). VLAN Gestión accede a todo. VLAN DB no inicia conexiones hacia afuera.
11. Connection tracking eficiente: primero aceptar established/related, luego filtrar new. Documentar por qué este orden importa para el rendimiento.
12. NAT masquerade para salida a internet.
13. Verificar: un servidor web puede conectar a la base de datos, pero no puede hacer SSH a la VLAN de gestión.

### Parte 3: WAF - Prueba de concepto en Docker
Desplieguen la herramienta WAF que eligieron en la fase de investigación. No se prescribe cuál usar: el alumno investiga, elige, justifica y despliega.

14. Desplegar el WAF elegido con Docker (docker-compose). Configurar para proteger al menos un servicio web de prueba. Pueden usar un NGINX simple, una app vulnerable tipo DVWA, o un WordPress de test.
15. Demostrar bloqueo de al menos 2 tipos de ataque OWASP Top 10. Ejemplos: SQL injection (con sqlmap o manualmente), XSS (inyección de script en parámetro), brute force / rate limiting (múltiples intentos de login).
16. Documentar: diagrama de arquitectura (dónde se ubica el WAF en la red), configuración aplicada, pruebas realizadas con capturas de pantalla, resultados (qué se bloqueó y qué no).
17. Mostrar los logs del WAF: cómo se ve un ataque bloqueado, qué información registra.
La idea es que experimenten con la herramienta. No hace falta que sea perfecta, pero sí que demuestren que entienden qué hace y cómo funciona.

### Parte 4: VRRP en MikroTik
18. Configurar /interface vrrp en ambos MikroTik con una virtual IP compartida que será el gateway de las VLANs.
19. MikroTik A: prioridad 200 (master). MikroTik B: prioridad 100 (backup).
20. Probar failover: apagar MikroTik A (o desconectar su interfaz) y verificar que MikroTik B toma el control.
21. Medir tiempo de convergencia: hacer ping continuo durante el failover y contar paquetes perdidos.
22. Documentar: tiempos de failover medidos, paquetes perdidos, estado VRRP antes y después.
---

## Fase 5: Reflexión y defensa
### Reflexión individual
Cada integrante responde por separado (medio párrafo por pregunta alcanza):
1. ¿Qué aprendí que no esperaba?
2. ¿Qué fue lo más difícil y cómo lo resolví?
3. ¿Cómo encaro un problema parecido la próxima vez?
4. ¿Dónde puedo aplicar esto profesionalmente?

### Defensa oral
* 10 minutos de presentación (qué hicieron, por qué, cómo).
* Demostración en vivo o video corto de la solución funcionando.
* 10 minutos de preguntas de la cátedra.
