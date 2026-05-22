# Plan de Implementación — Órbita CRM

Plan por fases para construir el CRM desde repo vacío hasta v1.0 entregable (Parte 1 + Parte 2). Bloque 9 (IA predictiva) queda fuera del alcance inicial — solo se prepara el modelo de datos.

Las fases están ordenadas por **dependencia** y **riesgo técnico**: lo que más bloquea va primero, IA/optimización al final. Cada fase es entregable y demostrable de forma aislada.

---

## Fase 0 — Fundaciones del repo

**Objetivo:** repositorio listo para que cualquier dev (o agente) pueda levantar `dev` en una máquina limpia con un comando.

**Entregables:**
- Monorepo o single-app: estructura `apps/web` (Next.js 15 App Router) + `apps/api` (Fastify) + `packages/db` (Drizzle schema compartido). Decisión a confirmar con el usuario antes de scaffolding.
- `package.json` raíz con scripts: `dev`, `build`, `lint`, `typecheck`, `db:push`, `db:studio`, `db:seed`.
- TypeScript strict en todo el monorepo. ESLint + Prettier compartidos.
- `.env.example` con todas las variables esperadas (DB, JWT secret, Órbita EDU base URL, Meta WA API tokens).
- Conexión a **PostgreSQL 18** en Railway desde dev local (vía `DATABASE_URL`).
- Drizzle inicializado con migraciones versionadas en `packages/db/migrations`.
- README mínimo con instrucciones de arranque.

**Riesgos:** decidir mono vs poly repo, single-app vs split web+api. Confirmar con usuario antes de scaffolding pesado.

---

## Fase 1 — Autenticación, roles y shell de la app

**Objetivo:** un usuario puede entrar, su rol se respeta, ve un layout autenticado.

**Entregables:**
- Tabla `users` con campos: id, email, name, password_hash, role, advisor_id (FK a sí mismo opcional para coordinadores), created_at.
- 6 roles del CLAUDE.md como enum: `direccion`, `coordinador`, `asesor`, `marketing`, `finanzas`, `admin`.
- Login con email/password + JWT (access + refresh). Endpoint preparado para **SSO con Órbita EDU** (stub que devuelve usuario por token externo).
- Middleware Fastify de auth + RBAC: helper `requireRole(...roles)` y `scopeToOwnedLeads(userId)`.
- Shell de la app web (sidebar + topbar + breadcrumbs) con shadcn/ui + Tailwind. Rutas protegidas según rol.
- Página `/login`, `/dashboard` (placeholder por rol), `/perfil`.
- Seed inicial: 1 admin + 1 dirección + 1 coordinador + 2 asesores.

**Decisión clave:** ¿password local o login solo via Órbita EDU desde día 1? Si EDU aún no tiene SSO, dejar password local + hook de SSO listo para conectar.

---

## Fase 2 — Bloque 1: Prospectos + Pipeline (núcleo del CRM)

**Objetivo:** capturar leads, gestionarlos en pipeline visual, asignarlos a asesores. Es **la mitad del valor del producto** — invertir tiempo aquí.

**Modelo de datos:**
- `prospects` (id, datos personales, contacto, empresa, ciudad, fuente, fecha_registro, programa_interes_id, modalidad, urgencia, presupuesto, ciclo_inicio, stage_id, primary_advisor_id, created_at, updated_at).
- `prospect_advisors` (M:N para asesores secundarios).
- `tags` + `prospect_tags`.
- `pipeline_stages` (id, name, order, color, is_terminal, is_won, is_lost) — **configurables por admin, no hardcodear**.
- `prospect_stage_history` (id, prospect_id, from_stage, to_stage, moved_by, moved_at).
- `prospect_notes` (id, prospect_id, author_id, body, is_important, mentions[], created_at).
- `audit_log` genérico (entity, entity_id, action, actor_id, diff_json, timestamp).

**Funcionalidades:**
1. CRUD de prospectos con validación zod en API y form.
2. Pipeline **Kanban con drag & drop** (recomendado: `@dnd-kit`). Contador por columna. Vista lista como alternativa.
3. Filtros avanzados (etapa, etiqueta, asesor, fuente, programa, rango fechas, ciudad).
4. Detección de duplicados al crear (email/teléfono) + UI de fusión que preserva historial.
5. Asignación múltiple de asesores (primario + secundarios).
6. Notas internas con menciones `@asesor` (autocomplete).
7. Movimiento entre pipelines (programas distintos) con registro de movimiento.
8. Exportación CSV/Excel/JSON con filtros aplicables.
9. Historial de cambios visible en la ficha.

**Criterio de aceptación:** un asesor puede capturar un lead, moverlo por el pipeline, agregarle notas y verlo correctamente filtrado en su vista personal vs la del coordinador.

---

## Fase 3 — Bloque 2: Actividades, seguimientos y comisiones

**Objetivo:** trazabilidad completa del trabajo del asesor + cálculo de comisiones.

**Modelo de datos:**
- `activities` (id, prospect_id, advisor_id, type [call|visit|appointment|guard_shift|info_sent], started_at, ended_at, duration_seconds, result, classification, summary, next_action_at, next_action_note, created_at).
- `appointments` (subset: location, mode [presencial|videollamada|llamada], status [pendiente|realizada|cancelada|reprogramada]).
- `guard_shifts` (sede_id, start_at, end_at, prospects_attended_count).
- `commission_rules` (id, program_id?, base_pct, bonus_meta, modifier_descuento, active_from, active_to).
- `commission_records` (id, advisor_id, prospect_id, enrollment_id, period, amount, status [pendiente|pagado|revisión], paid_at).
- `monthly_goals` (advisor_id|team_id, period, target_enrollments, target_revenue).
- `reminders` (id, owner_id, prospect_id?, due_at, channel [app|email|whatsapp], status).

**Funcionalidades:**
1. Formularios de registro por tipo de actividad — todos con **resultado obligatorio** y **próxima acción obligatoria** (regla no negociable según CLAUDE.md).
2. Calendario unificado del asesor (mensual/semanal) — `react-big-calendar` o equivalente.
3. Recordatorios in-app + email. (WhatsApp queda para Fase 6.)
4. Alertas de prospectos fríos: job programado (Bull/Quirrel/`pg_cron`) que evalúa inactividad por etapa.
5. Tabulador de comisiones configurable por admin.
6. Panel personal del asesor: inscritos del periodo, monto generado, estado de pago.
7. Meta mensual con barra de progreso en tiempo real.
8. Exportación de nómina a Excel/PDF.

**Criterio de aceptación:** un asesor cierra una llamada → resultado y próxima acción son obligatorios → recordatorio aparece en su calendario → si no actúa, sale como prospecto frío al día N.

---

## Fase 4 — Bloque 3: KPIs, reportes e integración Órbita EDU

**Objetivo:** dirección puede ver el embudo y los 6 endpoints con EDU están operativos.

**Reportes/KPIs (12 KPIs del CLAUDE.md):**
1. Capa de cálculo en SQL/vistas materializadas (preferir vistas para conversión, velocidad, ticket promedio). Evitar recalcular en cliente.
2. Endpoint `GET /reports/:type` con filtros por periodo, asesor, programa, fuente.
3. UI de reportes con exportación PDF/Excel/CSV.
4. Job nocturno de snapshots para histórico (`kpi_snapshots` por día/semana/mes/asesor/programa).

**Integración Órbita EDU (los 6 endpoints son no negociables, ver CLAUDE.md):**
- `POST /api/lead` — receptor de leads desde landing EDU. Webhook firmado (HMAC).
- `GET  /api/courses` — pull periódico + cache local de catálogo de programas.
- `GET  /api/availability` — pass-through con cache corto (5 min) por grupo/periodo.
- `POST /api/enroll` — disparado al marcar prospecto como "Inscrito". Idempotente con `prospect_id` como key.
- `POST /api/webhook/payment` — receptor. Actualiza `payment_status` del prospecto/alumno. Verificar firma.
- `GET  /api/student/:id` — pull bajo demanda desde ficha del prospecto-ya-alumno. Cache 1 min.

**Riesgo principal:** disponibilidad y contrato de los endpoints de Órbita EDU. **Acción:** acordar contrato OpenAPI 3.1 con el equipo de EDU antes de empezar la fase. Mientras tanto, mock server.

---

## Fase 5 — Bloque 4 + 5: Score Semanal + Control avanzado de llamadas

**Objetivo:** medir desempeño objetivo del asesor con score automático + endurecer registro de llamadas.

**Score Semanal:**
- Job semanal (lunes 02:00) que calcula score por asesor con la fórmula ponderada (35/20/15/15/10/5 — configurable por admin).
- Tabla `advisor_scores` (advisor_id, period_start, period_end, components_json, final_score, rank).
- UI: ranking visible para dirección/coordinador. Gráfica de tendencia por asesor.
- KPI dinámico: panel admin para reconfigurar pesos por periodo (ej. campaña activa).

**Control de llamadas (extiende Fase 3):**
- Cronómetro real obligatorio en el formulario (start/stop). No permitir entrada manual de duración.
- Resultado obligatorio (contactado/no contestó/buzón/número incorrecto).
- Clasificación: seguimiento / cierre / reactivación.
- Próxima acción con fecha obligatoria.
- Ratio llamada-conversión calculado en vista materializada.
- Grabación opcional: campo `recording_url` + integración pluggable (Twilio/Zadarma) — feature flag, no requerido para v1.0.

---

## Fase 6 — Bloque 6: WhatsApp Business API + Automatizaciones

**Objetivo:** centralizar comunicación + reducir trabajo manual con triggers automáticos.

**WhatsApp (Meta Business API obligatorio según CLAUDE.md):**
- Cuenta Meta Business + número verificado + plantillas preaprobadas (primer contacto, seguimiento, confirmación de cita, info de programa).
- Webhook receptor de mensajes entrantes → registra en `wa_messages` vinculado al prospecto por número.
- Envío de mensajes desde la ficha del prospecto con selector de plantilla.
- Etiquetado automático lead frío/caliente según frecuencia de respuesta.

**Automatizaciones (8 del CLAUDE.md):**
- Motor de reglas configurable por admin (tabla `automation_rules` con trigger + condition + action).
- Workers: cola persistente (BullMQ o `pg-boss` para no añadir Redis si no es necesario).
- Triggers iniciales: lead nuevo → asignación; cita agendada → confirmación + recordatorio 24h; pago confirmado → cambio a Inscrito + `POST /api/enroll`; lead abandonado → escalamiento; sin seguimiento N días → alerta.

**Riesgo:** aprobación de plantillas por Meta tarda 24-72h. Empezar este proceso al iniciar Fase 5.

---

## Fase 7 — Bloque 7: Dashboards ejecutivos y por programa

**Objetivo:** dirección ve el negocio en una pantalla.

**Dashboard dirección:**
- Tarjetas: leads totales, conversión global, ROI, asesor más eficiente, tiempo promedio de cierre, ingreso proyectado (forecast), pipeline global, tasa de pérdida, inscritos por programa.
- Forecast simple: `Σ(valor_prospecto × prob_etapa)` donde `prob_etapa` viene de tasas históricas por etapa.

**Dashboard por programa:**
- Funnel visual por programa con % de paso entre etapas.
- Comparativa mensual.
- Costo por inscripción (requiere input manual de gasto de captación por programa/periodo — agregar tabla `program_ad_spend`).

**UI:** Recharts o Tremor para gráficos. Filtros globales en topbar del dashboard.

---

## Fase 8 — Bloque 8: Fuga de leads + semaforización

**Objetivo:** prevención activa de pérdida con priorización visual.

- Job por minuto (ligero) que reevalúa el estado de semáforo de cada prospecto activo según última actividad, fecha próxima acción y umbrales por etapa.
- Campo `semaforo_color` denormalizado en `prospects` para queries rápidas.
- Reasignación automática con regla configurable (round-robin entre asesores activos, o coordinador decide).
- Vista "Leads en riesgo" para coordinadores con acción rápida de reasignación.
- Secuencia de reactivación: programable por etapa y programa, usa el motor de automatizaciones de Fase 6.
- Historial de auditoría completo (ya cubierto por `audit_log` desde Fase 2).

---

## Fase 9 — Preparación para IA predictiva (modelo de datos, no implementación)

Reservar columnas y tablas para que la fase futura no requiera migración disruptiva:
- `prospect_ai_scores` (prospect_id, score_conversion, score_abandon, computed_at, model_version).
- `prospect_interaction_features` (vectores agregados de interacciones — vacío por ahora).
- Webhook outbound a un servicio de scoring externo, deshabilitado por defecto.

**No construir el modelo en v1.0** — solo dejar las tablas y un endpoint stub.

---

## Tareas transversales (en paralelo a todas las fases)

- **Auditoría inmutable**: todo cambio sobre prospectos/actividades/comisiones pasa por `audit_log`. Esto se establece en Fase 2 y se respeta hasta el final.
- **Testing**: Vitest para unit, Playwright para e2e de los flujos críticos (crear lead → moverlo → inscribirlo → ver comisión).
- **Observabilidad**: logs estructurados (Pino), Sentry para errores, métricas básicas (request count, latency) expuestas en `/metrics`.
- **i18n**: español como único idioma del dominio. UI strings en archivo único por si se internacionaliza.
- **Backups**: snapshot diario de Postgres en Railway + retención 30 días.
- **CI**: GitHub Actions con typecheck + lint + test en cada PR; deploy automático a Railway preview en cada PR.

---

## Orden recomendado para empezar

1. **Fase 0** completa antes de cualquier otra cosa.
2. **Fase 1** (auth) — sin esto, todo lo demás es demo sin valor.
3. **Fase 2** (prospectos + pipeline) — la mitad del valor del producto.
4. **Fase 3** (actividades + comisiones) — la otra mitad operativa.
5. Punto de demo interna → validar con dirección antes de seguir.
6. **Fase 4** (KPIs + integración EDU) — requiere que EDU exponga sus endpoints. Empezar negociación de contrato en Fase 2.
7. **Fase 5** (Score + control de llamadas).
8. **Fase 6** (WhatsApp + automatizaciones) — iniciar trámite Meta en Fase 5.
9. **Fase 7** (dashboards).
10. **Fase 8** (fuga + semaforización).
11. **Fase 9** (preparación IA) — solo data model.

---

## Decisiones pendientes que bloquean Fase 0

Antes de scaffolding, confirmar con el usuario:

1. **Estructura de repo**: monorepo (`apps/web` + `apps/api` + `packages/db`) vs single Next.js full-stack con route handlers + workers separados.
2. **Base de datos**: ¿compartida con Órbita EDU o separada con sincronización por API? CLAUDE.md deja abiertas ambas opciones.
3. **SSO desde día 1**: ¿login local + EDU SSO en Fase 1, o solo EDU SSO?
4. **Cola de trabajos**: BullMQ (requiere Redis) vs `pg-boss` (solo Postgres). Recomendado `pg-boss` para minimizar infra.
5. **Contrato API con Órbita EDU**: ¿quién publica el OpenAPI 3.1, EDU o CRM consume el de EDU?
