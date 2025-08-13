# 🔐 AWS IAM (Identity and Access Management) - Guía Completa para Arquitectos

## 📋 Índice
1. [Introducción a AWS IAM](#introducción-a-aws-iam)
2. [Definición de Políticas](#definición-de-políticas)
3. [Definición de Roles](#definición-de-roles)
4. [Políticas y Roles - Relación](#políticas-y-roles---relación)
5. [Estructura de Políticas JSON](#estructura-de-políticas-json)
6. [Ejemplos Prácticos de Políticas](#ejemplos-prácticos-de-políticas)
7. [Simulador de Políticas IAM](#simulador-de-políticas-iam)
8. [Metadatos de EC2 e IAM](#metadatos-de-ec2-e-iam)
9. [SDK de AWS](#sdk-de-aws)
10. [Roadmap de Arquitectura IAM](#roadmap-de-arquitectura-iam)
11. [Mejores Prácticas](#mejores-prácticas)

---

## 🌟 Introducción a AWS IAM

**AWS Identity and Access Management (IAM)** es el servicio fundamental que controla el acceso a los recursos de AWS de forma segura. IAM permite crear y gestionar usuarios, grupos, roles y políticas para establecer quién puede acceder a qué recursos y bajo qué condiciones.

### 🎯 Características Principales
- ✅ **Control de acceso granular** a recursos AWS
- ✅ **Autenticación y autorización** centralizadas
- ✅ **Federación de identidades** con sistemas externos
- ✅ **Multi-Factor Authentication (MFA)**
- ✅ **Auditoría completa** con CloudTrail
- ✅ **Sin costo adicional** (incluido en AWS)

### 🏗️ Componentes Principales de IAM

```
┌─── AWS IAM ───────────────────────────────────────┐
│                                                   │
├── 👤 Usuarios                                     │
│   └── Identidad permanente                        │
│                                                   │
├── 👥 Grupos                                       │
│   └── Colección de usuarios                       │
│                                                   │
├── 🎭 Roles                                        │
│   └── Identidad asumible                          │
│                                                   │
└── 📜 Políticas                                    │
    └── Documento de permisos                       │
    ├── Se adjuntan a Usuarios                      │
    ├── Se adjuntan a Grupos                        │
    └── Se adjuntan a Roles                         │
└───────────────────────────────────────────────────┘
```

**Diagrama de Flujo IAM:**
```
Usuario/Servicio → Trust Policy → Asumir Rol → Permission Policy → Recursos AWS
     ↓               ↓              ↓              ↓               ↓
   Identidad    ¿Quién puede?   Credenciales   ¿Qué puede?    Acceso Final
```

---

## 📜 Definición de Políticas

### 🎯 ¿Qué son las Políticas IAM?

Las **políticas IAM** son documentos JSON que definen permisos. Especifican qué acciones están permitidas o denegadas sobre qué recursos de AWS y bajo qué condiciones.

### 🔧 Tipos de Políticas

#### 1️⃣ **Políticas Basadas en Identidad**
- 📎 **Políticas Gestionadas por AWS**: Creadas y mantenidas por AWS
- 📎 **Políticas Gestionadas por el Cliente**: Creadas y mantenidas por ti
- 📎 **Políticas en Línea**: Adjuntas directamente a un usuario, grupo o rol

#### 2️⃣ **Políticas Basadas en Recursos**
- 🗂️ **Políticas de Bucket S3**: Control de acceso a buckets específicos
- 🔑 **Políticas de Claves KMS**: Control de acceso a claves de cifrado
- 🚪 **Políticas de API Gateway**: Control de acceso a APIs

#### 3️⃣ **Políticas de Control de Servicios (SCPs)**
- 🏢 **AWS Organizations**: Controles de guardrail a nivel organizacional
- 🚫 **Máximo de permisos**: Define lo que NO se puede hacer

### 📊 Comparativa de Tipos de Políticas

| Tipo | Alcance | Gestión | Reutilización | Límites |
|------|---------|---------|---------------|---------|
| **AWS Managed** | Global | AWS | ✅ Alta | N/A |
| **Customer Managed** | Cuenta | Cliente | ✅ Alta | 5000 por cuenta |
| **Inline** | Entidad específica | Cliente | ❌ Baja | Ilimitadas |

### 🔄 Proceso de Evaluación de Políticas

```
┌─ Solicitud de Acceso ─┐
│                       │
└─────────┬─────────────┘
          │
          ▼
┌─────────────────────────┐    SÍ     ┌─────────────┐
│ ¿Denegación Explícita?  ├──────────▶│   DENEGAR   │
└─────────┬───────────────┘           └─────────────┘
          │ NO
          ▼
┌─────────────────────────┐    SÍ     ┌─────────────┐
│ ¿Permiso Explícito?     ├──────────▶│  PERMITIR   │
└─────────┬───────────────┘           └─────────────┘
          │ NO
          ▼
┌─────────────────────────┐
│ DENEGAR por Defecto     │
└─────────────────────────┘
```

**Regla de Oro:** 
- 🔴 **DENY siempre gana** (denegación explícita)
- 🟡 **ALLOW debe ser explícito** (permiso explícito)
- ⚫ **Sin ALLOW = DENY** (denegación por defecto)

---

## 👤 Definición de Roles

### 🎯 ¿Qué son los Roles IAM?

Los **roles IAM** son identidades de AWS que tienen políticas de permisos específicas pero que no están asociados permanentemente con una persona o aplicación específica. En su lugar, pueden ser "asumidos" temporalmente.

### 🔧 Características de los Roles

- 🔄 **Temporales**: Credenciales de seguridad temporales
- 🎭 **Asumibles**: Pueden ser asumidos por usuarios, servicios o aplicaciones
- 🔐 **Seguros**: No requieren credenciales de larga duración
- 🌐 **Cross-Account**: Permiten acceso entre cuentas AWS

### 📋 Tipos de Roles IAM

#### 1️⃣ **Roles de Servicio AWS**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### 2️⃣ **Roles Cross-Account**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/ExampleUser"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueSecretValue"
        }
      }
    }
  ]
}
```

#### 3️⃣ **Roles para Federación de Identidades**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/ExampleProvider"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

### 🏗️ Arquitectura de Roles

```
┌── Usuario/Servicio ──┐       ┌── STS Token ──┐       ┌── Recursos AWS ──┐
│                      │       │               │       │                   │
│  • EC2 Instance      │────┐  │ • AccessKeyId │────┐  │ • S3 Buckets      │
│  • Lambda Function   │    │  │ • SecretKey   │    │  │ • DynamoDB Tables │
│  • Usuario IAM       │    │  │ • SessionToken│    │  │ • EC2 Instances   │
│  • Aplicación        │    │  │ • Expiration  │    │  │ • RDS Databases   │
└──────────────────────┘    │  └───────────────┘    │  └───────────────────┘
                            ▼                       ▼
                   ┌────────────────────┐  ┌────────────────────┐
                   │   Asumir Rol       │  │   Acceso Temporal  │
                   │                    │  │                    │
                   │ 1. Trust Policy    │  │ 1. Permission      │
                   │    verifica QUIÉN  │  │    Policy verifica │
                   │ 2. Genera token    │  │    QUÉ puede hacer │
                   │    temporal        │  │ 2. Ejecuta acción  │
                   └────────────────────┘  └────────────────────┘
```

---

## 🔗 Políticas y Roles - Relación

### 🎯 Integración de Políticas y Roles

Los roles IAM funcionan con **dos tipos de políticas**:

#### 1️⃣ **Trust Policy (Política de Confianza)**
- 🤝 Define **quién** puede asumir el rol
- 📝 Especifica principals (usuarios, servicios, cuentas)
- ⚙️ Se configura durante la creación del rol

#### 2️⃣ **Permission Policy (Política de Permisos)**
- 🔐 Define **qué** puede hacer el rol
- 📋 Especifica acciones permitidas/denegadas
- 🎯 Se adjunta al rol después de crearlo

### 📊 Flujo de Trabajo Roles-Políticas

```
┌─────────────────┐    1. AssumeRole Request    ┌─────────────────┐
│ Usuario/Servicio├──────────────────────────────▶│    AWS STS      │
└─────────────────┘                              └─────┬───────────┘
                                                       │
          ┌─────────────────┐    2. Verificar Trust Policy    │
          │    Rol IAM      │◀───────────────────────────────┘
          └─────┬───────────┘                              
                │ 3. Trust Policy OK                       
                ▼                                          
          ┌─────────────────┐    4. Temporary Credentials ┌─────────────────┐
          │    AWS STS      ├──────────────────────────────▶│ Usuario/Servicio│
          └─────────────────┘                              └─────┬───────────┘
                                                                 │
                                                                 │ 5. Request con Credentials
                                                                 ▼
          ┌─────────────────┐    6. Verificar Permission Policy ┌─────────────────┐
          │    Rol IAM      │◀───────────────────────────────────│ Recursos AWS    │
          └─────┬───────────┘                                    └─────────────────┘
                │ 7. Permission Policy OK                        
                ▼                                                
          ┌─────────────────┐    8. Acceso Concedido            
          │ Recursos AWS    │◀───────────────────────────────────┘
          └─────────────────┘                                    
```

### 🔧 Ejemplo Práctico: EC2 accediendo a S3

**Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permission Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::mi-bucket/*"
    }
  ]
}
```

---

## 📄 Estructura de Políticas JSON

### 🏗️ Anatomía de una Política IAM

Una política IAM es un documento JSON con los siguientes componentes:

```json
{
  "Version": "2012-10-17",
  "Id": "PolicyId-Optional",
  "Statement": [
    {
      "Sid": "StatementId-Optional",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/username"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::example-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### 🔧 Componentes Detallados

#### 1️⃣ **Version** (Obligatorio)
```json
"Version": "2012-10-17"
```
- 📅 Especifica la versión del lenguaje de políticas
- 🎯 Siempre usar `"2012-10-17"` (versión actual)

#### 2️⃣ **Id** (Opcional)
```json
"Id": "S3-Console-Auto-Gen-Policy-1234567890"
```
- 🆔 Identificador único para la política
- 📝 Útil para organización y auditoría

#### 3️⃣ **Statement** (Obligatorio)
Array de declaraciones que definen los permisos:

```json
"Statement": [
  {
    // Declaración 1
  },
  {
    // Declaración 2
  }
]
```

#### 4️⃣ **Sid** (Opcional)
```json
"Sid": "AllowS3ReadWrite"
```
- 🏷️ Identificador para la declaración individual
- 📖 Mejora la legibilidad y debugging

#### 5️⃣ **Effect** (Obligatorio)
```json
"Effect": "Allow"  // o "Deny"
```
- ✅ **Allow**: Permite la acción
- ❌ **Deny**: Deniega la acción (tiene precedencia)

#### 6️⃣ **Principal** (Condicional)
```json
// Diferentes tipos de principals
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/username",
  "Service": "ec2.amazonaws.com",
  "Federated": "arn:aws:iam::123456789012:saml-provider/Provider"
}

// Principal universal (público)
"Principal": "*"
```

#### 7️⃣ **Action** (Obligatorio para Allow/Deny)
```json
// Acción específica
"Action": "s3:GetObject"

// Múltiples acciones
"Action": [
  "s3:GetObject",
  "s3:PutObject",
  "s3:DeleteObject"
]

// Wildcards
"Action": "s3:*"
"Action": "s3:Get*"
```

#### 8️⃣ **Resource** (Obligatorio para Allow/Deny)
```json
// Recurso específico
"Resource": "arn:aws:s3:::mi-bucket/archivo.txt"

// Múltiples recursos
"Resource": [
  "arn:aws:s3:::mi-bucket",
  "arn:aws:s3:::mi-bucket/*"
]

// Wildcards
"Resource": "*"
```

#### 9️⃣ **Condition** (Opcional)
```json
"Condition": {
  "StringEquals": {
    "s3:x-amz-server-side-encryption": "AES256"
  },
  "IpAddress": {
    "aws:SourceIp": "203.0.113.0/24"
  },
  "DateGreaterThan": {
    "aws:CurrentTime": "2023-12-01T00:00:00Z"
  }
}
```

### 🎯 Operadores de Condición Comunes

| Operador | Descripción | Ejemplo |
|----------|-------------|---------|
| `StringEquals` | Coincidencia exacta de cadena | `"aws:username": "alice"` |
| `StringLike` | Coincidencia con wildcards | `"s3:prefix": "home/*"` |
| `IpAddress` | Rango de direcciones IP | `"aws:SourceIp": "203.0.113.0/24"` |
| `DateGreaterThan` | Fecha posterior a | `"aws:CurrentTime": "2023-01-01T00:00:00Z"` |
| `Bool` | Valor booleano | `"aws:SecureTransport": "true"` |
| `NumericLessThan` | Valor numérico menor | `"s3:max-keys": "10"` |

---

## 💡 Ejemplos Prácticos de Políticas

### 🗂️ Ejemplo 1: Acceso Completo a S3 Bucket Específico

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mi-bucket-empresa"
    },
    {
      "Sid": "ObjectOperations",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mi-bucket-empresa/*"
    }
  ]
}
```

### 🏠 Ejemplo 2: Acceso de Usuario a su "Carpeta Personal" en S3

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUserToSeeRootBucketListing",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::empresa-users",
      "Condition": {
        "StringEquals": {
          "s3:prefix": [
            "",
            "home/",
            "home/${aws:username}/"
          ],
          "s3:delimiter": ["/"]
        }
      }
    },
    {
      "Sid": "AllowUserToListOwnFolder",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::empresa-users",
      "Condition": {
        "StringLike": {
          "s3:prefix": "home/${aws:username}/*"
        }
      }
    },
    {
      "Sid": "AllowUserToManageOwnFolder",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::empresa-users/home/${aws:username}/*"
    }
  ]
}
```

### 🛡️ Ejemplo 3: Rol para EC2 con Acceso Limitado a S3

**Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permission Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadOnlyAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mi-bucket-config",
        "arn:aws:s3:::mi-bucket-config/*"
      ]
    },
    {
      "Sid": "S3WriteToLogsFolder",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::mi-bucket-logs/app-logs/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### 🌐 Ejemplo 4: Política Cross-Account con Condiciones de Seguridad

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CrossAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/DataAnalysisRole"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::shared-data-bucket/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "true"
        },
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### 📱 Ejemplo 5: Política para Aplicación Web con MFA

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBasicActions",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::app-public-content",
        "arn:aws:s3:::app-public-content/*"
      ]
    },
    {
      "Sid": "AllowAdminActionsWithMFA",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::app-public-content/*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "3600"
        }
      }
    }
  ]
}
```

---

## 🧪 Simulador de Políticas IAM

### 🎯 ¿Qué es el Simulador de Políticas?

El **AWS IAM Policy Simulator** es una herramienta que permite probar y validar políticas IAM sin afectar recursos reales.

**URL de acceso:** https://policysim.aws.amazon.com/

### 🔧 Características Principales

- ✅ **Pruebas sin riesgo**: Simula acciones sin ejecutarlas
- ✅ **Validación de políticas**: Verifica antes de implementar
- ✅ **Debugging avanzado**: Identifica por qué se permite/deniega acceso
- ✅ **Múltiples escenarios**: Prueba diferentes combinaciones
- ✅ **Análisis de condiciones**: Evalúa condiciones complejas

### 📋 Casos de Uso del Simulador

#### 1️⃣ **Validación Antes de Implementar**
```bash
# Pregunta: ¿Puede el usuario "alice" eliminar objetos en "mi-bucket"?
# Acción a simular: s3:DeleteObject
# Recurso: arn:aws:s3:::mi-bucket/*
# Usuario: alice
```

#### 2️⃣ **Debugging de Problemas de Acceso**
```bash
# Escenario: Usuario reporta error 403 al acceder a S3
# Simular: s3:GetObject con las políticas actuales del usuario
# Resultado: Identificar política que está denegando acceso
```

#### 3️⃣ **Testing de Políticas Cross-Account**
```bash
# Simular: Usuario de cuenta A asumiendo rol en cuenta B
# Validar: Cadena completa de permisos entre cuentas
```

### 🛠️ Mejores Prácticas para el Simulador

#### ✅ **Preparación**
1. **Documenta el escenario** que quieres probar
2. **Identifica todos los usuarios/roles** involucrados
3. **Lista todas las políticas** que pueden aplicar
4. **Define condiciones específicas** (IP, MFA, tiempo, etc.)

#### ✅ **Durante la Simulación**
1. **Prueba casos positivos y negativos**
2. **Varía las condiciones** (IP, fecha, MFA)
3. **Simula con diferentes usuarios**
4. **Documenta los resultados**

#### ✅ **Interpretación de Resultados**
```
✅ Allowed: La acción está permitida
❌ Denied: La acción está denegada
ℹ️  implicitDeny: No hay política que permita (denegación por defecto)
⚠️  explicitDeny: Hay una política que explícitamente deniega
```

### 🧪 Ejemplo Práctico de Simulación

**Escenario:** Validar que un rol de EC2 puede leer de S3 pero no escribir

**Configuración de Simulación:**
```json
{
  "PolicyInputList": [
    {
      "InputPolicyDocument": "JSON_DE_LA_POLÍTICA_DEL_ROL"
    }
  ],
  "ActionNames": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "ResourceArns": [
    "arn:aws:s3:::mi-bucket/archivo.txt"
  ],
  "ContextKeys": [
    {
      "ContextKeyName": "aws:SourceIp",
      "ContextKeyValues": ["203.0.113.5"],
      "ContextKeyType": "string"
    }
  ]
}
```

**Resultados Esperados:**
- `s3:GetObject`: ✅ **Allowed**
- `s3:PutObject`: ❌ **Denied** (implicitDeny)

---

## 🖥️ Metadatos de EC2 e IAM

### 🎯 ¿Qué son los Metadatos de EC2?

Los **metadatos de EC2** son información sobre la instancia que está disponible desde dentro de la propia instancia. Incluyen información del IAM asociado.

**URL de acceso:** `http://169.254.169.254/latest/meta-data/`

### 🔧 Información Disponible en Metadatos

#### 📊 Metadatos Generales
```bash
# Información básica de la instancia
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/instance-type
curl http://169.254.169.254/latest/meta-data/local-ipv4
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/placement/availability-zone
```

#### 🔐 Información de IAM
```bash
# Información del rol IAM asociado
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Credenciales temporales del rol
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MI-ROL-EC2
```

#### 📄 Datos de Usuario
```bash
# Script de inicialización
curl http://169.254.169.254/latest/user-data
```

### 🔒 Consideraciones de Seguridad

#### ⚠️ **Limitaciones Importantes**
- ✅ **Puedes obtener**: Nombre del rol IAM
- ✅ **Puedes obtener**: Credenciales temporales
- ❌ **NO puedes obtener**: Políticas IAM del rol
- ❌ **NO puedes obtener**: Trust policies del rol

#### 🛡️ **Protección contra SSRF**
```bash
# IMDSv2 (más seguro) - requiere token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
```

### 💻 Ejemplo Práctico: Script de Inicialización

```bash
#!/bin/bash
# Script que usa metadatos para configuración dinámica

# Obtener información de la instancia
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
IAM_ROLE=$(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)

echo "Instancia: $INSTANCE_ID"
echo "Región: $REGION"
echo "Rol IAM: $IAM_ROLE"

# Usar AWS CLI sin configurar credenciales (usa el rol automáticamente)
aws s3 ls s3://mi-bucket-config/ --region $REGION

# Descargar configuración específica para esta instancia
aws s3 cp s3://mi-bucket-config/$INSTANCE_ID.conf /etc/myapp/config.conf
```

---

## 🛠️ SDK de AWS

### 🎯 ¿Qué son los SDKs de AWS?

Los **Software Development Kits (SDKs)** de AWS permiten realizar acciones de AWS directamente desde el código de aplicación, sin necesidad de usar la CLI.

### 🌐 SDKs Oficiales Disponibles

| Lenguaje | Nombre | Características |
|----------|--------|-----------------|
| **Java** | AWS SDK for Java | ☕ Soporte completo, alta performance |
| **Python** | Boto3 | 🐍 Muy popular, fácil de usar |
| **Node.js** | AWS SDK for JavaScript | 🟨 Async/await, TypeScript support |
| **PHP** | AWS SDK for PHP | 🐘 Composer, PSR compliance |
| **.NET** | AWS SDK for .NET | 🔷 C#, VB.NET, F# support |
| **Go** | AWS SDK for Go | ⚡ Alto rendimiento, concurrencia |
| **Ruby** | AWS SDK for Ruby | 💎 Sintaxis elegante, Rails integration |
| **C++** | AWS SDK for C++ | ⚡ Máximo rendimiento, sistemas embebidos |

### 🔧 Configuración de Credenciales

#### 1️⃣ **Orden de Precedencia de Credenciales**

```
SDK busca credenciales en este orden:

1. ┌─────────────────────────┐
   │ Código explícito        │ ← Más alta prioridad
   │ (hardcoded credentials) │
   └─────────────────────────┘
                ↓ Si no encuentra
2. ┌─────────────────────────┐
   │ Variables de entorno    │
   │ AWS_ACCESS_KEY_ID       │
   │ AWS_SECRET_ACCESS_KEY   │
   └─────────────────────────┘
                ↓ Si no encuentra
3. ┌─────────────────────────┐
   │ Archivo de credenciales │
   │ ~/.aws/credentials      │
   └─────────────────────────┘
                ↓ Si no encuentra
4. ┌─────────────────────────┐
   │ IAM Role en EC2         │
   │ (Instance Metadata)     │
   └─────────────────────────┘
                ↓ Si no encuentra
5. ┌─────────────────────────┐
   │ IAM Role en ECS         │
   │ (Task Role)             │
   └─────────────────────────┘
                ↓ Si no encuentra
6. ┌─────────────────────────┐
   │ Error                   │ ← Más baja prioridad
   │ No credentials found    │
   └─────────────────────────┘
```

#### 2️⃣ **Variables de Entorno**
```bash
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
```

#### 3️⃣ **Archivo de Credenciales** (`~/.aws/credentials`)
```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[desarrollo]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY

[produccion]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

### 💻 Ejemplos de Uso por Lenguaje

#### 🐍 **Python (Boto3)**
```python
import boto3
from botocore.exceptions import NoCredentialsError

# Cliente S3 (región por defecto: us-east-1)
s3_client = boto3.client('s3')

# Cliente S3 con región específica
s3_client = boto3.client('s3', region_name='eu-west-1')

# Ejemplo: Listar buckets
try:
    response = s3_client.list_buckets()
    for bucket in response['Buckets']:
        print(f"Bucket: {bucket['Name']}")
except NoCredentialsError:
    print("Error: Credenciales no configuradas")
```

#### ☕ **Java**
```java
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.ListBucketsResponse;
import software.amazon.awssdk.regions.Region;

public class S3Example {
    public static void main(String[] args) {
        // Cliente con región específica
        S3Client s3 = S3Client.builder()
            .region(Region.US_EAST_1)
            .build();
        
        // Listar buckets
        ListBucketsResponse response = s3.listBuckets();
        response.buckets().forEach(bucket -> 
            System.out.println("Bucket: " + bucket.name())
        );
    }
}
```

#### 🟨 **Node.js**
```javascript
const { S3Client, ListBucketsCommand } = require('@aws-sdk/client-s3');

// Cliente S3
const s3Client = new S3Client({ 
    region: 'us-east-1' 
});

async function listBuckets() {
    try {
        const command = new ListBucketsCommand({});
        const response = await s3Client.send(command);
        
        response.Buckets.forEach(bucket => {
            console.log(`Bucket: ${bucket.Name}`);
        });
    } catch (error) {
        console.error('Error:', error);
    }
}

listBuckets();
```

### 🌍 Configuración de Región

#### ⚙️ **Región por Defecto**
Si no especificas una región, los SDKs usan **us-east-1** por defecto.

#### 🎯 **Mejores Prácticas para Regiones**
```python
# ❌ Malo: Usar región por defecto
s3_client = boto3.client('s3')

# ✅ Bueno: Especificar región explícitamente
s3_client = boto3.client('s3', region_name='eu-west-1')

# ✅ Mejor: Usar variable de entorno
import os
region = os.environ.get('AWS_DEFAULT_REGION', 'us-east-1')
s3_client = boto3.client('s3', region_name=region)
```

### 🔐 Ejemplo: Uso de Roles IAM en EC2

```python
import boto3
import requests

# En una instancia EC2, este código funciona automáticamente
# sin configurar credenciales explícitas

def get_instance_info():
    # Obtener metadatos de la instancia
    metadata_url = "http://169.254.169.254/latest/meta-data/"
    instance_id = requests.get(f"{metadata_url}instance-id").text
    
    # Usar el rol IAM asociado a la instancia automáticamente
    s3 = boto3.client('s3')
    
    # El SDK usa automáticamente el rol IAM de la instancia
    buckets = s3.list_buckets()
    
    return {
        'instance_id': instance_id,
        'bucket_count': len(buckets['Buckets'])
    }
```

---

## 🗺️ Roadmap de Arquitectura IAM

### 🏗️ Niveles de Madurez en Arquitectura IAM

```
Evolución de IAM Architecture:

Nivel 1: Básico               Nivel 2: Intermedio           Nivel 3: Avanzado            Nivel 4: Enterprise
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ • Root account  │   ──▶    │ • Grupos        │   ──▶    │ • Federación    │   ──▶    │ • Organizations │
│ • Usuarios      │          │ • Roles         │          │ • Cross-account │          │ • SCPs          │
│ • Políticas     │          │ • MFA           │          │ • SAML/OIDC     │          │ • Identity      │
│   básicas       │          │ • Granularidad  │          │ • Boundaries    │          │   Center        │
└─────────────────┘          └─────────────────┘          └─────────────────┘          └─────────────────┘

Características:             Características:             Características:             Características:
• Acceso individual         • Organización              • SSO                       • Governance
• Políticas simples         • Estructura                • Integración AD            • Compliance
• Uso de root               • Automatización            • Políticas avanzadas       • Multi-cuenta
```

**Timeline de Implementación:**
```
Semanas:  1-2      2-4        4-8         8-12
         ┌──┐    ┌────┐    ┌────────┐  ┌──────────┐
         │██│    │████│    │████████│  │██████████│
         └──┘    └────┘    └────────┘  └──────────┘
        Básico Intermedio  Avanzado    Enterprise
```

### 📋 Fase 1: Fundamentos (Nivel Básico)

#### 🎯 **Objetivos**
- Eliminar uso de cuenta root
- Configurar usuarios IAM básicos
- Implementar MFA

#### ✅ **Checklist de Implementación**
- [ ] Crear usuario IAM administrativo
- [ ] Habilitar MFA en cuenta root
- [ ] Habilitar MFA en usuario admin
- [ ] Crear políticas básicas de lectura/escritura
- [ ] Configurar billing alerts
- [ ] Documentar credenciales de emergencia

#### 🔧 **Políticas Base Recomendadas**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### 📋 Fase 2: Estructuración (Nivel Intermedio)

#### 🎯 **Objetivos**
- Organizar usuarios en grupos
- Implementar roles para servicios
- Establecer políticas granulares

#### ✅ **Estructura Organizacional**
```
IAM Structure:
├── Groups/
│   ├── Administrators
│   ├── Developers
│   ├── ReadOnlyUsers
│   └── Auditors
├── Roles/
│   ├── EC2-S3-Access-Role
│   ├── Lambda-Execution-Role
│   └── CrossAccount-Role
└── Policies/
    ├── Company-S3-Access
    ├── Company-EC2-Management
    └── Company-Developer-Base
```

#### 🔧 **Política de Grupo: Desarrolladores**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDevEnvironment",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "lambda:*",
        "logs:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
        }
      }
    },
    {
      "Sid": "DenyProductionResources",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:ResourceTag/Environment": "production"
        }
      }
    }
  ]
}
```

### 📋 Fase 3: Federación (Nivel Avanzado)

#### 🎯 **Objetivos**
- Integrar con Active Directory
- Implementar Single Sign-On (SSO)
- Configurar acceso cross-cuenta

#### 🔧 **Configuración SAML**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/CompanyAD"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

#### 🌐 **Arquitectura Cross-Account**

```
Arquitectura de Múltiples Cuentas AWS:

                    ┌───────────────────────────────┐
                    │      Cuenta Central           │
                    │      (Identity Hub)           │
                    │                               │
                    │  ┌─────────────────────────┐  │
    ┌─────────┐     │  │   SAML Identity         │  │     ┌─────────────┐
    │Usuario  │────▶│  │   Provider              │  │◀────│ Active      │
    │  AD     │     │  └─────────────────────────┘  │     │ Directory   │
    └─────────┘     │                               │     └─────────────┘
                    │  ┌─────────────────────────┐  │
                    │  │   Cross-Account Roles   │  │
                    │  └─────────┬───────────────┘  │
                    └────────────┼───────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
    ┌──────────┐           ┌──────────┐           ┌──────────┐
    │ Cuenta   │           │ Cuenta   │           │ Cuenta   │
    │   Dev    │           │ Staging  │           │   Prod   │
    │          │           │          │           │          │
    │ ┌──────┐ │           │ ┌──────┐ │           │ ┌──────┐ │
    │ │ Dev  │ │           │ │Stage │ │           │ │ Prod │ │
    │ │Resources│           │ │Resources│           │ │Resources│
    │ └──────┘ │           │ └──────┘ │           │ └──────┘ │
    └──────────┘           └──────────┘           └──────────┘
```

### 📋 Fase 4: Governance Enterprise (Nivel Enterprise)

#### 🎯 **Objetivos**
- Implementar AWS Organizations
- Configurar Service Control Policies (SCPs)
- Establecer governance centralizada

#### 🏢 **Estructura Organizations**

```
AWS Organizations - Estructura Jerárquica:

                    Organization Root
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Security OU  │  │Production OU  │  │Non-Prod OU   │
│               │  │               │  │               │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │   Audit   │ │  │ │Prod App 1 │ │  │ │    Dev    │ │
│ │ Account   │ │  │ │ Account   │ │  │ │ Account   │ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
│               │  │               │  │               │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │Log Archive│ │  │ │Prod App 2 │ │  │ │   Test    │ │
│ │ Account   │ │  │ │ Account   │ │  │ │ Account   │ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
└───────────────┘  └───────────────┘  └───────────────┘
                                              │
                                              ▼
                                    ┌───────────────┐
                                    │  Sandbox OU   │
                                    │               │
                                    │ ┌───────────┐ │
                                    │ │Sandbox 1  │ │
                                    │ │ Account   │ │
                                    │ └───────────┘ │
                                    │               │
                                    │ ┌───────────┐ │
                                    │ │Sandbox 2  │ │
                                    │ │ Account   │ │
                                    │ └───────────┘ │
                                    └───────────────┘

SCPs aplicadas:
├── Root: Deny Root User Access
├── Production OU: Deny Dev Actions, Require MFA
├── Non-Prod OU: Deny Prod Resources Access
└── Sandbox OU: Limit Spend, Deny Sensitive Services
```

#### 🔒 **SCP Example: Deny Root Usage**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Principal": {
        "AWS": "*"
      },
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalType": "Root"
        }
      }
    }
  ]
}
```

### 📊 Timeline de Implementación

| Fase | Duración | Prioridad | Dependencias |
|------|----------|-----------|--------------|
| **Fase 1: Fundamentos** | 1-2 semanas | 🔴 Crítica | Ninguna |
| **Fase 2: Estructuración** | 2-4 semanas | 🟡 Alta | Fase 1 |
| **Fase 3: Federación** | 4-8 semanas | 🟢 Media | Fase 2, AD existente |
| **Fase 4: Enterprise** | 8-12 semanas | 🔵 Baja | Múltiples cuentas |

---

## ✅ Mejores Prácticas

### 🔐 Seguridad

#### 1️⃣ **Principio de Menor Privilegio**
```json
// ❌ Mal ejemplo: Demasiado permisivo
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// ✅ Buen ejemplo: Específico y limitado
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::mi-bucket-especifico/*"
}
```

#### 2️⃣ **Uso de Roles vs Usuarios**
- ✅ **Roles**: Para aplicaciones, servicios AWS, acceso temporal
- ✅ **Usuarios**: Para personas que necesitan acceso permanente
- ❌ **Evitar**: Credenciales hardcodeadas en aplicaciones

#### 3️⃣ **Multi-Factor Authentication (MFA)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### 🏗️ Arquitectura

#### 1️⃣ **Organización Jerárquica**
```
Organización Recomendada:
├── Root Account (solo billing)
├── Security Account (logs, auditoría)
├── Shared Services Account (DNS, AD)
├── Production Accounts (por aplicación)
├── Non-Production Accounts (dev, test)
└── Sandbox Accounts (experimentación)
```

#### 2️⃣ **Naming Conventions**
```bash
# Formato recomendado para recursos IAM
Usuarios: firstname.lastname
Grupos: {company}-{function}-{environment}
Roles: {service}-{purpose}-role
Políticas: {company}-{service}-{action}-policy

# Ejemplos
Usuario: john.doe
Grupo: acme-developers-dev
Rol: ec2-s3access-role
Política: acme-s3-readonly-policy
```

### 📊 Monitoreo y Auditoría

#### 1️⃣ **CloudTrail Configuration**
```json
{
  "Trail": {
    "Name": "company-management-trail",
    "S3BucketName": "company-cloudtrail-logs",
    "IncludeGlobalServiceEvents": true,
    "IsMultiRegionTrail": true,
    "EnableLogFileValidation": true,
    "EventSelectors": [
      {
        "ReadWriteType": "All",
        "IncludeManagementEvents": true,
        "DataResources": [
          {
            "Type": "AWS::S3::Object",
            "Values": ["arn:aws:s3:::sensitive-bucket/*"]
          }
        ]
      }
    ]
  }
}
```

#### 2️⃣ **Métricas Clave para Monitorear**
- 🔍 **Intentos de acceso fallidos**
- 🔍 **Uso de cuenta root**
- 🔍 **Cambios en políticas IAM**
- 🔍 **Creación/eliminación de usuarios**
- 🔍 **AssumeRole desde IPs inusuales**

#### 3️⃣ **Alertas Recomendadas**
```yaml
CloudWatch Alarms:
  - Root Account Usage
  - IAM Policy Changes
  - Failed Login Attempts (>5 in 5 minutes)
  - New IAM User Creation
  - MFA Device Deactivation
```

### 💰 Optimización de Costos

#### 1️⃣ **Access Analyzer**
```bash
# Usar AWS Access Analyzer para identificar permisos no utilizados
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/company-analyzer
```

#### 2️⃣ **Credential Report**
```bash
# Generar reporte de credenciales para auditar uso
aws iam generate-credential-report
aws iam get-credential-report --output text | base64 -d > credential-report.csv
```

#### 3️⃣ **Cleanup Automation**
```python
import boto3
from datetime import datetime, timedelta

def cleanup_unused_access_keys():
    iam = boto3.client('iam')
    
    # Obtener todos los usuarios
    users = iam.list_users()['Users']
    
    for user in users:
        username = user['UserName']
        access_keys = iam.list_access_keys(UserName=username)['AccessKeyMetadata']
        
        for key in access_keys:
            last_used = iam.get_access_key_last_used(AccessKeyId=key['AccessKeyId'])
            
            # Si la clave no se ha usado en 90 días
            if 'LastUsedDate' in last_used['AccessKeyLastUsed']:
                days_unused = (datetime.now(timezone.utc) - 
                             last_used['AccessKeyLastUsed']['LastUsedDate']).days
                
                if days_unused > 90:
                    print(f"Key {key['AccessKeyId']} for {username} unused for {days_unused} days")
                    # Opcional: Desactivar o eliminar clave
```

---

## 🎯 Resumen Ejecutivo

### 🔑 Puntos Clave de IAM

1. **🔐 Seguridad por Diseño**
   - Principio de menor privilegio
   - MFA obligatorio para acciones críticas
   - Rotación regular de credenciales

2. **🏗️ Arquitectura Escalable**
   - Uso de roles sobre usuarios para servicios
   - Federación con sistemas de identidad existentes
   - Governance centralizada con Organizations

3. **📊 Monitoreo Continuo**
   - CloudTrail para auditoría
   - Access Analyzer para optimización
   - Alertas proactivas para amenazas

4. **💰 Optimización de Costos**
   - Cleanup regular de recursos no utilizados
   - Políticas granulares para evitar sobrepermisos
   - Automatización para gestión eficiente

### 🚀 Próximos Pasos Recomendados

1. **Implementar Fase 1** del roadmap (fundamentos)
2. **Configurar monitoreo** básico con CloudTrail
3. **Establecer naming conventions** para la organización
4. **Planificar federación** con sistemas existentes
5. **Automatizar tareas** repetitivas de gestión IAM

---

*Documento creado por: **Roberto Ayra** - AWS Solutions Architect | Fecha: Agosto 2025 | Versión: 1.0*

---

## 📚 Referencias y Recursos Adicionales

### 🔗 **Enlaces Oficiales AWS**
- [Documentación oficial AWS IAM](https://docs.aws.amazon.com/iam/)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS IAM Policy Simulator](https://policysim.aws.amazon.com/)
- [AWS Organizations User Guide](https://docs.aws.amazon.com/organizations/)

### 📖 **Recursos de Estudio**
- [AWS Certified Solutions Architect Study Guide](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)

### 🛠️ **Herramientas Útiles**
- [AWS CLI Reference - IAM](https://docs.aws.amazon.com/cli/latest/reference/iam/)
- [AWS CloudFormation IAM Templates](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html)
- [AWS SDK Documentation](https://aws.amazon.com/tools/)

---

*Este documento forma parte del programa de capacitación en AWS Cloud Architecture desarrollado por Roberto Ayra.*










