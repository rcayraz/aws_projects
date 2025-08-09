# 🗄️ Almacenamiento en EC2: Guía Completa para Certificación AWS
*EBS, Snapshots, AMI, Instance Store, EFS*

---

## 📊 Arquitectura General de Almacenamiento EC2

```
┌─────────────────────────────────────────────────────────────────┐
│                        REGIÓN AWS                               │
│  ┌───────────────────┐           ┌─────────────────────────┐    │
│  │  Availability      │           │  Availability Zone B    │    │
│  │  Zone A           │           │                         │    │
│  │  ┌─────────────┐  │           │  ┌─────────────────┐    │    │
│  │  │    EC2      │  │◄──────────┼──┤      EFS        │    │    │
│  │  │ Instance 1  │  │           │  │  (Multi-AZ)     │    │    │
│  │  └─────────────┘  │           │  └─────────────────┘    │    │
│  │        │          │           │                         │    │
│  │  ┌─────▼─────┐    │           │  ┌─────────────────┐    │    │
│  │  │EBS Volume │    │           │  │    EC2          │    │    │
│  │  │  (gp3)    │    │           │  │  Instance 2     │    │    │
│  │  └───────────┘    │           │  └─────────────────┘    │    │
│  │                   │           │                         │    │
│  │  ┌─────────────┐  │           └─────────────────────────┘    │
│  │  │Instance     │  │                                          │
│  │  │Store        │  │           ┌─────────────────────────┐    │
│  │  │(Temporal)   │  │           │        Amazon S3        │    │
│  │  └─────────────┘  │           │    ┌─────────────┐      │    │
│  └───────────────────┘           │    │  Snapshots  │      │    │
│                                  │    │             │      │    │
│                                  │    └─────────────┘      │    │
│                                  │    ┌─────────────┐      │    │
│                                  │    │    AMIs     │      │    │
│                                  │    └─────────────┘      │    │
│                                  └─────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ EBS — Elastic Block Store

```
┌─────────────────────────────────────────────────────────────┐
│                    AVAILABILITY ZONE A                     │
│                                                             │
│  ┌─────────────┐     ┌─────────────────────────────────┐   │
│  │    EC2      │────▶│         EBS Volume              │   │
│  │  Instance   │     │  ┌─────────────────────────┐    │   │
│  │             │     │  │     Encrypted Data      │    │   │
│  │  - Running  │     │  │                         │    │   │
│  │  - Stopped  │     │  │  📊 IOPS: 3,000-16,000  │    │   │
│  │  - Restart  │     │  │  💾 Size: 1GB - 64TB    │    │   │
│  └─────────────┘     │  └─────────────────────────┘    │   │
│                      └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
      ❌ NO puede moverse a otra AZ sin Snapshot
```

### 🔑 Características Clave
- **Persistente**: Los datos sobreviven a paradas/reinicios de instancia
- **Zona específica**: Limitado a una Availability Zone
- **Red-attached**: Conectado via red (pequeña latencia)
- **Cifrado**: Encriptación en reposo y tránsito vía KMS
- **Facturable**: Pago por capacidad provisionada (no por uso)

### 💡 Casos de uso críticos para el examen
- **Bases de datos**: PostgreSQL, MySQL, MongoDB
- **File systems**: `/` (root), `/var`, `/opt`
- **Logs de aplicación**: Almacenamiento persistente
- **Boot volumes**: Volumen raíz del SO

---

## 2️⃣ Snapshots - Backup Incremental

```
EBS Volume (AZ-A)         Amazon S3 (Global)
┌─────────────────┐      ┌─────────────────────────────────┐
│ ┌─────────────┐ │      │  ┌─────────────────────────┐    │
│ │    Data     │ │ ───▶ │  │     Snapshot 1          │    │
│ │   Block 1   │ │      │  │   (Full Backup)         │    │
│ │   Block 2   │ │      │  └─────────────────────────┘    │
│ │   Block 3   │ │      │                                 │
│ └─────────────┘ │      │  ┌─────────────────────────┐    │
└─────────────────┘      │  │     Snapshot 2          │    │
                         │  │ (Only Changed Blocks)   │    │
        📅 Day 2         │  └─────────────────────────┘    │
┌─────────────────┐      │                                 │
│ ┌─────────────┐ │      │  ┌─────────────────────────┐    │
│ │    Data     │ │ ───▶ │  │     Snapshot 3          │    │
│ │   Block 1   │ │      │  │  (Incremental)          │    │
│ │   Block 2*  │ │      │  └─────────────────────────┘    │
│ │   Block 3   │ │      └─────────────────────────────────┘
│ │   Block 4   │ │              Cross-Region Copy
│ └─────────────┘ │         ┌─────────────────────────┐
└─────────────────┘         │      Different Region   │
                            └─────────────────────────┘
```

### 🚀 Operaciones con Snapshots

```
FLUJO DE TRABAJO TÍPICO:
1. EBS Volume (Producción) ──▶ Snapshot ──▶ S3
2. Snapshot ──▶ Nuevo EBS Volume (Cualquier AZ)
3. Snapshot ──▶ Copy to Region ──▶ DR/Backup
4. Snapshot ──▶ AMI Creation ──▶ Launch Template
```

### ⭐ Puntos clave para el examen
- **Almacenamiento**: Amazon S3 (internamente, no visible)
- **Tipo**: Incremental después del primero
- **Costo**: ~75% más barato que volumen original
- **Disponibilidad**: Sin downtime durante creación
- **Cross-region**: Posible copiar entre regiones

---

## 3️⃣ AMI — Amazon Machine Image

```
┌─────────────────────────────────────────────────────────────────┐
│                        AMI COMPONENTS                           │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Root Volume    │  │  Additional     │  │   Metadata      │ │
│  │    (EBS)        │  │  EBS Volumes    │  │                 │ │
│  │                 │  │                 │  │ • Architecture  │ │
│  │ • OS Files      │  │ • Application   │  │ • Kernel        │ │
│  │ • Software      │  │ • Data          │  │ • RAM Disk      │ │
│  │ • Configuration │  │ • Logs          │  │ • Permissions   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
            ┌─────────────────────────────────────┐
            │          LAUNCH INSTANCE            │
            │                                     │
            │  EC2 Instance 1  EC2 Instance 2     │
            │  ┌─────────────┐  ┌─────────────┐   │
            │  │   Same OS   │  │   Same OS   │   │
            │  │   Same App  │  │   Same App  │   │
            │  │Same Config  │  │Same Config  │   │
            │  └─────────────┘  └─────────────┘   │
            └─────────────────────────────────────┘
```

### 🏗️ Tipos de AMI y Casos de Uso

| Tipo | Fuente | Casos de Uso | Ejemplo |
|------|--------|--------------|---------|
| **AWS Public** | Amazon | Sistemas base | Amazon Linux 2, Ubuntu |
| **AWS Marketplace** | Partners | Software comercial | WordPress, SAP |
| **Community** | Usuarios | Configuraciones compartidas | LAMP Stack |
| **Private** | Mi cuenta | Aplicaciones propias | App corporativa |

### 🎯 Escenarios típicos del examen
1. **Auto Scaling**: AMI base → Launch Template → ASG
2. **Disaster Recovery**: AMI en múltiples regiones
3. **Entornos homogéneos**: Dev/Test/Prod con misma AMI
4. **Compliance**: AMIs con configuraciones de seguridad

### ⚡ AMI Lifecycle
```
Source Instance → Create Image → AMI → Launch Instances
      │                          │
      └── Snapshot EBS ──────────┘
```

---

## 4️⃣ Instance Store - Almacenamiento Efímero

```
┌─────────────────────────────────────────────────────────────┐
│                    PHYSICAL HOST                            │
│                                                             │
│  ┌─────────────────┐     ┌─────────────────────────────┐   │
│  │   EC2 Instance  │     │     Instance Store          │   │
│  │                 │     │                             │   │
│  │   App Running   │────▶│  ⚡ NVMe SSD                │   │
│  │                 │     │  🚀 Ultra High Performance  │   │
│  │                 │     │  💸 No Additional Cost      │   │
│  └─────────────────┘     │                             │   │
│           │               │  ⚠️  EPHEMERAL DATA         │   │
│           │               └─────────────────────────────┘   │
│           ▼                                                 │
│    ┌─────────────┐                                         │
│    │   STOP      │  ────▶  💥 DATA LOST                    │
│    │ TERMINATE   │                                         │
│    │  HARDWARE   │                                         │
│    │   FAILURE   │                                         │
│    └─────────────┘                                         │
└─────────────────────────────────────────────────────────────┘

        PERO: RESTART = DATA PERSISTS ✅
```

### ⚠️ Estados y Persistencia de Datos

| Acción en EC2 | Instance Store | EBS Volume |
|---------------|----------------|------------|
| **Restart** | ✅ Persiste | ✅ Persiste |
| **Stop** | ❌ Se pierde | ✅ Persiste |
| **Terminate** | ❌ Se pierde | ❌ Se pierde (default) |
| **Hardware Failure** | ❌ Se pierde | ✅ Persiste |

### 🎯 Casos de uso perfectos para examen
- **Cache**: Redis, Memcached
- **Buffers**: Kafka, streaming data
- **Swap space**: Memoria virtual
- **Temporary processing**: ETL, batch jobs
- **High-performance databases**: Donde se acepta pérdida

---

## 5️⃣ Tipos de Volumen EBS - Comparativa Detallada

```
┌─────────────────────────────────────────────────────────────────┐
│                    EBS VOLUME TYPES                            │
│                                                                 │
│  GENERAL PURPOSE SSD              HIGH PERFORMANCE SSD          │
│  ┌─────────────────┐              ┌─────────────────────────┐   │
│  │      gp3        │              │         io2             │   │
│  │  💰 $0.08/GB    │              │     💰 $0.125/GB        │   │
│  │  ⚡ 3K-16K IOPS │              │    ⚡ Up to 64K IOPS    │   │
│  │  📈 125-1000MB/s│              │   📈 Up to 1000 MB/s    │   │
│  └─────────────────┘              └─────────────────────────┘   │
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────────────┐   │
│  │      gp2        │              │         io1             │   │
│  │  💰 $0.10/GB    │              │     💰 $0.125/GB        │   │
│  │  ⚡ 3 IOPS/GB   │              │    ⚡ Up to 64K IOPS    │   │
│  │  📈 Up to 250MB │              │   📈 Up to 1000 MB/s    │   │
│  └─────────────────┘              └─────────────────────────┘   │
│                                                                 │
│  THROUGHPUT OPTIMIZED HDD         COLD HDD                     │
│  ┌─────────────────┐              ┌─────────────────────────┐   │
│  │      st1        │              │          sc1            │   │
│  │  💰 $0.045/GB   │              │     💰 $0.015/GB        │   │
│  │  ⚡ No IOPS     │              │    ⚡ No IOPS           │   │
│  │  📈 Up to 500MB │              │   📈 Up to 250 MB/s     │   │
│  │  🎯 Big Data    │              │   🎯 Cold Storage       │   │
│  └─────────────────┘              └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 📊 Matriz de Decisión por Caso de Uso

| Caso de Uso | Volumen Recomendado | Justificación |
|-------------|-------------------|---------------|
| **Boot Volume** | gp3 | Balance costo/performance |
| **Database OLTP** | io2 | Altas IOPS consistentes |
| **Database Analytics** | st1 | Alto throughput secuencial |
| **File Server** | gp3 | Uso general balanceado |
| **Log Archive** | sc1 | Acceso infrecuente, bajo costo |
| **Media Workstation** | io2 | Baja latencia, high IOPS |
| **Big Data** | st1 | Throughput optimizado |
| **Backup/Archive** | sc1 | Más económico para cold data |

### ⚡ Performance Characteristics

```
IOPS Performance:
io2: ████████████████████████████████████████ (64,000)
io1: ████████████████████████████████████████ (64,000)
gp3: ████████████████ (16,000)
gp2: ████████████████ (16,000)
st1: ████ (500 for 1TB+)
sc1: ██ (250 for 1TB+)

Throughput (MB/s):
io2: ████████████████████████████████████████ (1,000)
io1: ████████████████████████████████████████ (1,000)  
gp3: ████████████████████████████████████████ (1,000)
st1: ████████████████████████ (500)
gp2: ██████████ (250)
sc1: ██████████ (250)
```

---

## 6️⃣ Multi-Attach - Volúmenes Compartidos

```
┌─────────────────────────────────────────────────────────────────┐
│                    AVAILABILITY ZONE                           │
│                                                                 │
│  ┌─────────────┐                                               │
│  │EC2 Instance │ ──┐                                           │
│  │     #1      │   │    ┌─────────────────────────────────┐   │
│  │             │   ├───▶│      EBS io1/io2 Volume        │   │
│  └─────────────┘   │    │                                 │   │
│                    │    │  📝 Concurrent Read/Write       │   │
│  ┌─────────────┐   │    │  🔗 Up to 16 instances         │   │
│  │EC2 Instance │ ──┤    │  ⚠️  Cluster-aware filesystem  │   │
│  │     #2      │   │    │      required                  │   │
│  │             │   │    └─────────────────────────────────┘   │
│  └─────────────┘   │                                          │
│                    │                                          │
│  ┌─────────────┐   │                                          │
│  │EC2 Instance │ ──┘                                          │
│  │     #N      │                                              │
│  │(Up to 16)   │                                              │
│  └─────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

### ⚠️ Requisitos Críticos

```
FILESYSTEM COMPATIBLES:
✅ Cluster-aware filesystems:
   • Amazon FSx for Lustre
   • OCFS2 (Oracle)
   • GFS2 (Red Hat)
   
❌ NO compatibles:
   • ext4, ext3
   • NTFS
   • XFS (standard)
```

### 🎯 Casos de uso típicos del examen
- **Alta disponibilidad**: Failover entre instancias
- **Aplicaciones distribuidas**: Databases en cluster
- **Shared storage**: Media rendering farms
- **Analytics**: Spark clusters con storage compartido

### 🔧 Configuración típica
1. Crear volumen io1/io2 con Multi-Attach enabled
2. Attach a múltiples instancias en misma AZ
3. Configurar cluster-aware filesystem
4. Coordinar escrituras entre aplicaciones

---

## 7️⃣ Cifrado EBS - Seguridad de Datos

```
┌─────────────────────────────────────────────────────────────────┐
│                        EBS ENCRYPTION                          │
│                                                                 │
│ ┌─────────────────┐    KMS    ┌─────────────────────────────┐   │
│ │   EC2 Instance  │◄─────────▶│       AWS KMS              │   │
│ │                 │           │                             │   │
│ │ ┌─────────────┐ │           │  🔑 Customer Master Key     │   │
│ │ │ Application │ │           │  🔑 Data Encryption Key     │   │
│ │ └─────────────┘ │           │  🔑 Envelope Encryption     │   │
│ └─────────────────┘           └─────────────────────────────┘   │
│          │                                                     │
│          ▼ Encrypted in transit                                │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │                EBS Volume                               │    │
│ │                                                         │    │
│ │  🔒 Data encrypted at rest                             │    │
│ │  🔒 Snapshots automatically encrypted                  │    │
│ │  🔒 All volumes created from snapshot encrypted        │    │
│ │                                                         │    │
│ │  📊 Performance: No impact                             │    │
│ │  💰 Cost: Minimal additional cost                      │    │
│ └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 🔐 Encryption Workflow

```
1. EC2 Instance requests data
         ▼
2. EBS contacts KMS for DEK (Data Encryption Key)
         ▼  
3. KMS returns encrypted DEK + plaintext DEK
         ▼
4. EBS uses plaintext DEK to decrypt data
         ▼
5. Data sent to EC2 (encrypted in transit)
         ▼
6. EC2 receives plaintext data
```

### ⚡ Migración de Volúmenes No Cifrados

```
Unencrypted Volume → Snapshot → Encrypted Snapshot → New Encrypted Volume
        │                           ▲
        └─── Copy with encryption ───┘
```

### 🎯 Puntos clave para certificación
- **Transparente**: No cambios en aplicación
- **Performance**: Sin impacto significativo  
- **Automático**: Snapshots heredan cifrado
- **Granular**: Por volumen individual
- **Compliance**: FIPS 140-2 Level 2 validated

---

## 8️⃣ EFS — Elastic File System

```
┌─────────────────────────────────────────────────────────────────┐
│                           AWS REGION                           │
│                                                                 │
│ ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐ │
│ │  Availability    │  │  Availability    │  │ Availability    │ │
│ │    Zone A        │  │    Zone B        │  │    Zone C       │ │
│ │                  │  │                  │  │                 │ │
│ │ ┌──────────────┐ │  │ ┌──────────────┐ │  │┌──────────────┐ │ │
│ │ │EC2 Instance  │ │  │ │EC2 Instance  │ │  ││EC2 Instance  │ │ │
│ │ │              │ │  │ │              │ │  ││              │ │ │
│ │ └──────┬───────┘ │  │ └──────┬───────┘ │  │└──────┬───────┘ │ │
│ └────────┼─────────┘  └────────┼─────────┘  └───────┼─────────┘ │
│          │                     │                    │           │
│          └─────────────────────┼────────────────────┘           │
│                                │                                │
│ ┌──────────────────────────────┼──────────────────────────────┐ │
│ │                              ▼                              │ │
│ │               🗂️  EFS FILE SYSTEM                          │ │
│ │                                                            │ │
│ │  📁 /shared                  Performance Modes:            │ │
│ │  ├── application/            • General Purpose (default)   │ │
│ │  ├── logs/                   • Max I/O (higher throughput) │ │
│ │  └── media/                                                │ │
│ │                              Throughput Modes:             │ │
│ │  🔄 Auto-scaling             • Burst (default)             │ │
│ │  💰 Pay-per-use              • Provisioned                 │ │
│ │  🌐 Multi-AZ                                               │ │
│ └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 📊 EFS Storage Classes y Costos

```
STORAGE CLASS LIFECYCLE:

Standard Storage          Infrequent Access (IA)
┌─────────────────┐      ┌─────────────────────┐
│  💰 $0.30/GB    │ ───▶ │   💰 $0.025/GB      │
│  ⚡ Fast Access │ 30d  │   ⚡ Slower Access   │
│  📁 Frequent    │      │   📁 Infrequent     │
└─────────────────┘      └─────────────────────┘
        ▲                          │
        └─── Auto-move back ───────┘
             (if accessed)
```

### 🎯 Comparación con otros servicios

| Característica | EFS | EBS | S3 |
|----------------|-----|-----|-----|
| **Type** | File | Block | Object |
| **Protocol** | NFS v4.1 | Block-level | REST API |
| **Multi-AZ** | ✅ Native | ❌ (Snapshot) | ✅ Native |
| **Concurrent Access** | ✅ Thousands | ❌ Single instance | ✅ Unlimited |
| **Use Case** | Shared files | OS/Database | Web content |

### 💡 Casos de uso típicos del examen
- **Content repositories**: WordPress, Drupal sites
- **Container storage**: EKS persistent volumes  
- **Development environments**: Shared code repos
- **Media workflows**: Video editing, rendering
- **Big data analytics**: Shared datasets
- **Backup target**: Cross-region backup storage

---

## 9️⃣ EBS vs EFS vs Instance Store - Comparativa Completa

```
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE COMPARISON                          │
│                                                                 │
│     EBS                    EFS                Instance Store    │
│ ┌─────────────┐       ┌─────────────┐       ┌─────────────┐    │
│ │   Block     │       │    File     │       │   Block     │    │
│ │  Storage    │       │  Storage    │       │  Storage    │    │
│ │             │       │             │       │             │    │
│ │ 📦 1-to-1   │       │ 📦 1-to-N   │       │ 📦 1-to-1   │    │
│ │ 🏢 Single   │       │ 🌐 Multi    │       │ 🏢 Single   │    │
│ │    AZ       │       │    AZ       │       │    AZ       │    │
│ │ 💾 Network  │       │ 💾 Network  │       │ 💾 Local    │    │
│ │ 💰 Provision│       │ 💰 Pay-per  │       │ 💰 Free     │    │
│ │    based    │       │    use      │       │ (included)  │    │
│ └─────────────┘       └─────────────┘       └─────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 📊 Decision Matrix

| Escenario | EBS | EFS | Instance Store |
|-----------|-----|-----|----------------|
| **Database OLTP** | ✅ io2 | ❌ | ❌ |
| **Shared Content** | ❌ | ✅ | ❌ |
| **Boot Volume** | ✅ gp3 | ❌ | ❌ |
| **High IOPS** | ✅ io2 | ❌ | ✅ |
| **Multi-instance access** | ❌ | ✅ | ❌ |
| **Temporary cache** | ⚠️ Costly | ❌ | ✅ |
| **Cross-AZ availability** | ❌ | ✅ | ❌ |
| **Lowest latency** | ⚠️ | ❌ | ✅ |
| **Cost optimization** | ⚠️ | ✅IA | ✅ |

### 🎯 Patrones de Arquitectura Típicos

```
PATTERN 1: Web Application
┌──────────────┐    ┌──────────────┐
│    ELB       │    │     EFS      │
│              │    │ (Static      │
└──────┬───────┘    │  Content)    │
       │            └──────┬───────┘
   ┌───▼───┐           ┌───▼───┐
   │  EC2  │◄──────────┤  EC2  │
   │ +EBS  │           │ +EBS  │
   └───────┘           └───────┘

PATTERN 2: Database Cluster  
┌─────────────┐    ┌─────────────┐
│   Primary   │    │  Secondary  │
│     DB      │    │     DB      │
│   EBS io2   │    │   EBS io2   │
└─────────────┘    └─────────────┘
       │                   │
       └── Synchronous ────┘
           Replication

PATTERN 3: High Performance Computing
┌─────────────┐    ┌─────────────┐
│  Compute    │    │  Compute    │
│  Instance   │    │  Instance   │
│Store + EBS  │    │Store + EBS  │
└─────────────┘    └─────────────┘
       │                   │  
       └─── Shared EFS ────┘
           (Results)
```

### ⚡ Performance Comparison

| Metric | EBS (io2) | EFS | Instance Store |
|--------|-----------|-----|----------------|
| **IOPS** | 64,000 | 7,000+ | 3M+ |
| **Throughput** | 1,000 MB/s | 3+ GB/s | 30+ GB/s |
| **Latency** | Single-digit ms | Low ms | Sub-ms |
| **Durability** | 99.999% | 99.999999999% | ❌ |

---

## 🎯 Puntos Clave para Examen de Certificación

### 🔥 Preguntas Frecuentes del Examen

#### 1. Migración y Backup
```
❓ "¿Cómo mover un volumen EBS entre AZs?"
✅ EBS Volume → Snapshot → Create Volume in target AZ

❓ "¿Backup automático de EBS?"
✅ Data Lifecycle Manager (DLM) + CloudWatch Events

❓ "¿Restaurar instancia desde AMI en otra región?"
✅ Copy AMI to target region → Launch instance
```

#### 2. Performance y Troubleshooting
```
❓ "¿EBS lento? ¿Qué revisar?"
✅ 1. Volume type (gp2 vs gp3 vs io2)
✅ 2. IOPS vs instance capability  
✅ 3. EBS-optimized instance enabled
✅ 4. CloudWatch metrics

❓ "¿Necesito más IOPS?"
✅ gp3: Provision IOPS independently
✅ io1/io2: Up to 64,000 IOPS
```

#### 3. Security y Compliance
```
❓ "¿Cifrar volumen existente?"
✅ Create snapshot → Copy with encryption → New volume

❓ "¿Compartir snapshot entre cuentas?"
✅ Modify snapshot permissions → Add account IDs

❓ "¿Cifrado cross-region?"
✅ Different KMS keys per region required
```

### 🚨 Trampas Comunes del Examen

| ❌ Trampa | ✅ Realidad |
|-----------|-------------|
| "EBS Multi-Attach en cualquier tipo" | Solo io1/io2 |
| "Instance Store persiste en stop" | Solo en restart |
| "EFS solo para Linux" | También Windows (con NFS client) |
| "Snapshot bloquea el volumen" | Continúa funcionando |
| "gp2 tiene IOPS fijas" | 3 IOPS por GB (burst posible) |

### 📋 Checklist Pre-Examen

- [ ] **EBS Types**: gp3 vs io2 vs st1 vs sc1 decisiones
- [ ] **Snapshots**: Incremental, cross-region, sharing
- [ ] **AMI**: Components, marketplace, lifecycle  
- [ ] **Instance Store**: Cuándo se pierde vs persiste
- [ ] **EFS**: NFS, multi-AZ, storage classes
- [ ] **Multi-Attach**: Requisitos y limitaciones
- [ ] **Encryption**: En reposo, tránsito, KMS integration
- [ ] **Performance**: IOPS vs throughput optimization

---

## 🧠 Mapas Mentales y Técnicas de Memorización

### 🎨 Mapa Mental: Ecosistema de Almacenamiento AWS

```
                             ┌─────────────────────┐
                             │                     │
                             │   AWS STORAGE       │
                             │   ECOSYSTEM         │
                             │                     │
                             └──────────┬──────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
           ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
           │   BLOCK STORAGE │ │  FILE STORAGE   │ │ OBJECT STORAGE  │
           │                 │ │                 │ │                 │
           │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
           │ │     EBS     │ │ │ │     EFS     │ │ │ │     S3      │ │
           │ │             │ │ │ │             │ │ │ │             │ │
           │ │• gp3, io2   │ │ │ │• NFS v4.1   │ │ │ │• Standard   │ │
           │ │• Snapshots  │ │ │ │• Multi-AZ   │ │ │ │• IA, Glacier│ │
           │ │• Multi-     │ │ │ │• Auto-scale │ │ │ │• Versioning │ │
           │ │  Attach     │ │ │ │• Lifecycle  │ │ │ │• Cross-     │ │
           │ │• Encryption │ │ │ │• Mount      │ │ │ │  Region     │ │
           │ └─────────────┘ │ │ │  Targets    │ │ │ │• Static Web │ │
           │                 │ │ └─────────────┘ │ │ └─────────────┘ │
           │ ┌─────────────┐ │ │                 │ │                 │
           │ │ INSTANCE    │ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
           │ │   STORE     │ │ │ │     FSx     │ │ │ │ STORAGE     │ │
           │ │             │ │ │ │             │ │ │ │  GATEWAY    │ │
           │ │• NVMe SSD   │ │ │ │• Windows    │ │ │ │             │ │
           │ │• Ephemeral  │ │ │ │• Lustre     │ │ │ │• Hybrid     │ │
           │ │• High IOPS  │ │ │ │• NetApp     │ │ │ │• Volume     │ │
           │ │• No cost    │ │ │ │• OpenZFS    │ │ │ │• File       │ │
           │ └─────────────┘ │ │ └─────────────┘ │ │ │• Tape       │ │
           └─────────────────┘ └─────────────────┘ │ └─────────────┘ │
                                                   └─────────────────┘

🧠 MEMORY ASSOCIATIONS:
├── EBS = "Elastic Block" → Think LEGO blocks (attachable/detachable)
├── EFS = "Everyone's File System" → Think shared network drive
├── Instance Store = "In the Server" → Think internal hard drive
├── S3 = "Simple Storage Service" → Think cloud file cabinet
└── Multi-Attach = "Multiple Attachments" → Think USB hub
```

### 🎯 Acrónimos y Mnemonics para el Examen

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY DEVICES                              │
│                                                                 │
│  📚 EBS VOLUME TYPES (G.I.S.S):                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  G - General Purpose (gp3, gp2)                        │   │
│  │  I - IOPS Provisioned (io1, io2)                       │   │
│  │  S - Sequential Throughput (st1)                       │   │
│  │  S - Slow/Cold Storage (sc1)                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  🎵 PERSISTENCE SONG: "Stop, Drop and Roll"                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  STOP → Instance Store data DROPS (lost)               │   │
│  │  REBOOT → Everything ROLLS back (persists)             │   │
│  │  TERMINATE → EBS might DROP (depends on settings)      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  🔢 NUMBER PATTERNS:                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Powers of 2: 16K, 64K (IOPS limits)                   │   │
│  │  Quarters: 250, 500, 1000 (MB/s progression)           │   │
│  │  Magic 16: Multi-attach limit                          │   │
│  │  Binary: 1, 3, 16, 64 (gp3 base, gp2 ratio, limits)   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  🎭 CHARACTER ASSOCIATIONS:                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  gp3 = "Good Price" (cost-effective)                   │   │
│  │  io2 = "Intense Operations" (high performance)         │   │
│  │  st1 = "Stream Throughput" (sequential workloads)      │   │
│  │  sc1 = "Slow & Cheap" (infrequent access)             │   │
│  │  EFS = "Easy File Sharing" (multi-instance access)     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  🌍 GEOGRAPHIC THINKING:                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  EBS = City (single availability zone)                 │   │
│  │  EFS = Country (spans multiple availability zones)     │   │
│  │  S3 = Global (cross-region replication)               │   │
│  │  Instance Store = House (attached to specific host)    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 🎲 Gamificación: Storage Decision Game

```
┌─────────────────────────────────────────────────────────────────┐
│                   "CHOOSE YOUR STORAGE" GAME                   │
│                                                                 │
│  🎮 SCENARIO CARDS:                                            │
│                                                                 │
│  📱 CARD 1: "Mobile App Backend"                               │
│  ├── Requirements: Database, shared media files, auto-scaling  │
│  ├── Traffic: Variable, global users                          │
│  ├── Budget: Moderate                                         │
│  └── Answer: RDS (EBS io2) + EFS (shared media) + S3 (CDN)   │
│                                                                 │
│  🏦 CARD 2: "Financial Trading System"                        │
│  ├── Requirements: Ultra-low latency, high IOPS               │
│  ├── Traffic: Constant, millisecond critical                  │
│  ├── Budget: High                                             │
│  └── Answer: Instance Store (primary) + EBS io2 (backup)     │
│                                                                 │
│  📊 CARD 3: "Data Analytics Platform"                         │
│  ├── Requirements: Large sequential reads, cost-effective     │
│  ├── Traffic: Batch processing, overnight jobs                │
│  ├── Budget: Cost-conscious                                   │
│  └── Answer: EBS st1 (data processing) + S3 (data lake)      │
│                                                                 │
│  🏥 CARD 4: "Healthcare Archive System"                       │
│  ├── Requirements: Long-term storage, compliance              │
│  ├── Traffic: Infrequent access, audit requirements           │
│  ├── Budget: Minimal operational cost                         │
│  └── Answer: EBS sc1 (active) + S3 Glacier (archive)         │
│                                                                 │
│  🎯 SCORING SYSTEM:                                            │
│  ├── Correct storage type: +10 points                         │
│  ├── Cost optimization: +5 points                             │
│  ├── Performance match: +5 points                             │
│  ├── Compliance consideration: +3 points                      │
│  └── Disaster recovery plan: +2 points                        │
│                                                                 │
│  🏆 ACHIEVEMENT LEVELS:                                        │
│  ├── 0-15 points: "Storage Novice"                           │
│  ├── 16-30 points: "Storage Architect"                       │
│  ├── 31-45 points: "Storage Expert"                          │
│  └── 46+ points: "Storage Master"                            │
└─────────────────────────────────────────────────────────────────┘
```

### 🔗 Connections Map: Storage Integrations

```
┌─────────────────────────────────────────────────────────────────┐
│                 AWS SERVICES INTEGRATION MAP                   │
│                                                                 │
│         ┌─────────────┐                                         │
│      ┌─▶│     EC2     │◄─┐                                      │
│      │  └─────────────┘  │                                      │
│      │                   │                                      │
│  ┌───▼───┐           ┌───▼───┐                                  │
│  │  EBS  │           │  EFS  │                                  │
│  └───┬───┘           └───┬───┘                                  │
│      │                   │                                      │
│      ▼                   ▼                                      │
│ ┌─────────┐         ┌─────────┐                                 │
│ │ Lambda  │         │   EKS   │                                 │
│ │(EFS mnt)│         │   ECS   │                                 │
│ └─────────┘         └─────────┘                                 │
│      │                   │                                      │
│      ▼                   ▼                                      │
│ ┌──────────────────────────────┐                                │
│ │           S3                 │                                │
│ │ ┌─────────┬─────────────────┐│                                │
│ │ │Snapshots│   Lifecycle     ││                                │
│ │ │ AMIs    │   Policies      ││                                │
│ │ │ Backups │   Glacier       ││                                │
│ │ └─────────┴─────────────────┘│                                │
│ └──────────────────────────────┘                                │
│                                                                 │
│  INTEGRATION PATTERNS:                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 🔄 EBS → Snapshot → S3 → Cross-region copy              │ │
│  │ 🔄 EC2 → EFS mount → Lambda function access             │ │
│  │ 🔄 RDS → EBS storage → Automated S3 backup              │ │
│  │ 🔄 Instance Store → Application cache → EBS persistence │ │
│  │ 🔄 AMI → Launch Template → Auto Scaling Group           │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

*Última actualización: Julio 2025 - Preparación SAA-C03*
** Roberto Ayra**
