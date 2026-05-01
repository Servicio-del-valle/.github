# IEPS – Sistema de Gestión de CFDIs

Sistema para descarga, almacenamiento y consulta de Comprobantes Fiscales Digitales por Internet (CFDIs) desde el SAT (Servicio de Administración Tributaria de México).

---

## 📐 Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                      Usuario (Navegador)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────────┐
│                  IEPS (Frontend MVC .NET)                   │
│  Login · Razones Sociales · CFDIs · Anexo4 · Usuarios       │
└──────────────────────────┬──────────────────────────────────┘
                           │ REST API + JWT
┌──────────────────────────▼──────────────────────────────────┐
│                  IEPS.API (Backend .NET)                    │
│  Auth · RazonSocial · Cfdi · Anexo4 · Usuarios · Perfiles   │
│                    ↓ Al crear RS                            │
│         Encola 13 meses en DescargaQueue (26 jobs)          │
│                    ↓ Al actualizar RFC/.cer/.key/pass        │
│         Resetea jobs Error → Pendiente (reintento auto)     │
└──────────────────────────┬──────────────────────────────────┘
                           │ SQL Server (100.68.142.117\SQLEXPRESS · DB: Hmg)
┌──────────────────────────▼──────────────────────────────────┐
│         FiscalApi.Worker – 5 Shards en Paralelo             │
│                                                             │
│  Shard 1 (A-F) · Shard 2 (G-L) · Shard 3 (M-R)            │
│  Shard 4 (S-Z) · Shard 5 (Overflow)                        │
│                                                             │
│  Cada shard: hasta 8 jobs en paralelo = 40 descargas        │
│  concurrentes · pasó de horas a minutos                     │
└──────┬─────────────────────────────────────┬────────────────┘
       │                                     │
       ▼                                     ▼
FiscalApi.Core.dll                FiscalApi.PdfGenerator.dll
(Descarga XMLs del SAT)           (Genera PDFs desde XML)
(invocado como: dotnet <dll>)     (invocado como: dotnet <dll>)
```

---

## 📦 Proyectos

| Proyecto | Tipo | Puerto / Ruta |
|---|---|---|
| `IEPS` | MVC Frontend | https://localhost:xxxx |
| `IEPS.API` | REST API | https://localhost:xxxx/api |
| `FiscalApi.Worker` | Windows Service (5 shards) | `C:\FiscalApi\Worker.Shard{1-5}` |
| `FiscalApi.Core` | Console App | `C:\FiscalApi\shared-binaries\FiscalApi.Core.dll` |
| `FiscalApi.PdfGenerator` | Console App | `C:\FiscalApi\shared-binaries\FiscalApi.PdfGenerator.dll` |
| `FiscalApi.XmlCheck` | Console App | Herramienta auxiliar de validación XML |

---

## 🖥️ IEPS – Frontend MVC

Interfaz web para el usuario final. Requiere autenticación JWT.

### Vistas disponibles

| Ruta | Descripción |
|---|---|
| `/Auth` | Login (degradado azul corporativo `#005eb5`) |
| `/Home` | Dashboard con KPIs, estado del Worker, gráficas y tabla de errores |
| `/RazonSocial` | CRUD de razones sociales |
| `/Cfdi` | Consulta de CFDIs con filtros y visor de PDF |
| `/Anexo4` | Generador de reporte Anexo 4 por trimestre |
| `/Usuarios` | Administración de usuarios |
| `/Perfiles` | Roles y permisos |
| `/Regiones` | Catálogo de regiones |

---

## ⚙️ IEPS.API – Backend REST

Expone los endpoints que consume el frontend. Autenticación via JWT Bearer.

### Endpoints principales

| Controlador | Método | Ruta | Descripción |
|---|---|---|---|
| Auth | POST | `/api/auth/Authenticate` | Login → retorna JWT |
| RazonSocial | GET | `/api/razonsocial/ListRazonesSociales/{IdCliente}` | Lista RS activas |
| RazonSocial | POST | `/api/razonsocial/Create` | **Crea RS + dispara descarga automática** |
| RazonSocial | PUT | `/api/razonsocial/Update` | Actualiza RS + **resetea jobs en Error si cambian credenciales FIEL** |
| RazonSocial | DELETE | `/api/razonsocial/Delete/{id}` | Baja lógica RS |
| Cfdi | POST | `/api/cfdi/ListCfdis` | Lista CFDIs con filtros |
| Cfdi | GET | `/api/cfdi/Pdf/{uuid}` | Retorna PDF en base64 |
| Anexo4 | GET | `/api/anexo4/ListRazonSociales/{IdCliente}` | RS para Anexo 4 |
| Anexo4 | GET | `/api/anexo4/DescargarAnexo4` | Descarga reporte XLS |
| Menu | GET | `/api/menu/GetMenu` | Menú según perfil del usuario |

### 🔑 Disparador de descarga automática

**Cuando se crea una Razón Social**, el API encola automáticamente la descarga histórica de 13 meses (sin bloquear la respuesta HTTP):

```csharp
// RazonSocialController.cs – POST Create
await iRazonSocial.Create(dto, userId);       // 1. Guarda RS en BD

_ = Task.Run(async () =>                       // 2. Fire-and-forget (no bloquea)
{
    using var scope  = _scopeFactory.CreateScope();
    var queueService = scope.ServiceProvider.GetRequiredService<IDescargaQueueService>();
    await queueService.EnqueueAsync(dto.RFC);  // 3. Inserta 13 meses en DescargaQueue
                                               //    (Recibido + Emitido = 26 jobs)
});
```

El método `EnqueueAsync(RFC)` inserta en `DescargaQueue` los 13 meses anteriores al mes actual, para `Recibido` y `Emitido`, dando prioridad a los meses más recientes.

---

## 🤖 FiscalApi.Worker – Servicio de Fondo (Arquitectura Sharded)

Windows Service que orquesta toda la automatización de descargas SAT. Opera como **5 instancias independientes en paralelo (shards)**, cada una responsable de un rango de RFCs, logrando un rendimiento que pasó de **horas a minutos** para cientos de razones sociales.

### ¿Por qué Shards?

La versión anterior era un único servicio con 2-3 jobs en paralelo. Con un despacho de ~500 razones sociales descargando hasta 26 meses (Recibido + Emitido), la cola tardaba horas en procesarse. La solución fue **particionar** la cola por la primera letra del RFC y correr 5 instancias simultáneas con hasta 8 jobs en paralelo cada una, resultando en hasta **40 descargas concurrentes**.

### Partición por RFC (Shards)

| Shard | Rango de letras | Servicio Windows |
|---|---|---|
| **Shard 1** | A – F | `FiscalApi.Worker.Shard1` |
| **Shard 2** | G – L | `FiscalApi.Worker.Shard2` |
| **Shard 3** | M – R | `FiscalApi.Worker.Shard3` |
| **Shard 4** | S – Z | `FiscalApi.Worker.Shard4` |
| **Shard 5** | Overflow / carga pesada | `FiscalApi.Worker.Shard5` |

Cada job en `DescargaQueue` tiene la columna `Shard` (1–5) calculada al momento de encolarse. El Worker solo toma jobs cuyo `Shard` coincida con su configuración, garantizando que **nunca dos instancias procesan el mismo RFC al mismo tiempo**.

### Instalación como servicio (5 instancias)

```bat
REM Cada shard es un servicio Windows independiente
sc create "FiscalApi.Worker.Shard1" binPath= "C:\FiscalApi\Worker.Shard1\FiscalApi.Worker.exe" start= auto
sc create "FiscalApi.Worker.Shard2" binPath= "C:\FiscalApi\Worker.Shard2\FiscalApi.Worker.exe" start= auto
sc create "FiscalApi.Worker.Shard3" binPath= "C:\FiscalApi\Worker.Shard3\FiscalApi.Worker.exe" start= auto
sc create "FiscalApi.Worker.Shard4" binPath= "C:\FiscalApi\Worker.Shard4\FiscalApi.Worker.exe" start= auto
sc create "FiscalApi.Worker.Shard5" binPath= "C:\FiscalApi\Worker.Shard5\FiscalApi.Worker.exe" start= auto

net start "FiscalApi.Worker.Shard1"
net start "FiscalApi.Worker.Shard2"
net start "FiscalApi.Worker.Shard3"
net start "FiscalApi.Worker.Shard4"
net start "FiscalApi.Worker.Shard5"
```

### Estructura en servidor

```
C:\FiscalApi\
├── Worker.Shard1\          ← Binarios + appsettings con Shard=1
├── Worker.Shard2\          ← Binarios + appsettings con Shard=2
├── Worker.Shard3\          ← Binarios + appsettings con Shard=3
├── Worker.Shard4\          ← Binarios + appsettings con Shard=4
├── Worker.Shard5\          ← Binarios + appsettings con Shard=5
├── shared-binaries\        ← Fuente de binarios para actualizar shards
└── logs\
    ├── Shard1\             ← Logs de orquestación del Shard 1
    │   └── jobs\{RFC}\     ← Logs individuales por job
    ├── Shard2\
    ...
```

### Configuración por shard (appsettings.json)

```json
{
  "ConnectionStrings": {
    "Hmg": "data source=100.68.142.117\\SQLEXPRESS;Database=Hmg;..."
  },
  "Worker": {
    "Shard": 1,
    "RutaDll": "C:\\FiscalApi\\shared-binaries\\FiscalApi.Core.dll",
    "RutaPdfGeneratorDll": "C:\\FiscalApi\\shared-binaries\\FiscalApi.PdfGenerator.dll",
    "PollingIntervalSeconds": "30",
    "MaxJobsEnParalelo": "8"
  },
  "IEPSApi": {
    "Url": "https://localhost/api"
  }
}
```

> **Nota:** El Worker invoca `FiscalApi.Core` y `FiscalApi.PdfGenerator` como `dotnet <dll>` (no `.exe`) para evitar bloqueos de Windows Application Control (Device Guard / WDAC) sobre ejecutables no firmados.

### Logs – Dos niveles

El Worker genera dos tipos de logs:

1. **Log de orquestación** (`logs/ShardN/worker-YYYYMMDD.log`): arranque, polling, errores del propio Worker. Retención de 14 días (Serilog rolling).

2. **Log por job** (`logs/ShardN/jobs/{RFC}/{YYYY-MM}-{TipoDescarga}-{JobId}.log`): todo el stdout/stderr de `FiscalApi.Core` y `FiscalApi.PdfGenerator` de ese job específico. Permite diagnosticar errores de un RFC sin buscar en un archivo gigante mezclado con todos los demás.

---

### 🔁 Proceso 1 – Worker (Cola Principal)

Monitorea la tabla `DescargaQueue` y ejecuta las descargas del shard asignado.

```
Al arrancar:
  1. Resetea jobs "EnProceso" huérfanos de este shard → Pendiente
     (recuperación ante crash del proceso anterior)
  2. Por cada RFC en cola del shard: FiscalApi.Core.dll sync {RFC}
     → Escanea disco, inserta ZIPs no registrados en BD

Loop principal (cada 30 segundos, máx 8 jobs en paralelo por shard):
  ┌─ Espera slot libre (SemaphoreSlim)
  ├─ TakeNextAsync() → UPDATE atómico: Pendiente → EnProceso
  │   Prioridad: jobs manuales (EsManual=1) primero, luego por FechaCreacion
  │   (también procesa EnEsperaSAT con ProximoIntento vencido)
  │
  └─ EjecutarJobAsync():
       dotnet FiscalApi.Core.dll {Anio} {Mes:D2} {RFC} --tipo Recibido|Emitido [--force]
       Escribe stdout/stderr al log del job (logs/ShardN/jobs/{RFC}/...)
       ↓
       ExitCode = 0 ?
         ├─ Sí → ¿Job sigue EnProceso?
         │         Sí → MarcarCompletado()
         │              + await EjecutarPdfGeneratorAsync()   ← genera y guarda PDF en BD
         │         No → FiscalApi.Core lo cambió internamente (ej: EnEsperaSAT)
         └─ No → MarcarError(exitCode)
```

> **Descarga manual**: cuando el usuario solicita una descarga manual desde la UI (rango de fechas específico), el job se encola con `EsManual=1` y tiene prioridad sobre los jobs nocturnos. El Worker y FiscalApi.Core detectan este flag para saltarse los early-exits (no omiten la solicitud al SAT aunque ya existan ZIPs en disco).

#### Tabla de estatus DescargaQueue

| Estatus | Significado |
|---|---|
| `Pendiente` | En espera de ser procesado |
| `EnProceso` | FiscalApi.Core corriendo ahora |
| `EnEsperaSAT` | SAT aún no tiene la solicitud lista, reintento programado |
| `Completado` | Descarga exitosa + PDF generado |
| `CompletadoVacío` | SAT confirmó que el período no tiene facturas (0 CFDIs vigentes) |
| `Error` | Falló (ver columna `MensajeError` con descripción legible) |

#### Errores SAT detectados automáticamente

El Worker clasifica los errores del SAT y los almacena en `MensajeError` con texto legible para el usuario:

| Código SAT | Condición | Mensaje al usuario |
|---|---|---|
| `301` | RFC inválido (RfcEmisor/RfcReceptor) | "El RFC o contraseña son incorrectos. Verifique su información." |
| `304` | Certificado Revocado o Caduco | "El certificado FIEL está revocado o caducado. Renueve su FIEL en el SAT." |
| — | Error de certificado/firma | "Error de certificado FIEL. Verifique que .cer y .key correspondan al RFC." |
| `301` | XML Mal Formado | "Solicitud rechazada por el SAT (XML Mal Formado). Verifique las credenciales FIEL." |
| — | Sin RequestId / sin mensaje | "SAT no devolvió RequestId. Verifique credenciales FIEL y que el RFC esté habilitado." |

#### Reintento automático al actualizar credenciales FIEL

Cuando el usuario corrige el RFC, los archivos `.cer`/`.key` o la contraseña (`KeyPass`) de una Razón Social en la aplicación, el API **detecta automáticamente el cambio** y resetea todos los jobs en estado `Error` de ese RFC a `Pendiente`. El Worker los retoma en el siguiente ciclo de polling (~30 segundos), sin necesidad de esperar al proceso nocturno de las 2 AM.

```
Usuario edita RS → cambia .cer o .key o KeyPass
  → API detecta cambio comparando valor viejo vs nuevo
  → UPDATE DescargaQueue SET Estatus='Pendiente' WHERE RFC=? AND Estatus='Error'
  → Worker retoma en ~30 segundos
```

Si solo se cambian datos no relacionados con la autenticación SAT (domicilio, teléfono, etc.), **no se dispara ningún reintento**.

---

### 📅 Proceso 2 – MonthlyQueueScheduler (Detección de Facturas Nuevas)

Re-encola **TODOS** los jobs del mes anterior el **día 2 de cada mes** a las 02:05 AM para detectar facturas que llegaron después de la descarga inicial del día 1 (ej: facturas emitidas a las 5 PM cuando el scheduler ya pasó a las 2 AM).

```
Horario: Día 2 de cada mes a las 02:05 AM

Flujo:
  Si hoy.Day == 2:
    mesAnterior = hoy - 1 mes
    UPDATE DescargaQueue
    SET Estatus = 'Pendiente',
        EsActualizacion = 1,      ← Fuerza --force (borra ZIPs viejos, re-descarga)
        FechaInicio = NULL,
        RequestId = NULL,
        ...
    WHERE Anio = mesAnterior.Año
      AND Mes = mesAnterior.Mes
      AND Estatus IN ('Completado', 'Pendiente', 'Error', 'EnEsperaSAT')  ← TODOS

  Si hoy.Day >= 10 AND hoy.Day != 1 AND hoy.Day != 2:
    Por cada RFC activo (Cat_RazonSocial WHERE IdEstatus = 1):
      ResetOrInsertMesActualAsync(RFC, año_actual, mes_actual, "Recibido")
      ResetOrInsertMesActualAsync(RFC, año_actual, mes_actual, "Emitido")
      
      ┌─ Job no existe → INSERT Pendiente (EsActualizacion = 1)
      ├─ Job Completado → RESET a Pendiente (se re-descarga)
      ├─ Job Error → RESET a Pendiente
      ├─ Job Pendiente → Se omite (ya está en cola)
      └─ Job EnProceso/EnEsperaSAT → Se omite (se está ejecutando)
```

**¿Por qué día 2 en lugar de día 1?**
- Día 1 es para resetear el mes anterior (al fin del día anterior puede haber facturas pendientes de SAT)
- Día 2 garantiza que el mes anterior ya está completo en el cierre diario, capturando cualquier factura que se haya emitido entre las 2 AM del día anterior y la medianoche
- `EsActualizacion = 1` activa el modo `--force` en FiscalApi.Core, borrando ZIPs antiguos y forzando re-descarga para detectar cambios/correcciones del SAT

**Diferencia clave: Resetea TODOS, no solo Error**
- **Antes**: Solo reseteaba jobs con `Estatus = 'Error'`
- **Ahora**: Resetea `Completado`, `Pendiente`, `Error`, `EnEsperaSAT` para asegurar que **siempre se re-descarga** el mes anterior y se detectan facturas nuevas o corregidas por el SAT

---

### 🔍 Proceso 3 – WeeklyMetadataScheduler

Verifica con el SAT si algún CFDI fue cancelado.

```
Horario: Cada domingo a las 03:00 AM

Por cada RFC activo:
  FiscalApi.Core.exe metadata {RFC}
  (pausa 10 segundos entre RFC y RFC para no saturar el SAT)

FiscalApi.Core:
  → Consulta SAT: metadatos de los últimos 13 meses (Recibido + Emitido)
  → Actualiza columna EstatusSAT en tabla CFDIs
  → Si detecta un CFDI cancelado que ya tiene PDF:
       1. Elimina PdfBytes y RutaPdf en BD
       2. Borra el archivo .pdf del disco
       3. El PdfGenerator lo regenerará con sello CANCELADO en rojo
  → Si detecta CFDIs cancelados → envía email de alerta al despacho
```

---

## 📥 FiscalApi.Core – Descargador SAT

Consola que se comunica directamente con el SAT para descargar CFDIs.

### Modos de uso

```bash
# Descarga normal (invocado por el Worker)
FiscalApi.Core.exe {YYYY} {MM} {RFC} --tipo Recibido|Emitido
FiscalApi.Core.exe {YYYY} {MM} {RFC} --tipo Recibido|Emitido --force   # Re-descarga forzada

# Sincronización de disco a BD (al arrancar el Worker)
FiscalApi.Core.exe sync {RFC}

# Verificación de cancelaciones (semanal)
FiscalApi.Core.exe metadata {RFC}

# Prueba de email
FiscalApi.Core.exe test-email {RFC}
```

### Flujo de descarga

```
1. Autentica con SAT usando FIEL (archivos .CER / .KEY)
2. Solicita paquete de descarga (RequestId)
3. Verifica estatus del paquete (polling hasta que SAT lo prepare)
   → Si SAT no responde: marca job EnEsperaSAT + ProximoIntento = +30 minutos
      → Cada 5 intentos acumulados: RequestId=NULL + Pendiente + espera 1h (nueva solicitud al SAT)
      → Tras 15 intentos acumulados (3 ciclos × 5): marca Error definitivo
4. Descarga ZIP con XMLs
5. Extrae XMLs → parsea cada CFDI
6. Inserta/actualiza registros en tabla CFDIs (con XmlCFDI, UUID, RFC, fechas, totales, etc.)
7. Guarda ZIP en disco: C:\Sistema\Facturas\{RFC}\{AnioMes}\ (Recibidos)
                        C:\Sistema\Facturas\{RFC}\Emitido\{AnioMes}\ (Emitidos)
```

### Validaciones internas

- **Expiración de token**: renueva antes de continuar
- **Cache de Metadata SAT**: antes de autenticar con SAT, consulta `DescargaQueue.FacturasMetadata` y `FechaMetadata` para evitar solicitudes innecesarias al SAT:
  - Si `enBD >= FacturasMetadata` Y es descarga automática → período completo, skip inmediato (nunca re-verifica)
  - Si es descarga manual → **SIEMPRE procede**, ignorando el conteo cacheado (detecta nuevas facturas en rangos de fechas específicas)
  - Si `enBD < FacturasMetadata` y cache no ha expirado → usa el conteo cacheado, procede con descarga CFDI sin llamar a SAT para metadata
  - Si cache expirado → re-ejecuta pre-check de metadata antes de continuar
- **Intervalo de cache dinámico**:
  - Mes actual, días 20–31 → 1 día (re-verifica diario en cierre de mes, cuando más facturas llegan)
  - Mes actual, días 1–19 → 3 días
  - Meses históricos → 3 días
- **Pre-check de Metadata SAT**: justo después de autenticar, consulta SAT con `QueryType.Metadata` para obtener el conteo de CFDIs del período antes de lanzar la solicitud de descarga completa (más lenta y costosa en cuota SAT):
  - `NumeroCFDIs = 0` → marca job `CompletadoVacío`, no consume cuota de descarga
  - `NumeroCFDIs > enBD` → faltan facturas, procede con descarga CFDI
  - `NumeroCFDIs <= enBD` Y es automática → período ya completo, skip
  - `NumeroCFDIs <= enBD` Y es manual → **procede igual**, permite re-validar manualmente
  - Error/timeout del pre-check → **fail-open**: procede con descarga CFDI sin bloquear
- **Reintento con `--force`**: borra ZIPs existentes y re-descarga desde cero. Activado cuando:
  - Usuario solicita re-descarga manual desde UI (`--force` explícito)
  - MonthlyScheduler resetea job con `EsActualizacion = 1` (detección de facturas nuevas del mes anterior)
- **Early-exit por ZIP en disco**: si el ZIP ya existe y `--force` no está activo, lo salta (protege históricos)
- **RequestId vacío**: si el SAT devuelve `RequestId=""`, reintenta hasta 5 veces con ventana de tiempo ligeramente desplazada (offsets en segundos) antes de marcar Error
- **RequestId vacío en BD**: si el Worker retoma un job con `RequestId=""` guardado, se trata como null y crea una nueva solicitud al SAT
- **EnEsperaSAT con ciclos**: cada 5 intentos sin respuesta del SAT se descarta el RequestId y se pide solicitud nueva; tras 15 intentos acumulados (3 solicitudes distintas, ~7.5h) marca Error definitivo
- **Generación automática de PDFs**: tras procesar exitosamente todos los XMLs de un mes, invoca automáticamente `FiscalApi.PdfGenerator` para generar y guardar PDFs de los CFDIs descargados (no requiere ejecución manual posterior)

---

---

## 📄 FiscalApi.PdfGenerator – Generador de PDFs

Genera la representación impresa (PDF) de cada CFDI descargado.

### Modos de uso

```bash
# Genera PDFs de un mes específico (invocado por el Worker tras cada descarga exitosa)
FiscalApi.PdfGenerator.exe {RFC} {YYYY-MM} {TipoDescarga}

# Genera PDFs de todos los CFDIs sin PDF (retroactivo)
FiscalApi.PdfGenerator.exe --all

# Genera un PDF de ejemplo sin necesidad de BD (para pruebas de diseño)
FiscalApi.PdfGenerator.exe --test
```

### Flujo

```
1. Consulta CFDIs sin PDF (PdfBytes IS NULL) del RFC/mes/tipo dado
2. Por cada CFDI:
   a. Parsea el XML (Emisor, Receptor, Conceptos, Impuestos, TimbreFiscalDigital)
   b. Genera QR de verificación SAT
   c. Construye PDF con QuestPDF (formato representación impresa)
   d. Guarda en disco: C:\Sistema\Facturas\{RFC}\{AnioMes}\{UUID}.pdf
   e. Actualiza BD: columnas RutaPdf y PdfBytes en tabla CFDIs
```

### Estructura del PDF generado

- Datos del emisor (nombre, RFC, régimen fiscal, lugar de expedición)
- Tipo de comprobante + Serie/Folio + Fechas
- UUID (Folio Fiscal)
- Datos del receptor
- Tabla de conceptos
- Totales (Subtotal, IVA/Impuestos, Total)
- Sellos digitales (CFDI + SAT) y Cadena Original
- Código QR de verificación SAT
- **Sello CANCELADO** (banner rojo al inicio) cuando `EstatusSAT = 'Cancelado'`, con fecha de cancelación

---

## 🗄️ Base de Datos

**Servidor**: `100.68.142.117\SQLEXPRESS`  
**Base de datos**: `Hmg`

### Tablas principales

| Tabla | Descripción |
|---|---|
| `Cat_RazonSocial` | Razones sociales (RFC, domicilio fiscal, certificados CSD) |
| `CFDIs` | Todos los comprobantes descargados (XML, PDF, estatus SAT) |
| `DescargaQueue` | Cola de trabajos del Worker |
| `SendMail` | Configuración SMTP para notificaciones por email |
| `Cat_Clientes` | Clientes / unidades del sistema |
| `Cat_Usuarios` | Usuarios del sistema |
| `Cat_Perfiles` | Roles y permisos |

### Vistas principales

| Vista | Uso |
|---|---|
| `VRazonSocialCodigoPostales` | RS con datos de domicilio completo |
| `VUsuariosCliente` | Relación usuarios-clientes |

### Columnas clave de CFDIs

| Columna | Descripción |
|---|---|
| `UUID` | Folio fiscal del CFDI |
| `XmlCFDI` | XML original descargado del SAT |
| `PdfBytes` | PDF generado por FiscalApi.PdfGenerator |
| `RutaPdf` | Ruta en disco del PDF |
| `EstatusSAT` | Vigente / Cancelado (actualizado semanalmente) |
| `FechaCancelacion` | Fecha en que el SAT lo canceló (si aplica) |
| `FechaVerificacion` | Última vez que se verificó el estatus con el SAT |
| `AnioMes` | Período de descarga (ej: `2025-03`) |
| `TipoDescarga` | `Recibido` o `Emitido` |

### Columnas clave de DescargaQueue

| Columna | Descripción |
|---|---|
| `RFC` | RFC de la Razón Social |
| `Anio` / `Mes` | Período del job |
| `TipoDescarga` | `Recibido` o `Emitido` |
| `Estatus` | Estado actual del job (ver tabla de estatus) |
| `Shard` | Shard asignado (1–5, calculado por primera letra del RFC) |
| `FacturasMetadata` | Conteo de CFDIs reportado por SAT en el último pre-check de metadata |
| `FacturasDescargadas` | Conteo de CFDIs efectivamente descargados |
| `FechaMetadata` | Fecha en que se ejecutó el último pre-check de metadata (para cache) |
| `MensajeError` | Descripción legible del error (si `Estatus = 'Error'`) |
| `RequestId` | ID de solicitud SAT activo (para retomar jobs `EnEsperaSAT`) |
| `ProximoIntento` | DateTime del siguiente intento para jobs `EnEsperaSAT` |
| `EsManual` | `1` si fue solicitado desde la UI (prioridad sobre jobs nocturnos) |

---

## 🚀 Despliegue

### Worker + Core (servidor)

```powershell
# 1. Publicar Worker
cd FiscalApi.Worker
dotnet publish -c Release -r win-x64 --self-contained false -o .\publish

# 2. En el servidor: detener shards
FOR /L %i IN (1,1,5) DO net stop "FiscalApi.Worker.Shard%i"

# 3. Limpiar y reemplazar shared-binaries (eliminar antes para evitar DLLs obsoletos)
Remove-Item C:\FiscalApi\shared-binaries\* -Recurse -Force
# Copiar todo el publish al servidor → C:\FiscalApi\shared-binaries\

# 4. Copiar appsettings por shard
FOR /L %i IN (1,1,5) DO (
    copy C:\FiscalApi\configs\appsettings.Shard%i.json C:\FiscalApi\Worker.Shard%i\appsettings.json
)

# 5. Reiniciar shards
FOR /L %i IN (1,1,5) DO net start "FiscalApi.Worker.Shard%i"
```

> ⚠️ `Fiscalapi.Credentials.dll` es una versión **parchada localmente** (fix para razones sociales con comas en el nombre). Al hacer deploy, copiar la DLL compilada desde `fiscalapi-credentials-net\bin\Release\net8.0\Fiscalapi.Credentials.dll` a `shared-binaries\`, sobreescribiendo la que viene del NuGet.

> ⚠️ El frontend (`IEPS`) y el API (`IEPS.API`) se despliegan por separado (IIS / Visual Studio Publish).

---

## 📧 Notificaciones por Email

Cuando el verificador semanal detecta CFDIs cancelados, envía un correo automático con:
- Lista de CFDIs cancelados (UUID, emisor, receptor, total, fecha de cancelación)
- Configurado en tabla `SendMail` (Proyecto = `'IEPS'`)
- SMTP: Gmail con App Password (`smtp.gmail.com:587`)

Para probar el envío de email sin correr el flujo completo:
```bash
FiscalApi.Core.exe test-email {RFC}
```

---

## 🔧 Tecnologías

| Componente | Tecnología |
|---|---|
| Frontend | ASP.NET Core MVC (.NET 10) |
| Backend API | ASP.NET Core Web API (.NET 10) |
| Worker / Core | .NET 8 (BackgroundService / Console) |
| PDF | QuestPDF 2024.10.4 + QRCoder |
| ORM | Entity Framework Core |
| Base de datos | SQL Server Express |
| Autenticación | JWT Bearer |
| Grid UI | Grid.js |
| CSS Framework | Bootstrap 5.3 + Nifty Theme |
| Logging (Worker) | Serilog (consola + archivo rolling) |
| Excel (Anexo 4) | NPOI 2.7.x (HSSF, formato `.xls`) |

---

## 📊 Anexo 4 – Reporte IEPS Diésel

El Anexo 4 es el formato oficial del SAT para declarar el crédito fiscal del IEPS por consumo de diésel agropecuario. El sistema genera automáticamente el archivo `.xls` oficial llenado con los datos de los CFDIs recibidos del período seleccionado.

### ¿Cómo funciona?

El endpoint `GET /api/anexo4/DescargarAnexo4?idRazonSocial={id}&anio={year}&trimestre={1-4}` carga la plantilla oficial `ANEXO4.xls` (NPOI/HSSF) y la llena con los CFDIs que cumplan las condiciones del período.

---

### 🗂️ Estructura de la plantilla

El archivo `ANEXO4.xls` contiene 5 hojas:

| Hoja | Descripción |
|---|---|
| `Instructivo de Llenado` | Hoja de instrucciones — se omite, no se escribe |
| `Anexo 4` | Datos del trimestre (hoja 1 de hasta 4) |
| `Anexo 4 (2)` | Continuación si hay más de 8 facturas |
| `Anexo 4 (3)` | Continuación si hay más de 16 facturas |
| `Anexo 4 (4)` | Continuación si hay más de 24 facturas |

Cada hoja de datos tiene **8 slots** (filas de datos), por lo que el reporte soporta un máximo de **32 facturas por trimestre**.

---

### 📐 Celdas que se llenan (índices 0-based)

#### Por cada hoja de datos activa

| Celda (fila, col) | Nombre en plantilla | Contenido |
|---|---|---|
| (6, 1) | Fila 7, col B | RFC de la Razón Social |
| (8, 15) | Fila 9, col P | Número de hoja actual (ej. `1`) |
| (8, 17) | Fila 9, col R | Total de hojas usadas (ej. `3`) |

#### Por cada slot de factura (8 por hoja)

Los slots están en las filas 13, 19, 25, 31, 37, 43, 49 y 55 (`rfcRow = 12 + sIdx × 6`).  
La fila de fecha es `rfcRow + 3` (filas 16, 22, 28, 34, 40, 46, 52, 58).

| Celda relativa | Col | Contenido |
|---|---|---|
| (rfcRow, 1) | B | RFC del emisor (proveedor de diésel) |
| (rfcRow, 15) | P–S (merged) | Folio de la factura (solo dígitos, ej. `239338`) |
| (fechaRow, 3) | D | Día de la fecha de emisión |
| (fechaRow, 5) | F | Mes de la fecha de emisión |
| (fechaRow, 7) | H | Año de la fecha de emisión |
| (fechaRow, 9) | J | Monto total de la factura (redondeado) |
| (fechaRow, 17) | R | IEPS calculado (`min(14,948, round(monto × 0.355))`) |

#### Totales por hoja

| Celda | Contenido |
|---|---|
| (60, 9) — Fila 61, col J | Suma de montos de las facturas de esta hoja |
| (60, 17) — Fila 61, col R | Suma de IEPS de las facturas de esta hoja |
| (64, 9) — Fila 65, col J | Misma suma de montos (fila de doble captura) |
| (64, 17) — Fila 65, col R | Misma suma de IEPS |

#### Datos del tractor (fijos por hoja)

| Celda | Contenido |
|---|---|
| (70, 4) — Fila 71, col E | Clave del tractor (`Cat_Tractores.Clave`) |
| (70, 12) — Fila 71, col M | RFC del representante legal (`Cat_RazonSocial.RFCRepresentanteLegal`) |
| (70, 18) — Fila 71, col S | Número de factura del tractor (`Cat_Tractores.FacturaTractor`) |
| (73, 3) — Fila 74, col D | Día de la fecha de la factura del tractor |
| (73, 5) — Fila 74, col F | Mes de la fecha de la factura del tractor |
| (73, 7) — Fila 74, col H | Año de la fecha de la factura del tractor |
| (73, 12) — Fila 74, col M | Nombre del tractor (`Cat_Tractores.Nombre`) |

#### Datos del solicitante (fila 124, solo en primera hoja o todas)

| Celda | Contenido |
|---|---|
| (123, 12) — Fila 124, col M | Número de socios de la persona moral (`Cat_RazonSocial.NoSocios`) |

---

### ✅ Condiciones de filtrado de CFDIs

Para que una factura sea incluida en el Anexo 4, debe cumplir **todas** las siguientes condiciones:

1. **Tipo de descarga**: `TipoDescarga = 'Recibido'` (solo facturas recibidas).
2. **RFC receptor**: coincide con el RFC de la Razón Social seleccionada.
3. **Período**: `AnioMes` cae dentro del trimestre seleccionado (ej. trimestre 1 → meses 1, 2, 3 del año dado).
4. **Concepto de diésel**: el XML contiene al menos un concepto cuya descripción incluye la palabra `diesel` (búsqueda insensible a mayúsculas, sin tilde).
5. **Impuesto IEPS presente**: el CFDI tiene al menos un traslado de impuesto de tipo `003` (IEPS).

---

### 🔢 Cálculo del IEPS

```
montoRedondeado = Round(Total del CFDI, 0)   // sin decimales, AwayFromZero
                                              // ej: 42,155.36 → 42,155
                                              // ej: 42,155.50 → 42,156

ieps = Min(14,948, Round(montoRedondeado × 0.355, 0))
```

El tope máximo de IEPS por factura es **$14,948**.  
La tasa aplicada es **35.5%** del monto total de la factura redondeado.

---

### 📋 Requisitos en catálogos antes de generar

Para que el Anexo 4 se genere correctamente, los siguientes datos deben estar capturados en el sistema:

| Dato | Dónde capturarlo | Obligatorio |
|---|---|---|
| RFC del representante legal | Razón Social → campo "RFC Rep. Legal" | Sí (fila 71) |
| Nombre del representante legal | Razón Social → campo "Nombre(s) RL" | No |
| Número de socios (personas morales) | Razón Social → campo "No. de Socios" | Recomendado (fila 124) |
| Clave del tractor | Tractores → campo "Clave" | Sí (fila 71) |
| Nombre del tractor | Tractores → campo "Nombre" | Sí (fila 74) |
| Número de factura del tractor | Tractores → campo "Factura Tractor" | Sí (fila 71) |
| Fecha de la factura del tractor | Tractores → campo "Fecha" | Sí (fila 74) |

> El sistema toma el **primer tractor activo** asociado a la Razón Social (`IdEstatus = 1`, ordenado por `IdTractor`).

---

### 🗓️ Trimestres

| Trimestre | Meses incluidos |
|---|---|
| 1 | Enero, Febrero, Marzo |
| 2 | Abril, Mayo, Junio |
| 3 | Julio, Agosto, Septiembre |
| 4 | Octubre, Noviembre, Diciembre |

---

### 📁 Plantilla

La plantilla oficial debe estar en:
```
IEPS.API/IEPS.API/Templates/ANEXO4.xls
```
Este archivo **no se incluye en el repositorio** (es el formato oficial del SAT). Debe copiarse manualmente al servidor antes de publicar.
