# b3l0w-seanime

Imagen propia de [Seanime](https://github.com/5rahim/seanime) con configuración por defecto incluida: extensiones instaladas, player web y actualizaciones automáticas deshabilitadas.

Alpine 3.24 · multi-stage · binario estático · rootless · ~323 MB.

## Qué incluye

| | |
|---|---|
| **Seed** | `seed/` se copia a `/data` la primera vez y nunca más |
| **Extensiones** | 8 plugins preinstalados (AniList, torrent providers, etc.) |
| **Player** | `web` — reproduce en el navegador, sin binarios externos |
| **Updates** | `disable_update_check = 1` forzado en cada arranque |
| **Rootless** | `USER 1000` |

El seed está saneado: sin cuentas, sin credenciales de debrid, sin biblioteca.

## Archivos

| Archivo | Para qué |
|---|---|
| `Dockerfile` | imagen multi-stage: web → Go → Alpine |
| `compose.yaml` | uso final: baja la imagen publicada de GHCR |
| `seed/` | DB, configuración y extensiones que se siembran la primera vez |
| `.github/workflows/image.yml` | build amd64, smoke test y push a GHCR |
| `.github/workflows/image-arm64.yml` | build nativo arm64 |

No hay compose de build local. La imagen se construye en GitHub Actions usando el
source oficial de Seanime como contexto; el Dockerfile vive en este repo.

## Usar la imagen publicada

```bash
mkdir -p data anime downloads
export SEANIME_VERSION=3.10.4
docker compose pull
docker compose up -d
docker compose ps
```

Primera ejecución: el entrypoint copia `seed/` a `data/` y crea `data/.seeded`.

```text
http://localhost:43211
```

Tags publicados en GHCR:

| Tag | Uso |
|---|---|
| `3.10.4` | versión fija; es el contrato para el usuario final |
| `latest` | alias de la última versión probada |
| `sha-<commit>` | trazabilidad exacta del build |

`latest` y `3.10.4` apuntan al mismo digest en la publicación actual.

## Volúmenes

| Host | Container | Contenido |
|---|---|---|
| `./data` | `/data` | DB, config, extensiones, cache |
| `./anime` | `/anime` | Biblioteca |
| `./downloads` | `/downloads` | Descargas en curso |

Los tres son bind mounts: `docker compose down` no borra nada.

## Configuración por defecto

El seed trae:

```sql
library_path         = '/anime'
default_player       = 'web'
disable_update_check = 1
download_dir         = '/downloads'
```

Para cambiarlos antes de la primera ejecución, editá `seed/seanime.db`:

```bash
sqlite3 seed/seanime.db "UPDATE settings SET default_player = 'native';"
```

## Actualizar

El workflow publica tags por versión y por SHA. Para actualizar, cambiá la versión
en `.env` y bajá la imagen:

```bash
export SEANIME_VERSION=3.11.0
docker compose pull
docker compose up -d
```

Las actualizaciones **no** se hacen desde la UI: `disable_update_check` está
forzado en el entrypoint para evitar que el self-updater escriba en la capa
efímera (y la actualización se pierda al recrear el container).

## Publicar una versión

Desde GitHub Actions, ejecutar `seanime-image` con el tag de upstream (sin la
`v`). El workflow:

1. clona `5rahim/seanime` en el tag indicado;
2. construye el `Dockerfile` de este repo con `seed/` como contexto adicional;
3. hace un smoke test del servidor;
4. publica `3.10.4`, `sha-<sha>` y `latest` en GHCR.

Para arm64, ejecutar `seanime-image-arm64`; publica tags `arm64-<version>` y
`arm64-sha-<sha>`.

## Backup

```bash
docker run --rm --user "$(id -u):$(id -g)" \
  -v ./data:/data:ro \
  -v .:/backup \
  alpine tar czf /backup/seanime-data-$(date +%F).tar.gz -C /data .
```

## Estructura

```text
b3l0w-seanime/
├── Dockerfile
├── compose.yaml           pull-only (GHCR)
├── seed/
│   ├── seanime.db        settings saneadas
│   ├── config.toml
│   ├── entrypoint.sh     siembra + bloqueo de updates
│   └── extensions/       8 plugins JSON
├── .github/workflows/    build amd64 y arm64
├── data/                 (gitignored)
├── anime/                (gitignored)
└── downloads/            (gitignored)
```

## Pitfalls conocidos

| Problema | Causa | Solución |
|---|---|---|
| El seed no copia nada | `cp -rn` de BusyBox devuelve 0 pero no copia | El entrypoint itera con `[ ! -e "$dest" ]` |
| Descargas desaparecen | `download_dir` vacío → `/tmp` | El seed ya lo fija en `/downloads` |
| El build de CI no encuentra el seed | El contexto es el source de Seanime | El workflow pasa `seed=./seed` como build context |
| La imagen no actualiza desde la UI | El updater escribe en la capa efímera | `disable_update_check = 1` |

## Licencia

El Dockerfile, el compose y el entrypoint de este repo son MIT.

Seanime es **GPL-3.0** — el binario compilado y el source derivan de
[5rahim/seanime](https://github.com/5rahim/seanime).
