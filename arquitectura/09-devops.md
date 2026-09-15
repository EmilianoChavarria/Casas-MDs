# 9. DevOps


### 9.1 Git Flow simplificado

```
main        → producción (protegida, solo vía PR + CI verde)
develop     → integración / QA
feature/*   → features individuales
hotfix/*    → fixes urgentes desde main
```

### 9.2 Ambientes

| Ambiente | Propósito | Deploy |
|---|---|---|
| Local | Docker Compose local | manual |
| Desarrollo | rama `develop`, DB de prueba | automático al hacer push a `develop` |
| QA | mirror de prod con datos anonimizados | automático al mergear a `qa` (opcional) |
| Producción | rama `main` | automático tras PR aprobado + CI verde |

### 9.2.1 Entorno local hoy (sin Docker)

La tabla dice Docker Compose, pero hoy el desarrollo corre con los servicios **nativos en Windows**. Para probar el sistema completo tienen que estar arriba **todos** estos procesos; faltar uno no da error, solo hace que algo "no pase":

| Proceso | Comando | Si no corre… |
|---|---|---|
| MySQL | servicio en `3306` | nada funciona |
| API | `php artisan serve --port=8000` | el front no carga datos |
| Cola | `php artisan queue:work --tries=3` | no sale ningún correo, no se traduce nada, el chat no se difunde |
| Scheduler | `php artisan schedule:work` | no vencen los apartados, no salen recordatorios ni la lista del guía |
| WebSockets | `php artisan reverb:start --port=8080` | el chat no llega en tiempo real (el invitado sigue con polling) |
| Front | `npm run dev` (Next, `3000`) | — |
| Correo | `mailpit.exe` (SMTP `1025`, bandeja http://localhost:8025) | los correos fallan al enviarse (ver `servicios/07`) |
| Webhooks de Stripe | `stripe listen --forward-to http://127.0.0.1:8000/api/v1/webhooks/stripe` | **se paga en Stripe pero la reserva se queda "Pendiente de pago"** hasta vencer |

Notas que ya costaron tiempo:

- ⚠️ **El worker no ve cambios de código ni de `.env`**: carga todo al arrancar. Después de cambiar cualquiera de los dos, `php artisan queue:restart` y volver a levantarlo. Un worker viejo ejecuta código que ya no existe y falla con errores que no corresponden al código actual.
- **`stripe listen` imprime el secreto** del webhook; tiene que coincidir con `STRIPE_WEBHOOK_SECRET` (`stripe listen --print-secret`). Si un pago se hizo sin el listener, se recupera reenviando el evento: `stripe events resend evt_…` (el de `checkout.session.completed` de esa sesión).
- **Un solo Mailpit.** Con dos instancias los correos llegan a una y el navegador abre la otra.
- En producción nada de esto se levanta a mano: cola, scheduler y Reverb van bajo Supervisor (sección 8) y el webhook se registra en el panel de Stripe con su propio secreto.
### 9.3 GitHub Actions (esqueleto, reutilizando tu experiencia con FTP/CI ya construida)

```yaml
name: Deploy Backend
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: composer install && php artisan test
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/backend
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            sudo supervisorctl restart all
```

A diferencia de tu pipeline actual por FTP (cPanel shared hosting), aquí al tener VPS con acceso SSH puedes hacer deploy por `git pull` + `artisan migrate`, mucho más robusto que sincronizar archivos por FTP.

**Hoy (15-sep-2026), en cada PR a `main` o `develop`:**

| Repo | Pasos | Bloquea el merge si… |
|---|---|---|
| `Casas_back` | Pint · `composer audit` · migraciones · PHPUnit contra MySQL 8 | hay formato pendiente, una dependencia con vulnerabilidad conocida o una prueba roja |
| `Casas_front` | `tsc --noEmit` · ESLint · `next build` | hay error de tipos o de lint, o el build falla |

- El build del front corre **sin backend** (`NEXT_PUBLIC_API_URL` apunta a un puerto cerrado): comprueba que ninguna página dependa de la API al compilar.
- ⚠️ Tres reglas del React Compiler (`set-state-in-effect`, `immutability`, `purity`) van en **aviso** hasta migrar las cargas con `useEffect` a react-query (`Casas_front#48`). Todo lo demás es error.
- El deploy automático por SSH de arriba sigue pendiente: necesita el VPS. Mientras tanto, el despliegue es el de `deploy/README.md`.

---

