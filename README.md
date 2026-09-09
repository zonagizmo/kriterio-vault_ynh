# kriterio-vault_ynh

Paquete YunoHost para el servidor central de sincronización de
[Kriterio Vault](https://github.com/zonagizmo/kriterio-vault). Es la vía
**alternativa** al despliegue Debian a pelo (`deploy/` en el repo principal)
— sirven para instancias distintas, no se sustituyen entre sí.

## Estado: escrito, sin probar contra una instancia YunoHost real

No he tenido acceso a un YunoHost real para validarlo con
`yunohost app install --debug`. Está basado en las convenciones de
`packaging_format = 2` / helpers 2.1, pero nombres exactos de helpers o
comportamientos de detalle pueden haber cambiado según la versión de
YunoHost del servidor destino. **Instálalo primero en modo debug y revisa
el log paso a paso** antes de darlo por bueno.

## Por qué es un repositorio aparte

`yunohost app install <url>` clona el repo y busca `manifest.toml` **en la
raíz**. El código de la app vive en `kriterio-vault` (repo principal); este
paquete solo lo *envuelve* (nginx, systemd, Postgres, .env) y le indica de
dónde bajar el código real (`resources.sources.main.url` en
`manifest.toml`). Es el patrón estándar de YunoHost: `<app>` + `<app>_ynh`
como dos repos separados.

## Pendiente antes de poder instalarlo (no lo puedo hacer yo)

1. **Crear este repositorio en GitHub** (`kriterio-vault_ynh` o el nombre
   que prefieras) y subir esta carpeta — igual que hicimos con el repo
   principal: lo creas tú en github.com, me pasas la URL SSH, y hago el
   `git init` + push.
2. **Hacer público el repo principal `kriterio-vault`** (o al menos la rama
   `main`): `ynh_setup_source` descarga el tarball de GitHub, y GitHub no
   sirve tarballs de repos privados sin autenticación. Es la decisión que ya
   tomamos al hablar de esto — revisa antes que no haya nada sensible en el
   historial de commits (no debería: `.gitignore` excluye bases de datos,
   backups y `.env` desde el primer commit).
3. **Calcular el checksum real** del tarball, una vez el repo sea público:
   ```bash
   curl -sL https://github.com/zonagizmo/kriterio-vault/archive/refs/heads/main.tar.gz | sha256sum
   ```
   y pegar el resultado en `manifest.toml`, campo `resources.sources.main.sha256`
   (hoy tiene el placeholder `REEMPLAZAR_TRAS_PUBLICAR_EL_REPO`).
4. **Decidir la licencia** del paquete (`manifest.toml` tiene `license = "free"`
   como placeholder — cámbialo por un identificador SPDX si vas a publicarlo
   en el catálogo oficial de apps de YunoHost; si es solo para tu propio uso,
   puedes dejarlo así).

## Instalación (una vez resuelto lo anterior)

```bash
yunohost app install https://github.com/<tu-usuario>/kriterio-vault_ynh --debug
```

Te pedirá dominio, ruta (usa `/`) y un usuario YunoHost como "admin" de la
app (no se usa para nada especial hoy, es un requisito del formulario). El
permiso de acceso por defecto es `all_users`: solo usuarios de ese YunoHost,
logueados vía SSO, no acceso público — es la protección que decidimos usar
en vez de escribir autenticación propia.

Después de instalarlo, da de alta la primera instalación cliente por SSH en
el propio servidor:

```bash
sudo -u kriteriovault /var/www/kriteriovault/backend/venv/bin/python \
    /var/www/kriteriovault/backend/scripts/crear_instalacion.py --nombre "PC Oficina"
```

(La ruta exacta de `install_dir` te la dice `yunohost app info kriteriovault`.)

## Diferencias con `deploy/` (Debian a pelo)

| | `deploy/` (Debian) | Este paquete (YunoHost) |
|---|---|---|
| Proxy/TLS | Caddy, dominio propio | NGINX + Let's Encrypt gestionados por YunoHost |
| Autenticación | Ninguna (solo la clave de API entre instalaciones) | + SSO de YunoHost delante de toda la app |
| Backups | Script propio (`backups/`, retención 30 días) | Integrado con `yunohost backup` |
| Actualización | `deploy/actualizar.sh` | `yunohost app upgrade kriteriovault` |

Ambos caminos comparten exactamente el mismo código de aplicación
(`app/api/sync.py`, `app/services/sync_replay.py`, etc.) — solo cambia cómo
se despliega y se protege.
