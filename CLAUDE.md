# Órbita CRM

CRM comercial + operativo + supervisión para **Academia INTRA**, integrado con la plataforma educativa **Órbita EDU** vía REST API.

Desarrollo custom, web responsive, deploy en **Railway** (mismo entorno que Órbita EDU). Versión 1.0 Unificada — Mayo 2026.

## Propósito

Gestionar el ciclo comercial completo de captación, seguimiento y conversión de prospectos a alumnos, con un módulo extendido de supervisión, automatización y analítica para dirección comercial.

Usuarios: asesores comerciales, coordinadores, marketing, finanzas, dirección y admin de sistema.

## Stack técnico

| Capa | Tecnología |
|---|---|
| Frontend | Next.js 15 (App Router) |
| UI | Tailwind CSS + shadcn/ui |
| Backend | Node.js + Fastify |
| Base de datos | PostgreSQL 18 (Railway) — compartida o separada de Órbita EDU |
| ORM | Drizzle ORM |
| Autenticación | JWT + SSO con Órbita EDU |
| API interna | REST (OpenAPI 3.1) |
| WhatsApp | Meta Business API (WhatsApp Business API) |
| Deploy | Railway |

## Arquitectura por dominios

El sistema se divide en **2 partes** y **9 bloques funcionales**:

### Parte 1 — Gestión Comercial Base

1. **Gestión de Prospectos y Pipeline** — Ficha de prospecto, etiquetado, filtros, pipeline Kanban con drag & drop, asignación múltiple de asesores, seguimientos ilimitados, duplicados/fusión, historial de cambios, notas internas con menciones.
2. **Actividades, Seguimiento y Comisiones** — Registro de llamadas/visitas/citas/guardias/envíos, recordatorios y alertas de leads fríos, tabulador de comisiones, trackeo individual, reporte de nómina, metas mensuales, calendario unificado.
3. **KPIs, Reportería e Integración con Órbita EDU** — 12 KPIs (conversión, velocidad pipeline, actividad por asesor, fuente, caída, eficiencia de seguimientos, ingresos, ticket promedio, cumplimiento de meta), 6 reportes ejecutivos, endpoints de integración con Órbita EDU.

### Parte 2 — Supervisión Comercial Avanzada

4. **Productividad Comercial — Score de Asesores** — Score semanal ponderado, ranking automático, KPI dinámico configurable, historial de desempeño.
5. **Control Avanzado de Llamadas** — Duración automática, resultado y próxima acción obligatorios, clasificación, grabación opcional, ratio llamada-conversión.
6. **WhatsApp Business API y Automatizaciones** — Integración WA API centralizada, plantillas Meta-aprobadas, etiquetado automático, secuencias programadas, 8 automatizaciones operativas (asignación, recordatorios, reactivación, confirmaciones, escalamiento).
7. **Dashboards Ejecutivos y por Programa** — Dashboard dirección (leads, conversión global, ROI, forecast, pipeline) + dashboard por programa (leads, conversión, costo por inscripción, funnel visual).
8. **Control de Fuga de Leads y Semaforización** — Detección de leads abandonados, alertas, reasignación automática, detección de duplicados, semaforización (Verde/Amarillo/Naranja/Rojo/Azul), auditoría.
9. **Analítica Avanzada e IA Predictiva** (fase futura) — Probabilidad de inscripción, predicción de abandono, leads prioritarios, mejor horario/canal de contacto, forecast financiero.

## Pipeline de prospectos

Etapas configurables por administrador:

```
Nuevo Lead → Contactado → Cita Agendada → Propuesta Enviada → En Negociación → Inscrito
                                                                              ↘ Perdido
```

Cada prospecto se mueve libremente con drag & drop o desde su ficha. Las transiciones quedan auditadas (quién, cuándo, desde/hacia).

## Score Semanal de Asesores

Fórmula ponderada:

| Variable | Peso |
|---|---|
| Inscripciones cerradas | 35% |
| Citas concretadas | 20% |
| Seguimientos realizados | 15% |
| Llamadas realizadas | 15% |
| Tiempo de respuesta | 10% |
| Actualización del CRM | 5% |

Calculado automáticamente cada semana → ranking visible para dirección y coordinación.

## Semaforización de prospectos

| Color | Estado |
|---|---|
| Verde | Lead caliente — actividad reciente, alta probabilidad |
| Amarillo | Seguimiento pendiente — próximo contacto próximo |
| Naranja | Riesgo de pérdida — inactividad detectada |
| Rojo | Lead frío — requiere reactivación urgente |
| Azul | Recontactar después — fecha de seguimiento futura |

## Integración con Órbita EDU (REST API)

| Endpoint | Propósito |
|---|---|
| `POST /api/enroll` | Conversión a inscripción → dispara matrícula automática en Órbita EDU |
| `GET  /api/courses` | Sincronización del catálogo de programas activos |
| `GET  /api/student/:id` | Consulta de progreso académico, asistencia y pagos del alumno |
| `POST /api/webhook/payment` | Órbita EDU notifica al CRM cuando se registra un pago |
| `POST /api/lead` | Ingreso de lead desde landing page pública de Órbita EDU |
| `GET  /api/availability` | Consulta de cupos disponibles por grupo y periodo |

La sincronización es **bidireccional**: el CRM dispara matrículas e ingresos de leads, Órbita EDU notifica pagos y comparte catálogo/cupos/estado del alumno.

## Roles y permisos

| Rol | Acceso |
|---|---|
| **Dirección** | Acceso total. Todos los dashboards, reportes, configuraciones, datos |
| **Coordinador Comercial** | Equipo asignado: leads, scores, actividades, reportes de sus asesores |
| **Asesor / Comisionista** | Solo sus leads, su score, su calendario, su tabulador de comisiones |
| **Marketing** | Métricas de captación, conversión por fuente, ROI, rendimiento por campaña |
| **Finanzas** | Tabulador, nómina, ingresos por programa, estado de pagos |
| **Admin Sistema** | Configuración técnica: integraciones, automatizaciones, roles, pipelines |

Cada asesor ve solo sus prospectos (configurable). El admin ve todos.

## Reglas de negocio clave

- **Resultado obligatorio** al cerrar una llamada (contactado / no contactado). Sin resultado, el registro no se cierra.
- **Próxima acción obligatoria** con fecha al cerrar cualquier registro de llamada/visita/cita. No se permite dejar prospectos sin siguiente paso.
- **Duración automática** de llamadas: evita registro manual y simulación de actividad.
- **Detección de duplicados** automática por email/teléfono con opción de fusión que preserva historial completo.
- **Reasignación automática** de leads que superan umbral crítico de inactividad → otro asesor disponible según regla configurada.
- **Historial inmutable** de cambios por prospecto: cambios de etapa, reasignaciones, ediciones y seguimientos quedan auditados con autor y timestamp.
- **Seguimientos ilimitados** por prospecto, cada uno con: fecha, tipo, asesor, resultado, próximo paso y fecha siguiente.
- **Etapas del pipeline configurables** por el admin del sistema.

## Automatizaciones operativas

| Trigger | Acción |
|---|---|
| Lead nuevo | Asignación automática por turno/carga/regla |
| Sin respuesta en N horas | Recordatorio al asesor |
| Prospecto frío | Campaña de reactivación (WhatsApp/email) |
| Cita agendada | Confirmación + recordatorio 24h antes |
| Pago confirmado | Cambio a "Inscrito" + matrícula en Órbita EDU |
| Lead abandonado | Escalamiento a coordinación |
| Nuevo programa | Campaña automática a leads interesados en área relacionada |
| Sin seguimiento | Alerta visual de riesgo en dashboard |

## Exportación y datos

- Exportación completa de base de prospectos a **CSV / Excel / JSON** sin límite de volumen, con filtros previos aplicables.
- Reportes ejecutivos (PDF / Excel): pipeline, actividad por asesor, comisiones, prospectos perdidos, ingresos por programa.
- Reporte de nómina compatible con sistemas de RRHH externos.

## Convenciones para colaboración con Claude

- **Idioma**: la UI, los datos de negocio y los nombres de dominio están en **español** (prospecto, asesor, inscripción, seguimiento, guardia, comisión, sede). Mantener consistencia — no traducir términos del dominio al inglés.
- **Identificadores técnicos** (variables, funciones, tablas, endpoints): inglés estándar (`prospect`, `advisor`, `enrollment`, `followUp`, `shift`, `commission`).
- **Roles**: usar exactamente los 6 nombres definidos arriba.
- **Etapas del pipeline**: las 7 etapas son la configuración base — el sistema debe permitirlas modificar por admin, no hardcodear.
- **Score Semanal**: los pesos (35/20/15/15/10/5) son la configuración base; el módulo debe permitir KPI dinámico configurable por dirección.
- **Integración Órbita EDU**: cualquier cambio de estado relevante (inscripción, pago, nuevo lead) debe respetar los 6 endpoints definidos. No inventar endpoints alternativos.
- **Fase futura (Bloque 9)**: las funciones de IA predictiva están explícitamente marcadas como fase futura — no implementar sin pedirlo, pero el modelo de datos debería contemplar su llegada (campos para score IA, probabilidad, patrones de respuesta).
- **WhatsApp**: usar exclusivamente **Meta Business API** con plantillas preaprobadas, no WhatsApp Web ni integraciones no oficiales.
- **Auditoría**: cualquier acción sobre un prospecto (cambio de etapa, reasignación, edición, seguimiento) debe quedar registrada con autor y timestamp. Es un requisito no negociable.

## Referencia

La ficha técnica completa con tablas funcionales detalladas, KPIs y descripciones operativas está en `ficha-tecnica-orbita-crm-completa.html`. Es la fuente de verdad del producto — consultarla ante cualquier duda de alcance.
