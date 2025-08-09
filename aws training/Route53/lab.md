# Route 53 Lab: Netflix-Style Streaming Platform

## 🎬 Caso de Uso Empresarial

### Contexto del Negocio

**StreamFlix Entertainment Inc.** es una startup que busca competir directamente con Netflix, Amazon Prime Video y Disney+ en el mercado global de streaming. Después de obtener una ronda de financiación Serie B de $150 millones, la empresa tiene los siguientes objetivos estratégicos:

**Visión**: Convertirse en la plataforma de streaming líder en entretenimiento personalizado para mercados emergentes y desarrollados.

**Misión Técnica**: Crear una infraestructura DNS inteligente que garantice la mejor experiencia de usuario posible, independientemente de la ubicación geográfica, mientras cumple con regulaciones locales.

### Requerimientos del Negocio

#### 📊 **Requerimientos Funcionales**

1. **Distribución Geográfica Inteligente**
   - **RF001**: Los usuarios deben ser direccionados automáticamente al servidor más cercano según su ubicación
   - **RF002**: Contenido específico por región debe ser servido según licencias y restricciones legales
   - **RF003**: Usuarios de Europa deben acceder exclusivamente a servidores EU para cumplir GDPR
   - **RF004**: Latencia máxima de resolución DNS: 50ms a nivel global

2. **Disponibilidad y Recuperación**
   - **RF005**: Sistema debe mantener 99.99% de disponibilidad (máximo 52.6 minutos de downtime/año)
   - **RF006**: Failover automático en caso de fallo regional debe completarse en < 2 minutos
   - **RF007**: Detección automática de servicios degradados en < 90 segundos
   - **RF008**: Capacidad de mantener servicio con 1 región completamente fuera de línea

3. **Experimentación y Desarrollo**
   - **RF009**: Capacidad de dirigir porcentajes específicos de tráfico a versiones beta
   - **RF010**: A/B testing granular por características de usuario (país, dispositivo, suscripción)
   - **RF011**: Rollback instantáneo en caso de detección de problemas en versión beta
   - **RF012**: Blue/Green deployments para actualizaciones sin downtime

#### ⚡ **Requerimientos No Funcionales**

1. **Performance**
   - **RNF001**: Resolución DNS global < 50ms en 95% de consultas
   - **RNF002**: Tiempo de carga inicial de aplicación < 3 segundos
   - **RNF003**: Soporte para 10M usuarios concurrentes en hora pico
   - **RNF004**: Escalabilidad automática ante picos de tráfico (ej: lanzamientos exclusivos)

2. **Seguridad y Compliance**
   - **RNF005**: Cumplimiento GDPR para usuarios europeos
   - **RNF006**: Cumplimiento COPPA para contenido infantil en US
   - **RNF007**: Geo-blocking para contenido con restricciones de distribución
   - **RNF008**: Protección DDoS a nivel DNS

3. **Monitoreo y Observabilidad**
   - **RNF009**: Métricas en tiempo real de health de servicios
   - **RNF010**: Alertas automáticas ante degradación de servicio
   - **RNF011**: Dashboard ejecutivo con KPIs de negocio
   - **RNF012**: Capacidad de troubleshooting en < 5 minutos

### 🌍 Escenarios de Negocio Críticos

#### **Escenario 1: Lanzamiento Global de Serie Original**
- **Situación**: Estreno de serie original "Digital Crown" simultáneamente en 50 países
- **Desafío**: Tráfico 500% superior al normal en primeras 24 horas
- **Requerimiento**: Sistema debe escalar automáticamente sin degradación de servicio
- **Success Criteria**: 
  - 0% de timeouts durante el pico
  - Latencia < 200ms mantenida globalmente
  - Distribución automática de carga entre regiones

#### **Escenario 2: Fallo Regional Catastrófico**
- **Situación**: Región US-East-1 completamente fuera de servicio (simulando outage AWS)
- **Desafío**: 40% de la base de usuarios (América) debe ser redirigida transparentemente
- **Requerimiento**: Failover automático sin intervención manual
- **Success Criteria**:
  - Detección de fallo en < 90 segundos
  - Redirección completa en < 2 minutos
  - Experiencia de usuario sin interrupciones perceptibles

#### **Escenario 3: Expansión a Nuevo Mercado**
- **Situación**: Lanzamiento en Brasil con contenido localizado
- **Desafío**: Usuarios brasileños deben acceder a catálogo específico y servidores locales
- **Requerimiento**: Configuración DNS que soporte geo-segmentación granular
- **Success Criteria**:
  - 100% precisión en direccionamiento geográfico
  - Soporte para contenido exclusivo regional
  - Integración sin impacto en regiones existentes

#### **Escenario 4: Testing de Nueva Funcionalidad**
- **Situación**: Implementación de sistema de recomendaciones con IA
- **Desafío**: Probar con 10% de usuarios antes de rollout completo
- **Requerimiento**: A/B testing controlado y medible
- **Success Criteria**:
  - Distribución exacta 90/10 de tráfico
  - Métricas en tiempo real de performance
  - Rollback en < 30 segundos si es necesario

### 🎯 Arquitectura de Solución Propuesta

La solución StreamFlix implementará una arquitectura DNS multinivel utilizando Amazon Route 53 con las siguientes políticas:

```
📍 GEOLOCATION ROUTING
├── América del Norte → us-east-1 (Primary)
├── América del Sur → us-east-1 (Primary)  
├── Europa → eu-west-1 (GDPR Compliant)
├── Asia-Pacífico → ap-southeast-1
└── Default/Fallback → us-east-1

⚡ LATENCY-BASED ROUTING  
├── API Endpoints → Menor latencia automática
├── CDN Distribution → CloudFront optimizado
└── Database Connections → Read replicas regionales

⚖️ WEIGHTED ROUTING
├── Producción → 90% tráfico
├── Beta Testing → 10% tráfico  
└── Canary Releases → 1% para nuevas features

🔄 FAILOVER ROUTING
├── Primary → us-east-1 (Active Health Check)
├── Secondary → eu-west-1 (Standby Health Check)
└── Emergency → ap-southeast-1 (Manual activation)

📊 MULTIVALUE ROUTING
├── Load Distribution → Múltiples IPs por región
├── Health Check Integration → Solo IPs saludables
└── Random Selection → Balanceo automático
```

### 💼 Stakeholders y Responsabilidades

#### **Equipo Técnico**
- **DevOps Engineer**: Implementación y mantenimiento de infraestructura Route 53
- **Site Reliability Engineer**: Monitoreo 24/7 y respuesta a incidentes
- **Security Engineer**: Cumplimiento GDPR y geo-blocking
- **Data Engineer**: Análisis de métricas de performance y usage patterns

#### **Equipo de Negocio**
- **Product Manager**: Definición de requerimientos de A/B testing
- **Content Manager**: Gestión de restricciones geográficas de contenido
- **Legal/Compliance**: Validación de cumplimiento regulatorio por región
- **Customer Success**: Monitoreo de experiencia de usuario y escalación de issues

### 📋 Criterios de Aceptación del Proyecto

#### **Fase 1: MVP (Minimum Viable Product)**
- [ ] Resolución DNS funcional en 3 regiones principales
- [ ] Health checks configurados y funcionando
- [ ] Failover básico US → EU operativo
- [ ] Monitoreo básico con CloudWatch

#### **Fase 2: Production Ready**
- [ ] Geolocation routing con 95% precisión
- [ ] A/B testing funcional para 2 variantes
- [ ] SLA 99.9% demostrado en 30 días
- [ ] Compliance GDPR validado y documentado

#### **Fase 3: Enterprise Scale**
- [ ] Soporte para 10M usuarios concurrentes
- [ ] Latencia < 50ms en 95% de consultas globalmente  
- [ ] Automated scaling durante picos de tráfico
- [ ] Advanced monitoring con alertas predictivas

### 🎯 Métricas de Éxito del Negocio

#### **KPIs Técnicos**
- **DNS Resolution Time**: < 50ms (Objetivo: 30ms)
- **Service Availability**: 99.99% (Objetivo: 99.995%)
- **Geographic Accuracy**: > 98% (Objetivo: 99.5%)
- **Failover Recovery Time**: < 2 min (Objetivo: 90 segundos)

#### **KPIs de Negocio**
- **User Experience Score**: > 4.5/5.0 
- **Content Load Time**: < 3 segundos (Objetivo: 2 segundos)
- **Revenue Impact**: 0% pérdida durante incidentes
- **Market Expansion**: Soporte para nuevos países en < 48 horas

#### **KPIs de Compliance**
- **GDPR Compliance**: 100% usuarios EU en servidores EU
- **Content Geo-blocking**: 100% precisión en restricciones
- **Data Residency**: 100% cumplimiento por región
- **Audit Trail**: 100% de eventos DNS loggeados y auditables

---

## 🏗️ Arquitectura Objetivo

```
                                    streamflix.com
                                         │
                                    ┌─────────┐
                                    │Route 53 │
                                    │ Global  │
                                    │   DNS   │
                                    └────┬────┘
                                         │
                         ┌───────────────┼───────────────┐
                         │               │               │
                    ┌─────────┐     ┌─────────┐     ┌─────────┐
                    │Americas │     │ Europe  │     │  Asia   │
                    │(Primary)│     │(Secondary)│   │(Secondary)│
                    └─────────┘     └─────────┘     └─────────┘
                         │               │               │
                    ┌─────────┐     ┌─────────┐     ┌─────────┐
                    │us-east-1│     │eu-west-1│     │ap-south-1│
                    │         │     │         │     │         │
                    │CloudFront│    │CloudFront│    │CloudFront│
                    │   CDN    │    │   CDN    │    │   CDN    │
                    └─────────┘     └─────────┘     └─────────┘
                         │               │               │
                    ┌─────────┐     ┌─────────┐     ┌─────────┐
                    │   ALB   │     │   ALB   │     │   ALB   │
                    │Primary  │     │Standby  │     │Standby  │
                    └─────────┘     └─────────┘     └─────────┘
                         │               │               │
                ┌─────────────────┐ ┌─────────────┐ ┌─────────────┐
                │┌─────┐ ┌─────┐ │ │┌─────┐ ┌─────┐│ │┌─────┐ ┌─────┐│
                ││EC2-1│ │EC2-2│ │ ││EC2-1│ │EC2-2││ ││EC2-1│ │EC2-2││
                │└─────┘ └─────┘ │ │└─────┘ └─────┘│ │└─────┘ └─────┘│
                └─────────────────┘ └─────────────┘ └─────────────┘
```

---

## 📋 Lab Prerequisites

### Recursos AWS Necesarios

```yaml
Regions:
  Primary: us-east-1 (N. Virginia)
  Secondary: eu-west-1 (Ireland)  
  Tertiary: ap-southeast-1 (Singapore)

Components per Region:
  - VPC con subnets públicas
  - 2 EC2 instances (web servers)
  - Application Load Balancer
  - CloudFront distribution
  - Health checks configurados
```

### Domain Setup
```bash
# Dominio para el lab (usar uno de prueba o existente)
Domain: streamflix-lab.com
Subdomains:
  - www.streamflix-lab.com (web principal)
  - api.streamflix-lab.com (API backend)
  - admin.streamflix-lab.com (panel admin)
  - beta.streamflix-lab.com (versión beta)
```

---

## 🚀 Implementación Paso a Paso

### Fase 1: Preparación de Infraestructura

#### 1.1 Crear EC2 Instances en Múltiples Regiones

**US-East-1 (Virginia):**
```bash
#!/bin/bash
# User Data Script para EC2 US-East-1
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>StreamFlix - Americas</title>
    <style>
        body { font-family: Arial; background: #141414; color: white; text-align: center; padding: 50px; }
        .container { max-width: 800px; margin: 0 auto; }
        .region { color: #e50914; font-size: 24px; font-weight: bold; }
        .status { color: #46d369; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎬 StreamFlix</h1>
        <div class="region">Americas Region (Primary)</div>
        <div class="status">✅ Service Online</div>
        <p>Server: us-east-1a</p>
        <p>Instance ID: <span id="instance"></span></p>
        <p>Load Time: <span id="time"></span>ms</p>
        
        <div style="margin-top: 40px;">
            <h3>🎭 Featured Content - Americas</h3>
            <p>• Marvel Movies Collection</p>
            <p>• Breaking Bad Series</p>
            <p>• NFL Sunday Night Football</p>
        </div>
    </div>
    
    <script>
        // Simular datos dinámicos
        document.getElementById('instance').innerHTML = 'i-americas-' + Math.random().toString(36).substr(2, 9);
        document.getElementById('time').innerHTML = Math.floor(Math.random() * 50) + 20;
    </script>
</body>
</html>
EOF

# Health check endpoint
cat > /var/www/html/health << 'EOF'
{
  "status": "healthy",
  "region": "us-east-1",
  "service": "streamflix-web",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "checks": {
    "database": "connected",
    "cache": "active", 
    "storage": "available"
  }
}
EOF

# API endpoint simulado
mkdir /var/www/html/api
cat > /var/www/html/api/status << 'EOF'
{
  "api_version": "2.1.0",
  "region": "americas",
  "status": "operational",
  "latency_ms": 45,
  "active_streams": 15420,
  "content_library": {
    "movies": 8500,
    "series": 2100,
    "documentaries": 950
  }
}
EOF
```

**EU-West-1 (Ireland):**
```bash
#!/bin/bash
# User Data Script para EC2 EU-West-1
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>StreamFlix - Europe</title>
    <style>
        body { font-family: Arial; background: #141414; color: white; text-align: center; padding: 50px; }
        .container { max-width: 800px; margin: 0 auto; }
        .region { color: #e50914; font-size: 24px; font-weight: bold; }
        .status { color: #46d369; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎬 StreamFlix</h1>
        <div class="region">Europe Region (Secondary)</div>
        <div class="status">✅ Service Online</div>
        <p>Server: eu-west-1a</p>
        <p>Instance ID: <span id="instance"></span></p>
        <p>Load Time: <span id="time"></span>ms</p>
        
        <div style="margin-top: 40px;">
            <h3>🎭 Featured Content - Europe</h3>
            <p>• Premier League Highlights</p>
            <p>• BBC Documentaries</p>
            <p>• Eurovision Collection</p>
        </div>
    </div>
    
    <script>
        document.getElementById('instance').innerHTML = 'i-europe-' + Math.random().toString(36).substr(2, 9);
        document.getElementById('time').innerHTML = Math.floor(Math.random() * 40) + 25;
    </script>
</body>
</html>
EOF

cat > /var/www/html/health << 'EOF'
{
  "status": "healthy",
  "region": "eu-west-1", 
  "service": "streamflix-web",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "checks": {
    "database": "connected",
    "cache": "active",
    "storage": "available"
  }
}
EOF
```

**AP-Southeast-1 (Singapore):**
```bash
#!/bin/bash
# User Data Script para EC2 AP-Southeast-1
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>StreamFlix - Asia Pacific</title>
    <style>
        body { font-family: Arial; background: #141414; color: white; text-align: center; padding: 50px; }
        .container { max-width: 800px; margin: 0 auto; }
        .region { color: #e50914; font-size: 24px; font-weight: bold; }
        .status { color: #46d369; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎬 StreamFlix</h1>
        <div class="region">Asia Pacific Region</div>
        <div class="status">✅ Service Online</div>
        <p>Server: ap-southeast-1a</p>
        <p>Instance ID: <span id="instance"></span></p>
        <p>Load Time: <span id="time"></span>ms</p>
        
        <div style="margin-top: 40px;">
            <h3>🎭 Featured Content - Asia Pacific</h3>
            <p>• K-Drama Collection</p>
            <p>• Anime Series</p>
            <p>• Bollywood Movies</p>
        </div>
    </div>
    
    <script>
        document.getElementById('instance').innerHTML = 'i-asia-' + Math.random().toString(36).substr(2, 9);
        document.getElementById('time').innerHTML = Math.floor(Math.random() * 60) + 30;
    </script>
</body>
</html>
EOF
```

#### 1.2 Configurar Application Load Balancers

```bash
# US-East-1 ALB
aws elbv2 create-load-balancer \
    --name streamflix-alb-us \
    --subnets subnet-12345 subnet-67890 \
    --security-groups sg-web \
    --region us-east-1

# EU-West-1 ALB  
aws elbv2 create-load-balancer \
    --name streamflix-alb-eu \
    --subnets subnet-abc123 subnet-def456 \
    --security-groups sg-web \
    --region eu-west-1

# AP-Southeast-1 ALB
aws elbv2 create-load-balancer \
    --name streamflix-alb-ap \
    --subnets subnet-xyz789 subnet-uvw012 \
    --security-groups sg-web \
    --region ap-southeast-1
```

### Fase 2: Configuración Route 53

#### 2.1 Crear Hosted Zone

```bash
# Crear zona alojada principal
aws route53 create-hosted-zone \
    --name streamflix-lab.com \
    --caller-reference "streamflix-$(date +%s)" \
    --hosted-zone-config Comment="StreamFlix Lab Zone"

# Anotar Hosted Zone ID para uso posterior
HOSTED_ZONE_ID="Z1234567890ABC"
```

#### 2.2 Configurar Health Checks

```bash
# Health Check US-East-1
aws route53 create-health-check \
    --caller-reference "hc-us-east-$(date +%s)" \
    --health-check-config '{
        "Type": "HTTP",
        "ResourcePath": "/health",
        "RequestInterval": 30,
        "FailureThreshold": 3,
        "Port": 80,
        "IPAddress": "ALB-US-EAST-IP"
    }' \
    --region us-east-1

# Health Check EU-West-1
aws route53 create-health-check \
    --caller-reference "hc-eu-west-$(date +%s)" \
    --health-check-config '{
        "Type": "HTTP", 
        "ResourcePath": "/health",
        "RequestInterval": 30,
        "FailureThreshold": 3,
        "Port": 80,
        "IPAddress": "ALB-EU-WEST-IP"
    }' \
    --region eu-west-1

# Health Check AP-Southeast-1
aws route53 create-health-check \
    --caller-reference "hc-ap-southeast-$(date +%s)" \
    --health-check-config '{
        "Type": "HTTP",
        "ResourcePath": "/health", 
        "RequestInterval": 30,
        "FailureThreshold": 3,
        "Port": 80,
        "IPAddress": "ALB-AP-SOUTHEAST-IP"
    }' \
    --region ap-southeast-1
```

#### 2.3 Implementar Políticas de Enrutamiento

**Registro Principal - Geolocalización + Latencia:**

```bash
# Americas (Geolocation)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "www.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "Americas-Primary",
                "GeoLocation": {
                    "ContinentCode": "NA"
                },
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-US-EAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-US-ID"
            }
        }]
    }'

# Europe (Geolocation)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE", 
            "ResourceRecordSet": {
                "Name": "www.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "Europe-Primary",
                "GeoLocation": {
                    "ContinentCode": "EU"
                },
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-EU-WEST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-EU-ID"
            }
        }]
    }'

# Asia Pacific (Geolocation)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "www.streamflix-lab.com", 
                "Type": "A",
                "SetIdentifier": "Asia-Primary",
                "GeoLocation": {
                    "ContinentCode": "AS"
                },
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-AP-SOUTHEAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-AP-ID"
            }
        }]
    }'

# Default (Fallback Global)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "www.streamflix-lab.com",
                "Type": "A", 
                "SetIdentifier": "Global-Default",
                "GeoLocation": {
                    "CountryCode": "*"
                },
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-US-EAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-US-ID"
            }
        }]
    }'
```

#### 2.4 API Backend - Latency-Based Routing

```bash
# API US-East-1 (Latency-based)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "API-US-East",
                "Region": "us-east-1",
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-US-EAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-US-ID"
            }
        }]
    }'

# API EU-West-1 (Latency-based)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "API-EU-West", 
                "Region": "eu-west-1",
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-EU-WEST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-EU-ID"
            }
        }]
    }'

# API AP-Southeast-1 (Latency-based)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "API-AP-Southeast",
                "Region": "ap-southeast-1", 
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-AP-SOUTHEAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-AP-ID"
            }
        }]
    }'
```

#### 2.5 A/B Testing - Weighted Routing

```bash
# Versión Producción (90% tráfico)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "beta.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "Production-90",
                "Weight": 90,
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-US-EAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-US-ID"
            }
        }]
    }'

# Versión Beta (10% tráfico)  
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "beta.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "Beta-10",
                "Weight": 10,
                "TTL": 60, 
                "ResourceRecords": [{"Value": "ALB-EU-WEST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-EU-ID"
            }
        }]
    }'
```

#### 2.6 Disaster Recovery - Failover

```bash
# Primary (Active)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "admin.streamflix-lab.com",
                "Type": "A", 
                "SetIdentifier": "Admin-Primary",
                "Failover": "PRIMARY",
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-US-EAST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-US-ID"
            }
        }]
    }'

# Secondary (Standby)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "admin.streamflix-lab.com",
                "Type": "A",
                "SetIdentifier": "Admin-Secondary", 
                "Failover": "SECONDARY",
                "TTL": 60,
                "ResourceRecords": [{"Value": "ALB-EU-WEST-IP"}],
                "HealthCheckId": "HEALTH-CHECK-EU-ID"
            }
        }]
    }'
```

---

## 🧪 Escenarios de Prueba

### Prueba 1: Geolocalización
```bash
# Probar desde diferentes ubicaciones
# Simular desde US
curl -H "X-Forwarded-For: 192.0.2.1" http://www.streamflix-lab.com

# Simular desde Europa  
curl -H "X-Forwarded-For: 198.51.100.1" http://www.streamflix-lab.com

# Verificar resolución DNS
dig www.streamflix-lab.com @8.8.8.8
```

### Prueba 2: Latencia
```bash
# Medir latencia desde múltiples regiones
for region in us-east-1 eu-west-1 ap-southeast-1; do
    echo "Testing from $region:"
    dig api.streamflix-lab.com @8.8.8.8
    curl -w "Total time: %{time_total}s\n" -o /dev/null -s http://api.streamflix-lab.com/api/status
done
```

### Prueba 3: A/B Testing (Weighted)
```bash
# Ejecutar múltiples requests para verificar distribución
for i in {1..100}; do
    curl -s http://beta.streamflix-lab.com | grep "region" >> results.log
done

# Analizar distribución
grep -c "americas" results.log  # Debería ser ~90
grep -c "europe" results.log    # Debería ser ~10
```

### Prueba 4: Failover
```bash
# Simular fallo del primary
# 1. Detener instancias en US-East-1
aws ec2 stop-instances --instance-ids i-us-east-1-id --region us-east-1

# 2. Esperar health check failure (30-90 segundos)
watch -n 10 'dig admin.streamflix-lab.com @8.8.8.8'

# 3. Verificar failover a EU-West-1
curl http://admin.streamflix-lab.com
```

### Prueba 5: TTL Impact
```bash
# Cambiar TTL y medir propagación
# 1. Cambiar registro con TTL bajo (60s)
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "test-ttl.streamflix-lab.com",
                "Type": "A", 
                "TTL": 60,
                "ResourceRecords": [{"Value": "1.2.3.4"}]
            }
        }]
    }'

# 2. Esperar y cambiar a nueva IP
sleep 70
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT", 
            "ResourceRecordSet": {
                "Name": "test-ttl.streamflix-lab.com",
                "Type": "A",
                "TTL": 60,
                "ResourceRecords": [{"Value": "5.6.7.8"}]
            }
        }]
    }'

# 3. Monitorear cambio
watch -n 5 'dig test-ttl.streamflix-lab.com @8.8.8.8'
```

---

## 📊 Monitoreo y Métricas

### Dashboard CloudWatch

```yaml
StreamFlix Monitoring Dashboard:
  DNS Metrics:
    - Route53 Query Count por región
    - Health Check Status
    - Failover Events
    
  Application Metrics:
    - Request latency por región
    - Error rate por endpoint
    - Geographic distribution
    
  Business Metrics:  
    - Active users por región
    - Content delivery performance
    - A/B test conversion rates
```

### Alertas Configuradas

```bash
# Alert para Health Check failure
aws cloudwatch put-metric-alarm \
    --alarm-name "StreamFlix-HealthCheck-US-Failure" \
    --alarm-description "US region health check failed" \
    --metric-name HealthCheckPercentHealthy \
    --namespace AWS/Route53 \
    --statistic Average \
    --period 60 \
    --threshold 100 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-east-1:123456:streamflix-alerts

# Alert para alta latencia  
aws cloudwatch put-metric-alarm \
    --alarm-name "StreamFlix-High-Latency" \
    --alarm-description "API latency above 2 seconds" \
    --metric-name TargetResponseTime \
    --namespace AWS/ApplicationELB \
    --statistic Average \
    --period 300 \
    --threshold 2.0 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2
```

---

## 🎯 Escenarios de Optimización

### Optimización 1: Peak Traffic (Black Friday)

```yaml
Scenario: Viernes Negro - Tráfico 10x normal
Strategy:
  1. Pre-scaling:
     - Reducir TTL a 30s
     - Auto Scaling groups configurados
     - CloudFront pre-warming
     
  2. Traffic Management:
     - Weighted routing para load balancing
     - Geoproximity con bias hacia regiones menos cargadas
     - Multivalue responses para distribución
     
  3. Monitoring:
     - Real-time dashboards
     - Automated scaling triggers
     - Proactive failover thresholds
```

### Optimización 2: New Content Launch

```yaml
Scenario: Lanzamiento serie popular
Strategy:
  1. Gradual Rollout:
     - Weighted routing: 5% → 25% → 50% → 100%
     - A/B testing para nuevas features
     - Canary deployment por región
     
  2. Geographic Prioritization:
     - Geolocation para contenido regional
     - Latency-based para mejor experiencia
     - Failover preparado para sobrecarga
```

### Optimización 3: Compliance (GDPR)

```yaml
Scenario: Cumplimiento GDPR Europa
Strategy:
  1. Data Sovereignty:
     - Geolocation routing para usuarios EU
     - Datos procesados solo en eu-west-1
     - Backup y DR dentro de Europa
     
  2. Content Filtering:
     - Diferentes bibliotecas por región
     - Geolocation para contenido apropiado
     - Regional CDN configurations
```

---

## 🧪 Troubleshooting Guide

### Problema 1: Usuario en España accede a servidor US

```bash
# Diagnóstico
dig www.streamflix-lab.com @8.8.8.8
nslookup www.streamflix-lab.com 1.1.1.1

# Verificar configuración geolocation
aws route53 list-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --query "ResourceRecordSets[?Name=='www.streamflix-lab.com']"

# Solución: Verificar GeoLocation hierarchy
# 1. País específico (más específico)
# 2. Continente  
# 3. Default (menos específico)
```

### Problema 2: Failover no funciona

```bash
# Verificar health checks
aws route53 get-health-check --health-check-id HEALTH-CHECK-ID

# Verificar estado del health check
aws route53 get-health-check-status --health-check-id HEALTH-CHECK-ID

# Posibles causas:
# - Security groups bloquean health checkers
# - Endpoint /health no responde 200
# - TTL demasiado alto (usuarios ven cached)
# - Health check threshold muy alto
```

### Problema 3: A/B Testing distribución incorrecta

```bash
# Verificar weights
aws route53 list-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --query "ResourceRecordSets[?Name=='beta.streamflix-lab.com']"

# Calcular distribución real
for i in {1..1000}; do
    dig beta.streamflix-lab.com @8.8.8.8 +short >> distribution_test.log
done

# Analizar resultados
sort distribution_test.log | uniq -c
```

---

## 📈 Métricas de Éxito

### KPIs del Lab

```yaml
Performance KPIs:
  DNS Resolution Time: < 50ms
  Failover Time: < 2 minutos
  Geographic Accuracy: > 95%
  Health Check Detection: < 90 segundos
  
Business KPIs:
  A/B Test Distribution: ±2% of target
  Regional Content Compliance: 100%
  Uptime: > 99.9%
  User Experience: Latency < 200ms
```

### Reportes Automatizados

```python
#!/usr/bin/env python3
# StreamFlix Route53 Health Report

import boto3
import json
from datetime import datetime, timedelta

def generate_health_report():
    route53 = boto3.client('route53')
    cloudwatch = boto3.client('cloudwatch')
    
    # Obtener health checks
    health_checks = route53.list_health_checks()
    
    report = {
        'timestamp': datetime.utcnow().isoformat(),
        'overall_status': 'HEALTHY',
        'regions': {}
    }
    
    for hc in health_checks['HealthChecks']:
        hc_id = hc['Id']
        status = route53.get_health_check_status(HealthCheckId=hc_id)
        
        region = hc.get('HealthCheckConfig', {}).get('Region', 'unknown')
        report['regions'][region] = {
            'status': 'HEALTHY' if len([s for s in status['CheckStatus'] if s['Status'] == 'Success']) > 0 else 'UNHEALTHY',
            'last_check': max([s['CheckedAt'] for s in status['CheckStatus']]).isoformat()
        }
    
    # Generar alerta si alguna región está unhealthy
    unhealthy_regions = [r for r, data in report['regions'].items() if data['status'] == 'UNHEALTHY']
    if unhealthy_regions:
        report['overall_status'] = 'DEGRADED'
        send_alert(f"Regions unhealthy: {', '.join(unhealthy_regions)}")
    
    return report

def send_alert(message):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456:streamflix-alerts',
        Message=message,
        Subject='StreamFlix Health Alert'
    )

if __name__ == '__main__':
    report = generate_health_report()
    print(json.dumps(report, indent=2))
```

---

## 🎓 Conclusiones del Lab

### Conceptos Aplicados

1. **Geolocalización**: Contenido diferente por región
2. **Latencia**: API endpoints optimizados globalmente  
3. **Weighted Routing**: A/B testing controlado
4. **Failover**: Disaster recovery automático
5. **Health Checks**: Monitoreo proactivo
6. **TTL Strategy**: Balance entre performance y flexibilidad

### Lessons Learned

```yaml
Key Takeaways:
  DNS Strategy:
    - Geolocation para compliance y localización
    - Latency-based para APIs críticas
    - Failover para servicios críticos
    - Weighted para testing gradual
    
  Operational Excellence:
    - Health checks obligatorios para production
    - TTL bajo durante cambios, alto para estabilidad
    - Monitoring y alertas automatizadas
    - Disaster recovery probado regularmente
    
  Cost Optimization:
    - Consolidar health checks donde posible
    - Optimizar TTL para reducir consultas
    - Usar alias records para recursos AWS
    - Monitorear costos por consulta
```

### Próximos Pasos

1. **CloudFormation Templates**: Automatizar deployment
2. **Route 53 Application Recovery Controller**: Advanced DR
3. **Private Hosted Zones**: Para servicios internos
4. **Route 53 Resolver**: Para hybrid DNS
5. **Integration con CI/CD**: Automated blue/green deployments

---

## 📚 Recursos del Lab

### Scripts de Automatización

- `deploy-infrastructure.sh`: Deploy completo multi-región
- `configure-route53.sh`: Configuración DNS automatizada  
- `health-check-monitor.py`: Monitoreo health checks
- `traffic-test.sh`: Testing de distribución de tráfico
- `cleanup-lab.sh`: Limpieza de recursos

### Documentación Adicional

- Arquitectura de referencia StreamFlix
- Runbook para incident response
- Playbook para disaster recovery
- Guía de troubleshooting DNS

---

*StreamFlix Lab - Route 53 Masterclass - Agosto 2025*
** Roberto Ayra**
