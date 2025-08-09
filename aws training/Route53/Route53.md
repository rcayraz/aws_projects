# Amazon Route 53 - Guía Completa para Arquitectos

## 📋 Índice
1. [Introducción al Sistema DNS](#introducción-al-sistema-dns)
2. [Amazon Route 53 - Visión General](#amazon-route-53---visión-general)
3. [Registros DNS y Zonas de Alojamiento](#registros-dns-y-zonas-de-alojamiento)
4. [Políticas de Enrutamiento](#políticas-de-enrutamiento)
5. [Controles de Salud (Health Checks)](#controles-de-salud-health-checks)
6. [Registro de Dominios](#registro-de-dominios)
7. [Mejores Prácticas para Arquitectos](#mejores-prácticas-para-arquitectos)
8. [Tips para el Examen SAA-C03](#tips-para-el-examen-saa-c03)

---

## 🌐 Introducción al Sistema DNS

### ¿Qué es DNS?
El **Sistema de Nombres de Dominio (DNS)** traduce nombres de host amigables para humanos en direcciones IP de las máquinas.

```
www.google.com => 172.217.18.36
```

> 💡 **DNS es la columna vertebral de Internet** y utiliza una estructura jerárquica de nombres.

### Estructura Jerárquica DNS

```
http://api.www.example.com.
|      |   |   |       |   |
|      |   |   |       |   └── Raíz
|      |   |   |       └────── TLD (.com)
|      |   |   └────────────── SLD (example)
|      |   └────────────────── Subdominio (www)
|      └────────────────────── Subdominio (api)
└───────────────────────────── Protocolo
```

### Terminología DNS Fundamental

| Término | Descripción | Ejemplo |
|---------|-------------|---------|
| **Registrador de Dominio** | Servicio que registra dominios | Amazon Route 53, GoDaddy |
| **Registros DNS** | Tipos de entradas DNS | A, AAAA, CNAME, NS |
| **Archivo de Zona** | Contiene registros DNS | zona-example.com |
| **Servidor de Nombres** | Resuelve consultas DNS | ns-123.awsdns-12.com |
| **TLD** | Dominio de Primer Nivel | .com, .org, .gov |
| **SLD** | Dominio de Segundo Nivel | amazon.com, google.com |

### 🏗️ Flujo de Resolución DNS

```
┌─────────────┐    1. example.com    ┌─────────────────┐
│  Navegador  │ ────────────────────→│ DNS Local       │
│   Cliente   │                      │ (ISP/Empresa)   │
└─────────────┘                      └─────────┬───────┘
       ↑                                       │ 2. Consulta
       │ 8. Respuesta IP                       ↓
       │                              ┌─────────────────┐
       │                              │ Servidor DNS    │
       │                              │ Raíz (ICANN)    │
       │                              └─────────┬───────┘
       │                                       │ 3. Consulta .com
       │                                       ↓
       │                              ┌─────────────────┐
       │                              │ Servidor TLD    │
       │                              │ (.com)          │
       │                              └─────────┬───────┘
       │                                       │ 4. Consulta example.com
       │                                       ↓
       │                              ┌─────────────────┐
       │                              │ Servidor        │
       │                              │ Autoritativo    │
       │                              │ (Route 53)      │
       │                              └─────────┬───────┘
       │                                       │ 5. IP: 9.10.11.12
       │                                       ↓
┌─────────────┐    7. IP Address     ┌─────────────────┐
│ Servidor    │ ←────────────────────│ Respuesta DNS   │
│ Web         │    6. Respuesta      │ Completa        │
│ 9.10.11.12  │                      └─────────────────┘
└─────────────┘
```

---

## ☁️ Amazon Route 53 - Visión General

### ¿Qué es Route 53?

Amazon Route 53 es un **servicio DNS altamente disponible, totalmente gestionado y autoritativo**.

```
✅ Autoritativo: El cliente puede actualizar los registros DNS
✅ Alta Disponibilidad: SLA del 100% (único servicio AWS)
✅ Escalabilidad Global: Red global de servidores DNS
✅ Integración AWS: Conexión nativa con servicios AWS
```

### 🏗️ Arquitectura Route 53

```
┌─────────────┐    DNS Query     ┌─────────────────┐    Route to    ┌─────────────┐
│   Cliente   │ ───────────────→ │   Route 53      │ ─────────────→ │ AWS Resources│
│  (Browser)  │                  │  DNS Service    │                │  (EC2, ELB)  │
└─────────────┘                  └─────────────────┘                └─────────────┘
                                          │
                                          ↓
                                 ┌─────────────────┐
                                 │ Health Checks   │
                                 │ (Monitoring)    │
                                 └─────────────────┘
```

### ¿Por qué Route 53?
> 🔢 **Puerto DNS tradicional: 53** - De ahí el nombre del servicio

### Funciones Principales
1. **Servicio DNS**: Resolución de nombres a IPs
2. **Registrador de Dominios**: Compra y gestión de dominios
3. **Health Checks**: Monitoreo de recursos
4. **Traffic Flow**: Políticas complejas de enrutamiento

---

## 📝 Registros DNS y Zonas de Alojamiento

### 🧪 Experimento Inicial: Configuración de Laboratorio

#### Preparación del Entorno de Pruebas
```bash
# PASO 1: Registrar dominio (ejemplo: ayraz.com)
# Consideración: Esto requiere pago

# PASO 2: Crear 3 instancias EC2 en diferentes regiones
# - EEUU (us-east-1)
# - Europa (eu-west-1) 
# - Asia (ap-southeast-1)

# Script para cada instancia - Hola Mundo con hostname y AZ
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Obtener metadatos de la instancia
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>Route 53 Lab</title></head>
<body>
    <h1>🌍 Route 53 Testing Lab</h1>
    <h2>Servidor: $REGION</h2>
    <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
    <p><strong>Availability Zone:</strong> $AZ</p>
    <p><strong>Región:</strong> $REGION</p>
    <p><strong>Timestamp:</strong> $(date)</p>
</body>
</html>
EOF

# PASO 3: Crear ALB en una región (ejemplo: us-east-1)
# Target Group: tg_alb_rt53
# Seleccionar la instancia de esa región
```

#### 📋 Inventario de Recursos (Anotar)
```yaml
Instancias EC2:
  EEUU:
    - Instance ID: i-1234567890abcdef0
    - IP Pública: 3.208.123.45
    - Región: us-east-1
    
  Europa:
    - Instance ID: i-0987654321fedcba0  
    - IP Pública: 18.130.234.56
    - Región: eu-west-1
    
  Asia:
    - Instance ID: i-abcdef1234567890
    - IP Pública: 13.250.345.67
    - Región: ap-southeast-1

ALB:
  - DNS Name: alb-rt53-123456789.us-east-1.elb.amazonaws.com
  - Target Group: tg_alb_rt53
  - Región: us-east-1
```

### Estructura de un Registro DNS

Cada registro DNS contiene:

```yaml
Registro DNS:
  Nombre: ejemplo.com
  Tipo: A | AAAA | CNAME | NS
  Valor: 12.34.56.78
  TTL: 300 segundos
  Política: Simple | Weighted | Failover | etc.
```

### Tipos de Registros Soportados

#### Registros Obligatorios
| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| **A** | IPv4 | `example.com → 192.0.2.1` |
| **AAAA** | IPv6 | `example.com → 2001:db8::1` |
| **CNAME** | Alias a otro hostname | `www.example.com → example.com` |
| **NS** | Name servers de la zona | `example.com → ns-123.awsdns-12.com` |

#### Registros Avanzados
| Tipo | Descripción | Uso Común |
|------|-------------|-----------|
| **MX** | Mail exchange | Servidores de email |
| **TXT** | Texto libre | Verificación SPF, DKIM |
| **SRV** | Service records | Servicios específicos |
| **CAA** | Certificate Authority | Autorización SSL |

### 🏗️ Zonas de Alojamiento

#### Zona Pública vs Privada - Diferencias Clave

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZONA PÚBLICA                                 │
│  ┌─────────────────────────────────┐                           │
│  │ example.com                     │                           │
│  │ ├── A: 192.0.2.1               │  ←──────── Internet       │
│  │ ├── AAAA: 2001:db8::1          │            Público        │
│  │ ├── CNAME: www → example        │                           │
│  │ └── MX: mail.example.com        │                           │
│  └─────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        VPC (10.0.0.0/16)                       │
│  ┌─────────────────────────────────┐                           │
│  │ ZONA PRIVADA                    │                           │
│  │ application.company.internal    │  ←──────── Solo VPC       │
│  │ ├── A: api → 10.0.1.100         │            Privada        │
│  │ ├── A: db → 10.0.2.50           │                           │
│  │ └── CNAME: app → api            │                           │
│  └─────────────────────────────────┘                           │
│                                                                 │
│  ┌─────────┐    ┌─────────┐                                    │
│  │   EC2   │    │   RDS   │                                    │
│  │10.0.1.100│   │10.0.2.50│                                    │
│  └─────────┘    └─────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 🧪 Experimento 1: Registro Básico y TTL

#### Crear Registro de Prueba
```bash
# En Route 53 Console:
# 1. Nuevo registro subdominio = test
# 2. Tipo: A
# 3. Valor IP: 11.22.33.55 (o IP de instancia EEUU)
# 4. TTL: 300 segundos
# 5. Tipo direccionamiento: sencillo

# Probar en Cloud Shell
sudo yum install -y bind-utils
nslookup test.ayraz.com
dig test.ayraz.com
```

#### Experimento TTL
```bash
# Crear registro 'demo' con TTL corto
# 1. Registro: demo.ayraz.com
# 2. Tipo: A  
# 3. Valor: IP instancia EEUU
# 4. TTL: 120 segundos

# Probar en navegador - mostrar región EEUU

# ANTES que termine TTL (< 120s), cambiar a IP Europa
# Experimentar el comportamiento del TTL
# Ver cómo tarda en cambiar debido al caché DNS
```

### TTL (Time To Live)

#### ¿Qué es TTL?
El TTL determina cuánto tiempo se almacena un registro DNS en caché.

```
Cliente → Solicitud DNS → Route 53
       ← Respuesta con TTL ←
```

#### Estrategias TTL

| TTL | Ventajas | Desventajas | Caso de Uso |
|-----|----------|-------------|-------------|
| **Alto (24h)** | • Menos tráfico<br>• Menos costo | • Cambios lentos<br>• Registros obsoletos | Servicios estables |
| **Bajo (60s)** | • Cambios rápidos<br>• Flexibilidad | • Más tráfico<br>• Más costo | Deployments frecuentes |

### CNAME vs ALIAS - Diferencias Prácticas

#### Características CNAME
```yaml
CNAME (Canonical Name):
  Limitaciones:
    - NO puede usarse en el apex (example.com)
    - SÍ puede usarse en subdominios (www.example.com)
    - Apunta a otro nombre DNS
    - Tiene costo adicional por resolución
    - Solo para dominios no root

  Ejemplo:
    www.example.com CNAME → example.com
    app.midominio.com CNAME → lb1-1223.us-east-2.elb.amazonaws.com
```

#### Características ALIAS (Exclusivo AWS)
```yaml
ALIAS (AWS Extension):
  Ventajas:
    - SÍ puede usarse en el apex (example.com)
    - Apunta directamente a recursos AWS
    - Sin costo adicional
    - Health checks automáticos
    - Funciona para dominio raíz y no raíz
    - Reconoce automáticamente cambios de IP
    - Siempre tipo A/AAAA para recursos AWS

  Ejemplo:
    example.com ALIAS → my-alb-123456789.us-east-1.elb.amazonaws.com
    app.mydomain.com ALIAS → cloudfront-distribution.amazonaws.com
```

#### � Objetivos ALIAS Soportados
```yaml
Recursos AWS Compatibles:
  ✅ Elastic Load Balancer (ALB/NLB/CLB)
  ✅ CloudFront Distributions  
  ✅ API Gateway
  ✅ Elastic Beanstalk environments
  ✅ S3 buckets (website hosting)
  ✅ VPC Interface endpoints
  ✅ Global Accelerator
  ✅ Route 53 records (misma zona alojada)
  
  ❌ NO se puede: EC2 DNS nombres
```

### 🧪 Experimento 2: CNAME vs ALIAS

#### Parte A: Crear CNAME
```bash
# En Route 53 Console:
# 1. Registro: myapp.ayraz.com
# 2. Tipo: CNAME
# 3. Valor: [ALB-DNS-NAME] (ej: alb-rt53-123.us-east-1.elb.amazonaws.com)
# 4. TTL: 300

# Probar:
curl myapp.ayraz.com
dig CNAME myapp.ayraz.com
```

#### Parte B: Crear ALIAS
```bash
# En Route 53 Console:
# 1. Registro: myalias.ayraz.com
# 2. Tipo: A
# 3. Alias: SÍ
# 4. Buscar: Application Load Balancer
# 5. Región: us-east-1
# 6. Seleccionar: ALB creado anteriormente

# Probar:
curl myalias.ayraz.com
dig A myalias.ayraz.com

# Comparar respuestas - ALIAS devuelve IP directamente
```

### 🏗️ Diagrama CNAME vs ALIAS

```
CNAME (Indirecto):
myapp.ayraz.com ──CNAME──→ alb-123.us-east-1.elb.amazonaws.com ──A──→ 1.2.3.4
     (Subdominio)              (Resolución adicional)              (IP final)
                                      ↑
                              Costo adicional por lookup

ALIAS (Directo):
myalias.ayraz.com ──ALIAS──→ ALB ──→ IP Automática (1.2.3.4)
      (Cualquier nivel)           Sin costo adicional
                                  Health check incluido
```

---

## 🚦 Políticas de Enrutamiento

Las políticas de enrutamiento definen **cómo responde Route 53 a las consultas DNS**.

> ⚠️ **Importante**: No confundir con enrutamiento de tráfico de Load Balancers. DNS solo responde consultas, no enruta tráfico real.

### 1. Enrutamiento Simple

#### Características
- Dirige tráfico a un solo recurso
- Puede especificar múltiples valores en el mismo registro
- El cliente elige uno al azar si hay múltiples valores
- **NO** se puede asociar con Health Checks

#### 🏗️ Diagrama Simple

```
┌─────────────┐    DNS Query     ┌─────────────┐
│   Cliente   │ ──────────────→  │  Route 53   │
└─────────────┘                  └──────┬──────┘
       ↑                                │
       │                                ↓
       │ Respuesta Aleatoria    ┌───────────────┐
       └────────────────────────│ app.example   │
                                │ A: 1.2.3.4    │
                                │ A: 5.6.7.8    │ ← Múltiples IPs
                                │ A: 9.10.11.12 │
                                └───────────────┘
                                        │
                         ┌──────────────┼──────────────┐
                         ↓              ↓              ↓
                  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
                  │    EC2-1    │ │    EC2-2    │ │    EC2-3    │
                  │  1.2.3.4    │ │  5.6.7.8    │ │ 9.10.11.12  │
                  └─────────────┘ └─────────────┘ └─────────────┘
```

#### 🧪 Experimento Simple
```bash
# PASO 1: Crear registro simple
# En Route 53 Console:
# - Nombre: simple.ayraz.com
# - Tipo: A
# - Valor: IP de instancia EEUU (ej: 3.208.123.45)
# - Direccionamiento: sencillo

# Probar en Cloud Shell:
dig simple.ayraz.com

# PASO 2: Editar registro - añadir múltiples valores
# Editar el registro simple.ayraz.com
# Añadir IP de instancia Europa (ej: 18.130.234.56)
# Ahora el registro tiene 2 IPs

# PASO 3: Esperar TTL y probar
# Esperar que expire el TTL
# Probar múltiples veces:
for i in {1..10}; do
  dig +short simple.ayraz.com
  sleep 2
done

# Observar: el cliente elige IP aleatoriamente
```

### 2. Enrutamiento Ponderado (Weighted)

#### Características
- Controla el **porcentaje** de solicitudes que van a cada recurso
- Asigna un peso relativo a cada registro
- Los pesos NO necesitan sumar 100
- **SÍ** se puede asociar con Health Checks

#### Fórmula Matemática
```
Porcentaje de Tráfico = (Peso del Registro / Suma Total de Pesos) × 100

Ejemplo:
Registro A: Peso 10
Registro B: Peso 20  
Registro C: Peso 70
Total: 100

Tráfico A = (10/100) × 100 = 10%
Tráfico B = (20/100) × 100 = 20%
Tráfico C = (70/100) × 100 = 70%
```

#### 🏗️ Diagrama Weighted

```
                              Route 53
                           Política Weighted
                         ┌─────────────────┐
Cliente ──DNS Query──→   │ app.example.com │
                         │                 │
                         │ Weight: 70 (70%)│ ──→ ┌─────────────┐
                         │ Weight: 20 (20%)│ ──→ │    EC2-1    │
                         │ Weight: 10 (10%)│ ──→ │   (Prod)    │
                         └─────────────────┘     │ us-east-1   │
                                  │              └─────────────┘
                                  ↓
                         ┌─────────────────┐     ┌─────────────┐
                         │ Distribución:   │     │    EC2-2    │
                         │ 70% → Prod      │     │   (Test)    │
                         │ 20% → Test      │     │ us-west-2   │
                         │ 10% → Dev       │     └─────────────┘
                         └─────────────────┘
                                                 ┌─────────────┐
                                                 │    EC2-3    │
                                                 │    (Dev)    │
                                                 │  eu-west-1  │
                                                 └─────────────┘
```

#### Casos de Uso Weighted
- **Blue/Green Deployments**: Migración gradual de tráfico
- **A/B Testing**: Distribución controlada para pruebas
- **Load Balancing**: Entre regiones geográficas
- **Canary Releases**: Probar nuevas versiones con poco tráfico

#### 🧪 Experimento Weighted
```bash
# EXPERIMENTO CON LAS 3 INSTANCIAS CREADAS

# REGISTRO 1: EEUU (70% del tráfico)
# En Route 53 Console:
# - Nombre: ponderado.ayraz.com
# - Tipo: A
# - Valor: IP instancia EEUU
# - TTL: 3 segundos
# - Peso: 70
# - ID de registro: eeuu

# REGISTRO 2: Asia (10% del tráfico)  
# - Nombre: ponderado.ayraz.com
# - Tipo: A
# - Valor: IP instancia Asia
# - TTL: 3 segundos
# - Peso: 10  
# - ID de registro: asia

# REGISTRO 3: Europa (20% del tráfico)
# - Nombre: ponderado.ayraz.com
# - Tipo: A
# - Valor: IP instancia Europa
# - TTL: 3 segundos
# - Peso: 20
# - ID de registro: europa

# PRUEBA: Ejecutar múltiples consultas
for i in {1..100}; do
  dig +short ponderado.ayraz.com | tee -a resultados.log
  sleep 1
done

# ANÁLISIS: Contar distribución
grep -c "3.208.123.45" resultados.log    # EEUU ~70
grep -c "13.250.345.67" resultados.log   # Asia ~10  
grep -c "18.130.234.56" resultados.log   # Europa ~20

# RESULTADO ESPERADO: Siempre va más a EEUU (mayor peso)
```

### 3. Enrutamiento por Latencia

#### Características
- Dirige tráfico al recurso con **menor latencia**
- Basado en latencia entre usuario y regiones AWS
- Útil para aplicaciones globales
- **SÍ** se puede asociar con Health Checks

#### 🏗️ Diagrama Latency-Based

```
                          Route 53
                     Política de Latencia
                    ┌───────────────────┐
Usuario Europa ──→  │ app.example.com   │
                    │                   │
                    │ Medir latencia:   │
                    │ ✓ eu-west-1: 15ms │ ← Menor latencia
                    │ ✗ us-east-1: 95ms │
                    │ ✗ ap-south-1:140ms│
                    └─────────┬─────────┘
                              │
                              ↓ Redirige a menor latencia
                    ┌─────────────────┐
                    │      EC2        │
                    │   eu-west-1     │
                    │  (Irlanda)      │
                    └─────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Latencias Globales                          │
│                                                                 │
│  Usuario US-East    Usuario Europe    Usuario Asia-Pacific     │
│       │                  │                    │                │
│       ↓                  ↓                    ↓                │
│   us-east-1          eu-west-1          ap-southeast-1         │
│    (10ms)             (15ms)              (25ms)              │
└─────────────────────────────────────────────────────────────────┘
```

#### Casos de Uso Latency
- **Aplicaciones Globales**: Mejor experiencia de usuario
- **Gaming**: Reducir lag en juegos online
- **CDN Personalizado**: Cuando CloudFront no es suficiente
- **APIs Críticas**: Minimizar tiempo de respuesta

#### 🧪 Experimento Latency
```bash
# EXPERIMENTO CON LAS 3 REGIONES

# REGISTRO 1: Asia
# En Route 53 Console:
# - Nombre: latencia.ayraz.com
# - Tipo: A
# - Valor: IP instancia Asia
# - Política: Latencia
# - Región: ap-southeast-1
# - ID: asia-latency

# REGISTRO 2: EEUU
# - Nombre: latencia.ayraz.com  
# - Tipo: A
# - Valor: IP instancia EEUU
# - Política: Latencia
# - Región: us-east-1
# - ID: eeuu-latency

# REGISTRO 3: Europa
# - Nombre: latencia.ayraz.com
# - Tipo: A  
# - Valor: IP instancia Europa
# - Política: Latencia
# - Región: eu-west-1
# - ID: europa-latency

# PRUEBAS DE LATENCIA:
# 1. Desde tu ubicación actual
dig latencia.ayraz.com
curl -w "Tiempo total: %{time_total}s\n" http://latencia.ayraz.com

# 2. Simular desde diferentes ubicaciones con VPN
# - Conectar VPN a Asia → debería resolver a instancia Asia
# - Conectar VPN a Europa → debería resolver a instancia Europa  
# - Conectar VPN a EEUU → debería resolver a instancia EEUU

# 3. Verificar latencia real:
ping $(dig +short latencia.ayraz.com)
```

### 4. Enrutamiento por Geolocalización

#### Características
- Basado en la **ubicación geográfica** del usuario
- Control granular: Continente → País → Estado (US)
- Útil para localización y restricciones legales
- **SÍ** se puede asociar con Health Checks

#### 🏗️ Diagrama Geolocation

```
                         Route 53
                    Política Geolocalización
                   ┌─────────────────────────┐
                   │ app.example.com         │
                   │                         │
Usuario España ──→ │ España → eu-west-1      │ ──→ Madrid DC
Usuario México ──→ │ México → us-west-2      │ ──→ Oregon DC  
Usuario Brasil ──→ │ Brasil → sa-east-1      │ ──→ São Paulo DC
Usuario Default──→ │ Default → us-east-1     │ ──→ Virginia DC
                   └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Jerarquía de Precisión                       │
│                                                                 │
│  1. Estado/Provincia (más específico)                          │
│     └── California → us-west-1                                 │
│                                                                 │
│  2. País                                                        │
│     └── España → eu-west-1                                     │
│                                                                 │
│  3. Continente                                                  │
│     └── Europa → eu-central-1                                  │
│                                                                 │
│  4. Default (menos específico)                                 │
│     └── Cualquier ubicación → us-east-1                       │
└─────────────────────────────────────────────────────────────────┘
```

#### Casos de Uso Geolocation
- **Cumplimiento Legal**: GDPR, restricciones por país
- **Localización**: Contenido en idioma local
- **Precios Regionales**: Diferentes modelos de negocio
- **Restricciones de Contenido**: Bloqueo por geografía

### 5. Conmutación por Error (Failover)

#### Características  
- Arquitectura **Activo-Pasivo**
- Instancia primaria y secundaria
- Requiere **Health Checks obligatorios**
- Conmutación automática en caso de fallo

#### 🏗️ Diagrama Failover

```
                         Estado Normal
                              │
Usuario ──DNS Query──→ ┌─────────────┐ ──Health Check OK──→ ┌─────────────┐
                       │  Route 53   │                      │   Primary   │
                       │  Failover   │                      │   EC2 (US)  │
                       └─────────────┘                      │ SALUDABLE   │
                              │                             └─────────────┘
                              │
                              ↓ Monitoreo Continuo
                       ┌─────────────┐
                       │ Health Check│
                       │   Status    │
                       └─────────────┘
                              │
                              ↓ Si Primary falla...

                         Estado Failover
                              │
Usuario ──DNS Query──→ ┌─────────────┐ ──Health Check FAIL─✗ ┌─────────────┐
                       │  Route 53   │                       │   Primary   │  
                       │  Failover   │ ──Redirect to──→      │   EC2 (US)  │
                       └─────────────┘     Secondary         │ NO SALUDABLE│
                              │                              └─────────────┘
                              ↓
                       ┌─────────────┐
                       │  Secondary  │
                       │ EC2 (Europa)│
                       │  SALUDABLE  │
                       └─────────────┘
```

#### 🧪 Experimento Failover
```bash
# EXPERIMENTO CON INSTANCIAS EEUU Y EUROPA

# REQUISITO PREVIO: Crear Health Checks
# Health Check Primario (EEUU):
# - Endpoint: IP instancia EEUU
# - Puerto: 80
# - Path: /
# - Intervalo: 30s

# Health Check Secundario (Europa):  
# - Endpoint: IP instancia Europa
# - Puerto: 80
# - Path: /
# - Intervalo: 30s

# REGISTRO PRIMARIO (Activo)
# En Route 53 Console:
# - Nombre: failover.ayraz.com
# - Tipo: A
# - Valor: IP instancia EEUU
# - Política: Failover
# - Tipo failover: Primario
# - Health Check: health-check-eeuu-id
# - ID: primary-eeuu

# REGISTRO SECUNDARIO (Pasivo)
# - Nombre: failover.ayraz.com
# - Tipo: A
# - Valor: IP instancia Europa  
# - Política: Failover
# - Tipo failover: Secundario
# - Health Check: health-check-europa-id
# - ID: secondary-europa

# PRUEBA DE FAILOVER:
# 1. Estado normal - apunta a EEUU
dig failover.ayraz.com
curl failover.ayraz.com

# 2. Simular fallo: detener instancia EEUU
aws ec2 stop-instances --instance-ids i-1234567890abcdef0 --region us-east-1

# 3. Esperar detección health check (30-90 segundos)
watch -n 10 'dig +short failover.ayraz.com'

# 4. Verificar failover a Europa  
curl failover.ayraz.com  # Debería mostrar página Europa

# 5. Recuperación: reiniciar instancia EEUU
aws ec2 start-instances --instance-ids i-1234567890abcdef0 --region us-east-1

# 6. Verificar failback automático a primario
```

### 6. Geoproximidad

#### Características
- Enruta tráfico basado en la **ubicación geográfica** de usuarios y recursos
- Permite **bias** (sesgo) para modificar el área de influencia
- Control granular del tráfico geográfico
- Requiere Route 53 Traffic Flow (costo adicional)

#### 🏗️ Diagrama Geoproximity

```
                    Sin Bias (Distribución Natural)
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │  ┌─────────────┐                    ┌─────────────┐             │
   │  │   Recurso   │ ←── 50% tráfico ──→│   Recurso   │             │
   │  │  Virginia   │      Europa        │   Irlanda   │             │
   │  │             │                    │             │             │
   │  └─────────────┘                    └─────────────┘             │
   │         │                                   │                   │
   │         └─── Área Natural ───────────────────┘                  │
   └─────────────────────────────────────────────────────────────────┘

                    Con Bias Positivo (+50 Virginia)
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │  ┌─────────────┐                    ┌─────────────┐             │
   │  │   Recurso   │ ←── 75% tráfico    │   Recurso   │             │
   │  │  Virginia   │      Europa        │   Irlanda   │             │
   │  │  (Bias +50) │      25% ──────────→│             │             │
   │  └─────────────┘                    └─────────────┘             │
   │         │                                   │                   │
   │         └─── Área Expandida ─────────┐      │                   │
   │                                      └──────┘                   │
   └─────────────────────────────────────────────────────────────────┘
```

#### Casos de Uso Geoproximity
- **Optimización de Costos**: Dirigir más tráfico a recursos baratos
- **Balanceamento Regional**: Ajustar carga entre regiones
- **Testing Geográfico**: Probar nuevos recursos en áreas específicas
- **Disaster Recovery**: Expandir área de un recurso cuando otro falla

### 7. Respuesta Multivalor

#### Características
- Devuelve **múltiples valores/recursos** saludables
- Máximo **8 registros** por consulta
- **SÍ** se asocia con Health Checks
- **NO** es un reemplazo para ELB

#### 🏗️ Diagrama Multivalue

```
                         Route 53
                     Política Multivalor
                   ┌─────────────────────┐
Cliente ──Query──→ │ app.example.com     │
                   │                     │
                   │ Health Checks:      │
                   │ ✓ EC2-1 (Healthy)   │ ──→ Incluido en respuesta
                   │ ✓ EC2-2 (Healthy)   │ ──→ Incluido en respuesta  
                   │ ✗ EC2-3 (Unhealthy) │ ──→ Excluido de respuesta
                   │ ✓ EC2-4 (Healthy)   │ ──→ Incluido en respuesta
                   └─────────┬───────────┘
                             │
                             ↓ Respuesta DNS con múltiples IPs
                   ┌─────────────────────┐
                   │ DNS Response:       │
                   │ 1.2.3.4  (EC2-1)   │ ← Cliente elige
                   │ 5.6.7.8  (EC2-2)   │   aleatoriamente
                   │ 9.10.11.12 (EC2-4) │
                   └─────────────────────┘
                             │
                             ↓ Cliente selecciona IP
                   ┌─────────────────────┐
                   │    Instancia        │
                   │    Seleccionada     │
                   └─────────────────────┘
```

#### Diferencia con ELB
```
Multivalue DNS:               ELB:
┌─────────────────┐          ┌─────────────────┐
│ Cliente elige   │          │ ELB decide      │
│ IP de la lista  │    VS    │ instancia       │
│ (Layer 3)       │          │ (Layer 7)       │
└─────────────────┘          └─────────────────┘
```

#### 🧪 Experimento Multivalue
```bash
# EXPERIMENTO CON LAS 3 INSTANCIAS (EEUU, Europa, Asia)

# REQUISITO: Crear Health Checks para cada instancia
# Health Check 1: IP instancia EEUU
# Health Check 2: IP instancia Europa  
# Health Check 3: IP instancia Asia

# REGISTRO 1: EEUU con Health Check
# En Route 53 Console:
# - Nombre: multivalue.ayraz.com
# - Tipo: A
# - Valor: IP instancia EEUU
# - Política: Respuesta multivalor
# - Health Check: health-check-eeuu-id
# - ID: multivalue-eeuu

# REGISTRO 2: Europa con Health Check  
# - Nombre: multivalue.ayraz.com
# - Tipo: A
# - Valor: IP instancia Europa
# - Política: Respuesta multivalor
# - Health Check: health-check-europa-id
# - ID: multivalue-europa

# REGISTRO 3: Asia con Health Check
# - Nombre: multivalue.ayraz.com
# - Tipo: A
# - Valor: IP instancia Asia
# - Política: Respuesta multivalor  
# - Health Check: health-check-asia-id
# - ID: multivalue-asia

# PRUEBAS:
# 1. Estado normal - todas las instancias healthy
dig multivalue.ayraz.com
# Debería devolver máximo 8 IPs (en este caso, las 3)

# 2. Simular fallo de una instancia
aws ec2 stop-instances --instance-ids i-asia-instance --region ap-southeast-1

# 3. Esperar health check failure y probar
dig multivalue.ayraz.com
# Debería devolver solo 2 IPs (EEUU y Europa)

# 4. Diferencia con ELB:
# Cliente recibe múltiples IPs y elige uno
# ELB recibiría 1 request y él decide la instancia
```

### Comparación de Políticas

| Política | Health Checks | Casos de Uso | Complejidad |
|----------|---------------|--------------|-------------|
| **Simple** | ❌ | Configuración básica | Baja |
| **Weighted** | ✅ | A/B testing, gradual rollout | Media |
| **Latency** | ✅ | Apps globales, performance | Media |
| **Geolocation** | ✅ | Compliance, localización | Media |
| **Failover** | ✅ Obligatorio | DR, alta disponibilidad | Media |
| **Geoproximity** | ✅ | Control granular geográfico | Alta |
| **Multivalue** | ✅ | Redundancia simple | Baja |

---

## 🏥 Controles de Salud (Health Checks)

### Visión General

Los Health Checks de Route 53 permiten **conmutación automática por error de DNS**.

> ⚠️ **Importante**: Los Health Checks HTTP/HTTPS solo funcionan para recursos **públicos**.

### Tipos de Health Checks

#### 1. Health Checks de Endpoint

```
┌─────────────────────────────────────────────────────────────────┐
│                    Health Check Global                         │
│                                                                 │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│   │ Checker US  │  │Checker EU   │  │Checker ASIA │            │
│   │    East     │  │   West      │  │   Pacific   │            │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│          │                │                │                   │
│          └────────────────┼────────────────┘                   │
│                           │                                    │
│                           ↓                                    │
│                  ┌─────────────────┐                           │
│                  │   Target        │                           │
│                  │   Endpoint      │                           │
│                  │ (Tu servidor)   │                           │
│                  └─────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘

Configuración:
• ~15 verificadores globales
• Umbral: 3 fallos para marcar "unhealthy" (configurable)
• Intervalo: 30 segundos (puede ser 10s por costo extra)
• Protocolos: HTTP, HTTPS, TCP
• > 18% de verificadores deben reportar "healthy"
```

### 🧪 Experimento Completo: Health Checks para las 3 Instancias

#### Health Checks de Endpoint

```bash
# HEALTH CHECK 1: Instancia EEUU
# En Route 53 Console → Health Checks → Create:
# - Tipo: HTTP
# - IP Address: IP instancia EEUU  
# - Port: 80
# - Path: /
# - Request interval: 30 segundos
# - Failure threshold: 3
# - Nombre: health-check-eeuu

# HEALTH CHECK 2: Instancia Europa
# - Tipo: HTTP
# - IP Address: IP instancia Europa
# - Port: 80  
# - Path: /
# - Request interval: 30 segundos
# - Failure threshold: 3
# - Nombre: health-check-europa

# HEALTH CHECK 3: Instancia Asia
# - Tipo: HTTP
# - IP Address: IP instancia Asia
# - Port: 80
# - Path: /
# - Request interval: 30 segundos  
# - Failure threshold: 3
# - Nombre: health-check-asia
```

#### Health Check Calculado (Combinado)

```bash
# HEALTH CHECK PADRE: Combina los 3 anteriores
# En Route 53 Console → Health Checks → Create:
# - Tipo: Calculated
# - Health checks to monitor:
#   * health-check-eeuu
#   * health-check-europa  
#   * health-check-asia
# - Operator: OR (al menos 1 healthy)
# - Report healthy when: 1 of 3 health checks pass
# - Nombre: health-check-global

# Casos de uso del calculado:
# - Mantenimiento sin fallar health checks
# - Lógica AND: todos deben estar healthy
# - Lógica OR: al menos uno healthy  
# - Lógica NOT: invertir resultado
```

#### Health Checks para Recursos Privados

```bash
# PROBLEMA: Health checks están fuera de VPC
# NO pueden acceder a endpoints privados

# SOLUCIÓN: CloudWatch Alarm + Health Check
# 1. Crear métrica CloudWatch personalizada
aws cloudwatch put-metric-data \
  --namespace "Custom/Application" \
  --metric-data MetricName=HealthStatus,Value=1,Unit=Count

# 2. Crear alarma CloudWatch  
aws cloudwatch put-metric-alarm \
  --alarm-name "Private-Instance-Health" \
  --alarm-description "Private instance health status" \
  --metric-name HealthStatus \
  --namespace Custom/Application \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2

# 3. Health Check basado en CloudWatch Alarm
# En Route 53 Console:
# - Tipo: CloudWatch alarm
# - Region: us-east-1
# - Alarm name: Private-Instance-Health
```

### Configuración de Security Groups para Health Checks

```bash
# Los health checkers de Route 53 vienen desde IPs específicas
# Permitir acceso desde rangos IP de Route 53 health checkers

# Crear regla en Security Group:
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0  # En producción, usar rangos específicos Route 53

# Rangos IP Route 53 Health Checkers (documentados en AWS)
# - 15.177.0.0/18 (us-east-1)  
# - 52.86.0.0/15 (us-west-1)
# - Etc. (consultar documentación AWS actual)
```

### Mejores Prácticas Health Checks

#### 1. Endpoint de Health Check Dedicado
```python
# Ejemplo Flask
@app.route('/health')
def health_check():
    try:
        # Verificar base de datos
        db.execute('SELECT 1')
        
        # Verificar dependencias externas
        external_api_check()
        
        return {'status': 'healthy', 'timestamp': time.time()}, 200
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}, 503
```

#### 2. Monitoreo en Capas
```yaml
Arquitectura Health Checks:
  Load Balancer:
    - Health Check: /health
    - Timeout: 5s
    - Interval: 30s
    
  Route 53:
    - Health Check: /health
    - Timeout: 10s
    - Interval: 30s
    - Threshold: 3 failures
    
  CloudWatch:
    - Métrica personalizada
    - Alarma: > 5% error rate
    - Action: SNS notification
```

---

## 📝 Registro de Dominios

### Conceptos Fundamentales

**Registrador de Dominio ≠ Servicio DNS**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Conceptos Separados                         │
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │   REGISTRADOR   │              │  SERVICIO DNS   │          │
│  │   DE DOMINIO    │              │                 │          │
│  │                 │              │ • Resuelve      │          │
│  │ • Compra dominio│              │   consultas DNS │          │
│  │ • Renovación    │              │ • Gestiona      │          │
│  │ • Transferencia │              │   registros     │          │
│  │ • Whois info    │              │ • Health checks │          │
│  │ • Name servers  │              │ • Políticas     │          │
│  └─────────────────┘              └─────────────────┘          │
│          │                                 ↑                   │
│          │                                 │                   │
│          └── Configurar Name Servers ──────┘                   │
└─────────────────────────────────────────────────────────────────┘

Ejemplo Real:
Registrador: GoDaddy (compras ayraz.com por $12/año)
DNS: Route 53 (manejas resolución DNS por $0.50/mes)
```

### Escenarios de Configuración

#### Escenario 1: Route 53 Todo-en-Uno
```
┌─────────────────┐
│    Route 53     │
│                 │ ← Solución completa
│ • Compra dominio│   (más caro pero integrado)
│ • DNS hosting   │
│ • Health checks │
│ • Políticas DNS │
└─────────────────┘

Ventajas:
✅ Integración completa
✅ Gestión centralizada  
✅ Soporte AWS
✅ APIs unificadas

Desventajas:
❌ Más costoso para dominios
❌ Menos opciones de TLD
```

#### Escenario 2: Registrador Externo + Route 53 DNS
```
┌─────────────────┐    Configurar NS    ┌─────────────────┐
│    GoDaddy      │ ─────────────────→   │    Route 53     │
│   (Registrador) │                      │  (Servicio DNS) │
│                 │                      │                 │
│ • Compra dominio│                      │ • DNS hosting   │
│ • Renovación    │                      │ • Health checks │
│ • Whois info    │                      │ • Políticas DNS │
│ • Contactos     │                      │ • Monitoreo     │
└─────────────────┘                      └─────────────────┘

Ventajas:
✅ Dominios más baratos
✅ Más opciones de registradores
✅ Funcionalidades DNS avanzadas
✅ Flexibilidad

Desventajas:
❌ Gestión en 2 lugares
❌ Configuración adicional
```

### Proceso de Migración DNS (Registrador Externo → Route 53)

#### Paso 1: Crear Zona de Alojamiento en Route 53
```bash
# En Route 53 Console:
# 1. Hosted zones → Create hosted zone
# 2. Domain name: ayraz.com
# 3. Type: Public hosted zone
# 4. Create hosted zone

# Resultado: Obtienes 4 name servers:
# ns-123.awsdns-12.com
# ns-456.awsdns-45.net
# ns-789.awsdns-78.org  
# ns-012.awsdns-01.co.uk
```

#### Paso 2: Migrar Registros DNS Existentes
```bash
# Antes de cambiar name servers, migrar todos los registros:
# 1. Exportar registros del proveedor actual
# 2. Crear registros equivalentes en Route 53:

# Registros típicos a migrar:
# - A record: www → IP del servidor web
# - A record: @ (apex) → IP del servidor web  
# - MX records: mail servers
# - CNAME records: subdominios
# - TXT records: SPF, DKIM, verificaciones
```

#### Paso 3: Actualizar Name Servers en Registrador
```bash
# En panel de GoDaddy (o tu registrador):
# 1. DNS Management → Name servers
# 2. Cambiar a "Custom"
# 3. Introducir los 4 name servers de Route 53:
#    ns-123.awsdns-12.com
#    ns-456.awsdns-45.net  
#    ns-789.awsdns-78.org
#    ns-012.awsdns-01.co.uk
# 4. Guardar cambios

# ⚠️ IMPORTANTE: Este cambio puede tardar 24-48h en propagarse
```

#### Paso 4: Verificar Propagación DNS
```bash
# Verificar name servers globalmente:
dig NS ayraz.com @8.8.8.8
dig NS ayraz.com @1.1.1.1
dig NS ayraz.com @208.67.222.222

# Verificar registros específicos:
dig A www.ayraz.com @8.8.8.8
dig MX ayraz.com @8.8.8.8

# Herramientas online para verificación:
# - whatsmydns.net
# - dnschecker.org
# - nslookup.io

# Verificar desde múltiples ubicaciones:
nslookup ayraz.com 8.8.8.8
nslookup ayraz.com 1.1.1.1
```

#### Paso 5: Monitoreo Post-Migración
```bash
# Verificar logs Route 53:
# 1. CloudWatch → Logs → /aws/route53/ayraz.com
# 2. Monitorear volumen de consultas
# 3. Verificar health checks funcionando

# Rollback plan (si algo falla):
# 1. Cambiar name servers de vuelta al proveedor original
# 2. Esperar propagación (24-48h)
# 3. Investigar problemas en Route 53
```

### Consideraciones de Costo

| Servicio | Costo | Nota |
|----------|-------|------|
| **Hosted Zone** | $0.50/mes | Por zona alojada |
| **Consultas DNS** | $0.40/millón | Primeros 1B consultas/mes |
| **Health Checks** | $0.50/mes | Por health check |
| **Registro Dominio** | Variable | Depende del TLD |
| **Traffic Flow** | $50/mes | Para políticas complejas |

---

## ⚡ Mejores Prácticas para Arquitectos

### 1. Diseño de Alta Disponibilidad

#### Arquitectura Multi-Región
```
                            Route 53
                        (Global DNS Service)
                              │
                ┌─────────────┼─────────────┐
                │             │             │
         ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
         │  us-east-1  │ │ eu-west-1   │ │ap-southeast-1│
         │             │ │             │ │             │
         │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │
         │ │   ALB   │ │ │ │   ALB   │ │ │ │   ALB   │ │
         │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │
         │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │
         │ │Health   │ │ │ │Health   │ │ │ │Health   │ │
         │ │Check    │ │ │ │Check    │ │ │ │Check    │ │
         │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │
         └─────────────┘ └─────────────┘ └─────────────┘

Políticas recomendadas:
• Latency-based para performance
• Health checks obligatorios
• Failover entre regiones
```

#### TTL Strategy
```yaml
TTL Recommendations:
  Production:
    - Normal: 300s (5 minutos)
    - Durante deployment: 60s (1 minuto)
    - Post-deployment: volver a 300s
    
  Staging/Dev:
    - Siempre: 60s (cambios frecuentes)
    
  Critical Services:
    - Health check interval: 10s
    - TTL: 60s máximo
```

### 2. Seguridad y Compliance

#### DNS Security Extensions (DNSSEC)
```bash
# Habilitar DNSSEC en Route 53
aws route53 enable-hosted-zone-dnssec \
  --hosted-zone-id Z123456789
```

#### Logging de Consultas DNS
```yaml
Query Logging Configuration:
  CloudWatch Logs:
    - Log Group: /aws/route53/queries
    - Retention: 30 días
    - Monitoring: Query patterns, anomalías
    
  Analysis:
    - Volumen de consultas por región
    - Tipos de registros más consultados
    - Detección de ataques DNS
```

### 3. Monitoreo y Observabilidad

#### CloudWatch Metrics
```yaml
Route 53 Metrics:
  DNS Queries:
    - QueryCount por zona
    - QueryCount por tipo de registro
    - Latencia de respuesta
    
  Health Checks:
    - HealthCheckStatus
    - HealthCheckPercentHealthy
    - ConnectionTime
    
  Custom Dashboards:
    - DNS query patterns
    - Failover events
    - Regional distribution
```

#### Alertas Proactivas
```bash
# Alerta para health check failures
aws cloudwatch put-metric-alarm \
  --alarm-name "Route53-HealthCheck-Failure" \
  --alarm-description "Health check failing" \
  --metric-name HealthCheckPercentHealthy \
  --namespace AWS/Route53 \
  --statistic Average \
  --period 60 \
  --threshold 100 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:route53-alerts
```

### 4. Optimización de Costos

#### Strategies de Reducción
```yaml
Cost Optimization:
  Hosted Zones:
    - Consolidar subdominios en una zona
    - Eliminar zonas no utilizadas
    - Usar Private Hosted Zones para interno
    
  Health Checks:
    - Usar calculated health checks
    - Optimizar intervalos de verificación
    - Compartir health checks entre registros
    
  Queries:
    - Optimizar TTL para reducir consultas
    - Usar CloudFront para cacheo adicional
    - Implementar DNS caching en aplicaciones
```

### 5. Disaster Recovery

#### RTO/RPO Objectives
```yaml
DR Planning:
  RTO (Recovery Time Objective):
    - DNS Failover: < 1 minuto (TTL dependent)
    - Health Check Detection: 30-90 segundos
    - Total Switchover: < 3 minutos
    
  RPO (Recovery Point Objective):
    - DNS changes: Inmediato (eventual consistency)
    - Configuration backup: Diario
    - Health check history: 2 semanas
```

#### Automated DR
```bash
# Lambda function para failover automático
import boto3
import json

def lambda_handler(event, context):
    route53 = boto3.client('route53')
    
    # Detectar fallo en región primaria
    if event['source'] == 'aws.route53':
        if event['detail']['status'] == 'FAILURE':
            # Activar failover a región secundaria
            response = route53.change_resource_record_sets(
                HostedZoneId='Z123456789',
                ChangeBatch={
                    'Changes': [{
                        'Action': 'UPSERT',
                        'ResourceRecordSet': {
                            'Name': 'api.example.com',
                            'Type': 'A',
                            'SetIdentifier': 'Primary',
                            'Failover': 'SECONDARY',
                            'TTL': 60,
                            'ResourceRecords': [{'Value': '5.6.7.8'}]
                        }
                    }]
                }
            )
    
    return {'statusCode': 200}
```

---

## 🎯 Tips para el Examen SAA-C03

### Escenarios Frecuentes

#### 1. Selección de Política de Enrutamiento
```
Pregunta: "Una aplicación global necesita dirigir usuarios 
a la región con menor latencia"

Respuesta: Latency-based routing
✅ Latency-based (mínima latencia)
❌ Geolocation (ubicación específica)  
❌ Weighted (distribución porcentual)
❌ Simple (sin optimización)
```

#### 2. Health Check Configuration
```
Pregunta: "Recursos en VPC privada necesitan health monitoring"

Respuesta: CloudWatch Alarm + Route 53 Health Check
✅ CloudWatch metrics → Alarm → Health Check
❌ Direct health check (no acceso a privado)
❌ ELB health check (diferente propósito)
❌ EC2 status check (limitado)
```

#### 3. Failover Architecture
```
Pregunta: "Aplicación crítica necesita DR automático 
con Recovery Time < 2 minutos"

Respuesta: Failover routing + Low TTL
✅ Failover policy + TTL 60s + Health checks
❌ Manual DNS update (demasiado lento)
❌ ELB cross-region (no existe)
❌ CloudFront failover (diferentes casos)
```

#### 4. Optimización Global
```
Pregunta: "Distribuir tráfico 80% Producción, 20% Testing
en aplicación global"

Respuesta: Weighted routing policy
✅ Weighted (80/20 distribution)
❌ Simple (sin control porcentual)
❌ Latency (basado en performance)
❌ Geolocation (basado en ubicación)
```

### Palabras Clave del Examen

| Escenario | Pistas en Pregunta | Solución Route 53 |
|-----------|-------------------|-------------------|
| **Performance** | "menor latencia", "más rápido" | Latency-based |
| **Testing** | "A/B test", "gradual rollout" | Weighted |
| **Geography** | "usuarios europeos", "compliance GDPR" | Geolocation |
| **Disaster Recovery** | "failover", "backup region" | Failover |
| **Cost Optimization** | "reducir consultas DNS" | Higher TTL |
| **Private Resources** | "VPC privada", "internal monitoring" | CloudWatch Alarm |

### Errores Comunes a Evitar

```
❌ Confundir CNAME con ALIAS
   - CNAME: No puede usarse en apex
   - ALIAS: Sí puede usarse en apex

❌ Health Checks en recursos privados
   - No funciona directamente
   - Usar CloudWatch Alarm

❌ TTL cero para reducir costo
   - TTL debe ser > 0
   - Usar TTL apropiado para caso

❌ Múltiples políticas en mismo registro
   - Solo una política por registro
   - Usar Traffic Flow para complejas

❌ Olvidar Health Checks en Failover
   - Failover REQUIERE health checks
   - Sin health check no hay failover
```

### Preguntas de Práctica

#### Pregunta 1
```
Una empresa multinacional quiere que usuarios españoles 
accedan a servidores en Europa y usuarios mexicanos 
a servidores en América. ¿Qué política usar?

A) Latency-based routing
B) Weighted routing  
C) Geolocation routing ✅
D) Simple routing
```

#### Pregunta 2
```
Una aplicación necesita dirigir 90% tráfico a producción 
y 10% a nueva versión para testing. ¿Qué configurar?

A) Geolocation policy
B) Weighted policy ✅
C) Latency-based policy
D) Failover policy
```

#### Pregunta 3
```
Aplicación crítica con RTO < 1 minuto necesita failover 
automático entre us-east-1 y eu-west-1. ¿Qué configurar?

A) Manual DNS update
B) ELB cross-region
C) Failover policy + TTL 60s ✅
D) CloudFront origin failover
```

---

## 📚 Referencias y Recursos Adicionales

### Documentación Oficial
- [Route 53 Developer Guide](https://docs.aws.amazon.com/route53/)
- [Health Checks and DNS Failover](https://docs.aws.amazon.com/route53/latest/developerguide/dns-failover.html)
- [Routing Policies](https://docs.aws.amazon.com/route53/latest/developerguide/routing-policy.html)

### Herramientas Útiles
```bash
# DNS Testing Tools
dig example.com
nslookup example.com  
host example.com

# Multi-location Testing
curl -H "Host: example.com" http://1.2.3.4
```

### Labs Recomendados
1. **Basic DNS Setup**: Crear zona y registros básicos
2. **Health Check Configuration**: Monitoreo multi-región
3. **Policy Comparison**: Probar diferentes políticas
4. **Failover Testing**: Simular fallos y recuperación

---

*Última actualización: Agosto 2025 - Preparación SAA-C03*
** Roberto Ayra**
