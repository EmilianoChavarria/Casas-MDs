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

---

