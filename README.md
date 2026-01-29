# 📚 Práctica 03: Infraestructura de Servicios de Red (DNS, LDAP y Proxy)
## IES Luis Vives - Ciclo Superior de DAW

> 🎯 **Objetivo:** Desplegar un ecosistema completo de servicios de red usando contenedores Docker, incluyendo DNS, LDAP, Proxy Inverso y autenticación centralizada.

[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)](https://www.docker.com/)
[![BIND9](https://img.shields.io/badge/DNS-BIND9-0066CC)](https://www.isc.org/bind/)
[![OpenLDAP](https://img.shields.io/badge/Directory-OpenLDAP-00A69C)](https://www.openldap.org/)
[![Nginx](https://img.shields.io/badge/Proxy-Nginx-009639?logo=nginx)](https://nginx.org/)
[![Apache](https://img.shields.io/badge/Server-Apache-D22128?logo=apache)](https://httpd.apache.org/)

---

## 📑 Índice Rápido

- [Descripción General](#-descripción-general)
- [Arquitectura del Sistema](#-arquitectura-del-sistema)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Instalación y Despliegue](#-instalación-y-despliegue)
- [Verificación y Pruebas](#-verificación-y-pruebas)
- [Acceso y Credenciales](#-acceso-y-credenciales)
- [Troubleshooting](#-troubleshooting)
- [Documentación Adicional](#-documentación-adicional)

---

## 🌐 Descripción General

Este proyecto implementa una infraestructura completa de servicios de red para el IES Luis Vives, simulando un entorno empresarial real con:

- **Servidor DNS (BIND9):** Resolución de nombres de dominio local
- **Directorio LDAP (OpenLDAP):** Gestión centralizada de identidades
- **Proxy Inverso (Nginx):** Puerta de entrada única y balanceo de carga
- **Backend DAW (Nginx):** Servidor web ligero para el ciclo DAW
- **Backend DAM (Apache):** Servidor web con autenticación LDAP para el ciclo DAM
- **phpLDAPadmin:** Interfaz web para administración de LDAP

### 🎓 Conceptos Aplicados

- ✅ Resolución DNS directa e inversa (registros A, PTR, CNAME)
- ✅ Servicios de directorio y autenticación centralizada (LDAP)
- ✅ Proxy inverso y enrutamiento HTTP
- ✅ Autenticación LDAP en servidores web (mod_authnz_ldap)
- ✅ Orquestación multi-servicio con Docker Compose
- ✅ Redes personalizadas con IPs estáticas
- ✅ Gestión de volúmenes y configuraciones

---

## 🏗️ Arquitectura del Sistema

El sistema implementa la siguiente topología de red:

```
Internet (localhost:80)
          ↓
    Proxy Inverso (Nginx)
    192.168.200.10
          ↓
    ┌─────┴─────┐
    ↓           ↓
Backend DAW  Backend DAM
  (Nginx)     (Apache + LDAP Auth)
192.168.200.20  192.168.200.30
                     ↓
              OpenLDAP Server
              192.168.200.3
```

**📖 Para ver la arquitectura completa detallada, consulta:** [ARQUITECTURA.md](./ARQUITECTURA.md)

### Componentes del Sistema

| Servicio | Tecnología | IP | Puerto | Función |
|----------|-----------|-----|--------|---------|
| **DNS** | BIND9 | 192.168.200.2 | 53 | Resolución de nombres |
| **LDAP** | OpenLDAP | 192.168.200.3 | 389 | Directorio de usuarios |
| **phpLDAPadmin** | PHP | DHCP | 8080 | Admin web LDAP |
| **Proxy** | Nginx | 192.168.200.10 | 80 | Proxy inverso |
| **Backend DAW** | Nginx | 192.168.200.20 | - | Servidor DAW |
| **Backend DAM** | Apache 2.4 | 192.168.200.30 | - | Servidor DAM |

---

## 📋 Estructura del Proyecto

```
PracticaServiciosRed/
├── docker-compose.yml          # Orquestación de todos los servicios
├── dns/
│   ├── named.conf.local        # Configuración de zonas DNS
│   ├── db.iesluisvives.org     # Zona directa
│   └── db.192.168.200          # Zona inversa
├── ldap/
│   ├── Dockerfile              # Build personalizado de OpenLDAP
│   └── users.ldif              # Usuarios y grupos LDAP
├── nginx-proxy/
│   ├── default.conf            # Configuración del proxy inverso
│   └── .htpasswd               # Credenciales para /daw/calificaciones
├── backend-daw/
│   ├── index.html              # Página principal DAW
│   └── calificaciones/
│       └── index.html          # Página de calificaciones DAW
└── backend-dam/
    ├── Dockerfile              # Apache con módulos LDAP
    ├── index.html              # Página principal DAM
    └── calificaciones/
        ├── index.html          # Página de calificaciones DAM
        └── .htaccess           # Autenticación LDAP para DAM
```

---

## 🚀 Instalación y Despliegue

### Prerrequisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop) instalado y en ejecución
- Docker Compose v2.0+
- PowerShell 5.1+ o PowerShell Core 7+
- Navegador web moderno
- Permisos de administrador (para editar archivo hosts)

### Paso 1: Clonar o Descargar el Proyecto

```powershell
cd C:\Users\angel\Desktop\2DAW\Despliegue
cd PracticaServiciosRed
```

### Paso 2: Levantar la Infraestructura

```powershell
# Construir y levantar todos los servicios
docker-compose up -d --build
```

**Tiempo estimado:** 2-3 minutos (primera vez, descarga de imágenes)

### Paso 3: Verificar Estado de Contenedores

```powershell
# Ver todos los servicios activos
docker-compose ps
```

Deberías ver 6 contenedores en estado **Up**:
- ✅ dns_server
- ✅ ldap_server  
- ✅ phpldapadmin
- ✅ server_daw
- ✅ server_dam
- ✅ proxy_main

### Paso 4: Configurar Resolución de Nombres (Opción A - hosts)

**Editar como Administrador:** `C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1       iesluisvives.org
```

**Opción B - Configurar DNS real:**

1. Panel de control → Redes e Internet → Conexiones de red
2. Click derecho en tu adaptador → Propiedades
3. IPv4 → Propiedades
4. DNS preferido: `127.0.0.1`
5. DNS alternativo: `8.8.8.8`

### Paso 5: Ejecutar Verificación Automática

```powershell
# Script de verificación completa
.\verificar.ps1
```

---

## 🧪 Verificación y Pruebas

---

## 🔐 Acceso y Credenciales

### Acceso Web Principal

| URL | Descripción | Autenticación |
|-----|-------------|---------------|
| http://iesluisvives.org/ | Página principal del proxy | ❌ No requiere |
| http://iesluisvives.org/daw | Portal DAW | ❌ No requiere |
| http://iesluisvives.org/dam | Portal DAM | ❌ No requiere |
| http://iesluisvives.org/daw/calificaciones/ | 📊 Calificaciones DAW | ✅ Auth Basic |
| http://iesluisvives.org/dam/calificaciones/ | 📊 Calificaciones DAM | ✅ Auth LDAP |
| http://localhost:8080 | phpLDAPadmin | ✅ Admin LDAP |

### 👤 Credenciales de Usuario (LDAP)

**Para acceder a las calificaciones:**

```
Usuario:    alumno1
Contraseña: LuisVives123!
```

Estas credenciales funcionan tanto en:
- `/daw/calificaciones/` (autenticación contra .htpasswd)
- `/dam/calificaciones/` (autenticación contra LDAP)

### 🔑 Credenciales de Administrador

**phpLDAPadmin (http://localhost:8080):**

```
Login DN:   cn=admin,dc=iesluisvives,dc=org
Password:   admin
```

**LDAP (desde línea de comandos):**

```bash
Base DN:     dc=iesluisvives,dc=org
Bind DN:     cn=admin,dc=iesluisvives,dc=org
Bind Pass:   admin
URL:         ldap://ldap:389 (interno) o ldap://localhost:389 (host)
```

---

## 🧪 Tests de Verificación Manual

### ✅ Test 1: Acceso Público a DAW
### ✅ Test 1: Acceso Público a DAW

**Navegador:** http://iesluisvives.org/daw

**PowerShell:**
```powershell
curl http://localhost/daw
```

**✅ Resultado esperado:** Página "Bienvenido a DAW - IES Luis Vives"

---

### ✅ Test 2: Acceso Público a DAM

**Navegador:** http://iesluisvives.org/dam

**PowerShell:**
```powershell
curl http://localhost/dam
```

**✅ Resultado esperado:** Página "Bienvenido a DAM - IES Luis Vives"

---

### ✅ Test 3: Calificaciones DAW (Auth Basic - .htpasswd)

**Navegador:** http://iesluisvives.org/daw/calificaciones/

1. Se abre ventana de autenticación
2. Usuario: `alumno1`
3. Contraseña: `LuisVives123!`
4. ✅ Acceso concedido → Tabla de calificaciones

**PowerShell:**
```powershell
# Sin credenciales (debe fallar con 401)
curl http://localhost/daw/calificaciones/

# Con credenciales (debe funcionar)
$secpasswd = ConvertTo-SecureString "LuisVives123!" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("alumno1", $secpasswd)
Invoke-WebRequest -Uri "http://localhost/daw/calificaciones/" -Credential $credential
```

---

### ✅ Test 4: Calificaciones DAM (Auth LDAP Real)

**Navegador:** http://iesluisvives.org/dam/calificaciones/

1. Se abre ventana de autenticación
2. Usuario: `alumno1`
3. Contraseña: `LuisVives123!`
4. Apache verifica contra servidor LDAP
5. ✅ Acceso concedido → Tabla de calificaciones

**PowerShell:**
```powershell
# Con credenciales LDAP
$secpasswd = ConvertTo-SecureString "LuisVives123!" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("alumno1", $secpasswd)
Invoke-WebRequest -Uri "http://localhost/dam/calificaciones/" -Credential $credential
```

---

### ✅ Test 5: Resolución DNS

**Verificar zona directa:**
```powershell
docker exec dns_server host iesluisvives.org 127.0.0.1
docker exec dns_server host ldap.iesluisvives.org 127.0.0.1
docker exec dns_server host dns.iesluisvives.org 127.0.0.1
```

**Verificar zona inversa (PTR):**
```powershell
docker exec dns_server host 192.168.200.2 127.0.0.1
docker exec dns_server host 192.168.200.3 127.0.0.1
docker exec dns_server host 192.168.200.10 127.0.0.1
```

**✅ Resultado esperado:**
```
iesluisvives.org has address 192.168.200.10
ldap.iesluisvives.org has address 192.168.200.3
2.200.168.192.in-addr.arpa domain name pointer ns1.iesluisvives.org.
```

---

### ✅ Test 6: Estructura LDAP

**Buscar usuario en LDAP:**
```powershell
docker exec ldap_server ldapsearch -x -H ldap://localhost `
  -b "dc=iesluisvives,dc=org" `
  -D "cn=admin,dc=iesluisvives,dc=org" `
  -w admin "(uid=alumno1)"
```

**✅ Resultado esperado:**
```
dn: uid=alumno1,ou=users,dc=iesluisvives,dc=org
objectClass: inetOrgPerson
uid: alumno1
cn: Alumno Uno
sn: Vives
mail: alumno1@iesluisvives.org
```

**Interfaz Web:**
1. http://localhost:8080
2. Login: `cn=admin,dc=iesluisvives,dc=org` / `admin`
3. Navegar: `dc=iesluisvives,dc=org` → `ou=users` → `alumno1`

---

## 🐛 Troubleshooting

---

## 🐛 Troubleshooting

### ❌ Problema: "Connection refused" al acceder a iesluisvives.org

**Causas posibles:**
1. Archivo hosts no configurado
2. Proxy no está en ejecución

**Soluciones:**
```powershell
# Verificar que el proxy está activo
docker-compose ps proxy

# Usar localhost como alternativa
curl http://localhost/daw

# Verificar archivo hosts (debe tener la línea)
notepad C:\Windows\System32\drivers\etc\hosts
```

---

### ❌ Problema: Autenticación en /daw/calificaciones falla continuamente

**Causa:** Hash de contraseña incorrecto en `.htpasswd`

**Solución:**
```powershell
# Regenerar hash correcto
docker exec server_dam htpasswd -nb alumno1 'LuisVives123!'

# El resultado debe copiarse a nginx-proxy/.htpasswd
# Luego reiniciar el proxy
docker-compose restart proxy
```

---

### ❌ Problema: Backend DAM no autentica con LDAP

**Diagnóstico:**
```powershell
# 1. Verificar que LDAP está activo
docker-compose ps ldap

# 2. Ver logs de Apache
docker logs server_dam

# 3. Verificar conectividad desde Apache a LDAP
docker exec server_dam ping -c 2 ldap

# 4. Probar búsqueda LDAP directa
docker exec ldap_server ldapsearch -x -H ldap://localhost `
  -b "dc=iesluisvives,dc=org" -D "cn=admin,dc=iesluisvives,dc=org" `
  -w admin "(uid=alumno1)"
```

**Soluciones:**
- Verificar que el archivo `.htaccess` está en `backend-dam/calificaciones/`
- Comprobar que el módulo está activo: `docker exec server_dam apachectl -M | grep ldap`
- Reiniciar backend DAM: `docker-compose restart backend-dam`

---

### ❌ Problema: DNS no resuelve nombres

**Diagnóstico:**
```powershell
# Verificar contenedor DNS
docker-compose ps dns

# Ver logs
docker logs dns_server

# Probar resolución desde dentro del contenedor
docker exec dns_server host iesluisvives.org 127.0.0.1
```

**Soluciones:**
```powershell
# Reiniciar DNS
docker-compose restart dns

# Verificar archivos de zona
docker exec dns_server cat /etc/bind/zones/db.iesluisvives.org

# Verificar sintaxis de BIND
docker exec dns_server named-checkzone iesluisvives.org /etc/bind/zones/db.iesluisvives.org
```

---

### ❌ Problema: Contenedores no inician correctamente

**Solución nuclear (reinicio completo):**
```powershell
# Detener y eliminar todo
docker-compose down -v

# Limpiar imágenes y caché
docker system prune -f

# Reconstruir desde cero
docker-compose up -d --build

# Esperar 10 segundos y verificar
Start-Sleep -Seconds 10
docker-compose ps
```

---

### ❌ Problema: Puerto 80 ya está en uso

**Causa:** Otro servicio (IIS, Apache local) usa el puerto 80

**Soluciones:**

**Opción 1 - Cambiar puerto del proxy:**
Editar `docker-compose.yml`:
```yaml
proxy:
  ports:
    - "8000:8080"  # Cambiar 80 por 8000
```

Luego acceder a: http://localhost:8000/daw

**Opción 2 - Detener servicio conflictivo:**
```powershell
# Ver qué proceso usa el puerto 80
netstat -ano | findstr :80

# Detener IIS (si está instalado)
Stop-Service W3SVC
```

---

### 🔄 Comandos Útiles de Mantenimiento

```powershell
# Ver todos los contenedores y su estado
docker-compose ps

# Ver logs en tiempo real de un servicio
docker-compose logs -f proxy
docker-compose logs -f backend-dam

# Reiniciar un servicio específico
docker-compose restart ldap

# Entrar a un contenedor (shell interactivo)
docker exec -it server_dam bash
docker exec -it dns_server bash

# Ver uso de recursos
docker stats

# Limpiar volúmenes huérfanos
docker volume prune
```

---

## 📚 Documentación Adicional

### 📄 Archivos de Documentación

| Archivo | Descripción |
|---------|-------------|
| [README.md](./README.md) | **Este archivo** - Guía principal |
| [ARQUITECTURA.md](./ARQUITECTURA.md) | 🏗️ Diagrama detallado de componentes y flujos |
| [CORRECCIONES.md](./CORRECCIONES.md) | 🔧 Problemas detectados y soluciones aplicadas |
| [GUIA_PROFESOR.md](./GUIA_PROFESOR.md) | 🎓 Guía rápida de evaluación para profesores |

### 🔗 Referencias Técnicas

- **DNS (BIND9):** [Documentación oficial ISC BIND](https://bind9.readthedocs.io/)
- **LDAP:** [OpenLDAP Admin Guide](https://www.openldap.org/doc/admin25/)
- **Apache mod_authnz_ldap:** [Apache LDAP Auth](https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html)
- **Nginx Proxy:** [Nginx Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- **Docker Compose:** [Compose Specification](https://docs.docker.com/compose/compose-file/)

### 🎯 Temario Aplicado

Este proyecto implementa los conceptos del tema:
- **6.1** - DNS con BIND9: Zonas directas e inversas
- **6.2** - Servicios de Directorio LDAP
- **6.3** - Integración LDAP en aplicaciones
- **6.4** - Autenticación LDAP en servidores web (Apache mod_authnz_ldap)
- **6.5** - Orquestación con Docker Compose

---

## 📊 Criterios de Evaluación

| Criterio | Puntos | Verificación | Cumplido |
|----------|--------|--------------|----------|
| **DNS resuelve todos los nombres correctamente** | 2p | Zonas directa e inversa funcionales | ✅ |
| **LDAP con estructura correcta y usuario funcional** | 2p | phpLDAPadmin + ldapsearch | ✅ |
| **Proxy redirige /daw y /dam correctamente** | 2p | Navegador accede a ambas rutas | ✅ |
| **Auth LDAP bloquea /dam/calificaciones efectivamente** | 3p | Solo accesible con usuario LDAP válido | ✅ |
| **Docker Compose: redes, volúmenes, depends_on** | 1p | Código limpio y profesional | ✅ |
| **TOTAL** | **10p** | | ✅ |

---

## 📝 Notas Importantes

### 🔒 Seguridad

- ⚠️ **HTTP sin cifrar:** Las credenciales viajan en Base64. En producción usar **HTTPS obligatorio**.
- ⚠️ **Contraseñas por defecto:** Cambiar `admin` en entornos reales.
- ✅ **Volúmenes read-only:** Previene modificaciones accidentales.
- ✅ **Red aislada:** Los backends no están expuestos directamente al host.

### 🔄 Persistencia de Datos

Los datos de LDAP se pierden al ejecutar `docker-compose down -v`. Para persistencia real:

```yaml
ldap:
  volumes:
    - ldap-data:/var/lib/ldap
    - ldap-config:/etc/ldap/slapd.d

volumes:
  ldap-data:
  ldap-config:
```

### ⚠️ Limitación Conocida: Auth LDAP en Nginx

La imagen `bitnami/nginx` **no incluye** el módulo `nginx-auth-ldap` por defecto.

**Implementado:** `/daw/calificaciones` usa `auth_basic` con `.htpasswd`  
**LDAP Real:** `/dam/calificaciones` usa Apache con `mod_authnz_ldap` ✅

Para LDAP real en Nginx se requiere:
- Compilar Nginx con módulo `nginx-auth-ldap`, o
- Usar `nginx-extras` en Debian/Ubuntu

---

## 👨‍💻 Información del Proyecto

**Centro:** IES Luis Vives  
**Ciclo:** Desarrollo de Aplicaciones Web (DAW)  
**Módulo:** Despliegue de Aplicaciones Web  
**Práctica:** 03 - Servicios de Red e Infraestructura

**Fecha:** Enero 2026  
**Versión:** 1.0

---

## 🎉 ¡Práctica Completada!

Si has llegado hasta aquí y todos los tests pasan, **¡enhorabuena!** Has desplegado exitosamente una infraestructura completa de servicios de red con:

✅ DNS (BIND9)  
✅ LDAP (OpenLDAP)  
✅ Proxy Inverso (Nginx)  
✅ Autenticación LDAP (Apache)  
✅ Orquestación Docker Compose

**Para cualquier duda, consulta:**
- 📖 [ARQUITECTURA.md](./ARQUITECTURA.md) - Diagramas detallados
- 🔧 [CORRECCIONES.md](./CORRECCIONES.md) - Problemas comunes resueltos
- 🎓 [GUIA_PROFESOR.md](./GUIA_PROFESOR.md) - Evaluación rápida

---

<div align="center">

**Hecho con ❤️ y ☕ por Ángel Sánchez Gasanz**

[🔝 Volver arriba](#-práctica-03-infraestructura-de-servicios-de-red-dns-ldap-y-proxy)

</div>
