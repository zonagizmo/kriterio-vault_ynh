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

## Pendiente antes de poder instalarlo

Hecho: repo `kriterio-vault_ynh` creado y con el código; repo principal
`kriterio-vault` hecho público; checksum del tarball calculado y puesto en
`manifest.toml`.

Queda:

1. **Instalarlo en un YunoHost real** con `--debug` (ver más abajo) — no
   probado todavía contra ninguna instancia, es el paso que más puede
   necesitar ajustes.
2. **Recalcular el checksum tras cada cambio en el repo principal** (apunta
   a la rama `main`, no a un tag fijo — ver el comentario en `manifest.toml`):
   ```bash
   curl -sL https://github.com/zonagizmo/kriterio-vault/archive/refs/heads/main.tar.gz | sha256sum
   ```
3. **Licencia del paquete** (`manifest.toml` tiene `license = "free"` como
   placeholder — válido para uso privado; cámbialo por un identificador SPDX
   si algún día lo publicas en el catálogo oficial de apps de YunoHost).

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
