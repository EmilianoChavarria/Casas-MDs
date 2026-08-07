# Servicio: Autenticación con Google (OAuth 2.0)

## ¿Para qué se usa?

1. **"Continuar con Google" en el registro e inicio de sesión del huésped** — sin llenar formulario ni recordar otra contraseña.
2. **Precargar datos del perfil**: nombre, correo verificado, foto e **idioma preferido** (`locale`), que alimenta directo la columna usada para enviar los correos en el idioma correcto.
3. (Pendiente de decisión — ver [`dudas-cliente.md`](../dudas-cliente.md) **D6**) Acceso del personal al panel de administración.

## Justificación

**Es gratuito y no tiene cuota.** A diferencia de Google Maps, OAuth no factura por uso: no hay costo por inicio de sesión ni límite de usuarios.

**Reduce fricción justo donde más duele.** El proyecto atiende huéspedes de México, Estados Unidos y Canadá. Pedirle a alguien que nunca ha oído del negocio que cree una cuenta con contraseña, en un sitio en su segundo idioma, antes de poder reservar, es una barrera real. "Continuar con Google" la elimina en dos clics.

**Elimina superficie de ataque.** Cada contraseña que el sistema no almacena es una contraseña que no se puede filtrar. Las cuentas creadas por OAuth no tienen hash que robar.

**Aprovecha infraestructura ya prevista.** El proyecto ya tendrá un proyecto de Google Cloud por Places Autocomplete ([`08-google-maps.md`](08-google-maps.md)); el cliente OAuth vive en el mismo proyecto.

Del lado de Laravel el paquete es **Socialite**, mantenido por el equipo del framework.

## 💰 Precio y plan gratuito para desarrollo

| Concepto | Costo |
|---|---|
| **Google OAuth 2.0** | **$0** — sin cuota, sin límite de inicios de sesión, sin tarjeta |
| **Laravel Socialite** | $0 — paquete open source (MIT) |
| Verificación de marca (opcional) | $0 — trámite, no cobro |

### ⚠️ Sobre la verificación de la app — el punto que puede bloquear el lanzamiento

Google exige revisión para apps que piden **scopes sensibles** (leer Gmail, Drive, Calendar…). **Este proyecto no pide ninguno.**

Con solo `openid`, `email` y `profile` (scopes **no sensibles**):

| Estado de publicación | Consecuencia |
|---|---|
| **Testing** | ⚠️ **Máximo 100 usuarios**, y hay que registrarlos uno por uno como *test users*. Sirve para desarrollo, **no para producción** |
| **In production** | ✅ Sin límite de usuarios y **sin revisión de Google** para estos scopes |

**Publicar la app a "In production" es un cambio de estado en la consola, no una solicitud de revisión.** Es el paso que se olvida y que produce el error "esta app no está verificada / has alcanzado el límite de usuarios" en el lanzamiento.

**Verificación de marca (*brand verification*)** — trámite ligero y aparte: es lo que permite mostrar el **nombre y el logo del negocio** en la pantalla de consentimiento. Sin él la pantalla muestra el dominio en crudo, lo cual funciona pero se ve menos confiable. Conviene tramitarlo antes del lanzamiento; no bloquea el desarrollo.

## Ruta de creación

1. Ir a Google Cloud Console → **APIs y servicios → Pantalla de consentimiento de OAuth**.
2. Tipo de usuario: **External**.
3. Llenar nombre de la app, correo de soporte, logo y enlaces a política de privacidad y términos (obligatorios para publicar).
4. **Scopes:** añadir únicamente `openid`, `.../auth/userinfo.email`, `.../auth/userinfo.profile`. **No añadir ninguno más** — cualquier scope sensible dispara el proceso de revisión completo.
5. **Credenciales → Crear credenciales → ID de cliente de OAuth → Aplicación web.**
6. **URI de redirección autorizados** — exactos, sin barra final de más:
   ```
   http://localhost:8000/api/v1/auth/google/callback
   https://api.midominio.com/api/v1/auth/google/callback
   https://api-staging.midominio.com/api/v1/auth/google/callback
   ```
7. Copiar **Client ID** y **Client Secret**.
8. ⚠️ **Publicar la app**: pantalla de consentimiento → botón **"Publicar aplicación"** → estado *In production*. Sin esto queda el tope de 100 usuarios.
9. (Opcional, antes de lanzar) Solicitar la verificación de marca para que aparezcan nombre y logo.

## Contrato / plan recomendado

Sin contrato ni costo recurrente. Las únicas obligaciones son de cumplimiento, no económicas:

- Publicar una **política de privacidad** accesible en el dominio (requisito para publicar la app).
- Declarar en el aviso de privacidad que se usa autenticación de Google.
- Mantener los URI de redirección al día si cambian los dominios.

## Configuración

### Instalación
```bash
composer require laravel/socialite
```

### Variables de entorno (`.env` de Laravel)
```
GOOGLE_CLIENT_ID=<client id>
GOOGLE_CLIENT_SECRET=<client secret>
GOOGLE_REDIRECT_URI=https://api.midominio.com/api/v1/auth/google/callback

APP_FRONTEND_URL=https://midominio.com     # destino tras el callback

# Sanctum entre subdominios — sin esto la sesión no llega al frontend
SESSION_DOMAIN=.midominio.com
SANCTUM_STATEFUL_DOMAINS=midominio.com,api.midominio.com
```

⚠️ `GOOGLE_CLIENT_SECRET` **nunca** va al frontend. El flujo de redirección lo mantiene siempre en el servidor — es una de las razones para preferirlo sobre el flujo de cliente.

### `config/services.php`
```php
'google' => [
    'client_id'     => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect'      => env('GOOGLE_REDIRECT_URI'),
],
```

### Controlador — con las reglas de seguridad de la sección 7.1.1
```php
public function redirect(Request $request)
{
    // Regla 3: lista blanca — solo rutas relativas
    $to = $request->query('redirect_to', '/');
    if (! str_starts_with($to, '/') || str_starts_with($to, '//')) {
        $to = '/';
    }
    $request->session()->put('oauth_redirect_to', $to);

    return Socialite::driver('google')->redirect();   // Socialite añade el `state`
}

public function callback(Request $request)
{
    $googleUser = Socialite::driver('google')->user();

    // Regla 1: no confiar en un correo que Google no declare verificado
    if (! ($googleUser->user['email_verified'] ?? false)) {
        return redirect(config('app.frontend_url').'/login?error=email_unverified');
    }

    $user = DB::transaction(function () use ($googleUser) {
        $social = SocialAccount::firstWhere([
            'provider'         => 'google',
            'provider_user_id' => $googleUser->getId(),
        ]);

        if ($social) {
            return $social->user;
        }

        // Vinculación por correo verificado, o alta nueva
        $user = User::firstOrCreate(
            ['email' => $googleUser->getEmail()],
            [
                'name'              => $googleUser->getName(),
                'password'          => null,          // cuenta sin contraseña
                'email_verified_at' => now(),         // Google ya lo verificó
                'role_id'           => Role::guest()->id,
                'locale'            => $googleUser->user['locale'] ?? 'es',
            ]
        );

        $user->socialAccounts()->create([
            'provider'         => 'google',
            'provider_user_id' => $googleUser->getId(),
            'email'            => $googleUser->getEmail(),
            'avatar_url'       => $googleUser->getAvatar(),
        ]);

        // Enlaza con una ficha de huésped previa (reserva telefónica) o la crea
        Customer::firstOrCreate(
            ['email' => $googleUser->getEmail()],
            ['first_name' => $googleUser->getName(), 'locale' => $user->locale],
        )->update(['user_id' => $user->id]);

        return $user;
    });

    Auth::login($user, remember: true);
    $request->session()->regenerate();

    $to = $request->session()->pull('oauth_redirect_to', '/');

    return redirect(config('app.frontend_url').$to);
}
```

El `Customer::firstOrCreate` por correo es lo que hace que un huésped que reservó por teléfono recupere su historial al entrar con Google por primera vez.

### Frontend (Next.js)
```tsx
// Navegación real, NO fetch — el flujo OAuth necesita redirección del navegador
<a
  href={`${process.env.NEXT_PUBLIC_API_URL}/auth/google/redirect?redirect_to=/profile`}
  className="flex w-full items-center justify-center gap-2 rounded-xl border
             border-sand-200 px-4 py-2.5 text-sm font-medium
             hover:bg-sand-50 focus-visible:ring-2 focus-visible:ring-lagoon-400"
>
  <GoogleIcon className="h-4 w-4" />
  Continuar con Google
</a>
```

⚠️ Un `fetch()` al endpoint de redirección **no funciona**: la respuesta es un 302 hacia Google, que el navegador seguiría dentro de la petición AJAX en vez de navegar. Tiene que ser un `<a>` o `window.location.href`.

### Verificar que funciona
```bash
php artisan tinker
>>> config('services.google.redirect')   # debe coincidir EXACTO con la consola
```

El error más común es `redirect_uri_mismatch`: el URI registrado en Google y el de `.env` difieren en el esquema (`http`/`https`), en el puerto o en una barra final.

Referenciado desde: `../arquitectura/`, secciones 5.6, 6 y **7.1.1**.
