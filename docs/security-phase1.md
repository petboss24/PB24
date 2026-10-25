# Seguridad — Fase 1 del plan de tracking (2026-10-14)

Cambios en **Main (test)**. Nada se desplegó a Live.

- Savepoint antes: **"Antes de Fase 1 seguridad (privacy rules + webhook)"** — `1792008694212`
- Savepoint después: **"Después de Fase 1 seguridad: open-requests verificado, autobinding dgRequest vacío, call Retrieve Checkout Session (sin inicializar)"** — `1792009047530`

## Resumen

| # | Punto del plan | Estado | Detalle |
|---|---|---|---|
| 1 | Restringir la lectura de reservas pendientes | ✅ Aplicado | La regla `Open requests for petsitters` exige además `Current User's osStatusVerificationPetsitter is Enabled` (valor interno `aproved`). |
| 2 | Quitar el autobinding de estados y campos sensibles | ✅ Aplicado | Autobinding vacío en las reglas `Customer` y `Applied or assigned petsitter` de `dgRequest`. Antes permitían autoguardar status, sub-statuses, listOfPetsitters, petsitterConfirmed, duration, customer, etc. |
| 3 | Validar el webhook de pago | ✅ Aplicado | El webhook pregunta a Stripe por la sesión y se detiene si no está `complete` + `paid`. Ver "Webhook verificado". |
| 4 | `create-request-date` comprueba el customer | 🟢 Riesgo bajo, sin cambios | El backend `cmiCF` **no está expuesto** como API pública: solo se llama desde workflows propios. No es una vía de ataque externa. |
| 5 | Acceso a sesiones de tracking y puntos GPS | ✅ Ya correcto | `trackingNew` y `updateLocation`: lectura solo para Creator (petsitter), `customer` y Admin; sin autobinding. Se añadirá `assignedPetsitter` en Fase 2. |
| 6 | Validar escrituras en backend (ids de otras sesiones) | ⏭ Fase 3 | Hoy las escrituras de tracking son workflows de cliente ("Make changes"). Se mueven a backend (`walk-start/ping/end`) en Fase 3 con la comprobación de dueño. |
| — | `create-transaction` toma importe, fee y petsitter del cliente | ⚠️ Señalado | No se cambió. El importe debe calcularse en servidor (precio del servicio + tip) y compararse con `amount_total` de Stripe en el webhook. |

`check_app_issues` después de los cambios: los 6 errores preexistentes + el
aviso esperado de Google ("fill your app id and secret"). Ninguno nuevo.

## Hallazgo: clave de Stripe de test caducada (resuelto)

Al inicializar `Retrieve Checkout Session` Stripe devolvió:

```
401 api_key_expired — Expired API Key provided: sk_test_…jSaGAF
```

La clave está a nivel de colección (`cmVrD`, Stripe), así que **todas** las
llamadas de esa colección en test (Create Checkout Session, Refund, Retrieve
account…) fallan hoy. La última sesión de Checkout en test es de mayo 2026.

### Acción manual (usuario)
1. Stripe Dashboard (cuenta **PB24**, modo test) → Developers → API keys →
   generar / copiar la clave secreta de test (o una restricted key con lectura
   de Checkout Sessions y lo que usen las demás llamadas).
2. Bubble → Plugins → API Connector → colección **Stripe** → pegar la clave de
   **development** (no tocar la de Live sin revisarla antes).
3. ~~Avisarme~~ Hecho 2026-10-14: el usuario cambió las claves y la llamada
   se inicializó con la sesión `cs_test_a1wT0Kz…X4` de PB24 (encontrada, así
   que la clave es de la cuenta correcta).

## Webhook verificado

- Savepoint antes: **"Antes de verificar webhook checkout_session_completed con Retrieve Checkout Session"** — `1792009923616`
- Savepoint después: **"Después de verificar webhook checkout_session_completed (Retrieve + Terminate si no pagado)"** — `1792009984623`

Backend `checkout_session_completed` (`cmpFt`, expuesto sin autenticación;
endpoint de Stripe test `we_1S94XN…` → `version-test/api/1.1/wf/checkout_session_completed`):

| Paso | Acción | Cambio |
|---|---|---|
| 1 | `Stripe - Retrieve Checkout Session` (`cnBsq`) | **Nuevo.** `session_id = Request Data's object id`. Un id inventado → Stripe responde 404 → el workflow se detiene. |
| 2 | Terminate this workflow (`cnBsr`) | **Nuevo.** Solo cuando `status is not complete` **o** `payment_status is not paid` (datos de Stripe, no del payload). |
| 3 | Make changes to transaction (`cmpGW`) | Busca la transacción por el **id devuelto por Stripe** y guarda `payment_status` / `payment_intent` **de Stripe**. |
| 4-8 | chat-create, dgRequest → Booked, address, pets, notification | Sin cambios; ya solo corren con pago confirmado. |

Pendiente (siguiente iteración):
- Comparar `amount_total` de Stripe con el importe calculado en servidor
  (depende de arreglar `create-transaction`).
- Idempotencia: Stripe reintenta webhooks; hoy un reintento repetiría chat y
  notificación.
- **Live**: la misma lógica necesita la clave de Live válida en la colección y
  un endpoint de Live; no se ha tocado Live.

Prueba manual: hacer una reserva de test y pagar con la tarjeta `4242 4242 4242 4242`.
La reserva debe pasar a Booked/Upcoming. Una llamada a mano al endpoint con un
`object.id` falso no debe cambiar nada (ver Logs → Server logs).

## Pruebas de autorización (pendientes, manuales)

Hacen falta 4 cuentas de test: customer A, customer B, petsitter verificado
(Enabled), petsitter sin verificar (None), y admin.

| # | Caso | Esperado |
|---|---|---|
| 1 | Petsitter **Enabled** abre la lista de solicitudes pendientes | Ve las reservas Pending |
| 2 | Petsitter **None** o **Restricted** abre la misma lista | Lista vacía |
| 3 | Petsitter Enabled ve una reserva ya **Booked** de otro petsitter | No la ve |
| 4 | Customer A ve sus reservas | Las ve todas |
| 5 | Customer B busca reservas de A | No ve ninguna |
| 6 | Admin ve todas las reservas | Las ve |
| 7 | Customer crea, edita (bookingEditSheet) y cancela una reserva | Funciona igual que antes (no usa autobinding) |
| 8 | Petsitter acepta una reserva | Funciona igual que antes |
| 9 | Customer B intenta abrir el tracking de A | No ve nada |
| 10 | Petsitter asignado / customer dueño abren el tracking | Lo ven |

Si 7 u 8 fallan con un error de autobinding, restaurar el savepoint
`1792008694212` o devolver a la regla solo el campo concreto que use el input.
