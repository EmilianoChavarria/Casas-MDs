# 💰 Precio y plan gratuito para desarrollo

Cloudflare R2 tiene un **free tier permanente** (no es una prueba por tiempo limitado):

| Recurso | Incluido gratis cada mes | ¿Sirve para desarrollo? |
|---|---|---|
| Storage (Standard) | 10 GB-mes | ✅ Sí, de sobra para un catálogo de propiedades en desarrollo |
| Class A operations (escrituras: subir/listar) | 1,000,000 /mes | ✅ Sí |
| Class B operations (lecturas) | 10,000,000 /mes | ✅ Sí |
| Egress (transferencia de salida) | Siempre $0, incluso fuera del free tier | ✅ Sí |

**Recomendación:** usar la misma cuenta/bucket de R2 para desarrollo (con un bucket separado, ej. `rentas-casas-media-dev`) sin preocuparse por costo — es muy difícil salir del free tier en fase de desarrollo.

