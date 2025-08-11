# Soluciones Clásicas de Arquitectura AWS
## Guía de Preparación para Certificación

---

## 📋 Índice

1. [Introducción](#introducción)
2. [Caso 1: WhatsTheTime.com - Arquitectura Simple a Escalable](#caso-1-whatsthetimecom)
3. [Caso 2: MyClothes.com - Aplicación Web con Estado](#caso-2-myclothescom)
4. [Caso 3: MyWordPress.com - CMS con Almacenamiento Compartido](#caso-3-mywordpresscom)
5. [Mejores Prácticas de Lanzamiento Rápido](#mejores-prácticas-de-lanzamiento-rápido)
6. [Arquitectura de 3 Niveles y Elastic Beanstalk](#arquitectura-de-3-niveles)
7. [Pilares del Well-Architected Framework](#pilares-del-well-architected-framework)

## 🎨 Guía de Diagramas Profesionales

> **IMPORTANTE**: Los diagramas de este documento están optimizados para conversión a herramientas profesionales:
> 
> ### 📐 Herramientas Recomendadas:
> - **Draw.io (Gratuito)**: app.diagrams.net - AWS Icons incluidos
> - **Lucidchart (Comercial)**: Templates AWS profesionales 
> - **Visio (Empresarial)**: Stencils AWS oficiales
> 
> ### 📦 Recursos Oficiales:
> - **AWS Architecture Icons**: https://aws.amazon.com/architecture/icons/
> - **Plantillas Draw.io**: Ver archivo `plantillas-diagramas-aws.md`
> - **Color Palette AWS**: #FF9900 (Orange), #232F3E (Dark Blue), #4B92DB (Light Blue)
> 
> ### 🔄 Proceso de Conversión:
> 1. Copiar diagrama PlantUML/Mermaid
> 2. Importar en herramienta seleccionada
> 3. Sustituir cada servicio por icono oficial AWS
> 4. Aplicar colores y layout corporativo

---

## Introducción

Este documento presenta casos de estudio de arquitecturas AWS que evolucionan desde implementaciones básicas hasta soluciones empresariales robustas. Cada caso demuestra la aplicación práctica de los servicios AWS y los principios del Well-Architected Framework.

---

## Caso 1: WhatsTheTime.com
### Evolución de Arquitectura Simple a Escalable

#### 📊 Requisitos del Negocio
- **Función**: Servicio web que muestra la hora actual
- **Características**:
  - No requiere base de datos
  - Tolera breve inactividad
  - Necesita escalabilidad vertical y horizontal
  - Objetivo: cero tiempo de inactividad

#### 🏗️ Evolución Arquitectónica

##### **Fase 1: Arquitectura Básica**

```
🏗️ Fase 1: Single Instance
┌─────────────┐    ┌─────────────────┐
│   Usuario   │───▶│   EC2 t2.micro  │
│  (Cliente)  │    │   IP Elástica   │
└─────────────┘    │   Subred Pública│
                   └─────────────────┘
```

**Componentes:**
- 1 instancia EC2 t2.micro
- 1 IP Elástica asignada
- Subred pública

**Limitaciones:**
- Punto único de falla
- No escalable
- Dependencia de una sola instancia

---

##### **Fase 2: Escalado Vertical**

```
⬆️ Fase 2: Vertical Scaling
┌─────────────┐    ┌─────────────────┐
│   Usuario   │───▶│  EC2 m5.large   │
│  (Cliente)  │    │   IP Elástica   │
└─────────────┘    │   Subred Pública│
                   └─────────────────┘
```

**Mejoras:**
- Upgrade de t2.micro a m5.large
- Mayor capacidad de procesamiento
- Mantiene IP Elástica

**Limitaciones:**
- Sigue siendo un punto único de falla
- Tiempo de inactividad durante el cambio de instancia

---

##### **Fase 3: Escalado Horizontal Básico**

```
➡️ Fase 3: Horizontal Scaling (Problemático)
┌─────────────┐    ┌─────────────────┐
│   Usuarios  │───▶│  EC2 m5 + IP-1  │
│             │    └─────────────────┘
│             │    ┌─────────────────┐
│             │───▶│  EC2 m5 + IP-2  │
│             │    └─────────────────┘
│             │    ┌─────────────────┐
│             │───▶│  EC2 m5 + IP-3  │
└─────────────┘    └─────────────────┘
```

**Componentes:**
- 3 instancias EC2 m5.large
- 3 IPs Elásticas independientes

**Limitaciones:**
- Gestión manual de múltiples IPs
- No hay distribución automática de carga
- Complejidad operacional

---

##### **Fase 4: Introducción de Route 53**

```
🌐 Fase 4: DNS Load Balancing
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Usuarios  │───▶│    Route 53     │───▶│   EC2 Instance  │
│             │    │ Registro A      │    │      #1         │
└─────────────┘    │ TTL: 3600       │    └─────────────────┘
                   │ Round Robin     │    ┌─────────────────┐
                   │                 │───▶│   EC2 Instance  │
                   │                 │    │      #2         │
                   │                 │    └─────────────────┘
                   │                 │    ┌─────────────────┐
                   │                 │───▶│   EC2 Instance  │
                   └─────────────────┘    │      #3         │
                                          └─────────────────┘
```

**Mejoras:**
- Eliminación de IPs Elásticas
- DNS Round Robin con Route 53
- Registro tipo A con TTL de 1 hora

**Limitaciones:**
- TTL de 1 hora = 1 hora de indisponibilidad si falla una instancia
- No hay health checks automáticos

---

##### **Fase 5: Load Balancer + Route 53 Alias**

```
⚖️ Fase 5: Application Load Balancer
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Internet   │───▶│    Route 53     │───▶│  Application    │
│             │    │ Alias Record    │    │ Load Balancer   │
└─────────────┘    │ (No TTL cache)  │    │ Health Checks   │
                   └─────────────────┘    └─────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │              Multi-AZ               │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │ EC2 Private │   │ EC2 Private │   │ EC2 Private │
            │    AZ-1a    │   │    AZ-1b    │   │    AZ-1c    │
            └─────────────┘   └─────────────┘   └─────────────┘
```

**Componentes:**
- Application Load Balancer (ALB)
- Route 53 con registro Alias
- Instancias EC2 en subredes privadas
- Health checks automáticos

**Mejoras:**
- Health checks del ELB
- Failover automático
- Instancias en subredes privadas (seguridad)
- Sin TTL issues (Alias records)

---

##### **Fase 6: Auto Scaling Group**

```
🔄 Fase 6: Auto Scaling + Multi-AZ
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Usuarios   │───▶│    Route 53     │───▶│  Application    │
│             │    │                 │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    └─────────────────┘
                                                   │
                                           ┌─────────────────┐
                                           │ Auto Scaling    │
                                           │ Group           │
                                           │ Min: 2, Max: 10 │
                                           └─────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │          Managed Instances              │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │    EC2      │   │    EC2      │   │    EC2      │
            │   AZ-1a     │   │   AZ-1b     │   │   AZ-1c     │
            └─────────────┘   └─────────────┘   └─────────────┘
```

**Componentes:**
- Auto Scaling Group con mínimo 2, máximo 10 instancias
- Distribución en múltiples AZs
- Escalado automático basado en métricas

**Mejoras:**
- Escalado automático
- Gestión automática de instancias
- Distribución multi-AZ
- Resiliencia ante fallos de AZ

---

##### **Fase 7: Optimización de Costos**

```
💰 Fase 7: Mixed Instance Types
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Usuarios   │───▶│    Route 53     │───▶│  Application    │
│             │    │                 │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    └─────────────────┘
                                                   │
                                        ┌─────────────────────┐
                                        │  Auto Scaling Group │
                                        │  Mixed Instance     │
                                        │  Strategy           │
                                        └─────────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │        Cost Optimized Mix              │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │Reserved EC2 │   │Reserved EC2 │   │ Spot EC2    │
            │   AZ-1a     │   │   AZ-1b     │   │   AZ-1c     │
            │  (Base Load)│   │ (Base Load) │   │(Peak Load)  │
            └─────────────┘   └─────────────┘   └─────────────┘
```

**Optimizaciones:**
- Reserved Instances para capacidad base
- Spot Instances para capacidad adicional
- Mínimo 2 AZs para redundancia

---

#### 🎯 Conceptos Clave del Caso 1

| Concepto | Descripción | Beneficio |
|----------|-------------|-----------|
| **IP Pública vs Privada** | Instancias públicas vs privadas detrás de LB | Mayor seguridad |
| **IP Elástica vs Route 53** | DNS vs IP fija | Flexibilidad y gestión |
| **Route 53 TTL** | Tiempo de vida de registros DNS | Control de caché |
| **Registros A vs Alias** | IP vs recurso AWS | Integración nativa |
| **Auto Scaling** | Escalado automático vs manual | Eficiencia operacional |
| **Multi-AZ** | Distribución geográfica | Alta disponibilidad |
| **Health Checks** | Monitoreo automático | Detección de fallos |
| **Security Groups** | Firewall de instancias | Seguridad de red |
| **Reserved Instances** | Reserva de capacidad | Optimización de costos |

#### 🏛️ Pilares del Well-Architected Framework Aplicados
- **💰 Optimización de Costos**: Reserved Instances, Auto Scaling
- **⚡ Rendimiento**: Load Balancing, Multi-AZ
- **🛡️ Fiabilidad**: Auto Scaling, Health Checks, Multi-AZ
- **🔒 Seguridad**: Security Groups, subredes privadas
- **🔧 Excelencia Operacional**: Automatización, monitoreo

---

## Caso 2: MyClothes.com
### Aplicación de E-commerce con Gestión de Sesiones

#### 📊 Requisitos del Negocio
- **Función**: Tienda en línea con carrito de compras
- **Características**:
  - Cientos de usuarios simultáneos
  - Persistencia del carrito de compras
  - Aplicación web stateless
  - Base de datos para información de usuarios
  - Escalabilidad horizontal

#### 🏗️ Evolución Arquitectónica

##### **Fase 1: Problema - Pérdida de Sesión**

```
❌ Problema: Session Loss
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Cliente   │───▶│    Route 53     │───▶│  Application    │
│             │    │                 │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    └─────────────────┘
                                                   │
                                    Round Robin Distribution
                                                   │
                   ┌─────────────────────────────────────────┐
                   │                                         │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │    EC2 A    │   │    EC2 B    │   │    EC2 C    │
            │(sesión cart)│   │ (sin sesión)│   │ (sin sesión)│
            └─────────────┘   └─────────────┘   └─────────────┘
                   ▲                 ▲                 ▲
                   │                 │                 │
            Usuario agrega     Usuario refresh    Usuario pierde
            productos         redirige aquí      carrito completo
```

**Problema Identificado:**
- El Load Balancer redirige a diferentes instancias
- Cada instancia mantiene sesión local
- El carrito se pierde al cambiar de instancia

---

##### **Fase 2: Sesiones Persistentes (Sticky Sessions)**
```
🏗️ Sticky Sessions con ALB
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE                              │
│  ┌─────────────┐                                        │
│  │   Usuario   │                                        │
│  │     🧑‍💻     │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │ Session ID: XYZ123
┌─────────────────────────────────────────────────────────┐
│                  DNS + LOAD BALANCER                    │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │  Route 53   │  │           ALB                   │   │
│  │ Global DNS  │  │    (Sticky Sessions)            │   │
│  │     🌐      │  │ Siempre → Misma Instancia       │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │ Sticky Route
┌─────────────────────────────────────────────────────────┐
│                  EC2 INSTANCES                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  ⭐ EC2-A   │  │   EC2-B     │  │   EC2-C     │     │
│  │ (Selected)  │  │ (Inactive)  │  │ (Inactive)  │     │
│  │ Session     │  │             │  │             │     │
│  │ Storage     │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

⚠️ Limitaciones:
• Si EC2-A falla → Sesión perdida
• Distribución desigual de carga  
• No verdaderamente stateless
```

**Configuración:**
- Habilitación de Session Stickiness en ALB
- Cliente siempre dirigido a la misma instancia

**Limitaciones:**
- Si la instancia falla, se pierde la sesión
- Distribución desigual de carga
- No verdaderamente stateless

---

##### **Fase 3: Cookies del Cliente**
```
🏗️ Client-Side Cookie Storage
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE                              │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │   Browser   │  │         Cookies                 │   │
│  │     🌐      │  │   Cart: item1, item2, item3     │   │
│  │             │  │   Session: user123              │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │ HTTP + Cookies
┌─────────────────────────────────────────────────────────┐
│                  DNS + LOAD BALANCER                    │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │  Route 53   │  │           ALB                   │   │
│  │ Global DNS  │  │    (Round Robin)                │   │
│  │     🌐      │  │ Distribución Equitativa         │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │ Balanced Traffic
┌─────────────────────────────────────────────────────────┐
│                  EC2 INSTANCES                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   EC2-A     │  │   EC2-B     │  │   EC2-C     │     │
│  │ Lee Cookies │  │ Lee Cookies │  │ Lee Cookies │     │
│  │     ✅      │  │     ✅      │  │     ✅      │     │
│  │  Stateless  │  │  Stateless  │  │  Stateless  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

⚠️ Limitaciones:
• Cookies pueden ser alteradas por usuario
• Límite de tamaño (4KB max)
• Datos sensibles expuestos
```

**Implementación:**
- Almacenamiento del carrito en cookies del navegador
- Cualquier instancia puede leer las cookies

**Limitaciones:**
- Riesgo de seguridad (alteración de cookies)
- Tamaño limitado de cookies
- Dependencia del cliente

---

##### **Fase 4: Sesiones del Servidor con ElastiCache**

```
✅ Solución: External Session Store
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Cliente   │───▶│    Route 53     │───▶│  Application    │
│             │    │                 │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    └─────────────────┘
                                                   │
                                           ┌─────────────────┐
                                           │ Auto Scaling    │
                                           │ Group           │
                                           └─────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │          Stateless Apps                 │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │    EC2      │   │    EC2      │   │    EC2      │
            │ Stateless   │   │ Stateless   │   │ Stateless   │
            └─────────────┘   └─────────────┘   └─────────────┘
                   │                 │                 │
                   └─────────────────┼─────────────────┘
                                     │
                                     ▼
                              ┌─────────────────┐
                              │   ElastiCache   │
                              │  Redis Cluster  │
                              │ Session Storage │
                              └─────────────────┘
```

**Componentes:**
- ElastiCache Redis/Memcached para sesiones
- Todas las instancias acceden al mismo store de sesiones
- Aplicación completamente stateless

**Alternativa:**
- Amazon DynamoDB para persistencia de sesiones

---

##### **Fase 5: Base de Datos para Usuarios**
```
🏗️ Database-Driven Architecture
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE                              │
│  ┌─────────────┐                                        │
│  │   Usuario   │                                        │
│  │     🧑‍💻     │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                  LOAD BALANCER                          │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │  Route 53   │  │           ALB                   │   │
│  │ Global DNS  │  │         + ASG                   │   │
│  │     🌐      │  │    Auto Scaling                 │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│               APPLICATION TIER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   EC2-A     │  │   EC2-B     │  │   EC2-C     │     │
│  │  Stateless  │  │  Stateless  │  │  Stateless  │     │
│  │     📱      │  │     📱      │  │     📱      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
      │                    │                    │
┌─────────────────────────────────────────────────────────┐
│                 CACHE LAYER                             │
│          ┌─────────────────────────────────┐            │
│          │        ElastiCache              │            │
│          │    (Session Storage)            │            │
│          │           Redis                 │            │
│          └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                 DATABASE LAYER                          │
│          ┌─────────────────────────────────┐            │
│          │           RDS MySQL             │            │
│          │      (User Data Store)          │            │
│          │   Users, Products, Orders       │            │
│          └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘

✅ Beneficios:
• Persistencia completa de datos
• Aplicaciones totalmente stateless
• Escalabilidad horizontal ilimitada
• Recuperación ante fallos automática
```

**Componentes:**
- RDS MySQL para datos persistentes de usuarios
- ElastiCache para sesiones temporales
- Separación de responsabilidades

---

##### **Fase 6: Escalado de Lecturas**
```
🏗️ Read Scaling Architecture
┌─────────────────────────────────────────────────────────┐
│                  LOAD BALANCER                          │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │     ALB     │  │           ASG                   │   │
│  │    🎯       │  │      Auto Scaling               │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│               APPLICATION TIER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   EC2-W1    │  │   EC2-W2    │  │   EC2-W3    │     │
│  │  (Writes)   │  │  (Reads)    │  │  (Reads)    │     │
│  │     📝      │  │     👁️      │  │     👁️      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
      │                    │                    │
┌─────────────────────────────────────────────────────────┐
│                 CACHE LAYER                             │
│          ┌─────────────────────────────────┐            │
│          │        ElastiCache              │            │
│          │     (Session + Data)            │            │
│          │           Redis                 │            │
│          └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                 DATABASE LAYER                          │
│  ┌─────────────┐              ┌─────────────┐           │
│  │ RDS Master  │ Replicate    │ Read Replica│           │
│  │ (Writes)    │ ──────────>  │ (Reads Only)│           │
│  │     💾      │              │     📖      │           │
│  └─────────────┘              └─────────────┘           │
└─────────────────────────────────────────────────────────┘

🎯 Traffic Distribution:
• W1 → Master (INSERT, UPDATE, DELETE)
• W2, W3 → Read Replica (SELECT queries)
• All → ElastiCache (Session + Cache)
```
    classDef ec2 fill:#FF9900,stroke:#333;
    classDef cache fill:#DCFAF8,stroke:#0aa;
    classDef db fill:#B3E5FC,stroke:#036;
    class ALB alb; class ASG asg; class W1,W2,W3 ec2; class Cache cache; class Master,Replica db;
```

**Optimizaciones:**
- RDS Master para escrituras
- Read Replicas para lecturas
- Cache hit optimization en ElastiCache

---

##### **Fase 7: Arquitectura Final Multi-AZ**

```
🏗️ Arquitectura Final E-commerce Multi-AZ
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Internet   │───▶│    Route 53     │───▶│  Application    │
│             │    │                 │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    └─────────────────┘
                                                   │
                                        ┌─────────────────────┐
                                        │  Auto Scaling Group │
                                        │    Multi-AZ         │
                                        └─────────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │              Application Tier           │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │   EC2 App   │   │   EC2 App   │   │   EC2 App   │
            │    AZ-a     │   │    AZ-b     │   │    AZ-c     │
            └─────────────┘   └─────────────┘   └─────────────┘
                   │                 │                 │
                   └─────────────────┼─────────────────┘
                                     │
                   ┌─────────────────────────────────────────┐
                   │               Data Tier                 │
                   └─────────────────────────────────────────┘
                                     │
                   ┌─────────────────┼─────────────────┐
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │ElastiCache  │   │ RDS Master  │   │Read Replica │
            │Multi-AZ     │   │   MySQL     │   │    AZ-b     │
            │(Sessions)   │   │   AZ-a      │   └─────────────┘
            └─────────────┘   └─────────────┘   ┌─────────────┐
                                     │          │Read Replica │
                                     │          │    AZ-c     │
                                     └─────────▶└─────────────┘
```

---

#### 🔒 Configuración de Seguridad

##### **Security Groups:**

**ALB Security Group:**
```
Inbound:
- HTTP (80): 0.0.0.0/0
- HTTPS (443): 0.0.0.0/0

Outbound:
- All traffic to EC2 Security Group
```

**EC2 Security Group:**
```
Inbound:
- HTTP (80): ALB Security Group
- HTTPS (443): ALB Security Group

Outbound:
- MySQL (3306): RDS Security Group
- Redis (6379): ElastiCache Security Group
```

**RDS Security Group:**
```
Inbound:
- MySQL (3306): EC2 Security Group

Outbound:
- None required
```

**ElastiCache Security Group:**
```
Inbound:
- Redis (6379): EC2 Security Group

Outbound:
- None required
```

---

## Caso 3: MyWordPress.com
### CMS con Almacenamiento Compartido

#### 📊 Requisitos del Negocio
- **Función**: Sitio web WordPress
- **Características**:
  - Subida y visualización de imágenes
  - Base de datos MySQL para contenido
  - Escalabilidad horizontal
  - Persistencia de archivos entre instancias

#### 🏗️ Evolución Arquitectónica

##### **Fase 1: Problema - Almacenamiento Local**
```
🏗️ WordPress Almacenamiento Local (Problemático)
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE                              │
│  ┌─────────────┐                                        │
│  │   Usuario   │                                        │
│  │     🧑‍💻     │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                  LOAD BALANCER                          │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │  Route 53   │  │           ALB                   │   │
│  │ Global DNS  │  │         + ASG                   │   │
│  │     🌐      │  │    Auto Scaling                 │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│               APPLICATION TIER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   EC2 + EBS │  │   EC2 + EBS │  │   EC2 + EBS │     │
│  │ WordPress   │  │ WordPress   │  │ WordPress   │     │
│  │    📱💾     │  │    📱💾     │  │    📱💾     │     │
│  │ /wp-uploads │  │ /wp-uploads │  │ /wp-uploads │     │
│  │ (Aislado)   │  │ (Aislado)   │  │ (Aislado)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                 DATABASE LAYER                          │
│          ┌─────────────────────────────────┐            │
│          │           RDS MySQL             │            │
│          │         (Metadata)              │            │
│          │    Posts, Users, Config         │            │
│          └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘

❌ Problemas:
• Archivos almacenados localmente en EBS
• Otras instancias no pueden acceder a imágenes
• Pérdida de archivos al escalar
• Media no sincronizada entre instancias
```

**Problema Identificado:**
- Imágenes almacenadas localmente en EBS
- Otras instancias no pueden acceder a las imágenes
- Pérdida de archivos al escalar

---

##### **Fase 2: Solución con EFS**
```
🏗️ WordPress + EFS + Aurora Multi-AZ
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE                              │
│  ┌─────────────┐                                        │
│  │   Usuario   │                                        │
│  │     🧑‍💻     │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                  LOAD BALANCER                          │
│  ┌─────────────┐  ┌─────────────────────────────────┐   │
│  │  Route 53   │  │           ALB                   │   │
│  │ Global DNS  │  │         + ASG                   │   │
│  │     🌐      │  │    Auto Scaling                 │   │
│  └─────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│               APPLICATION TIER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   EC2-A     │  │   EC2-B     │  │   EC2-C     │     │
│  │ WordPress   │  │ WordPress   │  │ WordPress   │     │
│  │    📱       │  │    📱       │  │    📱       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
      │                    │                    │
┌─────────────────────────────────────────────────────────┐
│                 SHARED STORAGE                          │
│          ┌─────────────────────────────────┐            │
│          │             EFS                 │            │
│          │       Shared Files              │            │
│          │    /wp-content/uploads          │            │
│          │       Network File              │            │
│          └─────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────────┐
│                 DATABASE CLUSTER                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │Aurora Master│  │Aurora Replica│  │Aurora Replica│    │
│  │ (Writer)    │  │ (Reader)    │  │ (Reader)    │     │
│  │    💾       │  │     📖      │  │     📖      │     │
│  │   AZ-a      │  │    AZ-b     │  │    AZ-c     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

✅ Beneficios:
• Media files compartidos entre instancias
• Auto-scaling sin pérdida de archivos
• Aurora Multi-AZ para alta disponibilidad
• Backup automático de archivos y DB
```
    classDef storage fill:#E8F5E9,stroke:#2E7D32;
    classDef db fill:#B3E5FC,stroke:#036;
    class ALB alb; class ASG asg; class A1,A2,A3 ec2; class EFS storage; class Aurora,R1,R2 db;
```

**Componentes:**
- **EFS (Elastic File System)**: Sistema de archivos compartido
- **Aurora MySQL**: Base de datos con réplicas de lectura
- **ENI**: Cada instancia monta EFS via ENI

---

##### **Fase 3: Arquitectura Final Optimizada**

```
🏗️ WordPress + EFS + Aurora Multi-AZ
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│Content Mgr  │───▶│    Route 53     │───▶│  Application    │
│ (Uploads)   │    │  wordpress.com  │    │ Load Balancer   │
└─────────────┘    └─────────────────┘    │  SSL Terminate  │
                                          └─────────────────┘
                                                   │
                                        ┌─────────────────────┐
                                        │  Auto Scaling Group │
                                        │    WordPress        │
                                        └─────────────────────┘
                                                   │
                   ┌─────────────────────────────────────────┐
                   │            WordPress Tier               │
                   └─────────────────────────────────────────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │WordPress    │   │WordPress    │   │WordPress    │
            │  t3.medium  │   │  t3.medium  │   │  t3.medium  │
            │   AZ-a      │   │   AZ-b      │   │   AZ-c      │
            └─────────────┘   └─────────────┘   └─────────────┘
                   │                 │                 │
                   └─────────────────┼─────────────────┘
                                     │
                                     ▼
                              ┌─────────────────┐
                              │       EFS       │
                              │  Shared Storage │
                              │ wp-content/uploads │
                              │   Multi-AZ      │
                              └─────────────────┘
                                     │
                   ┌─────────────────────────────────────────┐
                   │            Database Tier                │
                   └─────────────────────────────────────────┘
                                     │
                   ┌─────────────────┼─────────────────┐
                   │                 │                 │
                   ▼                 ▼                 ▼
            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
            │Aurora Writer│   │Aurora Reader│   │Aurora Reader│
            │   MySQL     │   │    AZ-b     │   │    AZ-c     │
            │   AZ-a      │   │             │   │             │
            └─────────────┘   └─────────────┘   └─────────────┘
```

---

#### 🔧 Configuración Técnica

##### **EFS Mount Targets:**
```bash
# En cada AZ, crear mount target
AZ-1a: subnet-xxx (ENI en EC2s de AZ-1a)
AZ-1b: subnet-yyy (ENI en EC2s de AZ-1b)
AZ-1c: subnet-zzz (ENI en EC2s de AZ-1c)
```

##### **User Data Script para EC2:**
```bash
#!/bin/bash
yum update -y
yum install -y amazon-efs-utils

# Montar EFS
mkdir -p /var/www/html/wp-content/uploads
mount -t efs fs-xxxxxxxx:/ /var/www/html/wp-content/uploads

# Hacer permanente el mount
echo 'fs-xxxxxxxx.efs.region.amazonaws.com:/ /var/www/html/wp-content/uploads efs defaults,_netdev' >> /etc/fstab
```

---

## Mejores Prácticas de Lanzamiento Rápido

### 🚀 Optimización del Tiempo de Implementación

#### Problemática Común
Cuando se lanza un stack completo (EC2 + EBS + RDS), el tiempo de inicialización puede ser significativo debido a:
- Instalación de aplicaciones
- Inserción de datos iniciales
- Configuración del sistema
- Primera ejecución de la aplicación

---

#### 💡 Estrategias de Optimización

##### **1. EC2 Optimization**

**Golden AMI Approach:**
```
🚀 Golden AMI - Lanzamiento Rápido
┌─────────────────────────────────────────────────────────┐
│               IMAGEN PREPARATION                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Base AMI   │  │ Config +    │  │ Pre-Install │     │
│  │  Amazon     │  │ Hardening   │  │ Apps        │     │
│  │  Linux 2    │  │    ⚙️       │  │    📦       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         │                │                │            │
│         └────────────────┼────────────────┘            │
│                          ▼                             │
│                  ┌─────────────┐                       │
│                  │ Golden AMI  │                       │
│                  │   Custom    │                       │
│                  │     🥇      │                       │
│                  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   DEPLOYMENT                            │
│                  ┌─────────────┐                       │
│                  │ Launch EC2  │                       │
│                  │ from Golden │                       │
│                  │    AMI      │                       │
│                  │     🚀      │                       │
│                  └─────────────┘                       │
│                          │                             │
│                  ┌─────────────┐                       │
│                  │Ready in     │                       │
│                  │2-3 minutes  │                       │
│                  │     ✅      │                       │
│                  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

**Ventajas:**
- Tiempo de boot reducido
- Configuración pre-establecida
- Aplicaciones pre-instaladas

**User Data Approach:**
```
🐌 User Data - Lanzamiento Lento
┌─────────────────────────────────────────────────────────┐
│                   DEPLOYMENT                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Base AMI   │  │ Launch EC2  │  │User Data    │     │
│  │  Amazon     │  │ Instance    │  │Script Run   │     │
│  │  Linux 2    │  │     🚀      │  │    📜       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         │                │                │            │
│         └────────────────┼────────────────┘            │
│                          ▼                             │
│     ┌─────────────────────────────────────────────┐     │
│     │          User Data Execution               │     │
│     │  • yum update                              │     │
│     │  • Install packages                        │     │
│     │  • Configure services                      │     │
│     │  • Start applications                      │     │
│     │  ⏱️ 10-15 minutes                          │     │
│     └─────────────────────────────────────────────┘     │
│                          │                             │
│                  ┌─────────────┐                       │
│                  │Ready after  │                       │
│                  │ 10-15 min   │                       │
│                  │     ⏰      │                       │
│                  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

**Ventajas:**
- Flexibilidad en configuración
- Fácil versionado
- Menor mantenimiento de AMIs

**Enfoque Híbrido (Recomendado):**
```
🎯 Enfoque Híbrido - Balance Perfecto
┌─────────────────────────────────────────────────────────┐
│               BASE PREPARATION                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │            Golden AMI Base                      │   │
│  │     OS + Security + Core Apps                   │   │
│  │              📦⚙️🔒                             │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   DEPLOYMENT                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Launch EC2  │  │ User Data   │  │  Ready      │     │
│  │from Golden  │  │Final Config │  │ Fast Boot   │     │
│  │    AMI      │  │   Tweaks    │  │   3-5 min   │     │
│  │     🚀      │  │     🔧      │  │     ✅      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

✅ Mejor de ambos mundos:
• Base sólida con Golden AMI
• Flexibilidad con User Data
• Tiempo de boot optimizado
```

---

##### **2. RDS Optimization**

**Snapshot Restoration:**
```
💾 RDS Snapshot Restoration
┌─────────────────────────────────────────────────────────┐
│                   SOURCE                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │            RDS Production                       │   │
│  │         ┌─────────────┐                        │   │
│  │         │ Database    │                        │   │
│  │         │ + Data      │                        │   │
│  │         │    💽       │                        │   │
│  │         └─────────────┘                        │   │
│  │               │                                │   │
│  │               ▼                                │   │
│  │         ┌─────────────┐                        │   │
│  │         │  Snapshot   │                        │   │
│  │         │  Backup     │                        │   │
│  │         │     📸      │                        │   │
│  │         └─────────────┘                        │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                  RESTORATION                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Restore   │  │ New RDS     │  │   Ready     │     │
│  │ from Snap   │  │ Instance    │  │ with Data   │     │
│  │     🔄      │  │     🗄️      │  │   5-10 min  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

⚡ Ventajas: Datos pre-existentes, setup rápido
⚠️ Tiempo: 5-10 minutos vs 15-20 desde cero
```
    RST --> NRDS[Nuevo RDS con Datos]
```

**Beneficios:**
- Datos pre-poblados
- Configuración consistente
- Tiempo de inicialización mínimo

---

##### **3. EBS Optimization**

**Snapshot-based Volumes:**
```
💾 EBS Snapshot-based Volumes
┌─────────────────────────────────────────────────────────┐
│                   SOURCE                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │            EBS Volume                           │   │
│  │         ┌─────────────┐                        │   │
│  │         │ Application │                        │   │
│  │         │ Data +      │                        │   │
│  │         │ Content     │                        │   │
│  │         │    💽       │                        │   │
│  │         └─────────────┘                        │   │
│  │               │                                │   │
│  │               ▼                                │   │
│  │         ┌─────────────┐                        │   │
│  │         │ EBS Snap    │                        │   │
│  │         │   Backup    │                        │   │
│  │         │     📸      │                        │   │
│  │         └─────────────┘                        │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   RESTORATION                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │Create Volume│  │  Attach to  │  │ Data Ready  │     │
│  │from Snapshot│  │    EC2      │  │ Instantly   │     │
│  │     🔄      │  │     🔗      │  │     ⚡      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘

⚡ Ventajas: 
• Datos pre-configurados listos
• Sin inicialización de aplicación
• Boot time mínimo
```

**Casos de uso:**
- Datos estáticos pre-cargados
- Configuraciones de aplicación
- Logs históricos

---

#### 🛠️ Implementación con Elastic Beanstalk

```
🚀 Elastic Beanstalk Deployment
┌─────────────────────────────────────────────────────────┐
│                DEVELOPMENT                              │
│  ┌─────────────┐                                        │
│  │Application  │                                        │
│  │   Code      │                                        │
│  │    📦       │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │ Deploy
┌─────────────────────────────────────────────────────────┐
│              ELASTIC BEANSTALK                          │
│  ┌─────────────┐                                        │
│  │Deploy to    │                                        │
│  │Beanstalk    │                                        │
│  │Platform     │                                        │
│  │     🌱      │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                        │ Auto-Creates
┌─────────────────────────────────────────────────────────┐
│              COMPLETE ENVIRONMENT                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │     ALB     │  │Auto Scaling │  │    EC2      │     │
│  │Load Balance │  │   Groups    │  │ Instances   │     │
│  │     ⚖️      │  │     📈      │  │     🖥️      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐                     │
│  │ Security    │  │ Monitoring  │                     │
│  │  Groups     │  │CloudWatch   │                     │
│  │     🔒      │  │     📊      │                     │
│  └─────────────┘  └─────────────┘                     │
└─────────────────────────────────────────────────────────┘

✅ Ventajas:
• Infraestructura automatizada
• Escalado automático
• Monitoreo integrado
• Deploy simplificado
```

---

## Arquitectura de 3 Niveles

### 🏗️ Patrón Arquitectónico EstándAR

#### Estructura Típica

```
🏗️ Arquitectura 3-Tier Production Ready
┌─────────────────────────────────────────────────────────┐
│                PRESENTATION TIER                        │
│                 (Public Subnet)                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Route 53   │  │ CloudFront  │  │    ALB      │    │
│  │ Global DNS  │  │ Global CDN  │  │ Multi-AZ LB │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────┐
│               APPLICATION TIER                          │
│               (Private Subnet)                         │
│                                                        │
│        ┌─────────────────────────────────────────┐     │
│        │         Auto Scaling Group              │     │
│        │           2-20 instances                │     │
│        └─────────────────────────────────────────┘     │
│                        │                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   │ App Server  │  │ App Server  │  │ App Server  │   │
│   │   AZ-1a     │  │   AZ-1b     │  │   AZ-1c     │   │
│   └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────┐
│                  DATA TIER                              │
│               (Database Subnet)                        │
│                                                        │
│   ┌─────────────────────────────────────────────────┐  │
│   │              Cache Layer                        │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐        │  │
│   │   │ElastiC. │  │ElastiC. │  │ElastiC. │        │  │
│   │   │Primary  │  │Replica  │  │Replica  │        │  │
│   │   │ AZ-1a   │  │ AZ-1b   │  │ AZ-1c   │        │  │
│   │   └─────────┘  └─────────┘  └─────────┘        │  │
│   └─────────────────────────────────────────────────┘  │
│                                                        │
│   ┌─────────────────────────────────────────────────┐  │
│   │            Database Layer                       │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐        │  │
│   │   │   RDS   │  │  Read   │  │  Read   │        │  │
│   │   │ Master  │  │Replica  │  │Replica  │        │  │
│   │   │ AZ-1a   │  │ AZ-1b   │  │ AZ-1c   │        │  │
│   │   └─────────┘  └─────────┘  └─────────┘        │  │
│   └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

🔄 Traffic Flow:
1. DNS Query → Route 53
2. Route to CDN → CloudFront  
3. Origin Request → ALB
4. Load Balance → App Servers
5. Cache Lookup → ElastiCache
6. Data Operations → RDS
```

---

### 🔧 Elastic Beanstalk: Simplificando el Despliegue

#### Problemática para Desarrolladores
- **Gestión de infraestructura compleja**
- **Configuración de múltiples servicios**
- **Despliegue y versionado de código**
- **Monitoreo y escalado**
- **Repetición del mismo patrón ALB + ASG**

#### Solución: AWS Elastic Beanstalk

**Filosofía:**
> "Los desarrolladores solo quieren que su código funcione"

---

#### 🎯 Características Principales

| Característica | Descripción | Beneficio |
|----------------|-------------|-----------|
| **Gestión Automática** | Provisioning, load balancing, auto-scaling | Menos overhead operacional |
| **Control Granular** | Acceso a configuración subyacente | Flexibilidad cuando se necesita |
| **Costo Zero** | Solo pagas por los recursos | Sin costos adicionales |
| **Multi-plataforma** | Go, Java, .NET, Node.js, Python, PHP, Docker | Amplio soporte |

---

#### 🏗️ Componentes de Beanstalk

##### **1. Aplicación**
```
MyWebApp
├─ Version 1.0 (WAR file)
├─ Version 1.1 (ZIP file)
└─ Version 2.0 (Docker image)
```

##### **2. Versión de Aplicación**
- Etiqueta específica del código deployable
- Almacenado en S3
- Puede ser rollback point

##### **3. Entorno**
```
Production Environment
├─ ALB
├─ Auto Scaling Group
├─ EC2 Instances
├─ Security Groups
└─ CloudWatch Alarms

Development Environment
├─ Single EC2
└─ Basic monitoring
```

---

#### 🔄 Flujo de Trabajo

```
[Desarrollador] → [Upload Code] → [Create Application] → [Deploy Version] → [Launch Environment]
                                                                                      │
                                                                              [Monitor & Manage]
                                                                                      │
                                                                              [Deploy New Version]
```

**Proceso Detallado:**
1. **Crear Aplicación**: Contenedor lógico
2. **Cargar Versión**: Subir código a S3
3. **Lanzar Entorno**: Provisionar infraestructura
4. **Gestionar**: Monitorear, escalar, actualizar

---

#### 🎨 Tipos de Entorno

##### **Web Server Environment**
```
[Internet] → [ALB] → [EC2 Instances] → [Database]
```
**Casos de uso:**
- Aplicaciones web tradicionales
- APIs REST
- Sitios web con UI

##### **Worker Environment**
```
[SQS Queue] → [EC2 Instances] → [Process Jobs] → [Database]
```
**Casos de uso:**
- Procesamiento en background
- Tareas asíncronas
- Procesamiento de colas

---

#### 📋 Plataformas Soportadas

| Plataforma | Versiones | Casos de Uso |
|------------|-----------|--------------|
| **Java** | 8, 11, 17 | Aplicaciones empresariales |
| **Python** | 3.7, 3.8, 3.9 | Web apps, APIs, ML |
| **.NET** | Core, Framework | Aplicaciones Microsoft |
| **Node.js** | 14, 16, 18 | SPAs, APIs, microservicios |
| **PHP** | 7.4, 8.0, 8.1 | WordPress, Laravel |
| **Go** | 1.17, 1.18 | APIs de alto rendimiento |
| **Docker** | Customizable | Cualquier aplicación containerizada |

---

## Pilares del Well-Architected Framework

### 🏛️ Los 5 Pilares Fundamentales

#### 1. 💰 Optimización de Costos

**Principios:**
- **Pagar solo por lo que usas**
- **Medir la eficiencia general**
- **Eliminar gastos innecesarios**

**Estrategias en nuestros casos:**
```
Caso 1: Reserved Instances + Spot Instances
Caso 2: Auto Scaling basado en demanda
Caso 3: EFS con clases de almacenamiento
```

**Herramientas AWS:**
- Cost Explorer
- AWS Budgets
- Trusted Advisor
- Compute Optimizer

---

#### 2. ⚡ Eficiencia del Rendimiento

**Principios:**
- **Democratizar tecnologías avanzadas**
- **Globalizar en minutos**
- **Usar arquitecturas serverless**
- **Experimentar más frecuentemente**

**Implementaciones:**
```
- CloudFront para CDN global
- ElastiCache para baja latencia
- Auto Scaling para demanda variable
- Read Replicas para distribución de lecturas
```

---

#### 3. 🛡️ Fiabilidad

**Principios:**
- **Recuperación automática de fallos**
- **Procedimientos de recuperación escalables**
- **Escalado horizontal para disponibilidad**

**Estrategias Multi-AZ:**
```
RDS Multi-AZ:     Master/Standby failover
Aurora:           Distributed storage + replicas
ELB:              Health checks + traffic distribution
Auto Scaling:     Replace failed instances
```

---

#### 4. 🔒 Seguridad

**Principios:**
- **Implementar base sólida de identidad**
- **Aplicar seguridad en todas las capas**
- **Automatizar mejores prácticas**
- **Proteger datos en tránsito y reposo**

**Implementación por capas:**
```
Network Layer:     VPC, Subnets, NACLs
Instance Layer:    Security Groups
Application Layer: IAM Roles, policies
Data Layer:        Encryption at rest/transit
```

---

#### 5. 🔧 Excelencia Operacional

**Principios:**
- **Realizar operaciones como código**
- **Hacer cambios pequeños y reversibles**
- **Refinar procedimientos frecuentemente**
- **Anticipar fallos**

**Herramientas:**
```
Infrastructure as Code:  CloudFormation, CDK
Monitoring:             CloudWatch, X-Ray
Automation:             Systems Manager, Lambda
CI/CD:                  CodePipeline, CodeDeploy
```

---

### 📊 Matriz de Evaluación

| Pilar | Caso 1 | Caso 2 | Caso 3 |
|-------|--------|--------|--------|
| **Costos** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Rendimiento** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Fiabilidad** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Seguridad** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Operacional** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 🎓 Resumen para Certificación

### Conceptos Clave a Recordar

#### **Patrones de Escalabilidad**
1. **Vertical**: Aumentar recursos de instancia individual
2. **Horizontal**: Agregar más instancias
3. **Auto Scaling**: Escalado automático basado en métricas

#### **Patrones de Alta Disponibilidad**
1. **Multi-AZ**: Distribución en múltiples zonas
2. **Load Balancing**: Distribución de tráfico
3. **Health Checks**: Detección automática de fallos

#### **Patrones de Gestión de Estado**
1. **Stateless**: Aplicaciones sin estado local
2. **External Session Store**: ElastiCache/DynamoDB
3. **Shared Storage**: EFS para archivos compartidos

#### **Patrones de Base de Datos**
1. **Read Replicas**: Escalado de lecturas
2. **Multi-AZ**: Failover automático
3. **Caching**: ElastiCache para rendimiento

### 🔍 Preguntas Típicas de Examen

1. **¿Cuándo usar EFS vs EBS vs S3?**
2. **¿Diferencia entre Multi-AZ y Read Replicas?**
3. **¿Cómo manejar sesiones en aplicaciones escalables?**
4. **¿Cuándo usar ALB vs NLB vs CLB?**
5. **¿Estrategias de optimización de costos en Auto Scaling?**

---

**📚 Preparado para:** AWS Solutions Architect Associate/Professional  
**📅 Actualizado:** Agosto 2025  
**✅ Estado:** Listo para certificación
