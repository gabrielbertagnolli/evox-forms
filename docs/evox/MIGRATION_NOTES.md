# Evox Formbricks - Notas de Migración a Producción

Este documento detalla los pasos y precauciones para el despliegue final en Coolify (Lane 5), basado en el éxito del ensayo (sandbox local).

## 1. Datos y Volúmenes
Los datos originales residen en tres volúmenes clave de Docker extraídos del VPS antiguo:
- `formbricks_formbricks_pg` (Base de datos PostgreSQL)
- `formbricks_formbricks_minio_data` (Archivos S3/MinIO)
- `formbricks_formbricks_uploads` (Archivos locales en caso de fallback)

**Para Coolify:**
Si vas a montar los volúmenes externos en el nuevo servidor, asegúrate de utilizar los nombres referenciados en el `docker-compose.coolify.yml` (e.g. `formbricks_pg`, `formbricks_minio_data`). Deberás copiar el contenido de los directorios `_data` del backup original hacia el path de almacenamiento del nuevo nodo Coolify, asegurando que los permisos de Linux para la base de datos sigan siendo válidos (UID 999 para Postgres).

## 2. Variables de Entorno y Secrets Críticos

**NUNCA pegues valores reales de secretos en este archivo ni en ningún archivo
versionado en git — este repositorio es la fuente de verdad del código, no un
lugar para guardar contraseñas.** Los valores reales viven únicamente en:
`docs/evox/.env.sandbox` (local, gitignored, solo para el ensayo), o
directamente en la UI de variables de entorno de Coolify (producción).

Durante el ensayo (Lane 4) se confirmó, arrancando la app contra una copia del
volumen de Postgres original, que las siguientes variables **deben** ser las
mismas que en la instalación vieja — regenerarlas vuelve indescifrables los
datos ya cifrados en la base de datos restaurada:

- `NEXTAUTH_SECRET`
- `ENCRYPTION_KEY`
- `CRON_SECRET`

Origen de los valores reales: `.env` de la instalación vieja (el operador ya
lo tiene). Cárgalos en Coolify como variables de entorno del servicio
`formbricks`, nunca en un archivo commiteado.

## 3. Storage S3 / Minio
El servidor original dependía de un contenedor MinIO interno.
En Coolify, mantendremos esta estructura.
- **Hallazgo importante del ensayo (Lane 4):** si vas a reutilizar el volumen
  `formbricks_minio_data` original tal cual (copiado, no vacío), `S3_ACCESS_KEY`
  y `S3_SECRET_KEY` **deben coincidir exactamente con las credenciales root
  originales de MinIO**. MinIO persiste su identidad de usuario root dentro del
  propio volumen (`.minio.sys`) la primera vez que arranca; si le pasás
  credenciales nuevas contra un volumen que ya tiene un usuario root
  configurado, MinIO sigue esperando las credenciales originales y la app no
  podrá autenticarse contra el storage. Esto es la excepción a la regla
  general de "las credenciales de DB/S3 se pueden regenerar libremente" del
  `docs/evox/env.template` — aplica solo porque en este deploy se reutiliza el
  volumen real, no uno vacío.
- El servicio en Docker Compose se llama `formbricks-minio` y está en el puerto interno 9000. Formbricks usa internamente `S3_ENDPOINT_URL=http://formbricks-minio:9000`.

## 4. Dominios
Coolify maneja la redirección inversa (proxy) a través de Traefik/Caddy nativamente.
- Enlaza tu dominio `form.evox.com.uy` al servicio web de `formbricks` en el puerto `3000`.
- Enlaza tu dominio `files.evox.com.uy` al servicio `formbricks-minio` en el puerto `9000`.

## 5. Whitelabel
Las directivas de Whitelabel fueron incrustadas en el código (hardcoded en ciertos archivos y eliminadas del logo). La propiedad de frontend `NEXT_PUBLIC_HIDE_BRANDING=true` complementará el trabajo realizado, pero ya no dependes de una licencia de pago para quitar el logo, ya que el código del footer y loader fue modificado en la imagen compilada.
