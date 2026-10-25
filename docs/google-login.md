# Google Login

> Preparado en **Main (test)** el 2026-10-14. **No funciona todavía**: faltan
> el Client ID y el Client Secret en el plugin Google. Nada desplegado a Live.
> No se ha hecho ninguna autenticación real de extremo a extremo.
>
> Savepoints: antes `1792005527109`, después `1792005662824`.

## Mecanismo elegido

Autenticación nativa de Bubble con el plugin **Google** (de Bubble, ya
instalado, web y mobile) y la acción nativa **"Signup/login with OAuth
provider"** (proveedor `google`). No se instaló ningún plugin nuevo ni se hizo
OAuth manual: Bubble hace la redirección, el intercambio de tokens y guarda el
secreto en la configuración del plugin (servidor), nunca en el cliente.

Bubble lo confirma con este aviso en el comprobador, que desaparecerá al poner
las credenciales:
`Signup/login with Google - remember to fill your app id and secret for Google`.

## Qué se hizo en Bubble

### `authLogin`

| Elemento | Cambio |
|---|---|
| `Container buttons` (`cmjOg`) | Visible (estaba oculto) |
| `btnLoginGoogle` (`cmivT`, antes "Group Login Button Google") | Ya existía. Ahora: fondo `#131314`, borde 1 px `#8E918F`, radio 14, texto `#E3E3E3` (tema oscuro de los botones de Google). Al 50 % de opacidad mientras carga |
| Texto del botón (`cmivZ`) | "Continue with Google"; mientras carga: "Connecting to Google…" |
| `Button Continue with Apple` (`cmjCf`) | Oculto (no hay Apple Sign In configurado; ver Riesgos) |
| "or" (`cmjOa`) | Visible |
| `txtGoogleError` (`cnBsb`, nuevo) | `Mobile Text Error`: "We couldn't sign you in with Google. Please try again or use your email." Solo visible si `googleError` |
| Orden del bloque | Log in → or → Continue with Google → Don't have an account? Sign up |
| Estados nuevos de la vista | `googleLoading` (evita dobles toques) y `googleError` |

Workflow **`loginWithGoogle`** (`cnBsP`), solo si `googleLoading` es no:

1. `googleLoading` = sí, `googleError` = no.
2. **Signup/login with Google** (`cnBsS`).
3. Backend `auth-google-complete` — solo si hay sesión.
4. `googleLoading` = no.
5. `googleError` = sí — solo si **no** hay sesión (cancelado o fallido).
6. → `tabBookings` si hay sesión y es Petsitter; → `tabHome` si hay sesión y
   no es Petsitter.

Workflow **`loginWithGoogleError`** (`cnBsY`, "An element has an error running
a workflow" del botón): `googleLoading` = no, `googleError` = sí.

No usa `authVerificationSheet` ni el código de verificación.

### Backend `auth-google-complete` (`cnBsJ`)

Solo corre si **`Current User's Google` no está vacío**: la identidad que
enlaza el plugin en el servidor, no un dato que mande el cliente.

| Acción | Condición |
|---|---|
| `googleAccount` = sí, `googleId` = id de Google | — |
| `emailConfirmation` = sí | Solo si el email de la cuenta **es igual** al email de Google |
| `osRole` = Customer, `osStatusVerificationPetsitter` = None | Solo si `osRole` está vacío (usuario nuevo). Nunca toca un rol existente |
| Crear `profile` con nombre y apellido de Google | Solo si el usuario no tiene ningún `profile` (mismo enlace que el signup: *Created By*) |

`emailConfirmation` = sí hace falta: `welcomeStep1` cierra la sesión a quien
no esté confirmado, y un usuario de Google no tiene contraseña con la que
verificar por código. Google ya demostró que el email es suyo.

### Sin cambios

Login con email y contraseña, signup, `authVerificationSheet`, backends de
código, recuperación de contraseña, privacy rules y estructura de datos (los
campos `googleAccount` y `googleId` ya existían).

**`authSignup` no lleva botón de Google**: la acción de Bubble es
"Signup/login" a la vez, así que el botón del login ya sirve a usuarios nuevos
y existentes. Si se quiere también en el registro, copiar los pasos del
cliente; la lógica compartida ya vive en el backend.

## Casos

| Caso | Comportamiento |
|---|---|
| A. Ya entró antes con Google | Bubble inicia sesión en la misma cuenta. El backend no cambia rol ni crea otro perfil |
| B. Nuevo | Bubble crea el usuario. El backend pone Customer, confirma el email y crea el perfil con el nombre de Google → `tabHome` |
| C. Ya tiene cuenta con email y contraseña y el mismo email | **Sin verificar**: no se sabe si Bubble enlaza la cuenta, la rechaza o pide otra cosa. No se añadió ningún enlace manual por email. Si Bubble da error, se ve `txtGoogleError` y se puede entrar con email. Hay que probarlo antes de publicar |
| D. Cancela o falla | No hay sesión: se quita la carga y se muestra el error; sigue en `authLogin` |

## Lo que tienes que hacer en Google Cloud Console

No pude leer el manual de Bubble ni la configuración del plugin desde aquí
(red bloqueada), así que **los valores exactos de redirección los tienes que
copiar de Bubble**, no de este documento.

1. **Proyecto**: usa el proyecto de Google Cloud de Petboss 24 que ya tienes
   (el de Maps) o crea uno. Recomendado: el mismo.
2. **Google Auth Platform → Branding** (pantalla de consentimiento):
   - Nombre de la app: Petboss 24. Email de soporte. Logo (opcional; si lo
     subes, Google puede pedir verificación de marca).
   - Página principal, política de privacidad y términos: las páginas `privacy`
     y `terms` de tu dominio (comprueba que abren).
   - Dominio autorizado: tu dominio (`petboss24.com`).
   - Email de contacto del desarrollador.
3. **Audience**: *External*. Empieza en **Testing** y añade tus emails como
   *test users*; pasa a **In production** cuando funcione.
4. **Data access (scopes)**: `openid`, `.../auth/userinfo.email`,
   `.../auth/userinfo.profile` (no sensibles).
5. **Clients → Create client → Web application**. El plugin de Bubble pide
   *app id* y *secret*, que es lo que da este tipo de cliente.
   - **Authorized redirect URIs**: copia la URL que muestra Bubble en
     *Plugins → Google* (o la del manual de Bubble para el plugin Google).
     Como referencia **sin verificar**, Bubble suele usar
     `https://<tu-dominio>/api/1.1/oauth_redirect` y, para la versión de
     pruebas, `https://<tu-dominio>/version-test/api/1.1/oauth_redirect`.
     No la pegues sin confirmarla en Bubble.
   - **Authorized JavaScript origins**: tu dominio (`https://petboss24.com`)
     si Bubble lo pide.
6. Copia el **Client ID** y el **Client Secret**.
7. **En Bubble → Plugins → Google**: pega el Client ID en *App ID / Client
   ID* y el secreto en *App secret*. Si el plugin muestra campos separados para
   desarrollo y producción, o una sección para la app móvil, rellénalos todos.
   El secreto queda en la configuración del plugin (servidor).
8. Revisa si Bubble muestra en esa pantalla algún requisito extra para la app
   nativa (iOS/Android): en los foros hay casos de la ventana de Google que
   vuelve a la app de pruebas en vez de a la tuya.

**Desarrollo y producción**: la versión de pruebas y la publicada tienen URLs
distintas; las dos tienen que estar en *Authorized redirect URIs*. En modo
*Testing* de Google solo entran los *test users*.

## Riesgos y limitaciones

- **App Store (guía 4.8)**: si la app de iOS ofrece login con Google, Apple
  suele exigir también *Sign in with Apple* (o una alternativa equivalente).
  El botón de Apple existe pero está oculto y sin configurar. Puede bloquear
  la publicación en iOS.
- **Icono**: el botón usa la imagen que ya tenía (`flat-color-icons_google.png`,
  de un set de iconos de terceros). Las guías de Google piden el logo "G"
  oficial: descárgalo de la página de *Sign in with Google branding guidelines*
  y reemplaza la imagen `cmivY`. No lo pude descargar desde aquí.
- **Fuente**: Google recomienda Roboto Medium en su botón; se usa Poppins, la
  de la app.
- **Errores**: Bubble no distingue el tipo de error (cancelado, credenciales,
  redirect, dominio, red): todos muestran el mismo mensaje. No se sabe si la
  acción OAuth dispara "An element has an error running a workflow"; si al
  cancelar el workflow no continúa y tampoco dispara el error, el botón se
  queda en "Connecting to Google…" hasta salir de la vista.
- **Caso C** sin verificar (ver Casos).
- **Nativo**: el flujo de Google en la app nativa de Bubble no se pudo
  comprobar; los foros mencionan problemas de redirección en la app de pruebas.
- **Live**: nada de esto está en Live. No desplegar hasta probarlo con
  credenciales reales (el botón visible sin credenciales daría error).

## Pruebas

> 2026-10-15: credenciales configuradas por el usuario (Google Cloud + plugin
> Google en Bubble). Bubble ya no muestra el aviso de app id/secret. Las pruebas
> con credenciales siguen **pendientes**.

| # | Prueba | Sin credenciales | Con credenciales |
|---|---|---|---|
| 1 | Usuario de Google ya existente entra | — | Llega a su pestaña; no se crea otro usuario ni perfil |
| 2 | Usuario nuevo | — | Usuario con Customer, email confirmado, perfil con nombre; `tabHome` |
| 3 | Cancelar la ventana de Google | — | Error visible, botón usable de nuevo, sin usuario creado |
| 4 | Credenciales incorrectas | Ya hoy: tocar el botón debería dar error (comprobar mensaje y que el botón no se quede cargando) | Error visible |
| 5 | Redirect URI mal | — | Error de Google; volver a la app sin sesión |
| 6 | Email igual a una cuenta con contraseña | — | Ver qué hace Bubble (caso C); no debe crear una segunda cuenta |
| 7 | Onboarding | No hay onboarding extra en el signup actual; igual para Google | — |
| 8 | Perfil completo | — | Entra directo |
| 9 | Sesión al navegar y al reabrir la app | — | Sigue dentro; `welcomeStep1` no la cierra (email confirmado) |
| 10 | Cerrar sesión y volver con Google | — | Va a `welcomeStep1`; Google vuelve a entrar en la misma cuenta |
| 11 | Login con email y contraseña | Probar ya: igual que antes | Igual |
| 12 | Separación login / signup / código | Revisado en el editor: el workflow de Google no toca el sheet ni el código | No aparece el sheet con Google |
| 13 | Test y producción | — | Las dos URLs de redirect autorizadas |
| 14 | Sin duplicados ni perfiles a medias | — | Revisar *App data* tras varias entradas |
| 15 | Datos privados | Privacy rules sin cambios | Un usuario no ve datos de otro |

Pruebas hechas hasta ahora: solo lectura de la configuración en el editor
(workflows, condiciones y orden) y el comprobador de Bubble (los 6 errores de
siempre más el aviso de credenciales). Ninguna en dispositivo.
