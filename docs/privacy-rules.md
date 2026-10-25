# Privacy rules

> Aplicadas en **Main (test)** el 2026-10-09. Savepoint previo:
> "Antes de privacy rules y create-request" (`1791507626562`).
> **Aún no probadas con usuarios reales** — ver "Pruebas pendientes".

## Principio

Antes las reglas eran **por rol** ("si eres Customer, ves todo"), así que
cualquier usuario podía buscar los datos de todos los demás (incluidos lockbox y
códigos de acceso). Ahora son **por relación**: cada usuario ve lo suyo y lo de
las reservas en las que participa.

Una privacy rule de Bubble solo puede evaluar **campos del propio registro** y
el **Current User**; no puede hacer búsquedas. Por eso se agregaron campos de
relación (ver abajo).

Todas las condiciones de Admin son `Current User's osRole is Admin`.

## Reglas por tipo

| Tipo | Ve todo | Ve parcial | Nadie más |
|---|---|---|---|
| **User** | Self (`This User is Current User`; **sin** `tokenConfirmation` ni `pendingVerificationCode`), Admin | Logged in users: solo `osRole`, `osStatusVerificationPetsitter`, Created Date | ✔ |
| **profile** | Owner (`Creator is Current User`), Admin | Logged in users: firstName, lastName, profilePicture, about, experienceInMonths, osGender, details, country, Created Date, Creator. **Sin teléfono ni birthdate** | ✔ |
| **address** | Owner (`user` o Creator = me), **Authorized petsitter** (`authorizedPetsitters contains me`), Admin | — | ✔ |
| **dgRequest** | Customer (`customer = me`), Applied or assigned petsitter (`listOfPetsitters contains me`), Admin | **Open requests for petsitters**: rol Petsitter **y** status = Pending **y** verificación del petsitter = Enabled (`aproved`, desde 2026-10-14) → servicio, tipo, fechas, horas, duración, geographicAddress, mascotas, instrucciones, statuses, tip, customer. **Sin** address, conversation, trackingNew, reviews, requestAddTime | ✔ |
| **pets** | Owner (`user` o Creator = me), **Authorized petsitter** (`authorizedPetsitters contains me`: todo menos estado IA), Admin | Petsitters (basic info): name, photo, breed, size, gender, birthdate, spayedNeutered, personality, user, delete; búsqueda solo por user/delete | ✔ (2026-10-10) |
| **transaction** | Customer, Petsitter (del registro), Admin | — | ✔ |
| **requestAddTime** | Customer, Petsitter (del registro) | — | ✔ |
| **notifications** | Recipient or sender, Admin | Broadcast for my role: `all = yes` y `typeUser = my osRole` | ✔ |
| **trustedContact** | Owner, Admin | — | ✔ |
| **subscription** | Own subscription (`petsitter = me`), Admin | — | ✔ |
| **favorites** | Owner (`customer = me`) | — | ✔ |
| **dgDisponibility** | Owner petsitter (petsitter o Creator = me) | Logged in users: todo menos `address` | ✔ |
| **dgSubServices** | Petsitters | Customers: duration, price, osDgType, user | sin cambios |
| **review** | Customers y petsitters (rol) | — | sin cambios |
| **conversations / messages** | `participants contains me` | — | sin cambios (ya estaba bien) |
| **support / feedback** | Creator, Admin | — | **antes eran públicos sin login** |
| **trackingNew** | Petsitter (Creator) o customer, Admin | — | **antes era público sin login** |
| **updateLocation** | Petsitter (Creator) o customer, Admin | — | ✔ |
| **tracking** (legacy) | Admin | — | ✔ |
| **aiResponses** | Admin | — | sin cambios |

### Auto-binding

- **User:** auto-binding **desactivado** para el propio usuario. Antes un
  usuario podía cambiarse `osRole` (p. ej. a Admin) o
  `osStatusVerificationPetsitter` desde un input. El cambio de modo usa
  "Make changes to current user" en workflows, que no depende de esto.
- **address (petsitter), pets (petsitters), dgRequest (open requests):** sin
  auto-binding.

## Campos de relación agregados

| Campo | Se llena en | Se vacía en |
|---|---|---|
| `address.authorizedPetsitters` (lista User) | Backend `checkout_session_completed`: se agrega el `petsitter` de la transaction a la `address` de la reserva (solo si la reserva tiene address) | Backend `place-revoke-petsitter-access`, programado al terminar el paseo y al cancelar desde el chat (desde 2026-10-10; antes se hacía en el cliente) |
| `pets.authorizedPetsitters` (lista User) | Backend `checkout_session_completed`, solo con `payment_status = paid` | Backend `place-revoke-petsitter-access` (igual que la address) |
| `trackingNew.customer` | Vista `walkTrackingPetsitter`, al crear el trackingNew: `dgRequest's customer` | — |
| `updateLocation.customer` | Página `walk-tracking-petsitter`, al crear cada punto: `Current page's dgRequest's customer` | — |

> Las reservas pagadas **antes** de este cambio tienen `authorizedPetsitters`
> vacío: su petsitter no verá los códigos de acceso hasta rellenarlo.

## Efectos colaterales corregidos

- **Pago a Stripe Connect:** `create-transaction` recibía el `stripeAccountId`
  del petsitter como parámetro leído desde el cliente. Como ese campo ya no es
  visible para otros usuarios, ahora el backend lo lee de `Petsitter's
  stripeAccountId` (en la creación de la transaction y en
  `payment_intent_data[transfer_data][destination]`).

## Pruebas pendientes

No se pudieron ejecutar desde la sesión (el dominio `petboss24.com` está
bloqueado por la política de red del entorno). Probar en el preview con cada
rol:

- [ ] Customer A no encuentra direcciones, reservas, pagos ni mascotas de Customer B.
- [ ] Petsitter ve reservas Pending (sin dirección exacta) y puede postularse.
- [ ] Tras pagar, el petsitter asignado ve dirección y códigos; los demás postulantes no.
- [ ] Al terminar el paseo, el petsitter deja de ver los códigos.
- [ ] El customer ve el tracking en vivo de su paseo; otro customer no.
- [ ] Pantallas que leían datos de otros usuarios (teléfono, stripeAccountId,
      contactos de emergencia) siguen funcionando o se ajustan.
- [ ] El pago llega a la cuenta Connect correcta.
