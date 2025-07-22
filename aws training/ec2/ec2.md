# Amazon EC2 (Elastic Compute Cloud) - Guía Completa para Arquitectos

## 📋 Índice
1. [Introducción](#introducción)
2. [IP Privadas, Públicas y Elásticas](#ip-privadas-públicas-y-elásticas)
3. [Grupos de Colocación (Placement Groups)](#grupos-de-colocación-placement-groups)
4. [Interfaz de Red Elástica (ENI)](#interfaz-de-red-elástica-eni)
5. [Hibernación de EC2](#hibernación-de-ec2)
6. [Casos de Uso por Industria](#casos-de-uso-por-industria)
7. [Mejores Prácticas para Arquitectos](#mejores-prácticas-para-arquitectos)
8. [Tips para el Examen](#tips-para-el-examen)
9. [Diagramas y Arquitecturas](#diagramas-y-arquitecturas)

---

## 🌟 Introducción

Amazon EC2 es el servicio de computación en la nube más fundamental de AWS, proporcionando servidores virtuales escalables bajo demanda. Para arquitectos de soluciones, es crucial entender no solo cómo funciona, sino cuándo y cómo implementarlo de manera óptima.

### Conceptos Fundamentales
- **Instancias**: Servidores virtuales con recursos computacionales definidos
- **AMI (Amazon Machine Images)**: Plantillas que contienen el SO y aplicaciones
- **Tipos de Instancia**: Familias optimizadas para diferentes cargas de trabajo
- **Almacenamiento**: EBS (persistente) vs Instance Store (temporal)

---

## 🌐 IP Privadas, Públicas y Elásticas

### IP Privadas
```
Características:
✓ Solo dentro de la red (VPC)
✓ Permanecen durante el ciclo de vida de la instancia
✓ Rango RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
✓ No cambian al reiniciar la instancia
```

### IP Públicas
```
Características:
✓ Para identificar desde público
✓ CAMBIAN cuando se vuelve a iniciar la instancia
✓ Se liberan cuando la instancia se detiene
✓ Asignadas automáticamente si está habilitado en la subred
```

### IP Elásticas
```
Características:
✓ Son asignadas y NO cambian
✓ Se puede reemplazar al público
✓ Se paga desde que se tiene asignado
✓ Límite de 5 por región (ampliable)
✓ Persisten hasta que se liberen manualmente
```

### 🏗️ Diagrama de Conectividad IP

```
Internet Gateway
        |
    [IP Pública]
        |
┌─────────────────┐
│   EC2 Instance  │
│                 │
│ IP Privada      │ ←── Comunicación interna VPC
│ 10.0.1.100      │
│                 │
│ IP Pública      │ ←── Cambia al reiniciar
│ 203.0.113.25    │
│                 │
│ IP Elástica     │ ←── Estática (opcional)
│ 198.51.100.10   │
└─────────────────┘
```

### Casos de Uso IP Elásticas
- **Servidores Web**: Mantener la misma IP para DNS
- **Failover**: Reasignar rápidamente entre instancias
- **Licencias**: Software licenciado por IP
- **Whitelisting**: Cuando otros servicios necesitan IP fija

---

## 🔄 Grupos de Colocación (Placement Groups)

Los Placement Groups determinan cómo se distribuyen las instancias EC2 en el hardware físico subyacente.

### Tipos de Placement Groups

#### 1. Cluster Placement Group
```
Características:
✓ Están en un mismo rack
✓ Mejora la latencia pero si falla el rack fallan todos las instancias
✓ Casos de uso: big data, aplicación con baja latencia
```

**Diagrama Cluster:**
```
    Rack A
┌─────────────┐
│ [Instance1] │ ←── Misma AZ
│ [Instance2] │ ←── Baja latencia
│ [Instance3] │ ←── 10 Gbps network
└─────────────┘
```

#### 2. Spread Placement Group
```
Características:
✓ Distribución en diferentes zonas
✓ Reduce riesgos de fallos simultáneos, hw diferentes
✓ Pero limitado 7 instancias por zona
✓ Casos de uso: aplicación con máxima disponibilidad
```

**Diagrama Spread:**
```
AZ-1a        AZ-1b        AZ-1c
┌─────┐     ┌─────┐     ┌─────┐
│Inst1│     │Inst2│     │Inst3│
└─────┘     └─────┘     └─────┘
Hardware    Hardware    Hardware
Diferente   Diferente   Diferente
```

#### 3. Partition Placement Group
```
Características:
✓ Partición: tener particiones en diferentes zonas, zonas de disponibilidad
✓ Cada partición puede tener conjunto de instancias
✓ No comparten racks
✓ Casos de uso: kafka, hbase, cassandra
```

**Diagrama Partition:**
```
      AZ-1a                AZ-1b
┌─────────────────┐  ┌─────────────────┐
│   Partition 1   │  │   Partition 2   │
│  ┌───┐  ┌───┐   │  │  ┌───┐  ┌───┐   │
│  │EC2│  │EC2│   │  │  │EC2│  │EC2│   │
│  └───┘  └───┘   │  │  └───┘  └───┘   │
└─────────────────┘  └─────────────────┘
   Rack A              Rack B
```

### Selección de Placement Group

| Tipo | Latencia | Disponibilidad | Límite Instancias | Uso Ideal |
|------|----------|----------------|-------------------|-----------|
| **Cluster** | Ultra baja | Baja | 10 Gbps | HPC, ML training |
| **Spread** | Normal | Muy alta | 7 por AZ | Aplicaciones críticas |
| **Partition** | Baja | Alta | 100s por partición | Big Data distribuido |

---

## 🔌 Interfaz de Red Elástica (ENI)

### ¿Qué es ENI?
```
Es una tarjeta de red virtual, puedes crear
enis y adjuntarlas en instancias en marcha
esta enlazada en un az
```

### Características de ENI
- **IP Privada primaria**: Una por ENI
- **IPs Privadas secundarias**: Múltiples opcionales
- **IP Pública**: Una por IP privada
- **Grupos de Seguridad**: Uno o más por ENI
- **Dirección MAC**: Única y persistente

### 🏗️ Diagrama ENI Multi-attach

```
       ┌─────────────┐
       │ Instance A  │
       │   Primary   │ ──┐
       │     ENI     │   │
       └─────────────┘   │    ┌─────────────┐
                         ├────│ Shared ENI  │
       ┌─────────────┐   │    │   (eth1)    │
       │ Instance B  │   │    └─────────────┘
       │  Secondary  │ ──┘           │
       │     ENI     │               │
       └─────────────┘               │
                                     │
                              ┌─────────────┐
                              │   Subnet    │
                              │ 10.0.1.0/24 │
                              └─────────────┘
```

### Casos de Uso ENI
1. **Failover de Red**: Mover ENI entre instancias
2. **Licencias por MAC**: Software licenciado por dirección MAC
3. **Dual-homing**: Instancia en múltiples subredes
4. **Management Networks**: Separar tráfico de administración

### Ejemplo de Configuración ENI
```bash
# Crear ENI
aws ec2 create-network-interface \
    --subnet-id subnet-12345678 \
    --description "Management Interface" \
    --groups sg-12345678

# Adjuntar a instancia
aws ec2 attach-network-interface \
    --network-interface-id eni-12345678 \
    --instance-id i-12345678 \
    --device-index 1
```

---

## 💤 Hibernación de EC2

### ¿Qué es la Hibernación?
```
HIBERNACIÓN: conservar el estado en memoria,
el arranque de la instancia es mucho más rápido

Bajo el capo, el estado de la ram se
escribe en un archivo en el volumen EBS
raíz
```

### Proceso de Hibernación

```
┌─────────────┐    Hibernar    ┌─────────────┐
│   Running   │ ──────────────→│  Hibernated │
│             │                │             │
│ RAM: 8GB    │                │ EBS: +8GB   │
│ EBS: 20GB   │                │ File        │
└─────────────┘                └─────────────┘
       ↑                              │
       │           Despertar          │
       └──────────────────────────────┘
         (Carga RAM desde EBS)
```

### Requisitos para Hibernación
- **Tipos soportados**: M3, M4, M5, C3, C4, C5, R3, R4, R5
- **SO soportados**: Amazon Linux 2, Ubuntu, Windows
- **Tamaño RAM**: Máximo 150 GB
- **Volumen raíz**: Debe ser EBS cifrado
- **Tiempo máximo**: 60 días hibernado

### Casos de Uso Hibernación
```
CASOS DE USO: servicio que demoran en
iniciarse, procesamiento de larga duración,
guardar el estado de la ram
```

### Ejemplo Práctico
```bash
# Instancia con hibernación habilitada
aws ec2 run-instances \
    --image-id ami-12345678 \
    --instance-type m5.large \
    --hibernation-options Configured=true \
    --block-device-mappings '[{
        "DeviceName": "/dev/xvda",
        "Ebs": {
            "VolumeSize": 32,
            "Encrypted": true
        }
    }]'

# Hibernar instancia
aws ec2 stop-instances \
    --instance-ids i-12345678 \
    --hibernate
```

---

## 🏢 Casos de Uso por Industria

### Familias de Instancias
```
Familias de instancias: soportadas, tamaño de
ram, no soporta instancia de bare metal
```

### Por Tipo de Carga de Trabajo

#### 1. **Aplicaciones Web (General Purpose)**
```
Instancias: T3, T4g, M5, M6i
Casos: WordPress, e-commerce, microservicios
Arquitectura: Auto Scaling + Load Balancer
```

#### 2. **Análisis Big Data (Compute Optimized)**
```
Instancias: C5, C6g, C6gn
Casos: Apache Spark, procesamiento ETL
Placement: Cluster para máximo rendimiento
```

#### 3. **Bases de Datos en Memoria (Memory Optimized)**
```
Instancias: R5, R6g, X1e, z1d
Casos: Redis, SAP HANA, Apache Kafka
Características: High memory-to-vCPU ratio
```

#### 4. **Machine Learning (Accelerated Computing)**
```
Instancias: P3, P4, G4, Inf1
Casos: Training ML, inferencia, HPC
GPU: NVIDIA Tesla, AWS Inferentia
```

### 🏗️ Arquitectura Multi-Tier

```
Internet Gateway
        │
    ┌───────┐
    │  ALB  │ ← Application Load Balancer
    └───┬───┘
        │
┌───────┼───────┐
│   Web Tier    │ ← t3.medium (Auto Scaling)
│ ┌───┐ ┌───┐   │
│ │EC2│ │EC2│   │
│ └───┘ └───┘   │
└───────┼───────┘
        │
┌───────┼───────┐
│   App Tier    │ ← c5.large (Compute)
│ ┌───┐ ┌───┐   │
│ │EC2│ │EC2│   │
│ └───┘ └───┘   │
└───────┼───────┘
        │
┌───────┼───────┐
│   DB Tier     │ ← r5.xlarge (Memory)
│     ┌───┐     │
│     │RDS│     │
│     └───┘     │
└───────────────┘
```

---

## ⚡ Mejores Prácticas para Arquitectos

### 1. **Selección de Instancias**
```yaml
Principios:
  - Rightsizing: Monitorear y ajustar tamaños
  - Reserved Instances: Para cargas predecibles
  - Spot Instances: Para cargas fault-tolerant
  - Mixed Instance Types: En Auto Scaling Groups
```

### 2. **Networking y Seguridad**
```yaml
Configuración:
  Security Groups:
    - Principio de menor privilegio
    - Separar por función (web, app, db)
  
  ENI Strategy:
    - Management interfaces separadas
    - Failover automático con Lambda
  
  IP Management:
    - Elastic IPs solo cuando necesario
    - Documentar asignaciones
```

### 3. **Alta Disponibilidad**
```yaml
Estrategias:
  Multi-AZ:
    - Spread Placement Groups
    - Auto Scaling cross-AZ
  
  Backup y Recovery:
    - EBS Snapshots automatizados
    - AMIs con configuración completa
  
  Monitoring:
    - CloudWatch métricas detalladas
    - Alertas proactivas
```

### 4. **Optimización de Costos**
```yaml
Tácticas:
  Instance Optimization:
    - AWS Compute Optimizer
    - Scheduled scaling
  
  Storage Optimization:
    - GP3 vs GP2 evaluation
    - EBS optimization enabled
  
  Network Optimization:
    - Enhanced networking
    - Placement groups apropiados
```

---

## 🎯 Tips para el Examen

### Preguntas Frecuentes SAA-C03

#### 1. **Scenario: IP Management**
```
Pregunta: "Una aplicación necesita una IP que no cambie
al reiniciar la instancia"

Respuesta: Elastic IP (IP Elástica)
❌ IP Privada (solo interna)
❌ IP Pública (cambia al reiniciar)
✅ IP Elástica (persistente)
```

#### 2. **Scenario: High Performance Computing**
```
Pregunta: "Aplicación HPC requiere máximo rendimiento
de red entre instancias"

Respuesta: Cluster Placement Group
✅ Cluster (misma rack, 10 Gbps)
❌ Spread (máxima disponibilidad)
❌ Partition (distribución)
```

#### 3. **Scenario: Startup Time Optimization**
```
Pregunta: "Reducir tiempo de arranque manteniendo
estado de aplicación"

Respuesta: Hibernación EC2
✅ Hibernation (conserva RAM)
❌ Stop/Start (pierde estado)
❌ AMI (no conserva datos runtime)
```

#### 4. **Scenario: Network Failover**
```
Pregunta: "Failover automático de IP entre instancias
en diferentes AZ"

Respuesta: ENI + Lambda
✅ Detach/Attach ENI programáticamente
❌ Elastic IP (manual reassignment)
❌ Load Balancer (no IP específica)
```

### Palabras Clave del Examen

| Escenario | Pista en Pregunta | Solución |
|-----------|-------------------|----------|
| **IP Fija** | "misma IP", "no cambie" | Elastic IP |
| **Máximo rendimiento** | "baja latencia", "HPC" | Cluster Placement |
| **Alta disponibilidad** | "fault tolerance", "AZ failure" | Spread Placement |
| **Arranque rápido** | "startup time", "estado" | Hibernation |
| **Failover red** | "move IP", "network failover" | ENI |

### Errores Comunes a Evitar

```
❌ Confundir IP Pública con Elástica
❌ Usar Cluster PG para alta disponibilidad
❌ Hibernar instancias más de 60 días
❌ ENI cross-AZ attachment
❌ Placement Groups cross-región
```

---

## 📊 Diagramas y Arquitecturas

### Arquitectura de Referencia: E-commerce

```
                    ┌─────────────┐
                    │    Route53  │ ← DNS + Health Checks
                    └──────┬──────┘
                           │
                    ┌─────────────┐
                    │ CloudFront  │ ← CDN Global
                    └──────┬──────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                   ALB/NLB                   │
    │              Elastic IPs (if needed)        │
    └──────────────────────┼──────────────────────┘
                           │
    ┌─────────────────────────────────────────────┐
    │              Auto Scaling Group             │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
    │  │   AZ-a  │  │   AZ-b  │  │   AZ-c  │     │
    │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │     │
    │  │ │ EC2 │ │  │ │ EC2 │ │  │ │ EC2 │ │     │ ← Spread PG
    │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │     │
    │  └─────────┘  └─────────┘  └─────────┘     │
    └─────────────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ElastiCache│     │   RDS   │      │   S3    │
    │ (Redis) │      │Multi-AZ │      │(Static) │
    └─────────┘      └─────────┘      └─────────┘
```

### Monitoring y Troubleshooting

```yaml
CloudWatch Métricas Clave:
  Compute:
    - CPUUtilization
    - NetworkIn/Out
    - DiskReadOps/WriteOps
  
  Custom Métricas:
    - Application response time
    - Business metrics
    - Error rates

Logs Importantes:
  - /var/log/messages (sistema)
  - Application logs
  - CloudTrail (API calls)
  - VPC Flow Logs (network)
```

---

## 🚀 Conclusión

Amazon EC2 es fundamental en cualquier arquitectura AWS. Como arquitecto, debes dominar:

1. **Networking**: IPs, ENIs, y conectividad
2. **Placement**: Optimización de rendimiento y disponibilidad  
3. **Lifecycle**: Hibernation y gestión de estado
4. **Cost Optimization**: Right-sizing y instancias apropiadas
5. **Security**: Grupos de seguridad y acceso

### Próximos Pasos
- Revisar EBS y almacenamiento
- Estudiar Auto Scaling Groups
- Practicar con Load Balancers
- Implementar monitoring completo

---

## 📚 Referencias Adicionales

- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [Placement Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)
- [Elastic Network Interfaces](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)

---

*Última actualización: Julio 2025 - Preparación SAA-C03*
