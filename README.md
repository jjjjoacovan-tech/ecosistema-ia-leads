# Ecosistema de Automatización IA — Clasificación y respuesta de leads

**Proyecto Final · IA Automation, Comisión 2026 · Joaquín Van Rompaey**

Sistema de automatización que recibe consultas comerciales, las clasifica con un modelo de lenguaje usando el catálogo real de servicios como fuente de verdad, redacta un borrador de respuesta y **se detiene a esperar aprobación humana** antes de contactar al cliente.

---

## El problema que resuelve

Un estudio freelance de automatización recibe consultas por formulario web. Cada una hay que leerla, decidir si vale la pena, buscar qué servicio encaja y redactar una respuesta con precios y plazos correctos. Es trabajo repetitivo que consume las horas más productivas del día, y en el que se filtran errores: precios desactualizados, consultas que se responden tarde, spam que roba atención.

El sistema hace la parte mecánica y deja la decisión donde corresponde: en una persona.

---

## Arquitectura

```
Formulario web
      │
      ▼
[ Webhook ]──► Filtro anti-loop ──► Airtable: registrar lead (Pendiente)
                                            │
                                            ▼
                              Airtable: leer catálogo de servicios
                                            │
                                            ▼
                                  Agregador de contexto
                                            │
                                            ▼
                            Gemini: clasificar, puntuar, redactar
                                            │
                                            ▼
                              Airtable: guardar clasificación
                                            │
                                        ┌───┴───┐
                                     Router
                          ┌─────────────┴─────────────┐
                    VIP / Estándar                Descartado
                          │                            │
                          ▼                            ▼
                  Slack #aprobaciones          Estado: Rechazado
                          │                       (fin del ciclo)
                          ▼
              Estado: Esperando Aprobación
                          │
              ╔═══════════▼═══════════╗
              ║   PAUSA — El humano   ║
              ║   revisa y aprueba    ║
              ╚═══════════╤═══════════╝
                          ▼
              Gmail: enviar respuesta ──► Estado: Enviado ──► Log de éxito
```

### Las cuatro categorías tecnológicas

| Categoría | Herramienta | Rol en el sistema |
|---|---|---|
| Orquestador | Make.com | Dos escenarios encadenados por estado, no por llamada directa |
| Base de datos | Airtable | Tres tablas vinculadas: memoria, catálogo y bitácora |
| Procesamiento IA | Google Gemini (API HTTP) | Clasifica, puntúa y redacta con contexto recuperado (RAG) |
| Canal de salida | Slack + Gmail | Slack valida internamente, Gmail contacta al lead |

---

## Cómo funciona el Human-in-the-loop

El punto de validación no es una advertencia ni una configuración que se pueda desactivar: es **estructural**.

El Escenario 1 no contiene ningún módulo capaz de enviar correo. Termina su trabajo dejando el lead en estado `Esperando Aprobación` y ahí se acaba. El Escenario 2 vigila la tabla y solo actúa sobre registros que ya figuran como `Aprobado por Humano`, un estado que únicamente una persona puede escribir.

Aunque el modelo se equivoque por completo en su clasificación, no existe camino técnico por el que ese error alcance al lead sin que alguien lo haya leído.

---

## Estructura de datos

**Base:** `appyCPtzRP0Ytcz96`

| Tabla | Función |
|---|---|
| `Leads` | Registro maestro. 16 campos con máquina de estados de 7 valores |
| `Base de Conocimiento` | Catálogo de servicios con precios y plazos. Contexto factual para el modelo |
| `Log de Ejecuciones` | Bitácora de éxitos y fallos. Alimenta los KPIs del dashboard |

Dos relaciones bidireccionales: `Leads ↔ Base de Conocimiento` (qué servicio pidió cada lead) y `Leads ↔ Log de Ejecuciones` (qué eventos generó cada lead).

El detalle completo, con IDs de campo y esquemas JSON de transferencia, está en la documentación técnica.

---

## Resiliencia

Cuatro Error Handlers activos. La regla que los ordena: **se corta cuando continuar produciría un dato incorrecto, se continúa cuando el fallo solo afecta a un aviso interno.**

| Punto de fallo | Directiva | Comportamiento |
|---|---|---|
| Registrar lead en Airtable | `Break` | Registra el error y detiene: sin ID no hay nada que actualizar |
| Motor de IA | `Break` | Marca el lead como Error y detiene para reproceso manual |
| Notificación a Slack | `Resume` | Registra y continúa: el lead sigue visible en Airtable |
| Envío por Gmail | `Break` | Revierte a Error antes de marcar como Enviado |

Los Error Handlers fueron verificados en producción: durante las pruebas del 18/08/2026 el módulo de IA falló de forma sostenida y el sistema se comportó como estaba diseñado — el lead se guardó completo, el fallo quedó registrado con módulo y detalle, y **ningún mensaje se envió por error**.

---

## Optimización de costos

El hallazgo del análisis: **el gasto no está en la IA.**

| Componente | Costo mensual | Cobertura |
|---|---|---|
| Make (orquestación) | USD 9–12 | ~1.400 leads/mes |
| Google Gemini | USD 0 (free tier) | USD 0,09 cada 500 leads si se pasa a pago |
| Airtable | USD 0 | 1.000 registros |
| Slack | USD 0 | 90 días de historial |

Optimizar el diseño del escenario rindió **61% de ahorro real** (de 18 a 7 créditos por lead, agregando un módulo que colapsa el catálogo en un solo bloque). Optimizar el modelo hubiera rendido centavos.

La matriz completa de selección de modelo por tarea está en la documentación técnica.

---

## Dashboards

- **[KPIs del pipeline](https://airtable.com/appyCPtzRP0Ytcz96/shrHv6UqLgeaJRQFc)** — leads agrupados por estado, con conteo por etapa del embudo
- **[Tasa de errores](https://airtable.com/appyCPtzRP0Ytcz96/shrgFPdpgORH1XBh8)** — eventos agrupados por tipo, separando fallos técnicos de decisiones humanas

---

## Contenido del repositorio

```
├── README.md                      Este archivo
├── DOCUMENTACION_TECNICA.pdf      Los cuatro documentos unificados (13 páginas)
├── docs/
│   ├── 01_Mapa_de_Arquitectura.pdf
│   ├── 02_Manual_de_Datos.pdf
│   ├── 03_Matriz_de_Costos.pdf
│   └── 04_Seguridad_y_Resiliencia.pdf
├── blueprints/
│   ├── escenario_1_clasificacion.json
│   └── escenario_2_envio_hitl.json
└── screenshots/
    └── (evidencias de ejecución)
```

---

## Seguridad

Ninguna credencial está escrita dentro de los blueprints. Las conexiones aparecen como identificadores numéricos que solo tienen sentido dentro de la cuenta de Make donde fueron creadas. Quien clone este repositorio obtiene la lógica completa del sistema y **ningún acceso a los datos**.

El sistema recolecta cuatro campos: nombre, email, empresa y mensaje. Cada uno cumple una función verificable en el flujo. No se piden teléfono, dirección, documento ni datos de facturación, porque ninguno interviene en la decisión de clasificar ni en la de responder.
