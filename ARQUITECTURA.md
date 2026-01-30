# 🏗️ ARQUITECTURA DETALLADA DEL SISTEMA

## Vista General de Componentes

```
┌───────────────────────────────────────────────────────────────────┐
│                        INTERNET / LOCALHOST                       │
│                                                                   │
│  Browser/Client                                                   │
│      ↓                                                            │
│  http://iesluisvives.org                                          │
└──────────────────────────────────┬────────────────────────────────┘
                                   │ Puerto 80
                                   ↓
┌──────────────────────────────────────────────────────────────────┐
│                    PROXY INVERSO (Nginx Bitnami)                 │
│                        192.168.200.10:8080                       │
│                                                                  │
│  Rutas Configuradas:                                             │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ /daw         → proxy_pass backend-daw:80                 │    │
│  │ /daw/calificaciones → Auth Basic (.htpasswd) + proxy     │    │
│  │ /dam         → proxy_pass backend-dam:80                 │    │
│  │ /dam/calificaciones → proxy_pass (Auth en Apache)        │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────┬─────────────────────────┬──────────────────────────┘
              │                         │
              ↓                         ↓
┌─────────────────────────┐  ┌─────────────────────────────┐
│  BACKEND DAW (Nginx)    │  │  BACKEND DAM (Apache)       │
│  192.168.200.20:80      │  │  192.168.200.30:80          │
│                         │  │                             │
│  Archivos:              │  │  Módulos Activos:           │
│  ├─ index.html          │  │  ├─ mod_ldap.so             │
│  └─ calificaciones/     │  │  ├─ mod_authnz_ldap.so      │
│     └─ index.html       │  │  └─ mod_rewrite.so          │
│                         │  │                             │
│  ⚠️ Auth Basic Proxy    │  │  Archivos:                  │
│                         │  │  ├─ index.html              │
│                         │  │  └─ calificaciones/         │
│                         │  │     ├─ index.html           │
│                         │  │     └─ .htaccess            │
│                         │  │        └─ ✅ Auth LDAP Real │
└─────────────────────────┘  └──────────────┬──────────────┘
                                            │
                                            │ AuthLDAPURL
                                            ↓
                              ┌──────────────────────────────┐
                              │   SERVIDOR LDAP (OpenLDAP)   │
                              │   192.168.200.3:389          │
                              │                              │
                              │  Estructura DIT:             │
                              │  dc=iesluisvives,dc=org      │
                              │  ├─ ou=users                 │
                              │  │  └─ uid=alumno1           │
                              │  │     ├─ cn: Alumno Uno     │
                              │  │     ├─ sn: Vives          │
                              │  │     ├─ mail: @...         │
                              │  │     └─ userPassword: ***  │
                              │  └─ ou=groups                │
                              │     └─ cn=alumnos            │
                              │        └─ member: alumno1    │
                              └──────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              SERVICIOS AUXILIARES Y ADMINISTRACIÓN                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  📡 DNS (BIND9)                  🖥️ phpLDAPadmin                    │
│  192.168.200.2:53                localhost:8080                     │
│                                                                     │
│  Zonas Configuradas:             Interface Web:                     │
│  ├─ Directa: db.iesluisvives.org Login: cn=admin,dc=...             │
│  │  ├─ @ → 192.168.200.10                                           │
│  │  ├─ ns1 → 192.168.200.2                                          │
│  │  ├─ ldap → 192.168.200.3     Permite:                            │
│  │  ├─ proxy → 192.168.200.10   - Ver estructura DIT                │
│  │  ├─ daw → 192.168.200.20     - Gestionar usuarios                │
│  │  └─ dam → 192.168.200.30     - Validar configuración             │
│  └─ Inversa: db.192.168.200                                         │
│     ├─ 2 → ns1.iesluisvives.org                                     │
│     ├─ 3 → ldap.iesluisvives.org                                    │
│     └─ 10 → proxy.iesluisvives.org                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Flujo de Autenticación - Caso de Uso Real

### 📊 Caso 1: Acceso a /daw/calificaciones

```
Usuario (Navegador)
    ↓
    1. GET http://iesluisvives.org/daw/calificaciones/
    ↓
Proxy (192.168.200.10)
    ↓
    2. Detecta directiva auth_basic
    ↓
    3. Responde: 401 Unauthorized
       WWW-Authenticate: Basic realm="Área Restringida DAW"
    ↓
Usuario introduce: alumno1 / LuisVives123!
    ↓
    4. Reenvía con header: Authorization: Basic YWx1bW5vMTp...
    ↓
Proxy verifica contra: /opt/bitnami/nginx/conf/.htpasswd
    ↓
    5. ✅ Credenciales válidas
    ↓
    6. proxy_pass http://backend-daw:80/calificaciones/
    ↓
Backend DAW (192.168.200.20)
    ↓
    7. Devuelve: /usr/share/nginx/html/calificaciones/index.html
    ↓
    8. 200 OK → Contenido de calificaciones
```

---

### 🔐 Caso 2: Acceso a /dam/calificaciones (Auth LDAP Real)

```
Usuario (Navegador)
    ↓
    1. GET http://iesluisvives.org/dam/calificaciones/
    ↓
Proxy (192.168.200.10)
    ↓
    2. proxy_pass http://backend-dam:80/calificaciones/
       (pasa cabecera Authorization)
    ↓
Backend DAM - Apache (192.168.200.30)
    ↓
    3. Lee: /usr/local/apache2/htdocs/calificaciones/.htaccess
       AuthType Basic
       AuthBasicProvider ldap
       AuthLDAPURL "ldap://ldap:389/ou=users,dc=iesluisvives,dc=org?uid?sub"
    ↓
    4. Responde: 401 Unauthorized
       WWW-Authenticate: Basic realm="Acceso Restringido - Alumnos DAM"
    ↓
Usuario introduce: alumno1 / LuisVives123!
    ↓
    5. Reenvía con header: Authorization: Basic YWx1bW5vMTp...
    ↓
Apache extrae credenciales y consulta LDAP
    ↓
    6. BIND a LDAP:
       - Servidor: ldap:389
       - BindDN: cn=admin,dc=iesluisvives,dc=org
       - BindPassword: admin
    ↓
    7. SEARCH en LDAP:
       - Base: ou=users,dc=iesluisvives,dc=org
       - Filter: (uid=alumno1)
    ↓
LDAP (192.168.200.3)
    ↓
    8. Encuentra: uid=alumno1,ou=users,dc=iesluisvives,dc=org
    ↓
    9. Valida userPassword: LuisVives123!
    ↓
    10. ✅ LDAP responde: Autenticación exitosa
    ↓
Apache valida: Require valid-user → OK
    ↓
    11. Devuelve: /usr/local/apache2/htdocs/calificaciones/index.html
    ↓
    12. 200 OK → Contenido de calificaciones
```

---

## Red Docker - Topología

```
Red: vives-net (192.168.200.0/24)
├─ Driver: bridge
├─ Subnet: 192.168.200.0/24
└─ IPs Estáticas Asignadas:
   ├─ 192.168.200.2  → dns_server (BIND9)
   ├─ 192.168.200.3  → ldap_server (OpenLDAP)
   ├─ 192.168.200.10 → proxy_main (Nginx Proxy)
   ├─ 192.168.200.20 → server_daw (Nginx Backend)
   └─ 192.168.200.30 → server_dam (Apache Backend)

Servicios sin IP fija (DHCP interno):
├─ phpldapadmin → Resuelve 'ldap' por nombre
└─ [Cualquier servicio futuro]
```

---

## Dependencias de Inicio (depends_on)

```
Orden de Arranque:

1. dns (No depende de nadie)
   ↓
2. ldap (depends_on: dns)
   ↓
3. phpldapadmin (depends_on: ldap)
   
   backend-daw (depends_on: dns)
   backend-dam (depends_on: dns, ldap)
   ↓
4. proxy (depends_on: backend-daw, backend-dam, ldap, dns)
```

**Razón:** El proxy necesita que todos los backends estén activos antes de intentar hacer proxy_pass.

---

## Volúmenes y Persistencia

| Servicio | Volumen Host | Volumen Contenedor | Modo | Propósito |
|----------|--------------|-------------------|------|-----------|
| dns | `./dns/named.conf.local` | `/etc/bind/named.conf.local` | ro | Configuración de zonas |
| dns | `./dns/db.iesluisvives.org` | `/etc/bind/zones/db.iesluisvives.org` | ro | Zona directa |
| dns | `./dns/db.192.168.200` | `/etc/bind/zones/db.192.168.200` | ro | Zona inversa |
| ldap | `./ldap/users.ldif` | `/container/.../bootstrap/ldif/` | - | Usuarios iniciales |
| proxy | `./nginx-proxy/default.conf` | `/opt/bitnami/.../my_server_block.conf` | ro | Configuración proxy |
| proxy | `./nginx-proxy/.htpasswd` | `/opt/bitnami/nginx/conf/.htpasswd` | ro | Credenciales auth |
| backend-daw | `./backend-daw/` | `/usr/share/nginx/html` | ro | Contenido web DAW |
| backend-dam | `./backend-dam/` | `/usr/local/apache2/htdocs` | ro | Contenido web DAM |

**Nota:** `ro` (read-only) previene modificaciones accidentales desde el contenedor.

---

## Puertos Expuestos al Host

| Servicio | Puerto Interno | Puerto Host | Protocolo | Uso |
|----------|---------------|-------------|-----------|-----|
| dns | 53 | 53 | UDP/TCP | Consultas DNS |
| ldap | 389 | 389 | TCP | Consultas LDAP |
| phpldapadmin | 80 | 8080 | HTTP | Admin web LDAP |
| proxy | 8080 | 80 | HTTP | Entrada principal |

**Servicios sin puerto expuesto:**
- backend-daw: Solo accesible internamente
- backend-dam: Solo accesible internamente

---

## Variables de Entorno Críticas

### LDAP
```yaml
LDAP_DOMAIN: iesluisvives.org
  → Genera: dc=iesluisvives,dc=org

LDAP_ADMIN_PASSWORD: admin
  → Usuario: cn=admin,dc=iesluisvives,dc=org

LDAP_ORGANISATION: "IES Luis Vives"
  → Campo descriptivo en la base
```

### phpLDAPadmin
```yaml
PHPLDAPADMIN_LDAP_HOSTS: ldap
  → Conecta al servicio 'ldap' por nombre
```

---

## Archivos de Configuración Clave

### 1. Apache - .htaccess (Autenticación LDAP)
```apache
AuthName "Acceso Restringido - Alumnos DAM"
AuthType Basic
AuthBasicProvider ldap

AuthLDAPURL "ldap://ldap:389/ou=users,dc=iesluisvives,dc=org?uid?sub?(objectClass=inetOrgPerson)"
AuthLDAPBindDN "cn=admin,dc=iesluisvives,dc=org"
AuthLDAPBindPassword "admin"

Require valid-user
```

### 2. Nginx Proxy - default.conf
```nginx
location /dam/ {
    proxy_pass http://backend-dam:80/;
    
    # CRÍTICO: Pasar autenticación HTTP Basic
    proxy_set_header Authorization $http_authorization;
    proxy_pass_header Authorization;
}
```

### 3. DNS - db.iesluisvives.org
```bind
@       IN      NS      ns1.iesluisvives.org.
ns1     IN      A       192.168.200.2
ldap    IN      A       192.168.200.3
@       IN      A       192.168.200.10
```

---

## Seguridad Implementada

✅ **Autenticación LDAP real** en Apache (mod_authnz_ldap)  
✅ **Autenticación básica** en Nginx proxy (.htpasswd)  
✅ **Volúmenes read-only** para prevenir modificaciones  
✅ **Red aislada** (192.168.200.0/24) sin acceso externo  
✅ **Backends no expuestos** directamente al host  
⚠️ **Sin HTTPS** (⚠️ En producción usar SSL/TLS)

---

## Escalabilidad Futura

Para expandir el sistema:

```yaml
# Añadir nuevo backend
backend-dwec:
  image: nginx:alpine
  networks:
    vives-net:
      ipv4_address: 192.168.200.40
  
# Actualizar proxy
location /dwec {
    proxy_pass http://backend-dwec:80/;
}

# Actualizar DNS
dwec    IN      A       192.168.200.40
```

---

**Diagrama actualizado:** 30 de enero de 2026  
**Práctica:** 03 - Servicios de Red  
**IES Luis Vives - Ciclo DAW**

<div align="center">

**Hecho con ❤️ y ☕ por Ángel Sánchez Gasanz**


</div>
