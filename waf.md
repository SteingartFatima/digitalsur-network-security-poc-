# BunkerWeb WAF 

## Descripción

Laboratorio de pruebas orientado a la implementación y validación de un Web Application Firewall (WAF) utilizando BunkerWeb, **ModSecurity** y **OWASP Core Rule Set (CRS)**.

El objetivo fue desplegar aplicaciones web deliberadamente vulnerables y verificar la capacidad del WAF para detectar y bloquear ataques comunes como:

* SQL Injection (SQLi)
* Cross Site Scripting (XSS)

La infraestructura fue desplegada mediante Docker Compose sobre Ubuntu.

---

## Tecnologías utilizadas

* Docker
* Docker Compose
* BunkerWeb 1.6.10
* ModSecurity
* OWASP CRS v4
* PHP 8.2 + Apache
* MariaDB
* Ubuntu

---

## Arquitectura

```text
[Navegador]
      │
      ▼
[BunkerWeb WAF]
      │
      ├────────────► Cliente 1
      │             Aplicación PHP vulnerable
      │
      └────────────► Cliente 2
                    Mini Blog vulnerable

Servicios auxiliares:

- BunkerWeb UI
- BunkerWeb Scheduler
- MariaDB
```

---

## Despliegue

Clonar repositorio:

```bash
git clone <repositorio>
cd proyecto
```

Levantar servicios:

```bash
docker compose up -d
```

Verificar estado:

```bash
docker compose ps
```

### Estado de contenedores

<img width="886" height="614" alt="image" src="https://github.com/user-attachments/assets/0add5431-d8d2-457e-a90b-d0e237e258bd" />


---

## Servicios desplegados

| Servicio     | Función                          |
| ------------ | -------------------------------- |
| BunkerWeb    | WAF / Reverse Proxy              |
| BunkerWeb UI | Administración                   |
| Scheduler    | Gestión de configuración         |
| MariaDB      | Persistencia                     |
| Cliente 1    | Aplicación vulnerable SQLi y XSS |
| Cliente 2    | Mini Blog vulnerable             |

<img width="847" height="142" alt="image" src="https://github.com/user-attachments/assets/394f5a75-59db-49eb-ae0f-217d2343bb07" />

---

## Aplicación Cliente 1

Aplicación PHP vulnerable utilizada para pruebas de:

* SQL Injection
* XSS Reflectivo

<img width="879" height="869" alt="image" src="https://github.com/user-attachments/assets/74427afb-4202-4e0c-9c20-7899aa39acd0" />


---

## Aplicación Cliente 2

Mini Blog PHP utilizado para pruebas adicionales de filtrado y protección.

<img width="892" height="889" alt="image" src="https://github.com/user-attachments/assets/59598b11-398a-47cd-87e7-f8cc73cd1ddd" />


---

# Validación del WAF

## SQL Injection

Payload utilizado:

```sql
admin' OR '1'='1
```

Resultado:

* Ataque detectado por ModSecurity.
* Regla OWASP CRS activada.
* Solicitud bloqueada por el WAF.

Respuesta obtenida:

```text
HTTP 403 Forbidden
```

<img width="886" height="764" alt="image" src="https://github.com/user-attachments/assets/15f2c3f7-d915-41cc-9052-7efd99932c51" />


---

## Cross Site Scripting (XSS)

Payload utilizado:

```html
<script>alert('XSS')</script>
```

Resultado:

* Ataque detectado.
* Solicitud bloqueada por el WAF.
* La aplicación vulnerable no llegó a procesar la solicitud.

Respuesta obtenida:

```text
HTTP 403 Forbidden
```

<img width="883" height="766" alt="image" src="https://github.com/user-attachments/assets/b9572429-a2a9-4b9b-a784-d631609b243b" />

---

## Resultados obtenidos

Se logró:

* Implementar un entorno WAF funcional.
* Publicar múltiples aplicaciones detrás de un reverse proxy.
* Configurar ModSecurity y OWASP CRS.
* Detectar y bloquear ataques SQL Injection.
* Detectar y bloquear ataques XSS.
* Validar el funcionamiento del WAF mediante pruebas controladas.

---

## Evidencia de detección del WAF

Los eventos fueron registrados por ModSecurity y OWASP CRS.

La siguiente evidencia muestra la detección de un intento de Cross Site Scripting (XSS), identificado por la regla:

REQUEST-941-APPLICATION-ATTACK-XSS

y posteriormente bloqueado mediante HTTP 403.

<img width="890" height="873" alt="image" src="https://github.com/user-attachments/assets/6f0abfd4-7241-4bf1-a2a5-bf01a3412731" />


--- 

## Conclusiones

Las aplicaciones desplegadas contienen vulnerabilidades deliberadas para fines de laboratorio. Sin embargo, al ser publicadas detrás de BunkerWeb con ModSecurity y OWASP CRS habilitados, los intentos de explotación fueron interceptados y bloqueados antes de alcanzar la lógica vulnerable del backend.
Esto demuestra la utilidad de un Web Application Firewall como capa adicional de defensa para aplicaciones web expuestas a Internet.
