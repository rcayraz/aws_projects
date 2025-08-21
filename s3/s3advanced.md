# Amazon S3 Advanced Features

**Autor:** Roberto Ayra  
**Fecha:** Abril 2024

---

## 📊 MOVIMIENTO ENTRE CLASES DE ALMACENAMIENTO

Dependiendo de los requerimientos de acceso y costo, podemos mover los archivos entre distintos tipos de S3:

```
Standard → Standard-IA → One Zone-IA → Glacier → Glacier Deep Archive
   💰💰💰        💰💰           💰         💰/2        💰/5
  (Acceso        (Acceso       (Acceso    (Acceso     (Acceso
  frecuente)     ocasional)    ocasional) archivo)    archivo)
```

### 📋 Recomendaciones por Tipo de Uso:

- **Objetos con acceso poco frecuente** → **Standard-IA**
- **Objetos de archivo (backup/compliance)** → **Glacier o Glacier Deep Archive**
- **Datos críticos con acceso ocasional** → **Standard-IA**
- **Datos no críticos con acceso ocasional** → **One Zone-IA**

---

## ⚙️ REGLAS DE CICLO DE VIDA (Lifecycle Rules)

### 🔄 Acciones de Transición
Configuran objetos para que pasen automáticamente a otra clase de almacenamiento:

```
Día 0: Standard
    ↓ (30 días mínimo)
Día 30: Standard-IA
    ↓ (30 días mínimo)
Día 60: One Zone-IA
    ↓ (90 días mínimo desde Standard)
Día 90: Glacier Instant Retrieval
    ↓
Día 180: Glacier Flexible Retrieval
    ↓
Día 365: Glacier Deep Archive
```

### 🗑️ Acciones de Expiración
Configuran objetos para que se eliminen automáticamente:

- **Logs de acceso**: Eliminar después de 365 días
- **Versiones antiguas**: Eliminar versiones no actuales después de 30 días
- **Uploads incompletos**: Eliminar multipart uploads después de 7 días

### 📁 Casos Prácticos de Lifecycle Rules

#### **Caso 1: Empresa de Media y Entretenimiento**
**Situación**: Videos de streaming que se acceden frecuentemente los primeros 30 días, ocasionalmente hasta 6 meses, y luego se archivan.

**Solución**:
```json
{
  "Rules": [{
    "Id": "MediaLifecycle",
    "Status": "Enabled",
    "Filter": {"Prefix": "videos/"},
    "Transitions": [
      {
        "Days": 30,
        "StorageClass": "STANDARD_IA"
      },
      {
        "Days": 180,
        "StorageClass": "GLACIER"
      }
    ]
  }]
}
```

#### **Caso 2: Logs de Aplicación**
**Situación**: Logs que necesitan estar disponibles por 90 días para análisis, luego archivados por compliance durante 7 años.

**Solución**:
```json
{
  "Rules": [{
    "Id": "LogsLifecycle",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {
        "Days": 90,
        "StorageClass": "GLACIER"
      },
      {
        "Days": 365,
        "StorageClass": "DEEP_ARCHIVE"
      }
    ],
    "Expiration": {
      "Days": 2555
    }
  }]
}
``` 

---

## 📈 AMAZON S3 ANALYTICS

Herramienta que ayuda a tomar decisiones informadas sobre cuándo mover objetos a la clase de almacenamiento más eficiente en costos.

### 🎯 Características Principales:

- **Análisis de patrones de acceso** para Standard e Standard-IA
- **No incluye**: One Zone-IA ni clases Glacier
- **Actualización**: Informes diarios
- **Tiempo inicial**: 24-48 horas para ver los primeros datos
- **Beneficio**: Base sólida para crear Lifecycle Rules

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   S3 Analytics  │───▶│  Análisis de     │───▶│  Lifecycle      │
│   Monitoring    │    │  Patrones de     │    │  Rules          │
│                 │    │  Acceso          │    │  Optimization   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

---

## 💳 MODELO "REQUESTER PAYS" (El Solicitante Paga)

### 🏛️ Modelo Tradicional:
```
┌─────────────────┐    💰 Costos    ┌─────────────────┐
│  Propietario    │◀─────────────────│   AWS S3        │
│  del Bucket     │   • Storage     │   Bucket        │
│                 │   • Requests    │                 │
│                 │   • Data Transfer│                 │
└─────────────────┘                 └─────────────────┘
        ▲                                   ▲
        │                                   │
        └──────── Acceso ───────────────────┘
```

### 🔄 Modelo "Requester Pays":
```
┌─────────────────┐                 ┌─────────────────┐
│  Propietario    │    💰 Storage   │   AWS S3        │
│  del Bucket     │◀────────────────│   Bucket        │
└─────────────────┘                 └─────────────────┘
                                            ▲
┌─────────────────┐    💰 Requests          │
│   Solicitante   │    💰 Data Transfer     │
│   (Usuario)     │─────────────────────────┘
└─────────────────┘
```

### 📋 Casos de Uso:
- **Compartir datasets grandes** con otras organizaciones
- **Distribución de contenido** donde el consumidor paga el acceso
- **Colaboración entre empresas** con datos sensibles al costo

**⚠️ Requisito**: El solicitante debe estar autenticado en AWS (no puede ser anónimo)

---

## 🔔 NOTIFICACIONES DE EVENTOS S3

### 📝 Tipos de Eventos Principales:
- `s3:ObjectCreated:*` - Objeto creado
- `s3:ObjectRemoved:*` - Objeto eliminado
- `s3:ObjectRestore:*` - Objeto restaurado desde Glacier
- `s3:ObjectReplication:*` - Objeto replicado

### 🎯 Filtros Disponibles:
- **Por extensión**: `*.jpg`, `*.pdf`
- **Por prefijo**: `logs/`, `images/`
- **Por tamaño**: Objetos > 1MB
- **Metadatos personalizados**

### 🏗️ Arquitectura Básica para Generación de Miniaturas:

```
┌─────────────┐    🔔 Event     ┌─────────────┐    📤 Message    ┌─────────────┐
│   S3 Bucket │──────────────▶ │     SNS     │─────────────────▶│     SQS     │
│  (Images)   │   *.jpg        │   Topic     │                  │   Queue     │
└─────────────┘   Upload       └─────────────┘                  └─────────────┘
                                                                         │
                                                                         │ Poll
┌─────────────┐    📥 Process   ┌─────────────┐    📤 Trigger    ┌─────▼─────┐
│   S3 Bucket │◀──────────────  │   Lambda    │◀─────────────────│   Lambda   │
│(Thumbnails) │   Save Thumb    │  Function   │                  │  Function  │
└─────────────┘                 └─────────────┘                  └───────────┘
```

### 📊 Caso Práctico: Reorganización de Datos CDC

**Situación**: Archivos CDC llegando en tiempo real desde 20 bases de datos con 100 tablas cada una.

**Estructura Actual**:
```
bucket/
├── db1/
│   ├── table1/files.parquet
│   ├── table2/files.parquet
│   └── ...
├── db2/
│   ├── table1/files.parquet
│   └── ...
└── db20/...
```

**Estructura Objetivo**:
```
analytics-bucket/
├── table1/
│   ├── db1/files.parquet
│   ├── db2/files.parquet
│   └── ...
├── table2/
│   └── ...
└── table100/...
```

**Arquitectura de Solución**:
```
┌─────────────────┐    🔔 S3 Event      ┌─────────────────┐
│   Source S3     │───────────────────▶ │      SQS        │
│   CDC Bucket    │   ObjectCreated     │   Dead Letter   │
│ db*/table*/*.   │                     │     Queue       │
└─────────────────┘                     └─────────────────┘
         │                                       ▲
         │                                       │ Failed Messages
         │                              ┌─────────────────┐
         │                              │   Lambda DLQ    │
         │                              │   Processor     │
         │                              └─────────────────┘
         │
         │ Read Object
         ▼
┌─────────────────┐    📤 Reorganized   ┌─────────────────┐
│   Lambda        │───────────────────▶ │  Target S3      │
│   Function      │    File Structure   │ Analytics Bucket│
│ • Parse path    │                     │ table*/db*/     │
│ • Copy file     │                     └─────────────────┘
│ • Reorganize    │
└─────────────────┘
```

**Código Lambda (Python)**:
```python
import boto3
import json
from urllib.parse import unquote_plus

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    
    for record in event['Records']:
        # Parse S3 event
        bucket = record['s3']['bucket']['name']
        key = unquote_plus(record['s3']['object']['key'])
        
        # Extract db and table from path: db1/table1/file.parquet
        path_parts = key.split('/')
        db_name = path_parts[0]  # db1
        table_name = path_parts[1]  # table1
        file_name = path_parts[2]  # file.parquet
        
        # New structure: table1/db1/file.parquet
        new_key = f"{table_name}/{db_name}/{file_name}"
        
        # Copy to new structure
        copy_source = {'Bucket': bucket, 'Key': key}
        s3.copy_object(
            CopySource=copy_source,
            Bucket='analytics-bucket',
            Key=new_key
        )
        
    return {'statusCode': 200}
```

### 🌉 Integración con EventBridge

**Caso**: Sistema de procesamiento de documentos legales con múltiples workflows.

```
┌─────────────────┐    🔔 Event        ┌─────────────────┐
│   S3 Bucket     │─────────────────▶  │   EventBridge   │
│  Legal Docs     │   .pdf Upload      │   Custom Bus    │
└─────────────────┘                    └─────────────────┘
                                                │
                              ┌─────────────────┼─────────────────┐
                              │                 │                 │
                              ▼                 ▼                 ▼
                    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
                    │   Lambda    │    │Step Functions│    │   Kinesis   │
                    │ OCR Process │    │ Approval     │    │  Analytics  │
                    └─────────────┘    │ Workflow     │    │   Stream    │
                                       └─────────────┘    └─────────────┘
```

**Ventajas de EventBridge**:
- **Filtrado avanzado** con reglas JSON complejas
- **Múltiples destinos** simultáneos
- **Reintentos automáticos** y DLQ
- **Transformación de eventos**

---

## 🚀 RENDIMIENTO Y OPTIMIZACIÓN S3

### 📊 Métricas de Rendimiento Base:
- **Escalabilidad**: Automática para altas tasas de petición
- **Latencia**: 100-200 ms
- **Throughput por prefijo**:
  - **3,500** peticiones PUT/COPY/POST/DELETE por segundo
  - **5,500** peticiones GET/HEAD por segundo

### 📁 Optimización por Prefijos:

```
Bucket Structure:
├── /logs/2024/01/    → 5,500 GET/s
├── /logs/2024/02/    → 5,500 GET/s  
├── /images/products/ → 5,500 GET/s
└── /videos/stream/   → 5,500 GET/s
                        ═══════════
                       22,000 GET/s Total
```

**💡 Consejo**: Distribuye las cargas entre múltiples prefijos para maximizar el rendimiento.

---

## ⚡ TÉCNICAS DE OPTIMIZACIÓN DE PERFORMANCE

### 🔄 Multipart Upload

```
Archivo Grande (>100MB)
┌────────────────────────────┐
│     10GB Video File        │
└────────────────────────────┘
            │ Split
            ▼
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│Part1│ │Part2│ │Part3│ │Part4│
│ 2GB │ │ 2GB │ │ 2GB │ │ 4GB │
└─────┘ └─────┘ └─────┘ └─────┘
   │       │       │       │
   └───────┼───────┼───────┘
          Upload en Paralelo
   ┌───────▼───────▼───────┐
   │      S3 Bucket        │
   └───────────────────────┘
```

**📋 Beneficios**:
- **Obligatorio**: Archivos > 5GB
- **Recomendado**: Archivos > 100MB
- **Paralelización**: Múltiples parts simultáneamente
- **Resiliencia**: Reintentar solo parts que fallan

### 🌐 S3 Transfer Acceleration

```
Cliente (Madrid)
     │ Upload
     ▼
┌─────────────┐    Optimized     ┌─────────────┐
│   Madrid    │    Network       │   S3 Bucket │
│ Edge Loc.   │─────────────────▶│ us-east-1   │
└─────────────┘    (AWS Backbone)└─────────────┘
```

**🎯 Casos de Uso**:
- **Usuarios globales** subiendo a bucket en región específica
- **Archivos grandes** con conexiones lentas
- **Compatible** con Multipart Upload

### 📤 Byte-Range Fetches (Recuperación por Rangos)

#### Para Acelerar Descargas:
```
Archivo 100MB en S3
┌─────────┬─────────┬─────────┬─────────┐
│  0-25MB │ 25-50MB │ 50-75MB │75-100MB │
└─────────┴─────────┴─────────┴─────────┘
     │         │         │         │
     └─────────┼─────────┼─────────┘
         Peticiones Paralelas
┌─────────────────────────────────────────┐
│            Cliente                      │
│        Download 4x Faster               │
└─────────────────────────────────────────┘
```

#### Para Datos Parciales:
```
Log File 1GB
┌────────────┬──────────────────────────┐
│  Header    │       Log Data           │
│  (1KB)     │        (999MB)           │
└────────────┴──────────────────────────┘
     │
     └── GET Range: bytes=0-1023
┌─────────────────────────────────────────┐
│        Solo Header (1KB)                │
│      Ahorro 99.9% Transferencia         │
└─────────────────────────────────────────┘
```

---

## 🔍 S3 SELECT y GLACIER SELECT

### 💡 Concepto: "Server-Side Filtering"

```
Archivo CSV 100MB
┌─────────────────────────────────────────┐
│ timestamp,user_id,action,ip_address     │
│ 2024-01-01,123,login,192.168.1.1       │
│ 2024-01-01,456,logout,10.0.0.1         │
│ ... (millones de registros)             │
└─────────────────────────────────────────┘
         │
         │ S3 SELECT Query:
         │ SELECT user_id, action FROM s3object 
         │ WHERE timestamp LIKE '2024-01-01%'
         ▼
┌─────────────────────────────────────────┐
│         Resultado 2MB                   │
│ user_id,action                          │
│ 123,login                               │
│ 456,logout                              │
└─────────────────────────────────────────┘
```

**💰 Beneficios**:
- **98% menos transferencia** de datos
- **Menor costo de red**
- **Menos CPU** en el cliente
- **Soporte**: CSV, JSON, Parquet

### 📋 Caso Práctico: Análisis de Logs
```python
import boto3

s3 = boto3.client('s3')

response = s3.select_object_content(
    Bucket='logs-bucket',
    Key='access-logs/2024-01-01.csv',
    Expression="""
        SELECT ip_address, COUNT(*) as requests 
        FROM s3object 
        WHERE status_code = '404' 
        GROUP BY ip_address
    """,
    ExpressionType='SQL',
    InputSerialization={
        'CSV': {'FileHeaderInfo': 'USE'},
        'CompressionType': 'GZIP'
    },
    OutputSerialization={'CSV': {}}
)
```

---

## 🔄 S3 BATCH OPERATIONS

### 🎯 Operaciones Masivas Disponibles:

```
┌─────────────────────────────────────────┐
│           S3 Batch Operations           │
├─────────────────────────────────────────┤
│ • Copiar objetos entre buckets          │
│ • Modificar metadatos/propiedades       │
│ • Cambiar storage classes               │
│ • Aplicar/modificar ACLs                │
│ • Añadir/modificar tags                 │
│ • Cifrar objetos no cifrados            │
│ • Restaurar desde Glacier               │
│ • Invocar Lambda (custom actions)       │
└─────────────────────────────────────────┘
```

### 🏗️ Arquitectura de Batch Operations:

```
┌─────────────────┐    📋 Lista        ┌─────────────────┐
│  S3 Inventory   │─────────────────▶  │   Batch Job     │
│   Report        │   de Objetos       │   Definition    │
└─────────────────┘                    └─────────────────┘
         │                                      │
         │ ┌─────────────────┐                  │
         └▶│   S3 Select     │                  │
           │   Filtrado      │──────────────────┘
           └─────────────────┘         📋 Lista Filtrada
                                             │
                                             ▼
┌─────────────────┐    🔄 Ejecuta     ┌─────────────────┐
│     Report      │◀─────────────────  │  Batch Job      │
│  • Success      │   en Lotes        │  Execution      │
│  • Failed       │                   │  • Reintentos   │
│  • Manifest     │                   │  • Progreso     │
└─────────────────┘                   │  • Notificaciones│
                                      └─────────────────┘
```

### 📋 Caso Práctico: Migración de Storage Class

**Situación**: Migrar 10 millones de objetos de Standard a Standard-IA en un bucket de 100TB.

**Solución**:
```json
{
  "JobId": "batch-migration-2024",
  "Description": "Migrate old objects to Standard-IA",
  "Manifest": {
    "Spec": {
      "Format": "S3InventoryReport_CSV_20161130"
    },
    "Location": {
      "ObjectArn": "arn:aws:s3:::inventory-bucket/2024/inventory.csv",
      "ETag": "example-etag"
    }
  },
  "Operation": {
    "S3PutObjectCopy": {
      "StorageClass": "STANDARD_IA",
      "MetadataDirective": "COPY"
    }
  },
  "Priority": 10,
  "RoleArn": "arn:aws:iam::123456789012:role/batch-operations-role"
}
```

**💼 Beneficios del Caso**:
- **Sin código**: No necesitas escribir Lambda
- **Manejo de errores**: Automático con reportes
- **Progreso**: Seguimiento en tiempo real
- **Costo**: Solo pagas por las operaciones ejecutadas

---

## 📚 RECOMENDACIONES PARA EL EXAMEN AWS

### 🎯 Temas Clave que Aparecen Frecuentemente:

#### 📋 **Lifecycle Rules y Storage Classes**
- **Mínimos obligatorios**: 30 días en Standard antes de IA, 90 días antes de Glacier
- **Pregunta típica**: "¿Cuál es la configuración más costo-efectiva para datos que se acceden semanalmente por 2 meses y luego nunca?"
  - **Respuesta**: Standard → Standard-IA (30 días) → Glacier (90 días)

#### 🔔 **S3 Events y Arquitecturas**
- **Escenarios comunes**: Procesamiento automático de archivos subidos
- **Pregunta típica**: "Necesitas procesar imágenes automáticamente cuando se suben a S3"
  - **Respuesta**: S3 Event → SNS → SQS → Lambda (patrón de fan-out)

#### ⚡ **Performance Optimization**
- **Multipart Upload**: Obligatorio >5GB, recomendado >100MB
- **Transfer Acceleration**: Para usuarios globales
- **Pregunta típica**: "¿Cómo optimizar uploads de 10GB desde Asia a bucket en us-east-1?"
  - **Respuesta**: Transfer Acceleration + Multipart Upload

#### 🔍 **S3 Select vs Normal Transfer**
- **Pregunta típica**: "Necesitas solo 10% de un archivo de 1GB regularmente"
  - **Respuesta**: S3 Select para reducir transferencia y costos

### ⚠️ **Errores Comunes en el Examen**:

1. **Confundir Storage Classes**: One Zone-IA NO es para datos críticos
2. **Lifecycle mínimos**: No se puede ir directo a Glacier desde Standard (mínimo 90 días)
3. **Requester Pays**: El usuario DEBE estar autenticado
4. **S3 Analytics**: NO funciona con Glacier classes

### 🎯 **Palabras Clave del Examen**:
- "Cost-effective" → Lifecycle Rules
- "Automatically process" → S3 Events
- "Global users" → Transfer Acceleration
- "Large files" → Multipart Upload
- "Partial data" → S3 Select
- "Million objects" → Batch Operations

### 📊 **Preguntas de Ejemplo Típicas**:

**Pregunta 1**: Una empresa necesita almacenar logs de aplicación que se consultan frecuentemente durante los primeros 30 días, ocasionalmente durante los siguientes 60 días, y después solo para auditorías anuales. ¿Cuál es la estrategia más costo-efectiva?

**Respuesta**: 
- Standard (30 días) → Standard-IA (60 días) → Glacier (después de 90 días)
- Configurar expiración después de 7 años si no se requiere retención permanente

**Pregunta 2**: Necesitas procesar automáticamente videos subidos a S3 para generar diferentes resoluciones. El procesamiento puede fallar y necesita reintentos. ¿Cuál es la mejor arquitectura?

**Respuesta**:
- S3 Event → SQS (con DLQ) → Lambda → Step Functions
- Usar SQS para garantizar procesamiento y manejo de fallos

---

## 📖 RECURSOS ADICIONALES

### 📚 **Documentación Oficial**:
- [S3 Storage Classes](https://docs.aws.amazon.com/s3/latest/userguide/storage-class-intro.html)
- [S3 Event Notifications](https://docs.aws.amazon.com/s3/latest/userguide/NotificationHowTo.html)
- [S3 Performance Guidelines](https://docs.aws.amazon.com/s3/latest/userguide/optimizing-performance.html)
- [S3 Batch Operations](https://docs.aws.amazon.com/s3/latest/userguide/batch-ops.html)

### 🔗 **Labs Recomendados**:
1. **Configurar Lifecycle Rules** para un escenario de media company
2. **Implementar S3 Events** con Lambda para procesamiento de imágenes
3. **Probar Transfer Acceleration** con archivos grandes
4. **Usar S3 Select** para análisis de logs
5. **Configurar Batch Operations** para migración masiva de storage classes

### 💡 **Tips de Estudio**:
- **Practica** configurando Lifecycle Rules con diferentes escenarios
- **Entiende** las diferencias entre SNS, SQS y Lambda en eventos S3
- **Memoriza** los límites de tiempo para Storage Classes
- **Experimenta** con S3 Select usando datos reales
- **Familiarízate** con los casos de uso de cada técnica de optimización

---

## 👨‍💻 **Sobre el Autor**

**Roberto Ayra**  
*AWS Solutions Architect*  
📅 **Abril 2024**

Documentación creada como parte del material de estudio para certificaciones AWS. Este contenido está basado en las mejores prácticas oficiales de AWS y experiencias reales de implementación en proyectos empresariales.

### 🎯 **Especialidades del Autor**:
- Arquitecturas serverless con S3 y Lambda
- Optimización de costos en almacenamiento
- Migración de datos a gran escala
- Implementación de data lakes en AWS

---

### 🔖 **Notas Finales**

Esta documentación cubre los aspectos avanzados de Amazon S3 que son fundamentales para:
- **AWS Solutions Architect Associate/Professional**
- **AWS Developer Associate** 
- **AWS SysOps Administrator**

**🎓 Recomendación de Estudio**: Dedica especial atención a los casos prácticos y arquitecturas presentadas, ya que el examen AWS evalúa principalmente la capacidad de aplicar estos conceptos en escenarios reales.

Para consultas o actualizaciones de este material, contactar al autor o consultar la documentación oficial de AWS que se actualiza constantemente.

---
*Última actualización: Abril 2024*  
*Versión: 2.0*