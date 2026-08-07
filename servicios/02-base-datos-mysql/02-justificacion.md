# Justificación

- Transaccional (ACID), crítico para evitar doble-booking en `bookings`/`availability`.
- Ecosistema maduro con Laravel/Eloquent (migraciones, seeders, ya es tu stack actual en `notasCreditos`).
- Dos modalidades posibles:

| Modalidad | Cuándo usarla |
|---|---|
| **Self-hosted en el mismo VPS (Docker)** | Fase inicial (100–1,000 usuarios); menor costo, tú controlas backups |
| **Gestionado (ej. DigitalOcean Managed MySQL, Amazon RDS)** | A partir de crecimiento medio/alto; failover automático, backups gestionados, réplicas de lectura con 1 clic |

**Recomendación:** iniciar self-hosted en Docker dentro del mismo VPS de Hetzner (ver `01-hosting-vps.md`) y migrar a un servicio gestionado cuando se llegue a la fase de escalado (sección 11 del doc principal, >10,000 usuarios).

