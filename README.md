# NetApp ONTAP Adaptive QoS Policy Creation Script

## Descripción

Script para automatizar la creación de políticas adaptativas de calidad de servicio (AQoS) en sistemas NetApp ONTAP mediante la API REST Python Client Library.

## Características

- ✅ Creación de políticas adaptativas de QoS (AQoS)
- ✅ Configuración de IOPS esperados y pico por TB
- ✅ Configuración de IOPS mínimos absolutos
- ✅ Selección de tipo de asignación (allocated_space/used_space)
- ✅ Verificación automática de la política creada
- ✅ Registro de eventos en archivos JSON con timestamp
- ✅ Obtención de event logs de la cabina NetApp
- ✅ Gestión completa de errores y validaciones

## Requisitos

- NetApp ONTAP 9.6+
- Python 3.7+
- Bibliotecas Python:
  - `netapp-ontap`
  - `PyYAML`

## Instalación

```bash
pip install netapp-ontap PyYAML
```

## Configuración

Edita el archivo `config.yaml` con tus parámetros:

```yaml
# CLUSTER CONNECTION SETTINGS
cluster:
  host: cluster1.demo.netapp.com
  username: admin
  password: "Netapp1!"

# STORAGE VIRTUAL MACHINE (SVM) SETTINGS
svm:
  name: svm_1_cluster

# ADAPTIVE QOS POLICY SETTINGS
aqos_policy:
  name: NAS1200_SVMv2-XXXXXX-RHOSO_COR_NAS01-NAS
  vserver: svm_1_cluster
  expected_iops: 1200              # IOPS/TB esperados
  peak_iops: 3600                  # IOPS/TB pico
  expected_iops_allocation: allocated_space   # allocated_space o used_space
  peak_iops_allocation: allocated_space       # allocated_space o used_space
  absolute_min_iops: 1200          # IOPS mínimos absolutos
  block_size: ANY                  # Tamaño de bloque (ANY, 4k, 8k, etc.)
```

## Uso

```bash
python aqos_policies.py
```

## Parámetros de Configuración

### Política Adaptativa (aqos_policy)

| Parámetro | Descripción | Valores | Requerido |
|-----------|-------------|---------|-----------|
| `name` | Nombre único de la política | String | ✅ |
| `vserver` | Nombre de la SVM propietaria | String | ✅ |
| `expected_iops` | IOPS esperados por TB | Integer | ✅ |
| `peak_iops` | IOPS pico por TB | Integer | ✅ |
| `expected_iops_allocation` | Tipo de asignación para IOPS esperados | `allocated_space` / `used_space` | ✅ |
| `peak_iops_allocation` | Tipo de asignación para IOPS pico | `allocated_space` / `used_space` | ✅ |
| `absolute_min_iops` | IOPS mínimos absolutos | Integer | ✅ |
| `block_size` | Tamaño de bloque | `ANY`, `4k`, `8k`, `16k`, etc. | ❌ |

### Tipos de Asignación

- **`allocated_space`**: Los IOPS se calculan basándose en el espacio asignado del objeto
- **`used_space`**: Los IOPS se calculan basándose en el espacio utilizado del objeto

## Equivalencia CLI

El script crea una política equivalente al siguiente comando ONTAP CLI:

```bash
qos adaptive-policy-group create \
  -policygroup NAS1200_SVMv2-XXXXXX-RHOSO_COR_NAS01-NAS \
  -vserver SVMv2-XXXXXXRHOSO_COR_NAS01-NAS \
  -expected-iops 1200iops/tb \
  -peak-iops 3600iops/tb \
  -expected-iops-allocation allocated-space \
  -peak-iops-allocation allocated-space \
  -absolute-min-iops 1200iops/tb \
  -block-size ANY
```

## Archivos de Log

El script genera automáticamente archivos de log en el directorio `logs/`:

- `aqos_policy_created_YYYYMMDD_HHMMSS.json` - Información de la política creada
- `event_logs_YYYYMMDD_HHMMSS.json` - Event logs del cluster

## Funciones Principales

### `aqos_policies_creation(aqos_config)`

Crea una política adaptativa de QoS en NetApp ONTAP.

**Parámetros:**
- `aqos_config` (dict): Diccionario con la configuración de la política

**Retorna:**
- `bool`: True si exitoso, False si falla

**Ejemplo:**
```python
aqos_config = {
    'name': 'NAS1200_policy',
    'vserver': 'svm_1',
    'expected_iops': 1200,
    'peak_iops': 3600,
    'expected_iops_allocation': 'allocated_space',
    'peak_iops_allocation': 'allocated_space',
    'absolute_min_iops': 1200
}
aqos_policies_creation(aqos_config)
```

### `get_event_logs(max_records=100)`

Obtiene los event logs del sistema NetApp ONTAP.

**Parámetros:**
- `max_records` (int): Número máximo de eventos a recuperar

**Retorna:**
- `bool`: True si exitoso, False si falla

## Flujo de Trabajo

1. **Carga de configuración** - Lee y valida `config.yaml`
2. **Conexión al cluster** - Verifica acceso a la cabina NetApp
3. **Validación de parámetros** - Comprueba que existan todos los campos requeridos
4. **Creación de política** - Envía petición POST a la API REST
5. **Verificación** - Obtiene los detalles de la política creada
6. **Registro** - Guarda información en archivo JSON
7. **Event logs** - Obtiene logs del cluster

## Gestión de Errores

El script maneja los siguientes errores:

- **NetAppRestError 409**: La política ya existe
- **NetAppRestError 400**: Parámetros inválidos
- **KeyError**: Falta un campo de configuración requerido
- **ConnectionError**: No se puede conectar al cluster
- **FileNotFoundError**: No se encuentra config.yaml

## Documentación NetApp

Basado en la documentación oficial de NetApp ONTAP REST API:
- [QoS Policy Resource](https://library.netapp.com/ecmdocs/ECMLP3351667/html/resources/qos_policy.html)

## Estructura de Archivos

```
aqos_policies/
├── aqos_policies.py      # Script principal
├── config.yaml           # Archivo de configuración
├── README.md            # Esta documentación
└── logs/                # Directorio de logs (generado automáticamente)
    ├── aqos_policy_created_*.json
    └── event_logs_*.json
```

## Autor

NetApp ONTAP Automation

## Versión

1.0.0