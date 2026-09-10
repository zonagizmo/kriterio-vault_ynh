# kriterio-vault_ynh

Paquete YunoHost para el servidor central de sincronización de
[Kriterio Vault](https://github.com/zonagizmo/kriterio-vault). Es la vía
**alternativa** al despliegue Debian a pelo (`deploy/` en el repo principal)
— sirven para instancias distintas, no se sustituyen entre sí.

## Estado: en pruebas contra una instancia YunoHost real (arm64, bookworm)

Dos intentos de instalación real hasta ahora, cada uno encontró un bug ya
corregido:

1. **`ynh1`** — al script `install` le faltaba `ynh_setup_source`
   (`resources.sources` solo precarga/verifica el tarball, no lo coloca en
   `$install_dir`; eso lo hace el propio script). Corregido.
2. **`ynh2`** — dos bugs a la vez:
   - Ningún script (`install/upgrade/remove/backup/restore`) llamaba a
     `ynh_abort_if_errors` tras `source /usr/share/yunohost/helpers`. Sin
     eso el script no aborta cuando un comando falla, así que cuando
     `pip install` no pudo instalar `psycopg2-binary` el script siguió
     igualmente hasta el final, dejó el venv roto, y el fallo real quedó
     enterrado bajo un timeout de 5 minutos de `systemd` reiniciando el
     servicio en bucle. Añadido a los 5 scripts.
   - `psycopg2-binary` no tenía wheel precompilada para esta plataforma y
     pip necesitaba compilarlo desde código fuente, lo que requiere
     `libpq-dev` (cabeceras de PostgreSQL) — no estaba en `resources.apt`.
     Añadido.
3. **`ynh3`** — con los dos fixes de `ynh2` puestos, el segundo intento real
   volvió a fallar exactamente igual: `ModuleNotFoundError: No module named
   'psycopg2'`. La causa esta vez era mucho más tonta: **el repo principal
   `kriterio-vault` nunca se había empujado a GitHub** con el código de
   sincronización — el commit llevaba desde la sesión anterior solo en
   local. `resources.sources` descarga el tarball de la rama `main` de
   GitHub, así que la instalación seguía trayéndose la versión antigua
   `v1.07.00`, cuyo `requirements.txt` ni siquiera tiene `psycopg2-binary`
   (por eso `pip install` no daba ningún error: instalaba todo lo que
   *sí* estaba en esa lista). `libpq-dev` y `ynh_abort_if_errors` eran
   correcciones legítimas pero no la causa de este segundo fallo. Empujado
   el repo principal y recalculado el checksum en `manifest.toml`.
4. **`v1.10.1~ynh1`** — con todo lo anterior corregido, el tercer intento
   real llegó mucho más lejos (`pip install` instaló `psycopg2-binary` sin
   problema, systemd arrancó el proceso) pero la app crasheaba en el
   arranque: `psycopg2.errors.SyntaxError: syntax error at or near
   "PRAGMA"` en `_migraciones()` (`backend/app/main.py`). Esa función usa
   `PRAGMA table_info(...)`, sintaxis propia de SQLite, para poner al día
   esquemas SQLite antiguos — no tiene sentido y directamente no funciona
   contra Postgres. Como `crear_tablas()` ya crea el esquema completo
   actual en cualquier base de datos nueva, se corrigió para que
   `_migraciones()` se salte entera cuando el dialecto no es SQLite (bug
   de la app, corregido en el repo principal, versión de la app
   `1.10.00` → `1.10.01`). Checksum recalculado, versión de paquete
   `1.10.0~ynh3` → `1.10.1~ynh1` (a partir de aquí el número de paquete
   sigue el de la app, no un contador de intentos de packaging).

Con `ynh_abort_if_errors` en su sitio, cualquier fallo futuro debería
detener el script de inmediato con un mensaje claro, en vez de repetir este
patrón. **Pendiente: confirmar que la instalación con `1.10.1~ynh1`
completa correctamente.**

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

1. **Confirmar que la instalación con `1.10.1~ynh1` completa correctamente**
   en el YunoHost real de pruebas (ver sección "Estado" arriba) — si vuelve
   a fallar, revisar el log con `--debug` en vez de asumir que estos fixes
   fueron los únicos problemas. **Antes de cada intento, asegúrate de que
   el repo principal `kriterio-vault` está empujado a GitHub** (`git push`
   en `GestionMGD_web`) — el fallo de `ynh3` fue justo no haberlo hecho.
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
