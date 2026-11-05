# Plan 6 · Despliegue y monitoreo (Etapas 8 y 9)

> **Para quien lo ejecuta (persona o agente):** seguí las tareas en orden y marcá cada casilla `- [ ]`. Si tu agente
> tiene las skills de superpowers, usá `superpowers:executing-plans`. Leé antes `../LEEME.md`, la sección 13 de
> `../especificacion.md` y las Etapas 8 y 9 de la guía de la cátedra (`pp2 devs/pp3/Guia_Detallada_PP3_v1.0.docx`).

**Objetivo:** que el servicio se pueda instalar y correr en un servidor con un solo comando, que se pueda probar contra
un MySQL con el esquema real sin depender de Desarrollo, que avise solo cuando algo anda mal, y dejar escritas las
propuestas de las Etapas 8 (despliegue) y 9 (monitoreo) que pide la cátedra.

**Arquitectura:** una imagen Docker con el servicio y tres procesos en `docker-compose.yml` (API, sincronizador y
tablero) que comparten la base propia en un volumen. Un perfil `prueba` levanta un MySQL 8 y lo llena con el mundo
simulado usando los nombres reales de Java (`volcado.py`, con `origenes/vivso3.py` del Plan 2), así el origen MySQL se
prueba de punta a punta antes de tener la contraseña de Aiven. `monitoreo.py` reúne los controles de la Etapa 9 y el
informe semanal para el área.

**Tecnología:** Docker y Docker Compose, MySQL 8 (solo en el perfil de prueba), SQLAlchemy, Streamlit Community Cloud
para la demo pública con datos simulados.

**Especificación:** `../especificacion.md`, secciones 13 (despliegue y monitoreo), 11 (variables) y 16 (limitaciones);
decisiones 1, 2, 5 y 19.

**Depende de:** el Plan 2 (sincronizador, `vivso3.py`, origen MySQL), el Plan 4 (API y `salud.py`) y el Plan 5
(tablero). Las Tareas 1 a 3 se pueden empezar en cuanto esté el Plan 2.

## Restricciones globales

- Python 3.13; comandos de PowerShell desde `vivso-analitica/`; todo en español.
- **Nunca** se vuelca nada en la base real: `volcar` se niega si la URL es la de `ANALITICA_MYSQL_URL` o si el
  servidor termina en `.aivencloud.com`, y exige `--base-de-prueba`.
- El `.env`, los certificados y `datos_locales/` nunca entran a la imagen ni al repositorio (`.dockerignore`,
  `.gitignore`). La imagen corre con un usuario sin privilegios.
- Los puertos de la API y del tablero se publican solo en `127.0.0.1`; exponerlos a una red es una decisión del lugar
  donde se instale (con acceso restringido si hay datos reales).
- La demo pública (Streamlit Community Cloud) usa **solo** el origen simulado.
- Ninguna prueba usa la red ni necesita Docker (lo que exige Docker se verifica a mano y se anota).
- Documentos sin «Plan N», «Claude», «superpowers» ni el host de Aiven; diagramas Mermaid 11; lenguaje llano.
- Commits en español, sin coautoría ni menciones a IA; suite completa en verde antes de cada commit.

## Foco de revisión

1. **`volcar` apuntado por error a la base real:** tiene que negarse antes de conectarse (prueba de la Tarea 1).
2. **Dos contenedores escribiendo la misma base SQLite** (API con `POST /v1/sincronizar` y el sincronizador): el
   candado de corridas del Plan 2 lo evita; la Etapa 8 lo documenta y recomienda sincronizar solo desde el sincronizador.
3. **El servidor se reinicia:** los servicios tienen `restart: unless-stopped` y la base propia queda en el volumen.
4. **Diciembre sin los feriados del año siguiente:** el control de feriados avisa desde el 1 de diciembre.
5. **El informe semanal con datos reales:** sale con nombres de organizaciones y números de expediente (decisión 5),
   pero nunca con datos de las familias; se comparte solo dentro del área.

## Mapa

```mermaid
flowchart TD
    T1["1 · Volcado a un MySQL<br/>de prueba"] --> T3["3 · docker-compose<br/>y perfil de prueba"]
    T2["2 · Imagen Docker"] --> T3
    T4["4 · Controles de<br/>monitoreo y salud"] --> T5["5 · Informe semanal"]
    T3 --> T6["6 · Demo pública<br/>con el simulado"]
    T3 --> T7["7 · Etapa 8:<br/>propuesta de despliegue"]
    T5 --> T8["8 · Etapa 9:<br/>propuesta de monitoreo"]
    T6 --> T7
    T7 --> T8
```

Cómo queda instalado:

```mermaid
flowchart LR
    subgraph desarrollo["Sistema de Desarrollo"]
        base[("vivso3<br/>MySQL en Aiven")]
        java["Backend Java"]
    end
    subgraph servidor["Servidor con acceso restringido"]
        sinc["sincronizador<br/>cada 30 min"] --> propia[("base propia<br/>volumen datos_locales")]
        api["API /v1<br/>127.0.0.1:8000"] --> propia
        tablero["tablero<br/>127.0.0.1:8501"] --> api
    end
    base -->|"lector, solo lectura, TLS"| sinc
    java -->|"X-API-Key"| api
    area["Personal del área"] --> tablero
    demo["Demo pública<br/>Streamlit Cloud, simulado"] -.-> publico["Cualquier persona"]
```

## Archivos

| Archivo | Responsabilidad |
| :--- | :--- |
| `analitica/volcado.py` | Escribir una `Lectura` en tablas con los nombres de Java |
| `Dockerfile`, `.dockerignore` | La imagen |
| `docker-compose.yml` | API, sincronizador, tablero y el perfil de prueba |
| `analitica/monitoreo.py` | Controles de la Etapa 9 e informe semanal |
| `analitica/__main__.py` | Comandos `volcar`, `salud` e `informe` |
| `.env.example` | `ANALITICA_PRUEBA_MYSQL_CLAVE` |
| `docs/etapa-8-despliegue.md`, `docs/etapa-9-monitoreo.md` | Documentos del entregable |

---

### Tarea 1: Volcado a un MySQL de prueba

**Archivos:** `analitica/volcado.py`, `analitica/__main__.py`, `tests/test_volcado.py`

**Interfaces:**
- `BaseRealNoPermitida(ValueError)`
- `controlar_destino(url: str, cfg: Configuracion) -> None`: lanza `BaseRealNoPermitida` si `url` es igual a
  `cfg.mysql_url` o si el servidor de la URL termina en `.aivencloud.com`.
- `volcar(lectura: Lectura, motor: Engine) -> dict[str, int]`: crea las tablas de `vivso3.METADATA` (si no existen),
  borra sus filas y escribe la lectura con los nombres de Java (al revés de `vivso3.MAPEO`); `familias` → `familia`
  (`id_familia`, `antiguedadRancho`) y `familiar` (`cantidad_familiares` filas por familia, con ids correlativos);
  booleanos → `b"\x01"` / `b"\x00"` en las columnas `bit(1)`; `etapas.numero` no se escribe. Devuelve las filas
  escritas por tabla de Java.
- `python -m analitica volcar --url URL --base-de-prueba [--hasta AAAA-MM-DD] [--semilla N]`: controla el destino,
  vuelca `OrigenSimulado(semilla, hasta).leer()` e imprime las filas por tabla. Sin `--base-de-prueba`, sale con 1.

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_ida_y_vuelta_por_las_tablas_de_java():
    lectura = OrigenSimulado(8, hasta=date(2026, 10, 15)).leer()
    motor = sqlalchemy.create_engine("sqlite://")
    volcar(lectura, motor)
    leida = OrigenMySQL(motor=motor).leer()
    for tabla in TABLAS:
        orden = list(CLAVES[tabla])
        pd.testing.assert_frame_equal(leida.tabla(tabla).sort_values(orden).reset_index(drop=True),
                                      lectura.tabla(tabla).sort_values(orden).reset_index(drop=True), check_like=False)


@pytest.mark.parametrize("url", ["mysql+pymysql://lector:x@vivsosql-ejemplo.h.aivencloud.com:18379/vivso3",
                                 "mysql+pymysql://lector:x@servidor-real:3306/vivso3"])
def test_nunca_en_la_base_real(url):
    cfg = cargar_configuracion({"ANALITICA_MYSQL_URL": "mysql+pymysql://lector:x@servidor-real:3306/vivso3"})
    with pytest.raises(BaseRealNoPermitida):
        controlar_destino(url, cfg)


def test_el_comando_exige_la_marca_de_prueba(tmp_path):
    assert main(["volcar", "--url", f"sqlite:///{tmp_path / 'p.db'}"]) == 1
    assert main(["volcar", "--url", f"sqlite:///{tmp_path / 'p.db'}", "--base-de-prueba", "--hasta", "2026-10-15"]) == 0
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Volcar el mundo simulado en tablas con los nombres de Java para probar el origen MySQL"`.

### Tarea 2: Imagen Docker

**Archivos:** `Dockerfile`, `.dockerignore`, `tests/test_despliegue.py`

**Interfaces:**
- `Dockerfile`: `FROM python:3.13-slim`; `ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
  TZ=America/Argentina/Buenos_Aires`; `WORKDIR /app`; copia e instala `requirements.txt` con `--no-cache-dir`; copia
  `analitica/`, `tablero/` y `.streamlit/`; crea el usuario `vivso` sin privilegios y la carpeta `/app/datos_locales`
  de su propiedad; `USER vivso`; `EXPOSE 8000 8501`; `CMD ["uvicorn", "analitica.api.app:app", "--host", "0.0.0.0",
  "--port", "8000"]`.
- `.dockerignore`: `.env`, `.venv/`, `datos_locales/`, `salidas/`, `.git/`, `*.pem`, `*.crt`, `*.key`, `tests/`,
  `docs/`, `__pycache__/`, `.pytest_cache/`.

- [ ] **Paso 1: pruebas que fallan** (`tests/test_despliegue.py`, solo leen los archivos):

```python
def test_la_imagen_no_lleva_secretos_ni_corre_como_root():
    ignorados = (RAIZ / ".dockerignore").read_text(encoding="utf-8").split()
    assert {".env", ".venv/", "datos_locales/", "*.pem", "*.crt", "*.key"} <= set(ignorados)
    dockerfile = (RAIZ / "Dockerfile").read_text(encoding="utf-8")
    assert "USER vivso" in dockerfile and "python:3.13-slim" in dockerfile and ".env" not in dockerfile
```

- [ ] **Pasos 2 a 4:** correr, escribir, suite en verde.
- [ ] **Paso 5 (a mano, si hay Docker):** `docker build -t vivso-analitica .` y anotar en el commit si se pudo verificar.
- [ ] **Paso 6: commit** `"Armar la imagen Docker del servicio"`.

### Tarea 3: `docker-compose.yml` y perfil de prueba

**Archivos:** `docker-compose.yml`, `.env.example`, `tests/test_despliegue.py`, `tests/test_config.py` (si la prueba de
`.env.example` necesita conocer la variable nueva)

**Interfaces** (`docker-compose.yml`):

| Servicio | Qué corre | Detalles |
| :--- | :--- | :--- |
| `api` | `uvicorn analitica.api.app:app --host 0.0.0.0 --port 8000` | `env_file: .env`, `ANALITICA_DB_URL=sqlite:////app/datos_locales/analitica.db`, volumen `./datos_locales:/app/datos_locales`, puerto `127.0.0.1:8000:8000`, `restart: unless-stopped` |
| `sincronizador` | `python -m analitica sincronizar --cada ${ANALITICA_INTERVALO_MIN:-30}` | igual que `api`, sin puertos |
| `tablero` | `streamlit run tablero/app.py --server.address 0.0.0.0 --server.port 8501` | `ANALITICA_API_URL=http://api:8000`, `env_file: .env` (para la clave de lectura), puerto `127.0.0.1:8501:8501`, `depends_on: [api]` |
| `mysql-prueba` (perfil `prueba`) | `mysql:8.4` | `MYSQL_DATABASE=vivso3`, `MYSQL_ROOT_PASSWORD=${ANALITICA_PRUEBA_MYSQL_CLAVE}`, `--sql-mode=ANSI` (como Aiven), `healthcheck` con `mysqladmin ping`, sin puertos publicados |
| `volcado` (perfil `prueba`) | `python -m analitica volcar --url mysql+pymysql://root:${ANALITICA_PRUEBA_MYSQL_CLAVE}@mysql-prueba:3306/vivso3 --base-de-prueba --hasta 2026-10-15` | `depends_on: mysql-prueba: condition: service_healthy`, sin `restart` |

- `.env.example`: sumar `ANALITICA_PRUEBA_MYSQL_CLAVE=` con un comentario («solo para el MySQL de prueba de
  `docker compose --profile prueba`; inventar una clave»).

- [ ] **Paso 1: prueba que falla:**

```python
def test_compose_publica_solo_en_la_maquina_y_tiene_el_perfil_de_prueba():
    compose = (RAIZ / "docker-compose.yml").read_text(encoding="utf-8")
    for servicio in ("api:", "sincronizador:", "tablero:", "mysql-prueba:", "volcado:"):
        assert servicio in compose
    puertos = re.findall(r'-\s*"?([\d.]+:)?(\d+):(\d+)"?', compose)
    assert puertos and all(ip == "127.0.0.1:" for ip, _, _ in puertos)
    assert compose.count("profiles:") == 2 and "--base-de-prueba" in compose
```

- [ ] **Pasos 2 a 4:** correr, escribir, suite en verde.
- [ ] **Paso 5 (a mano, si hay Docker), y anotar el resultado en `docs/etapa-8-despliegue.md`:**
  1. `docker compose --profile prueba up -d mysql-prueba` y `docker compose --profile prueba run --rm volcado`.
  2. Con `ANALITICA_ORIGEN=mysql` y `ANALITICA_MYSQL_URL=mysql+pymysql://root:<clave>@127.0.0.1:<puerto>/vivso3` (o
     desde otro contenedor de la misma red), correr `python -m analitica sincronizar`: tiene que terminar `ok` y
     `verificar_esquema()` sin diferencias. **Es la prueba de que las consultas funcionan en MySQL de verdad** (comillas,
     `bit(1)`, `datetime(6)`, modo ANSI).
  3. `docker compose up -d --build` y abrir `http://127.0.0.1:8501` y `http://127.0.0.1:8000/docs`.
- [ ] **Paso 6: commit** `"Orquestar la API, el sincronizador y el tablero, con un MySQL de prueba"`.

### Tarea 4: Controles de monitoreo y comando `salud`

**Archivos:** `analitica/monitoreo.py`, `analitica/__main__.py`, `tests/test_monitoreo.py`

**Interfaces:**
- `Control` (dataclass congelada): `nombre: str`, `estado: str` (`"ok"` o `"alerta"`), `detalle: str`, `accion: str`.
- `controles(sesion: Session, cfg: Configuracion, ahora: datetime, carpeta_feriados: Path = Path("datos_locales/feriados")) -> list[Control]`,
  en este orden:

| `nombre` | Alerta si… | `accion` |
| :--- | :--- | :--- |
| `frescura` | `estado_de_salud(...)["estado"]` no es `ok` | «Revisar que el sincronizador esté corriendo y que el origen responda» |
| `corridas_fallidas` | las últimas 3 corridas del origen terminaron en `error` | «Revisar credenciales, certificado y conexión con la base de Desarrollo» |
| `esquema` | la última corrida falló con «esquema de vivso3» en `detalle.error` | «El esquema de Desarrollo cambió: actualizar `origenes/vivso3.py` y avisarles» |
| `volumen` | el total de filas leídas cayó más de un 20 % entre las dos últimas corridas correctas | «Posible problema de permisos del usuario lector o borrado masivo: consultar a Desarrollo» |
| `huerfanas` | la cantidad de advertencias de relaciones creció entre las dos últimas corridas correctas | «Revisar con Desarrollo las filas sin su dueño» |
| `feriados` | desde el 1 de diciembre, falta `carpeta_feriados/{año + 1}.json` | «Correr `python -m analitica feriados {año + 1}`» |

- `python -m analitica salud`: imprime una línea por control (`[ok] frescura: …` o `[ALERTA] corridas_fallidas: … →
  acción`) y sale con 1 si hay alguna alerta (para el Programador de tareas de Windows o `cron`).

- [ ] **Paso 1: pruebas que fallan** (una por control, armando corridas a mano en la fixture `sesion`):

```python
def test_tres_corridas_fallidas_seguidas(sesion):
    for hora in (8, 9, 10):
        sesion.add(Corrida(origen="simulado", inicio=datetime(2026, 11, 4, hora), fin=datetime(2026, 11, 4, hora),
                           resultado="error", detalle={"error": "ConnectionError: sin conexión"}))
    sesion.commit()
    por_nombre = {c.nombre: c for c in controles(sesion, cargar_configuracion({}), datetime(2026, 11, 4, 10, 5))}
    assert por_nombre["corridas_fallidas"].estado == "alerta" and por_nombre["frescura"].estado == "alerta"


def test_caida_de_volumen(sesion):
    for hora, filas in ((8, 1000), (9, 790)):
        sesion.add(Corrida(origen="simulado", inicio=datetime(2026, 11, 4, hora), fin=datetime(2026, 11, 4, hora),
                           resultado="ok", detalle={"filas": {"solicitudes": filas}, "advertencias": []}))
    sesion.commit()
    volumen = next(c for c in controles(sesion, cargar_configuracion({}), datetime(2026, 11, 4, 9, 5)) if c.nombre == "volumen")
    assert volumen.estado == "alerta"


def test_feriados_del_anio_siguiente(sesion, tmp_path):
    estado = lambda ahora: next(c for c in controles(sesion, cargar_configuracion({}), ahora, tmp_path) if c.nombre == "feriados").estado
    assert estado(datetime(2026, 11, 30, 9)) == "ok"
    assert estado(datetime(2026, 12, 1, 9)) == "alerta"
    (tmp_path / "2027.json").write_text("[]", encoding="utf-8")
    assert estado(datetime(2026, 12, 1, 9)) == "ok"
```

- [ ] **Pasos 2 a 4:** correr, implementar (más el comando y su prueba con `capsys`), suite en verde.
- [ ] **Paso 5: commit** `"Controlar la salud del servicio para la Etapa 9"`.

### Tarea 5: Informe semanal para el área

**Archivos:** `analitica/monitoreo.py`, `analitica/__main__.py`, `tests/test_monitoreo.py`

**Interfaces:**
- `armar_informe(ctx: Contexto, ahora: datetime) -> str`: Markdown con título «Informe semanal de obras · AAAA-MM-DD»,
  «Datos al …», el aviso de datos simulados si corresponde, y una sección por tema, cada una con la cantidad y una tabla
  (`num_exp`, organización, etapa, días hábiles): obras vencidas (los tres códigos de vencida), revisiones demoradas,
  prórrogas de más de 14 días hábiles, etapas rechazadas, etapas aprobadas sin fotos en la última semana y documentos
  pendientes de revisión hace más de 7 días corridos. Una sección sin casos dice «Sin casos esta semana». Nunca datos de
  familias.
- `python -m analitica informe [--salida RUTA]`: arma el `Contexto` desde la base propia (como la API) y escribe
  `salidas/informe-AAAA-MM-DD.md` por defecto.

- [ ] **Paso 1: prueba que falla** (con `tests/fabrica.py` del Plan 3: la obra 1 de `escenario_afo()` está vencida):

```python
def test_el_informe_lista_las_obras_vencidas():
    texto = armar_informe(escenario_afo(), datetime(2026, 9, 30, 18))
    assert texto.startswith("# Informe semanal de obras · 2026-09-30")
    vencidas = texto.split("## Obras vencidas")[1].split("##")[0]
    assert "EXP-2026-0001" in vencidas and "Datos simulados" in texto
    assert "Sin casos esta semana" in texto.split("## Prórrogas de más de 14 días hábiles")[1].split("##")[0]
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Armar el informe semanal de obras para el área"`.

### Tarea 6: Demo pública con el simulado

**Archivos:** `README.md` (enlace), `requirements.txt` (sin cambios si ya tiene `streamlit`)

- [ ] **Paso 1:** comprobar que el tablero corre sin `.env` (`ANALITICA_ORIGEN` por defecto es `simulado`) con
  `.venv\Scripts\python.exe -m streamlit run tablero/app.py`.
- [ ] **Paso 2:** en Streamlit Community Cloud, nueva app desde el repositorio de `vivso-analitica`, rama `main`,
  archivo `tablero/app.py`, Python 3.13, sin secretos. Si el repositorio es privado, darle acceso a la cuenta de
  Streamlit (o dejar la demo para ejecutarla en local durante la presentación).
- [ ] **Paso 3:** abrirla, recorrer las 11 páginas y anotar el tiempo de la primera carga (si tarda mucho, la Etapa 8
  lo registra como limitación de la demo).
- [ ] **Paso 4:** sumar el enlace al README («Demo con datos simulados: …»).
- [ ] **Paso 5: commit** `"Publicar la demo del tablero con datos simulados"`.

### Tarea 7: Documento de la Etapa 8 (`docs/etapa-8-despliegue.md`)

Responde la pregunta de la guía: **«¿Qué necesitamos para que la solución pueda ser utilizada?»**, con sus elementos
obligatorios (propuesta con entorno y condiciones, recursos y condiciones de acceso y funcionamiento, limitaciones).

1. **Propuesta:** el servicio corre en un servidor del área o junto al backend de Desarrollo cuando lo desplieguen;
   lee `vivso3` con el usuario `lector` por conexión cifrada; publica la API para el backend Java y el tablero para el
   personal. Alternativa sin Docker: Windows con el entorno de Python y el Programador de tareas.
2. **Entorno y recursos:** 1 vCPU, 1 GB de memoria, 2 GB de disco; Docker; acceso de salida a Aiven; los dos puertos
   solo en la red interna.
3. **Acceso:** quién entra al tablero (personal del área, con usuario o detrás de la red interna), quién consume la API
   (el backend de Desarrollo, con su clave), quién administra (claves de administración, `.env`).
4. **Datos personales:** la base propia guarda datos personales y sensibles (nombres, DNI, domicilios, el tipo de
   documento «certificado de discapacidad»); condiciones de la Ley 25.326 (finalidad, acceso restringido, seguridad,
   quién es responsable); la demo pública nunca usa datos reales.
5. **Cómo corre y quién lo mantiene:** el diagrama de este plan; sincronización cada 30 minutos (o lo que diga el
   experimento de frecuencia); feriados una vez por año; quién queda a cargo después de PP3.
6. **Lo que ya se probó:** la imagen, el `docker-compose` con el MySQL de prueba (Tarea 3, con fecha y resultado), la
   demo pública (Tarea 6) y, si llegó, la lectura real contra Aiven.
7. **Limitaciones para implementarla:** sin backend desplegado; la contraseña y el certificado de `lector` pendientes (si
   siguen pendientes); el esquema puede cambiar sin aviso (`ddl-auto`); SQLite compartido entre contenedores; la API
   pensada para consultas de servidor a servidor; sin usuarios propios en el tablero.
8. **Pasos para que el área lo implemente**, como lista de verificación.

Diagramas: el de instalación de este plan y uno de secuencia de una actualización (sincronizador → base propia → API →
tablero).

- [ ] **Paso 1:** sumar `"docs/etapa-8-despliegue.md": 2` a `DIAGRAMAS_MINIMOS`; correr → falla.
- [ ] **Paso 2:** escribir; sumarlo al índice y al README.
- [ ] **Paso 3:** releer como alguien del área que no programa: ¿podría decidir si lo implementa?; suite en verde.
- [ ] **Paso 4: commit** `"Escribir la propuesta de despliegue de la Etapa 8"`.

### Tarea 8: Documento de la Etapa 9 (`docs/etapa-9-monitoreo.md`)

Responde **«¿Qué debemos observar para saber si la solución continúa funcionando adecuadamente?»**. La guía la marca
como condicional: el documento dice al principio en qué condición se activa (si la propuesta de la Etapa 8 se
implementa) y qué parte ya funciona (el comando `salud` y el informe se pueden correr hoy con el simulado).

1. **Qué se observa:** la tabla de controles de la Tarea 4 (qué, cómo, umbral, acción) más los datos: el indicador
   `calidad_datos` en el tiempo (obras con notificaciones incompletas, desfase, huérfanas, límites distintos de Java).
2. **Cada cuánto:** `salud` cada hora (Programador de tareas o `cron`, con el comando exacto); el informe semanal los
   lunes a las 8; la revisión de feriados en diciembre.
3. **Qué hacer ante cada alerta** y quién lo hace.
4. **Cuándo actualizar la solución:** cambios en las reglas de Java (comparar con el código de `develop`), nuevas
   tablas, cambios de criterio del área.
5. Ejemplo de salida de `salud` y un recorte del informe semanal generado con el simulado.

Diagramas: el flujo «observar → detectar → actuar» (`flowchart`) y la secuencia de una alerta (`sequenceDiagram`).

- [ ] **Paso 1:** sumar `"docs/etapa-9-monitoreo.md": 2` a `DIAGRAMAS_MINIMOS`; correr → falla.
- [ ] **Paso 2:** escribir; sumarlo al índice y al README.
- [ ] **Paso 3:** releer; suite en verde.
- [ ] **Paso 4: commit** `"Escribir la propuesta de monitoreo de la Etapa 9"`.

---

## Decisiones para tomar al revisar

1. **Dónde se instalaría:** servidor del área, junto al backend de Desarrollo o solo como propuesta (recomendado:
   preguntarlo al área en la presentación; la Etapa 8 se cumple con la propuesta).
2. **Demo pública:** Streamlit Community Cloud con el simulado (recomendado) o solo en local durante la presentación.
3. **Repositorio público o privado:** si es privado, la demo pública necesita darle acceso a Streamlit; decidirlo con
   el grupo.
