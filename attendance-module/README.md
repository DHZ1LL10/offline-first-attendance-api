# attendance-module/

Este directorio contiene el módulo de asistencia (reloj checador)
desarrollado como demostración de portafolio.

## Lo que se implementará aquí:

- `schema.prisma`          → Modelos Employee (con hora_oficial_entrada) y Attendance
- `attendances.service.ts` → Motor de reglas: retardos, horas extra, tiempo del servidor
- `attendances.controller.ts`
- `attendances.routes.ts`  → check-in, check-out, /sync (offline-first)
- `attendances.schemas.ts` → Validaciones Zod
- `cleanup-photos.ts`      → Script de purga para crontab

## Próximos pasos

Ver el README principal del proyecto para el plan completo.
