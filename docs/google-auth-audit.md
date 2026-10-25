# Auditoría Google Authentication — Login y Sign Up (2026-10-15)

> Solo lectura. **No se modificó nada** en Bubble. Main (test).

## Estado encontrado

| Punto | Hallazgo |
|---|---|
| Vistas | `authLogin` (`cmitN`) y `authSignup` (`cmiyd`), mobile nativo. Se llega a ambas desde `welcomeStep3` y entre sí. |
| Botón en Login | `btnLoginGoogle` (`cmivT`) visible, workflow `loginWithGoogle` (`cnBsP`) + `loginWithGoogleError` (`cnBsY`). |
| Botón en Sign Up | Existe `Group Login Button Google` (`cmjLs`) dentro de `Container buttons` (`cmjNq`), **oculto** y **sin workflow**; estilo antiguo. "or" (`cmizJ`) oculto. Apple (`cmjLx`) también ahí. |
| Mecanismo | Plugin oficial **Google** de Bubble (web + mobile) + acción nativa **Signup/login with OAuth provider** (`cnBsS`, proveedor `google`). Es una acción **combinada**: si la identidad no existe, Bubble crea el usuario; si existe, inicia sesión. No hay acciones separadas de login y de alta. |
| Credenciales | Bubble ya no muestra el aviso "fill your app id and secret" → hay valores en el plugin. **No se puede confirmar que sean correctos** sin un login real. El secreto vive en la configuración del plugin (servidor); no aparece en elementos, workflows ni datos. |
| Uso real | **Ningún login con Google ejecutado**: 0 entradas OAuth/Google en los logs (14–15 oct) y ningún usuario con `googleAccount = yes`. |
| Backend compartido | `auth-google-complete` (`cnBsJ`), solo si `Current User's Google` no está vacío: (1) `googleAccount`/`googleId` siempre; (2) `emailConfirmation = yes` si el email coincide; (3) rol Customer + verificación None solo si `osRole` vacío; (4) `profile` con nombre de Google si no tiene. |
| Login tradicional | `Log the user in` → si no verificado: código + `verificationCheck` y termina; si verificado → `tabBookings` (Petsitter) / `tabHome`. |
| Sign Up tradicional | Valida contraseña → `Sign the user up` con **osRole = Customer**, `googleAccount = no`, `emailConfirmation = no` → `profile` con el nombre → código de verificación. |
| Modelo User | `emailConfirmation`, `googleAccount`, `googleId`, `osRole`, `osStatusVerificationPetsitter`, `tokenConfirmation`, `tokenExpiresAt`, `verificationAttempts`, Stripe ids. Privacy: "Self" no ve el código; "Logged in users" solo rol/verificación; Admin todo. |
| Logout / reset | Logout → `welcomeStep1`. Reset → email de Bubble (`authForgotPassword`). Un usuario solo-Google no tiene contraseña: "Forgot password" le crearía una. |

## Señal fiable de "usuario nuevo creado por Google"

El alta tradicional **siempre** pone `osRole = Customer`. Un usuario que Bubble
acaba de crear por OAuth llega con **`osRole` vacío**. Es un dato del servidor,
no del cliente → sirve para distinguir nuevo vs existente.

## Problemas

1. **Sign Up sin Google** (oculto, sin workflow).
2. **Vinculación solo por email** (contra la regla pedida): si Bubble enlaza una
   cuenta de contraseña con el mismo email, el backend marca `googleAccount`,
   `googleId` y `emailConfirmation` sin más comprobación. Comportamiento real de
   Bubble en este caso **sin verificar** (puede enlazar o dar error).
3. **Login crea cuentas**: la acción es combinada; desde Login una identidad
   nueva crea usuario. Hay que decidir la política.
4. La navegación tras Google no distingue "nuevo", "existente vinculado" ni
   "existente sin vincular".
5. Email verificado: el plugin expone `id`, `email`, `given_name`,
   `family_name`, `picture`; **no** expone `verified_email`. Se confía en que
   Google exige email verificado en la cuenta (hipótesis razonable, no
   comprobable desde Bubble).
6. Nombre: si Google no da `given_name`, el perfil queda sin nombre.
7. App nativa: vuelta desde Google a la app **sin probar**.

## Plan propuesto (pendiente de autorización)

Backend `auth-google-complete` (una sola lógica compartida):
- Vincular (`googleAccount`, `googleId`), confirmar email, rol Customer y
  perfil **solo si `osRole` está vacío** (usuario recién creado por Google).
- Cuenta existente con `googleAccount = yes` → no tocar nada.
- Cuenta existente sin vincular → no tocar nada (el cliente la rechaza).
- Nombre: si falta `given_name`, usar la parte del email antes de `@`.

Login (`cnBsP`):
- Guardar si el usuario era nuevo (`osRole` vacío) antes del backend.
- Sesión + `googleAccount = yes` → navegación actual por rol.
- Sesión + `googleAccount` no → **Log out** + mensaje "This email already has a
  password account. Log in with your password." (sin éxito falso).
- Identidad nueva desde Login → según decisión 1.

Sign Up (`authSignup`):
- Mostrar `cmjNq` y "or"; ocultar Apple; mismo estilo que el botón del Login;
  estados `googleLoading` / `googleError`; workflow `signupWithGoogle` +
  `OnError`.
- Nuevo → `tabHome`. Existente vinculado → entra con aviso "You already have an
  account". Existente sin vincular → Log out + mensaje + enlace a Login.

Pruebas: todas requieren navegador/dispositivo y cuentas de Google de prueba;
desde este entorno no se pueden ejecutar.

## Decisiones del usuario (2026-10-15)

1. **Identidad nueva desde Login** → se trata como alta (Customer + perfil) y entra.
2. **Cuenta existente con contraseña y mismo email** → se **bloquea**: cerrar
   sesión y pedir que entre con contraseña. No se vincula solo por email.
   Vincular Google desde Ajustes queda como mejora futura.

## Implementación (2026-10-15)

Savepoints: antes `1792085576162` · Login/backend `1792085680814` · después `1792085774217`.

### Backend `auth-google-complete` (`cnBsJ`)
- Condición del evento: `Current User's Google is not empty` **y `osRole is empty`**.
  Solo inicializa usuarios recién creados por Google. Cuentas existentes
  (vinculadas o no) no se tocan.
- Perfil: nombre = `given_name` de Google, *defaulting to* la parte del email
  antes de `@`.

### Login (`authLogin`, `loginWithGoogle` `cnBsP`)
1. `googleLoading` = sí; `googleError`, `googleBlocked` = no.
2. Signup/login with Google.
3. **Nuevo** `cnBsu`: `googleBlocked` = sesión **y** `osRole` no vacío **y**
   `googleAccount` ≠ sí (cuenta existente no creada por Google).
4. **Nuevo** `cnBsv`: Log out si `googleBlocked`.
5. Backend (si hay sesión) → `googleLoading` = no → `googleError` si no hay
   sesión **y no** bloqueado → navegación por rol (sin cambios).
- `txtGoogleError` (`cnBsb`): nuevo condicional con `googleBlocked` → "This
  email already has an account with a password. Please log in with your email
  and password."
- Identidad nueva desde Login → alta (decisión 1).

### Sign Up (`authSignup`)
- `Container buttons` (`cmjNq`) y "or" (`cmizJ`) visibles; Apple (`cmjLx`)
  oculto. Orden: Sign up → or → Continue with Google → "You already have an
  account? Log in".
- `btnSignupGoogle` (`cmjLs`): mismo estilo que el de Login (`#131314`, borde
  `#8E918F`, radio 14, texto `#E3E3E3`), 50 % y "Connecting to Google…" al cargar.
- `txtGoogleError` (`cnBsw`, estilo error): error genérico o "You already have
  an account with this email. Please log in with your email and password."
- Estados de la vista: `googleLoading`, `googleError`, `googleBlocked`.
- Workflow `signupWithGoogle` (`cnBsy`): mismos pasos que Login (`cnBtA`…`cnBtI`).
  `signupWithGoogleError` (`cnBtJ`).
- Existente ya vinculado a Google → entra con su rol (sin duplicar).

`check_app_issues`: solo los 6 errores preexistentes.

### Pruebas — **todas pendientes** (no ejecutables desde este entorno)

| # | Prueba | Resultado |
|---|---|---|
| L1 | Login: usuario Google existente | Pendiente |
| L2 | Login: cancelar ventana | Pendiente |
| L3 | Login: error del proveedor | Pendiente |
| L4 | Login: identidad nueva → alta Customer + perfil | Pendiente |
| L5 | Login: email con cuenta de contraseña → bloqueado, sin sesión | Pendiente |
| S1 | Sign Up: usuario nuevo → tabHome, perfil con nombre | Pendiente |
| S2 | Sign Up: cancelar | Pendiente |
| S3 | Sign Up: usuario Google existente → entra sin duplicar | Pendiente |
| S4 | Sign Up: email con cuenta de contraseña → bloqueado | Pendiente |
| S5 | Reintento tras cancelar/error (botón vuelve a funcionar) | Pendiente |
| C1 | Login y Sign Up tradicionales siguen igual | Pendiente |
| C2 | Recuperar contraseña, logout y volver a entrar | Pendiente |
| C3 | App data: sin usuarios/perfiles duplicados | Pendiente |
| C4 | App nativa (TestFlight): vuelta desde Google a la app | Pendiente |

### Riesgos pendientes
- Si en la app nativa los datos de `Current User` llegan tarde tras volver de
  Google, `googleBlocked` podría evaluarse con datos vacíos. Verificar en C4.
- Comportamiento exacto de Bubble con email ya registrado (enlaza o error):
  ambos casos acaban sin sesión y con mensaje, pero hay que confirmarlo (L5/S4).
- Usuario solo-Google que usa "Forgot password" crea una contraseña: aceptable,
  documentado.
- Vincular Google desde Ajustes: mejora futura.
- Sign in with Apple antes de publicar en iOS (guía 4.8).

## Corrección 2026-10-23: botones no interactivos

- Síntoma: Google (Login y Sign Up) y el botón Apple de Login no reaccionaban
  en Preview Web ni en Bubble Go; los logs no tenían ni un evento de esos
  workflows. El botón nativo de Apple de Sign Up sí se ejecutaba.
- Causa (error introducido en la implementación anterior): en mobile, la
  propiedad `button_disabled` significa **"This element isn't interactive"**
  (el MCP la muestra como "Make interactive"). Estaba a `yes` en
  `btnLoginGoogle` (`cmivT`), `btnSignupGoogle` (`cmjLs`) y el botón Apple de
  Login (`cmjCf`).
- Arreglo: `button_disabled = no` en los tres. Savepoints: antes
  `1792783621352`, después `1792783649358`. Bubble: sin errores nuevos.
- Apple (Login y Sign Up) y `auth-apple-complete` los añadió otra sesión/persona;
  no se revisó su lógica aquí.
- TestFlight usa **Live**: hay que desplegar el arreglo a Live; si los botones
  siguen sin verse, generar un build nuevo de iOS.

## 2026-10-25: "inicia sesión con Google pero no avanza"

- Logs de **Live** (TestFlight), 23 oct: `signupWithGoogle` → *Signup/login
  with Google* falla con **"This email address is already registered:
  js.grisales03@gmail.com"**. Ese email ya tiene cuenta con contraseña en Live.
- Confirmado: **Bubble no vincula** una cuenta de contraseña con Google por
  email; rechaza el login. Coincide con la decisión 2 (bloquear), pero la
  pantalla solo mostraba el error genérico (o nada) y parecía "no avanzar".
- Arreglo (test): los `OnError` de Login (`cnBsa`) y Sign Up (`cnBtL`) ponen
  `googleBlocked` si el mensaje de error contiene "already registered" (muestra
  el aviso de "ya tienes cuenta con contraseña") y `googleError` en otro caso.
  Savepoints: antes `1792966231713`, después `1792966277661`.
- Apple en Live: la acción termina sin sesión y sin error (cancelado o
  configuración); pendiente de revisar.
- Para probar un login real con Google hay que usar una cuenta cuyo email **no**
  exista en la base de datos de esa versión (Live y test tienen datos distintos).

## 2026-10-25: botón nativo de Apple en Login y Sign Up

- Antes: Login usaba un `Button` de Bubble con icono `fa-apple` (`cmjCf`);
  Sign Up, el elemento nativo `AppleSignInButton` (`cnBtP`). El botón viejo de
  Sign Up (`cmjLx`) ya no existía.
- Ahora: `AppleSignInButton` en ambos — `btnLoginApple` (`cnBvl`, alto 60 como
  Google en Login) y `btnSignupApple` (`cnBtP`, alto 55 como Google en Sign Up).
  Estilo blanco, "Continue with Apple", radio 14, centrado. Orden: Google →
  Apple → texto de error.
- El workflow existente `loginWithApple` (`cnBuB`) apunta al botón nuevo (sin
  cambiar su lógica); se borró el evento vacío que Bubble crea al añadir el
  elemento. `cmjCf` borrado. Sign Up: `signupWithApple` (`cnBtV`).
- Savepoints: antes `1792966384298`, después (ver changelog). Sin errores nuevos.
- Pendiente: que Apple inicie sesión (en Live vuelve sin sesión ni error).

## 2026-10-25: arreglos de Apple (backend y workflows)

Savepoint antes: `1792966745452`.

- **Backend `auth-apple-complete` (`cnBtw`)**: condición `logged in` **y
  `osRole` vacío` (solo usuarios recién creados por Apple). Antes marcaba
  `appleAccount` y **`emailConfirmation = yes` a cualquier usuario con sesión**
  (salto de verificación de email).
- Perfil (`cnBuA`): nombre = `apple_first_name` *defaulting to* parte del email
  antes de `@`; apellido = `apple_last_name` (Apple solo los envía la primera vez).
- **Login `loginWithApple` (`cnBuB`)** y **Sign Up `signupWithApple` (`cnBtV`)**:
  - Condición del evento: `googleLoading` es no (sin doble toque).
  - Paso inicial: loading = sí, error/blocked = no; loading = no tras el backend.
  - `OnError` nuevos: `loginWithAppleError` (`cnBvu`) y `signupWithAppleError`
    (`cnBvw`), con el mismo tratamiento de "already registered" que Google.
- Textos neutros: `txtGoogleError` → "We couldn't sign you in/up. Please try
  again or use your email."; botón Google al cargar → "Connecting…" (el estado
  de carga es compartido entre Google y Apple).
- Sin errores nuevos en Bubble. **Sin probar en dispositivo.**

### Configuración de Apple (manual, no verificable desde Bubble)
- Logs de Live (24 oct): Apple vuelve a la app sin intercambio de token con
  Apple ni sesión → hoja cancelada o build/IDs mal configurados.
- Comprobar: App ID con *Sign in with Apple*; Services ID con dominio
  `petboss24.com` y return URL `https://petboss24.com/api/1.1/oauth_redirect`;
  Key `.p8` con Sign in with Apple; mismos valores en Bubble → Plugins → Apple;
  build de iOS nuevo si se activó después del último build.
