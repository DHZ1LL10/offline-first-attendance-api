# offline-first-attendance-api

<div align="center">

**Módulo de Asistencia (Reloj Checador) — Contribución de Portafolio**

*Extraído de un sistema POS privado en producción para demostrar arquitectura real*

![Node.js](https://img.shields.io/badge/Node.js-20+-339933?style=flat-square&logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Express](https://img.shields.io/badge/Express-5-000000?style=flat-square&logo=express&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-6-2D3748?style=flat-square&logo=prisma&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8-4479A1?style=flat-square&logo=mysql&logoColor=white)

</div>

---

## Contexto: ¿De dónde viene este código?

Soy parte del equipo que desarrolla un **sistema de punto de venta (POS) para restaurantes** compuesto por:

| Componente | Stack | Mi rol |
|---|---|---|
| API REST central | Node.js + Express 5 + Prisma + MySQL | ✅ **Aquí trabajo** |
| Panel web de administración | React 19 + React Router v7 SSR | Equipo frontend |
| App de caja (TPV) | Flutter + BLoC + Isar | Equipo móvil |
| Dashboard de reportes | Flutter + Riverpod | Equipo móvil |

Este repositorio es **mi aportación específica al backend**: el módulo de control de asistencia que diseñé e implementé de principio a fin. El sistema real es privado, así que extraje la lógica en un microservicio público para mostrarlo en mi portafolio, sanitizando los datos del cliente.

> **¿Por qué importa el contexto?**
> Trabajar en un ecosistema de 4 aplicaciones conectadas te obliga a tomar decisiones de diseño que van más allá del código — tienes que pensar en contratos de API, tolerancia a fallos en el cliente móvil, y cómo el backend puede proteger la integridad de los datos sin importar qué pase en el dispositivo.

---

## El Problema que Me Asignaron

Los restaurantes operan con WiFi inestable. Los empleados fichan entrada desde una tablet (TPV Flutter) en la caja. El requerimiento llegó con tres restricciones técnicas no negociables:

1. **El sistema no puede perder fichajes** aunque la tablet pierda internet por horas
2. **El empleado no puede manipular su hora de entrada** desde el dispositivo
3. **Las fotos de evidencia no deben reventar la base de datos** con el tiempo

Cada restricción llevó a una decisión de arquitectura concreta. Las explico abajo.

---

## Mis Decisiones de Diseño

### 1. El Servidor es la Única Fuente de Verdad del Tiempo

**Problema:** Si la app Flutter genera el timestamp del fichaje, cualquier persona con acceso al dispositivo puede adelantar el reloj del sistema operativo y llegar tarde sin consecuencias.

**Mi decisión:** El backend descarta completamente cualquier campo de tiempo enviado por el cliente. El registro siempre usa `new Date()` del servidor en el instante que llega la petición.

```typescript
// ❌ Esto NUNCA entra al service — el controller no lo pasa
// body.timestamp, body.hora, body.client_time → ignorados en el schema Zod

// ✅ El service solo conoce esto
const serverNow = new Date(); // ← única fuente de verdad
const { delayMinutes, status } = this.calculateDelay(employee, serverNow);
```

El campo `client_timestamp` que acepta el endpoint `/sync` es **exclusivamente para auditoría** — permite ver cuándo ocurrió el evento en el dispositivo, pero nunca entra al motor de cálculo de retardos.

---

### 2. Sincronización Offline-First con Bulk Insert

**Problema:** Si el registro de asistencia depende de conexión, un corte de WiFi de 2 horas significa que todos los empleados "desaparecen" del sistema ese día.

**Mi decisión:** diseñé el contrato de API pensando en el cliente Flutter desde el principio, no como un afterthought.

**Flujo completo:**

```
TABLET (Flutter + Isar)
        │
        ├─ Con internet → POST /api/attendances/check-in
        │                  └─ respuesta inmediata ✅
        │
        └─ Sin internet → DioException capturado
                           ├─ Guardar en Isar: { synced: false }
                           ├─ Mostrar: "Guardado sin conexión" 🟡
                           └─ ConnectivitySubscriber escucha reconexión
                                └─ POST /api/attendances/sync
                                   { records: [ ...todos los pendientes ] }
                                        │
                                        ▼
                               BACKEND (este módulo)
                               • Procesa el array registro por registro
                               • Fallos PARCIALES no abortan el lote
                               • Responde: { success: N, failed: M, errors: [...] }
                                        │
                                        ▼
                               TABLET — post sync
                               • Elimina de Isar solo los exitosos
                               • Reintenta los fallidos en el próximo sync
```

**Por qué respondo siempre 200 en `/sync`:**
Si el servidor responde 4xx o 5xx, el cliente Flutter no sabe qué registros fallaron y cuáles no. Responder 200 con un payload estructurado `{ success, failed, errors }` permite al cliente tomar decisiones granulares: eliminar los exitosos de Isar y conservar solo los fallidos para reintento.

---

### 3. Fotos en Disco, Solo la Ruta en BD

**Problema:** Muchos sistemas almacenan imágenes como Base64 en MySQL. Funciona al principio. Con 50 empleados fichando diario durante un año, la tabla de asistencias se infla con decenas de GB de BLOBs, las queries se vuelven lentas y los backups se eternizan.

**Mi decisión:**

```
Entrada (tiempo real):   multipart/form-data → disk → photo_url en BD
Entrada (sync offline):  base64 en JSON      → decode → disk → photo_url en BD

En base de datos: photo_url = "/media/attendances/3f9a-uuid.jpg"
En disco:         /media/attendances/3f9a-uuid.jpg  (el archivo real)
```

Un script (`cleanup-photos.ts`) corre como cronjob en el servidor y purga archivos con más de N días de antigüedad (configurable en `.env`). La BD siempre es liviana.

---

## Arquitectura del Módulo

```
attendance-module/
├── schema.prisma              ← Modelos: Tenant, Branch, Employee, Attendance
│
├── attendances.schemas.ts     ← Validación Zod
│   ├── checkInSchema          ← employeeId + branchId (SIN timestamp)
│   ├── checkOutSchema
│   ├── syncSchema             ← array de hasta 100 registros
│   └── attendanceQuerySchema  ← filtros de reporte paginado
│
├── attendances.service.ts     ← Motor de reglas de negocio
│   ├── calculateDelay()       ← scheduled_start_time vs new Date() del servidor
│   ├── calculateExtraHours()  ← check_out - check_in vs jornada base
│   ├── savePhoto()            ← buffer → disco → ruta relativa
│   ├── checkIn()
│   ├── checkOut()
│   ├── syncRecords()          ← bulk insert con tolerancia a fallos parciales
│   └── getAttendances()       ← query con filtros + paginación
│
├── attendances.controller.ts
└── attendances.routes.ts      ← Swagger inline en cada endpoint
```

### Modelo de Datos

```prisma
model Employee {
  scheduled_start_time  String?   // "HH:mm" — hora oficial de entrada
  // String sobre Time de MySQL: evita ambigüedad de timezone en cálculos
}

model Attendance {
  check_in_time   DateTime?   // ← siempre new Date() del servidor
  check_out_time  DateTime?   // ← siempre new Date() del servidor
  delay_minutes   Int         // calculado al recibir check-in
  extra_hours     Decimal     // calculado al recibir check-out
  status          ON_TIME | LATE
  photo_url       String?     // ruta en disco, nunca Base64
  synced          Boolean     // false = vino del endpoint /sync
}
```

---

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/api/auth/sign-in` | Login → JWT |
| `GET` | `/api/employees` | Listar empleados del tenant |
| `POST` | `/api/employees` | Crear empleado con PIN + hora oficial de entrada |
| `POST` | `/api/employees/find-by-pin` | Validar PIN (usado por el TPV Flutter) |
| `POST` | `/api/attendances/check-in` | Registrar entrada + foto (multipart) |
| `PUT` | `/api/attendances/check-out` | Registrar salida + calcular horas extra |
| `POST` | `/api/attendances/sync` | **Sincronización offline bulk** |
| `GET` | `/api/attendances` | Reporte filtrado por fecha / empleado / status |

Documentación interactiva completa en `/api-docs` (Swagger UI).

---

## Reglas de Negocio

```
RETARDO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
tolerancia  = ATTENDANCE_LATE_TOLERANCE_MINUTES (default: 10)
limite      = employee.scheduled_start_time + tolerancia
ahora       = new Date()  ← reloj del SERVIDOR

delay_minutes = max(0, floor((ahora − limite) / 60_000))
status        = delay_minutes > 0 ? "LATE" : "ON_TIME"

HORAS EXTRA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
horas_trabajadas = (check_out_time − check_in_time) / 3_600_000
extra_hours      = max(0, horas_trabajadas − 8).toFixed(2)
```

El frontend (panel web React y dashboard Flutter) solo **consume** el `status` y `delay_minutes` pre-calculados. No hace ningún cálculo propio. La fuente de verdad vive 100% en el backend.

---

## Setup

```bash
pnpm install
cp .env.example .env   # configurar DATABASE_URL y JWT_SECRET
docker-compose up -d   # MySQL local
pnpm setup             # migraciones + seed con datos demo
pnpm dev               # http://localhost:4001
```

Variables clave en `.env`:

```env
ATTENDANCE_LATE_TOLERANCE_MINUTES=10   # minutos de gracia antes de marcar retardo
PHOTO_MAX_SIZE_MB=5
PHOTO_RETENTION_DAYS=30                # el cronjob limpia después de este tiempo
```

---

## Progreso

- [ ] Schema Prisma — modelos y relaciones
- [ ] Auth básico (login → JWT de 8h)
- [ ] CRUD Employees con `scheduled_start_time` y PIN
- [ ] `POST /check-in` con foto multipart + motor de retardos
- [ ] `PUT /check-out` + cálculo de horas extra
- [ ] `POST /sync` — bulk insert offline-first
- [ ] `GET /attendances` — reporte paginado con filtros
- [ ] Script `cleanup-photos.ts` para crontab
- [ ] Swagger completo en todos los endpoints

---

<div align="center">

*Este módulo forma parte de un sistema POS privado para restaurantes.*
*El código aquí presente es mi contribución individual, publicada con fines de portafolio.*

</div>
