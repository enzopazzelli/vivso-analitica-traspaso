# Plan 2 · Base propia, sincronizador, historia y origen MySQL

> **Para quien lo ejecuta (persona o agente):** seguí las tareas en orden y marcá cada casilla `- [ ]`. Si tu agente
> tiene las skills de superpowers, usá `superpowers:subagent-driven-development` (recomendado) o
> `superpowers:executing-plans`. Leé antes `../LEEME.md` y las secciones 6, 7 y 11 de `../especificacion.md`.

**Objetivo:** que el servicio guarde en su propia base cada lectura del origen como versiones, deduzca los eventos que
el sistema Java no guarda, arme la línea de tiempo de cada expediente, lea la base real de Desarrollo (MySQL) sin
tocarla, y medir cuánto de la historia se recupera según cada cuánto se sincroniza.

**Arquitectura:** una base propia (SQLite por defecto, cualquier motor de SQLAlchemy) con tres tablas: `corrida`,
`analitica_version` y `analitica_evento`. `sincronizar()` lee el origen, compara cada fila con su versión vigente por
huella, cierra y abre versiones, deriva eventos con funciones puras y deja todo en una sola transacción. La historia de
etapas de las notificaciones (Plan 1b) se combina con estos eventos en `instantanea.py`. Las tablas de Java se
describen una sola vez en `origenes/vivso3.py`; de ahí salen las consultas del origen MySQL y el esquema de prueba.

**Tecnología:** SQLAlchemy 2 (ORM y Core), PyMySQL, pandas; pytest con SQLite en carpetas temporales (sin red).

**Especificación:** `../especificacion.md`, secciones 6.2, 6.3, 7 (7.1 a 7.5), 11 y criterio 2 de la Etapa 6.

**Depende de:** el Plan 1b (`esquema.CLAVES`, `esquema.clave_de`, `historia.py`, la tabla `notificaciones`).
**Lo necesitan:** el Plan 4 (API), el Plan 6 (despliegue) y el Plan 7 (evaluación).

## Restricciones globales

- Python 3.13 con `.venv`; comandos de PowerShell desde `vivso-analitica/`; todo en español.
- **El servicio nunca escribe en `vivso3`:** solo `SELECT`, en una transacción de solo lectura, nunca `SELECT *`.
- Ninguna prueba usa la red; la base propia de las pruebas es SQLite en `tmp_path`.
- La contraseña de MySQL y la ruta del certificado salen solo de `ANALITICA_MYSQL_URL` y `ANALITICA_MYSQL_SSL_CA`; nunca
  aparecen en mensajes, registros ni en `detalle` de una corrida (usar `config._sin_clave` para mostrar una URL).
- Fechas y horas: siempre hora de Buenos Aires sin zona (`calendario.ahora_local()`), sin microsegundos.
- Un origen por base: datos de `simulado` y de `mysql` nunca se mezclan.
- El núcleo no importa `fastapi` ni `streamlit`.
- Documentos sin «Plan N», «Claude», «superpowers» ni el host de Aiven; diagramas Mermaid 11.
- Commits en español, sin coautoría ni menciones a IA; suite completa en verde antes de cada commit.

## Foco de revisión

1. **Una corrida que falla a la mitad** (por ejemplo, un error al escribir versiones): no puede quedar una corrida `ok`
   con versiones a medias. Todo lo de los pasos 3 a 5 va en una transacción; la corrida queda `error` con el detalle.
2. **Corrida abandonada** (el proceso se cortó y quedó `en_curso` hace más de 10 minutos): la siguiente la cierra como
   `error` («abandonada») y sigue; si es de hace menos, aborta con «corrida en curso».
3. **Valores vacíos que cambian de tipo** (un `NaN` de pandas contra un `None`, un `Int64` contra un `int`): la huella
   tiene que ser la misma; si no, cada corrida crearía versiones nuevas de todo.
4. **La base real con una columna menos o una tabla opcional ausente** (`visitaObra`, `notificacion`): error claro con
   la lista de lo que falta, o tabla vacía si es opcional; nunca una excepción de SQL cruda.
5. **Una URL de MySQL con contraseña en un mensaje de error:** ningún mensaje, ni el `detalle` de la corrida, la muestra.

## Mapa

```mermaid
flowchart TD
    T1["1 · Base propia<br/>corrida, versión, evento"] --> T2["2 · Versiones<br/>normalizar y huella"]
    T2 --> T3["3 · Eventos de etapas"]
    T3 --> T4["4 · Eventos del resto"]
    T4 --> T5["5 · Sincronizador"]
    T5 --> T6["6 · Instantánea<br/>y línea de tiempo"]
    T5 --> T7["7 · Comandos<br/>sincronizar y vaciar"]
    T8["8 · Tablas de Java<br/>vivso3.py"] --> T9["9 · Origen MySQL"]
    T9 --> T7
    T9 --> T10["10 · Origen API REST<br/>(opcional)"]
    T5 --> T11["11 · Experimento<br/>de frecuencia"]
    T6 --> T12["12 · Documento<br/>sincronizacion.md"]
    T11 --> T12
```

Una corrida (sección 7.2 de la especificación):

```mermaid
sequenceDiagram
    participant C as Comando o API
    participant S as sincronizar()
    participant O as Origen
    participant B as Base propia
    C->>S: sincronizar(origen, sesión, ahora)
    S->>B: ¿corridas de otro origen? ¿una en curso hace < 10 min?
    alt hay otro origen o una en curso
        S-->>C: error, sin escribir nada
    else sigue
        S->>B: corrida en_curso (commit)
        S->>O: leer()
        alt el origen falla
            S->>B: corrida error con el detalle
        else lectura válida
            S->>B: versiones nuevas y cerradas (una transacción)
            S->>B: eventos derivados
            S->>B: corrida ok o sin_cambios
        end
        S-->>C: ResultadoCorrida
    end
```

## Archivos

| Archivo | Responsabilidad |
| :--- | :--- |
| `analitica/base_de_datos.py` | Motor, sesiones, las tres tablas propias, `vaciar` |
| `analitica/versiones.py` | Normalizar una fila a JSON, huella, volver de JSON a tipos de Python |
| `analitica/eventos.py` | `derivar()`: eventos que surgen de comparar dos versiones (puro) |
| `analitica/sincronizador.py` | Una corrida completa |
| `analitica/instantanea.py` | Lo vigente en la base: lectura, eventos guardados, línea de tiempo |
| `analitica/origenes/vivso3.py` | Las tablas de Java con sus nombres reales (SQLAlchemy Core) y el mapeo a las nuestras |
| `analitica/origenes/mysql.py` | `OrigenMySQL` |
| `analitica/origenes/api_rest.py` | `OrigenApiRest` (opcional) |
| `analitica/origenes/simulado.py` | `OrigenSimulado.al_corte(corte)` |
| `analitica/origenes/fijo.py` | `OrigenFijo`: un origen que devuelve siempre la misma lectura |
| `analitica/origenes/simulado_proyeccion.py` | `proyectar_por_cortes(...)` para el experimento |
| `analitica/evaluacion/__init__.py`, `analitica/evaluacion/frecuencia.py` | Experimento de frecuencia |
| `analitica/__main__.py` | Comandos `sincronizar`, `vaciar`, `experimento-frecuencia` |
| `tests/ayudas_origenes.py` | `OrigenQueFalla` para las pruebas |
| `docs/sincronizacion.md` | Documento del entregable |

---

### Tarea 1: Base propia

**Archivos:** `requirements.txt`, `analitica/base_de_datos.py`, `tests/test_base_de_datos.py`

**Interfaces:**
- `crear_motor(url: str) -> sqlalchemy.Engine`: si es `sqlite:///ruta`, crea la carpeta de la ruta.
- `crear_tablas(motor) -> None` (idempotente; `Base.metadata.create_all`).
- `fabrica_de_sesiones(motor) -> sqlalchemy.orm.sessionmaker`
- `vaciar(motor) -> dict[str, int]`: borra todas las filas de las tres tablas (eventos, versiones, corridas, en ese
  orden) y devuelve cuántas borró de cada una.
- Clases ORM (`DeclarativeBase` de SQLAlchemy 2, `Mapped[...]`):

| Clase · tabla | Columnas |
| :--- | :--- |
| `Corrida` · `corrida` | `id` (int, PK), `origen` (str 20), `inicio` (DateTime), `fin` (DateTime, nulo), `resultado` (str 20: `en_curso`, `ok`, `sin_cambios`, `error`), `detalle` (JSON, `{}` por defecto) |
| `Version` · `analitica_version` | `id`, `entidad` (str 40), `clave` (str 200), `huella` (str 64), `datos` (JSON), `vigente_desde` (DateTime), `vigente_hasta` (DateTime, nulo), `id_corrida` (FK `corrida.id`); índice (`entidad`, `clave`, `vigente_hasta`) |
| `Evento` · `analitica_evento` | `id`, `entidad`, `clave`, `id_solicitud` (int, nulo), `numero` (int, nulo), `tipo` (str 40), `ronda` (int, nulo), `momento` (DateTime), `precision` (str 10), `fuente` (str 15), `detectado_en` (DateTime), `id_corrida` (FK), `detalle` (JSON); índice (`id_solicitud`, `momento`) |

- [ ] **Paso 1:** sumar a `requirements.txt` `SQLAlchemy` (2.x) y `PyMySQL` (1.x), instalar con
  `.venv\Scripts\python.exe -m pip install -r requirements-dev.txt` y fijar con `==` las versiones que quedaron
  (`pip freeze`), con un comentario por línea como el resto.
- [ ] **Paso 2: pruebas que fallan** (`tests/test_base_de_datos.py`):

```python
def test_crea_la_carpeta_y_las_tablas(tmp_path):
    motor = crear_motor(f"sqlite:///{tmp_path / 'sub' / 'analitica.db'}")
    crear_tablas(motor)
    crear_tablas(motor)                                      # idempotente
    assert set(sqlalchemy.inspect(motor).get_table_names()) == {"corrida", "analitica_version", "analitica_evento"}


def test_guardar_y_vaciar(tmp_path):
    motor = crear_motor(f"sqlite:///{tmp_path / 'a.db'}")
    crear_tablas(motor)
    with fabrica_de_sesiones(motor)() as sesion:
        corrida = Corrida(origen="simulado", inicio=datetime(2026, 11, 4, 10), resultado="ok", detalle={"filas": {}})
        sesion.add(corrida)
        sesion.flush()
        sesion.add(Version(entidad="usuarios", clave="1", huella="x" * 64, datos={"id_usuario": 1},
                           vigente_desde=datetime(2026, 11, 4, 10), id_corrida=corrida.id))
        sesion.commit()
    assert vaciar(motor) == {"analitica_evento": 0, "analitica_version": 1, "corrida": 1}
```

- [ ] **Paso 3:** correr → falla. **Paso 4:** implementar. **Paso 5:** suite en verde.
- [ ] **Paso 6: commit** `"Crear la base propia del servicio: corridas, versiones y eventos"`.

### Tarea 2: Versiones (normalizar, huella y vuelta)

**Archivos:** `analitica/versiones.py`, `tests/test_versiones.py`

**Interfaces:**
- `normalizar(tabla: str, fila: Mapping) -> dict`: solo las columnas del esquema, en su orden; `date` → `"AAAA-MM-DD"`,
  `datetime` → `"AAAA-MM-DDTHH:MM:SS"`; vacíos (`None`, `NaN`, `pd.NA`, `NaT`) → `None`; enteros y booleanos de numpy →
  `int` y `bool`; `float` → `float`.
- `huella(datos: dict) -> str`: sha256 hexadecimal de `json.dumps(datos, sort_keys=True, ensure_ascii=False,
  separators=(",", ":"))`.
- `filas_de(lectura: Lectura, tabla: str) -> dict[str, dict]`: `esquema.clave_de` → fila normalizada.
- `desnormalizar(tabla: str, datos: Mapping) -> dict`: la vuelta, con los tipos del esquema (`date`, `datetime`).

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_los_vacios_y_los_tipos_de_numpy_dan_la_misma_huella():
    a = normalizar("usuarios", {"id_usuario": np.int64(3), "username": "x", "rol": "TECNICO", "email": None,
                                "activo": np.bool_(True), "nombre": float("nan"), "apellido": pd.NA})
    b = normalizar("usuarios", {"id_usuario": 3, "username": "x", "rol": "TECNICO", "email": None, "activo": True,
                                "nombre": None, "apellido": None, "columna_de_mas": 1})
    assert a == b and huella(a) == huella(b)
    assert list(a) == nombres("usuarios")


def test_fechas_y_vuelta():
    fila = {"id_etapa_obra": 1, "id_solicitud": 1, "etapa": "ETAPA_1", "numero": 1, "estado": "APROBADA",
            "fecha_inicio": date(2026, 10, 16), "fecha_envio": datetime(2026, 10, 19, 15, 0)}
    datos = normalizar("etapas", fila)
    assert datos["fecha_inicio"] == "2026-10-16" and datos["fecha_envio"] == "2026-10-19T15:00:00"
    vuelta = desnormalizar("etapas", datos)
    assert vuelta["fecha_inicio"] == date(2026, 10, 16) and vuelta["fecha_envio"] == datetime(2026, 10, 19, 15, 0)


def test_filas_de_una_lectura_por_clave():
    lectura = OrigenSimulado(8, hasta=date(2026, 10, 15)).leer()
    filas = filas_de(lectura, "fotos_visita")
    assert len(filas) == len(lectura.tabla("fotos_visita"))      # la clave compuesta no repite
```

- [ ] **Pasos 2 a 4:** correr (falla), implementar, suite en verde.
- [ ] **Paso 5: commit** `"Normalizar las filas y calcular su huella para versionarlas"`.

### Tarea 3: Eventos de etapas

**Archivos:** `analitica/eventos.py`, `tests/test_eventos.py`

**Interfaces:**
- `Ventana` (dataclass congelada): `desde: datetime | None` (la última corrida correcta), `hasta: datetime` (esta).
- `derivar(entidad: str, anterior: Mapping | None, actual: Mapping | None, ventana: Ventana,
  id_solicitud: int | None = None) -> list[dict]`: filas con las claves de `historia.COLUMNAS_EVENTOS`
  (`fuente = "versiones"`, `ronda` vacía: la pone el sincronizador). `anterior` y `actual` vienen con tipos de Python
  (`versiones.desnormalizar`). `anterior is None` se trata como una fila con todo vacío (la fila es nueva).
- Los eventos de `intervalo` llevan `momento = ventana.hasta` y `detalle["desde"]` = `ventana.desde` en ISO (o `None`).

Reglas para `entidad == "etapas"` (especificación 7.3, en este orden; pueden salir varios eventos de un mismo cambio):

| # | Si… | Evento | Momento · precisión |
| ---: | :--- | :--- | :--- |
| 1 | `fecha_inicio` pasa de vacía a fecha | `inicio` | `fecha_inicio` 00:00 · `dia` |
| 2 | `fecha_revision` cambia y no queda vacía | `aprobacion` si el estado es `APROBADA`, `rechazo` si es `RECHAZADA`, si no `observacion` | `fecha_revision` · `exacto` |
| 3 | `fecha_envio` cambia y no queda vacía | `envio` | `fecha_envio` · `exacto` |
| 4 | cambia `observacion_tecnico` sin que cambie `fecha_revision` | `observacion` | `intervalo` |
| 5 | `fecha_limite` cambia y la anterior no era vacía | `prorroga`, `detalle` = `limite_anterior`, `limite_nuevo` (ISO), `habiles_agregados` (`CALENDARIO_JAVA.habiles_entre`), `motivo` | `intervalo` |
| 6 | cambia `estado` y no salió ningún evento de 1 a 5 | `cambio_estado`, `detalle` = `de`, `a` | `intervalo` |
| 7 | `actual is None` | `baja` | `intervalo` |

`numero` = el de la fila; `clave` = `str(id_etapa_obra)`. Los eventos se devuelven ordenados por `momento` (un `observacion`
anterior al `envio` que lo sigue queda primero).

```mermaid
flowchart TD
    A["anterior y actual<br/>de una etapa"] --> I{"¿empezó?"}
    I -->|sí| E1["inicio · día"]
    A --> R{"¿cambió fecha_revision?"}
    R -->|sí| E2["aprobación, rechazo u observación · exacto"]
    A --> V{"¿cambió fecha_envio?"}
    V -->|sí| E3["envío · exacto"]
    A --> O{"¿cambió solo el texto del técnico?"}
    O -->|sí| E4["observación · intervalo"]
    A --> L{"¿cambió fecha_limite?"}
    L -->|sí| E5["prórroga · intervalo<br/>(neta si hubo varias)"]
    A --> S{"¿cambió solo el estado?"}
    S -->|sí| E6["cambio de estado · intervalo"]
```

- [ ] **Paso 1: pruebas que fallan** (`tests/test_eventos.py`), una por regla. Ayuda: `etapa(**cambios)` arma una
  fila de la etapa 1 de la solicitud 1 (`id_etapa_obra=1`, `numero=1`, `estado="EN_CURSO"`,
  `fecha_inicio=date(2026, 10, 16)`, `fecha_limite=fecha_limite_original=date(2026, 10, 20)`, el resto vacío);
  `V = Ventana(datetime(2026, 10, 19, 8), datetime(2026, 10, 19, 18))`.

```python
def test_una_etapa_que_aparece_empezada():
    eventos = derivar("etapas", None, etapa(), V)
    assert [(e["tipo"], e["momento"], e["precision"]) for e in eventos] == [("inicio", datetime(2026, 10, 16), "dia")]


def test_envio():
    eventos = derivar("etapas", etapa(), etapa(estado="EN_REVISION", fecha_envio=datetime(2026, 10, 19, 9, 5)), V)
    assert [(e["tipo"], e["momento"], e["precision"], e["fuente"]) for e in eventos] == [
        ("envio", datetime(2026, 10, 19, 9, 5), "exacto", "versiones")]


def test_observacion_y_reenvio_entre_dos_corridas():
    antes = etapa(estado="EN_REVISION", fecha_envio=datetime(2026, 10, 19, 9, 5))
    despues = etapa(estado="EN_REVISION", fecha_envio=datetime(2026, 10, 19, 15, 0),
                    fecha_revision=datetime(2026, 10, 19, 12, 0), observacion_tecnico="falta la capa aisladora")
    assert [e["tipo"] for e in derivar("etapas", antes, despues, V)] == ["observacion", "envio"]


@pytest.mark.parametrize("estado, tipo", [("APROBADA", "aprobacion"), ("RECHAZADA", "rechazo"), ("OBSERVADA", "observacion")])
def test_revision(estado, tipo):
    antes = etapa(estado="EN_REVISION", fecha_envio=datetime(2026, 10, 19, 9, 5))
    eventos = derivar("etapas", antes, antes | {"estado": estado, "fecha_revision": datetime(2026, 10, 19, 11, 0)}, V)
    assert [(e["tipo"], e["precision"]) for e in eventos] == [(tipo, "exacto")]


def test_dos_prorrogas_entre_corridas_quedan_como_una_neta():
    eventos = derivar("etapas", etapa(), etapa(estado="PRORROGADA", fecha_limite=date(2026, 10, 27), motivo_prorroga="lluvias"), V)
    assert len(eventos) == 1 and eventos[0]["tipo"] == "prorroga" and eventos[0]["precision"] == "intervalo"
    assert eventos[0]["detalle"] == {"desde": "2026-10-19T08:00:00", "limite_anterior": "2026-10-20",
                                     "limite_nuevo": "2026-10-27", "habiles_agregados": 5, "motivo": "lluvias"}


def test_solo_cambia_el_texto_del_tecnico():
    antes = etapa(estado="OBSERVADA", fecha_revision=datetime(2026, 10, 18, 9), observacion_tecnico="a")
    eventos = derivar("etapas", antes, antes | {"observacion_tecnico": "b"}, V)
    assert [(e["tipo"], e["momento"], e["precision"]) for e in eventos] == [("observacion", V.hasta, "intervalo")]


def test_cambio_de_estado_sin_otra_explicacion_y_baja():
    assert [e["tipo"] for e in derivar("etapas", etapa(), etapa(estado="PENDIENTE"), V)] == ["cambio_estado"]
    assert [e["tipo"] for e in derivar("etapas", etapa(), None, V)] == ["baja"]
```

- [ ] **Pasos 2 a 4:** correr (falla), implementar, suite en verde.
- [ ] **Paso 5: commit** `"Deducir los eventos de las etapas comparando dos versiones"`.

### Tarea 4: Eventos del resto de las entidades

**Archivos:** `analitica/eventos.py`, `tests/test_eventos.py`

Reglas (especificación 7.3, «Resto de las entidades»; `anterior is None` = la fila aparece):

| Entidad | Si… | Evento | Momento · precisión |
| :--- | :--- | :--- | :--- |
| `solicitudes` | aparece | `solicitud_registrada` | `fecha_solicitud` · `dia` |
| | `num_exp` de vacío a valor | `expediente_asignado` | `intervalo` |
| | `id_tecnico_asignado` cambia a un valor | `tecnico_asignado` (`detalle.id_tecnico`) | `intervalo` |
| | estado → `SemiAprobada` / `Rechazada` | `solicitud_semiaprobada` / `solicitud_rechazada` | `intervalo` |
| | estado `SemiAprobada` → `Pendiente` | `correccion_confirmada` | `intervalo` |
| | estado → `Aprobada` | `solicitud_aprobada` | `fecha_activacion` · `dia` |
| `viviendas` | aparece | `vivienda_creada` | `ventana.hasta` · `intervalo` (el sincronizador la reemplaza por la `fecha_activacion` de la solicitud, `dia`, si la conoce) |
| | `tipo_vivienda` de vacío a valor | `vivienda_completada` | `intervalo` |
| `fotos_etapa` | aparece / desaparece | `foto_subida` / `foto_borrada` | `fecha_subida` · `exacto` / `intervalo` |
| `visitas_iniciales` | aparece | `visita_inicial` | `fecha_visita` · `exacto` |
| `visitas_obra` | aparece | `visita_obra` | `fecha` · `dia` |
| `documentos` | aparece | `documento_subido` | `fecha_subida` · `exacto` |
| | estado → `A_CORREGIR` | `documento_a_corregir` | `fecha_revision` · `exacto` |
| | estado `A_CORREGIR` → `PENDIENTE` | `documento_reemplazado` | `intervalo` |
| | estado → `APROBADO` / `RECHAZADO` | `documento_aprobado` / `documento_rechazado` | `intervalo` |
| `solicitudes_organizacion` | aparece | `alta_registrada` | `fecha_solicitud` · `dia` |
| | estado → `SemiAprobada` | `alta_observada` | `intervalo` |
| | estado `SemiAprobada` → `Pendiente` | `alta_corregida` | `intervalo` |
| | estado → `Aprobada` / `Rechazada` | `alta_aprobada` / `alta_rechazada` | `fecha_aprobacion` / `fecha_rechazo` · `dia` |
| `integrantes`, `usuarios` | aparece activo, o `activo` pasa a verdadero / a falso | `integrante_alta` / `integrante_baja` (`usuario_alta` / `usuario_baja`) | `intervalo` |
| `notificaciones` | — | ninguno (la historia de etapas sale de `historia.py`) | |
| cualquiera (salvo `notificaciones`) | cambian otros campos y no salió ningún evento | `cambio` (`detalle.campos` = `{campo: [antes, después]}`, fechas en ISO) | `intervalo` |
| cualquiera (salvo `fotos_etapa` y `notificaciones`) | desaparece | `baja` | `intervalo` |

- [ ] **Paso 1: pruebas que fallan:** una por fila de la tabla (parametrizadas por entidad cuando se pueda), más:

```python
def test_cambio_generico_lista_los_campos():
    antes = {"cuit": "30-1-1", "nombre": "A", "tipo": "Otra", "dom_legal": "x", "cpe": None,
             "fecha_vencimiento_vigencia": date(2027, 1, 1), "fecha_ultima_asamblea": None}
    eventos = derivar("organizaciones", antes, antes | {"fecha_vencimiento_vigencia": date(2028, 1, 1)}, V)
    assert [(e["tipo"], e["detalle"]["campos"]) for e in eventos] == [
        ("cambio", {"fecha_vencimiento_vigencia": ["2027-01-01", "2028-01-01"]})]


def test_las_notificaciones_no_generan_eventos():
    fila = {"id_notificacion": 1, "id_usuario": 5, "tipo": "ETAPA_ENVIADA", "id_referencia": 1,
            "fecha": datetime(2026, 11, 3, 10), "leida": False}
    assert derivar("notificaciones", None, fila, V) == []
    assert derivar("notificaciones", fila, fila | {"leida": True}, V) == []
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Deducir los eventos de solicitudes, documentos, altas, fotos y usuarios"`.

### Tarea 5: Sincronizador

**Archivos:** `analitica/sincronizador.py`, `analitica/origenes/simulado.py`, `analitica/origenes/fijo.py`,
`tests/ayudas_origenes.py`, `tests/test_sincronizador.py`

**Interfaces:**
- Consume: `Corrida`, `Version`, `Evento` (Tarea 1); `versiones.filas_de`, `huella`, `desnormalizar` (Tarea 2);
  `eventos.derivar`, `Ventana` (Tareas 3 y 4); `historia.eventos_de_la_lectura`, `historia.vinculos` (Plan 1b);
  `origenes.base.verificar_relaciones`.
- Produce: `ResultadoCorrida` (dataclass): `id_corrida: int | None`, `resultado: str`, `filas: dict[str, int]`,
  `versiones_nuevas: int`, `versiones_cerradas: int`, `eventos: int`, `advertencias: list[str]`, `error: str | None`.
- Produce: `sincronizar(origen: Origen, sesion: Session, ahora: datetime) -> ResultadoCorrida`.
- Produce: `MINUTOS_CANDADO = 10`.
- Produce: `OrigenSimulado.al_corte(corte: date | datetime) -> Origen` (un objeto con `codigo = "simulado"` cuyo
  `leer()` es `self.leer_al(corte)`; reutiliza el mismo mundo).
- Produce: `analitica/origenes/fijo.py` con `OrigenFijo(lectura)` (devuelve siempre esa lectura, `codigo` =
  `lectura.origen`); lo usan las pruebas y la evaluación del Plan 7.
- En `tests/ayudas_origenes.py`: `OrigenQueFalla(codigo="mysql")` (`leer()` lanza `ConnectionError("sin conexión")`).

Pasos de una corrida (especificación 7.2): (0) si hay corridas de otro origen → `error` «la base tiene datos de otro
origen (…): correr `python -m analitica vaciar`», sin escribir; (1) una `en_curso` de hace menos de `MINUTOS_CANDADO` →
`error` «corrida en curso», sin escribir; una más vieja se cierra como `error` con `detalle.error = "abandonada"`;
(2) corrida `en_curso` y commit; `origen.leer()`; si falla, la corrida queda `error` con `detalle.error` = el tipo y el
mensaje de la excepción (sin URLs con contraseña) y se devuelve; (3) en una transacción: versiones (nuevas, cambiadas
—se cierra la vigente con `vigente_hasta = ahora` y se abre otra—, desaparecidas —se cierran—); (4) eventos: si es la
**primera corrida correcta** de ese origen, los de `historia.eventos_de_la_lectura(lectura)` con
`detalle.linea_de_base = True`; si no, `derivar()` por cada fila nueva, cambiada o desaparecida, con
`Ventana(desde=fin de la última corrida correcta, hasta=ahora)` e `id_solicitud` de `historia.vinculos` (de la lectura
nueva, o de la anterior para las filas que desaparecen); a cada `envio` se le pone `ronda` = envíos ya guardados de esa
etapa + 1; (5) `resultado` = `ok` si hubo versiones nuevas o cerradas, si no `sin_cambios`; `fin = ahora`; `detalle` =
`{"filas", "versiones_nuevas", "versiones_cerradas", "eventos", "advertencias"}`. Si algo de 3 a 5 falla: rollback y la
corrida queda `error` con el detalle.

- [ ] **Paso 1: pruebas que fallan** (`tests/test_sincronizador.py`, con una fixture `sesion` sobre SQLite en
  `tmp_path`, escrita en `tests/conftest.py` porque la usan también las Tareas 6 y 7 y el Plan 4):

```python
REF = date(2026, 10, 15)

def test_dos_corridas_iguales(sesion):
    origen = OrigenFijo(OrigenSimulado(8, hasta=REF).leer())
    primera = sincronizar(origen, sesion, datetime(2026, 10, 16, 8))
    assert primera.resultado == "ok" and primera.versiones_nuevas == sum(primera.filas.values())
    assert primera.eventos > 0
    segunda = sincronizar(origen, sesion, datetime(2026, 10, 16, 8, 30))
    assert (segunda.resultado, segunda.versiones_nuevas, segunda.eventos) == ("sin_cambios", 0, 0)


def test_dos_cortes_del_simulado_dan_los_eventos_de_lo_que_paso_en_el_medio(sesion):
    mundo = OrigenSimulado(8)
    sincronizar(mundo.al_corte(datetime(2026, 10, 14, 8)), sesion, datetime(2026, 10, 14, 8))
    sincronizar(mundo.al_corte(datetime(2026, 10, 15, 8)), sesion, datetime(2026, 10, 15, 8))
    envios_reales = {(x.datos["id_etapa_obra"], x.momento_registrado.replace(second=0)) for x in mundo.verdad()
                     if x.accion == "enviar_etapa" and datetime(2026, 10, 14, 8) < x.momento_registrado <= datetime(2026, 10, 15, 8)}
    guardados = {(int(e.clave), e.momento) for e in sesion.scalars(select(Evento).where(
        Evento.tipo == "envio", Evento.detectado_en == datetime(2026, 10, 15, 8)))}
    assert envios_reales and envios_reales <= guardados


def test_el_origen_caido_no_escribe_versiones(sesion):
    resultado = sincronizar(OrigenQueFalla(), sesion, datetime(2026, 10, 16, 8))
    assert resultado.resultado == "error" and "sin conexión" in resultado.error
    assert sesion.scalar(select(func.count()).select_from(Version)) == 0


def test_un_origen_por_base(sesion):
    sincronizar(OrigenFijo(OrigenSimulado(8, hasta=REF).leer()), sesion, datetime(2026, 10, 16, 8))
    otra = sincronizar(OrigenQueFalla("mysql"), sesion, datetime(2026, 10, 16, 9))
    assert otra.resultado == "error" and "vaciar" in otra.error and otra.id_corrida is None


def test_candado_y_corrida_abandonada(sesion):
    sesion.add(Corrida(origen="simulado", inicio=datetime(2026, 10, 16, 8), resultado="en_curso", detalle={}))
    sesion.commit()
    origen = OrigenFijo(OrigenSimulado(8, hasta=REF).leer())
    assert "en curso" in sincronizar(origen, sesion, datetime(2026, 10, 16, 8, 5)).error
    assert sincronizar(origen, sesion, datetime(2026, 10, 16, 8, 30)).resultado == "ok"
    abandonada = sesion.scalars(select(Corrida).order_by(Corrida.id)).first()
    assert (abandonada.resultado, abandonada.detalle["error"]) == ("error", "abandonada")


def test_una_fila_que_desaparece(sesion):
    lectura = OrigenSimulado(8, hasta=REF).leer()
    sincronizar(OrigenFijo(lectura), sesion, datetime(2026, 10, 16, 8))
    una_foto = lectura.tabla("fotos_etapa").iloc[[0]]
    sin_ella = Lectura(lectura.origen, lectura.leido_en, lectura.tablas | {
        "fotos_etapa": lectura.tabla("fotos_etapa").iloc[1:].reset_index(drop=True)})
    resultado = sincronizar(OrigenFijo(sin_ella), sesion, datetime(2026, 10, 16, 9))
    assert resultado.versiones_cerradas == 1
    borrada = sesion.scalars(select(Evento).where(Evento.tipo == "foto_borrada")).one()
    assert borrada.clave == str(int(una_foto["id_foto"].item()))
```

- [ ] **Pasos 2 a 4:** correr (falla), implementar, suite en verde. Para no leer todas las versiones fila por fila,
  cargar las vigentes de una entidad con una sola consulta (`vigente_hasta IS NULL`) en un diccionario por clave.
- [ ] **Paso 5: commit** `"Sincronizar: versionar cada lectura y guardar los eventos deducidos"`.

### Tarea 6: Instantánea y línea de tiempo

**Archivos:** `analitica/instantanea.py`, `tests/test_instantanea.py`

**Interfaces (las usa el Plan 4):**
- `ultima_corrida(sesion, origen: str | None = None, solo_correctas: bool = False) -> Corrida | None`
  (correctas = `ok` o `sin_cambios`).
- `lectura_vigente(sesion, origen: str) -> Lectura`: las 14 tablas desde las versiones vigentes (con
  `versiones.desnormalizar` y `armar_tabla`), `leido_en` = `fin` de la última corrida correcta; sin corridas correctas,
  tablas vacías y `leido_en = ahora_local()`. Pasa `verificar_lectura`.
- `eventos_guardados(sesion, origen: str) -> pd.DataFrame`: `analitica_evento` de las corridas de ese origen, con el
  contrato `historia.COLUMNAS_EVENTOS` (usar `historia.tabla_eventos`).
- `linea_de_tiempo(sesion, origen: str, id_solicitud: int) -> pd.DataFrame`: de
  `historia.eventos_para_contexto(lectura_vigente, eventos_guardados)`, los de esa solicitud más los del alta de su
  organización (`solicitudes_organizacion` con el mismo `cuit` que `solicitudes.cuit_org`), ordenados por `momento`.

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_la_lectura_vigente_es_la_ultima_lectura(sesion):
    lectura = OrigenSimulado(8, hasta=date(2026, 10, 15)).leer()
    sincronizar(OrigenFijo(lectura), sesion, datetime(2026, 10, 16, 8))
    vigente = lectura_vigente(sesion, "simulado")
    for tabla in TABLAS:
        pd.testing.assert_frame_equal(
            vigente.tabla(tabla).sort_values(list(CLAVES[tabla])).reset_index(drop=True),
            lectura.tabla(tabla).sort_values(list(CLAVES[tabla])).reset_index(drop=True))
    assert vigente.leido_en == datetime(2026, 10, 16, 8)


def test_sin_corridas_todo_vacio(sesion):
    assert set(lectura_vigente(sesion, "simulado").filas().values()) == {0}
    assert eventos_guardados(sesion, "simulado").empty


def test_linea_de_tiempo_de_una_obra(sesion):
    mundo = OrigenSimulado(8)
    sincronizar(mundo.al_corte(datetime(2026, 10, 1)), sesion, datetime(2026, 10, 1))
    sincronizar(mundo.al_corte(datetime(2026, 10, 15)), sesion, datetime(2026, 10, 15))
    etapas = lectura_vigente(sesion, "simulado").tabla("etapas")
    obra = int(etapas.loc[etapas["estado"] == "APROBADA", "id_solicitud"].iloc[0])
    linea = linea_de_tiempo(sesion, "simulado", obra)
    assert linea["momento"].is_monotonic_increasing
    assert {"solicitud_registrada", "visita_inicial", "inicio", "envio", "aprobacion", "alta_registrada"} <= set(linea["tipo"])
    assert set(linea.loc[linea["entidad"] == "etapas", "fuente"]) == {"notificaciones"}
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Leer lo vigente de la base propia y armar la línea de tiempo de cada expediente"`.

### Tarea 7: Comandos `sincronizar` y `vaciar`

**Archivos:** `analitica/__main__.py`, `tests/test_comandos_base.py`

**Interfaces:**
- `python -m analitica sincronizar [--cada MINUTOS] [--veces N]`: arma el origen con `crear_origen()` y la base con
  `ANALITICA_DB_URL` (`crear_motor` + `crear_tablas`), corre `sincronizar(..., ahora_local())` e imprime una línea:
  `Corrida 3 · ok · 1234 filas · 12 versiones nuevas · 3 cerradas · 15 eventos`. Con `--cada`, repite cada N minutos
  (`--veces` limita las repeticiones; sin `--veces`, sigue hasta Ctrl+C). Código de salida 0 si la última corrida no fue
  `error`, 1 si lo fue. La espera usa una función de módulo `_dormir = time.sleep` (para reemplazarla en las pruebas).
- `python -m analitica vaciar --si`: borra la base propia (`base_de_datos.vaciar`) e imprime una línea por tabla con
  la forma `corrida: 2`; sin `--si`, explica qué haría y sale con 1 sin borrar.

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_sincronizar_dos_veces_y_vaciar(tmp_path, monkeypatch, capsys):
    monkeypatch.setenv("ANALITICA_DB_URL", f"sqlite:///{tmp_path / 'a.db'}")
    monkeypatch.setattr("analitica.__main__._dormir", lambda segundos: None)
    assert main(["sincronizar", "--cada", "30", "--veces", "2"]) == 0
    salida = capsys.readouterr().out
    assert salida.count("Corrida ") == 2 and "sin_cambios" in salida.splitlines()[-1]
    assert main(["vaciar"]) == 1
    assert main(["vaciar", "--si"]) == 0
    assert "corrida: 2" in capsys.readouterr().out
```

  (Con el origen simulado y el corte «ahora», dos corridas seguidas pueden diferir si pasó un minuto con hechos: si la
  prueba resulta inestable, fijar el corte con `monkeypatch` sobre `analitica.__main__.ahora_local`.)
- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Agregar los comandos sincronizar y vaciar"`.

### Tarea 8: Las tablas de Java con sus nombres reales (`origenes/vivso3.py`)

**Archivos:** `analitica/origenes/vivso3.py`, `tests/test_vivso3.py`

**Interfaces (las usan la Tarea 9 y el Plan 6):**
- `METADATA: sqlalchemy.MetaData` con estas tablas (solo las columnas que se leen; nombres exactos, sensibles a
  mayúsculas). Tipos: `Bit1 = mysql.BIT(1).with_variant(LargeBinary, "sqlite")` para los `bit(1)`;
  `DateTime(6)` = `mysql.DATETIME(fsp=6).with_variant(DateTime, "sqlite")`; `Numeric(10, 2)` para `superficie`.

| Tabla de Java | Columnas |
| :--- | :--- |
| `solicitud` | `id_solicitud`, `fechaSolicitud` (date), `CUIT_org`, `numExp`, `estado`, `observacion`, `camposObservados`, `fechaActivacion` (date), `id_familia`, `id_revisor`, `id_tecnico_asignado` |
| `familia` | `id_familia`, `antiguedadRancho` |
| `familiar` | `id_familiar`, `id_familia` |
| `vivienda` | `id_vivienda`, `departamento`, `localidad`, `direccion`, `barrio`, `superficie`, `fechaInic`, `fechaFin`, `latitud`, `longitud`, `id_familia`, `id_solicitud`, `tipoVivienda`, `cant_dormitorios`, `avance_obra` |
| `etapa_obra` | `id_etapa_obra`, `id_solicitud`, `etapa`, `estado`, `fecha_inicio`, `fecha_limite`, `fecha_limite_original` (date), `motivo_prorroga`, `fecha_envio` (datetime 6), `observacion_organizacion`, `id_usuario_envio`, `id_revisor`, `fecha_revision` (datetime 6), `observacion_tecnico` |
| `foto_etapa` | `id_foto`, `id_etapa`, `nombre_archivo`, `ruta_archivo`, `fecha_subida` (datetime 6), `id_usuario_subida` |
| `visitaInicial` | `id`, `id_solicitud`, `fecha_visita` (datetime), `id_tecnico`, `clasificacion_preliminar`, `latitud`, `longitud`, `localidad`, `departamento`, `observaciones`, `cantHabitantesReal`, `tienePrioridadSocial` (bit) |
| `visita_fotos` | `id_visita`, `url_foto` |
| `visitaObra` (opcional) | `id_visita_obra`, `id_vivienda`, `fecha` (date), `afo`, `observaciones`, `id_usuario_tecnico` |
| `organizacion` | `CUIT`, `nombre`, `tipo`, `dom_legal`, `cpe`, `fecha_vencimiento_vigencia`, `fecha_ultima_asamblea` |
| `solicitud_organizacion` | `id_solicitud_org`, `cuit`, `nombre`, `tipo`, `domLegal`, `cpe`, `fechaVencimientoVigencia`, `fechaUltimaAsamblea`, `dniPresidente`, `nombrePresidente`, `apellidoPresidente`, `correoPresidente`, `telefonoPresidente`, `fechaAltaCargoPresidente`, `domicilioPresidente`, los siete del tesorero (`dniTesorero` … `domicilioTesorero`), `estado`, `motivo`, `fechaRechazo`, `fechaAprobacion`, `fechaSolicitud`, `campos_observados`, `id_revisor` |
| `integrante` | `id_integranteOrganizacion`, `CUIT_org`, `DNI`, `nombre`, `apellido`, `correo`, `telefono`, `cargo`, `domicilio`, `activo` (bit), `fecha_alta_cargo`, `id_usuario` |
| `usuario` | `id_usuario`, `username`, `rol`, `email`, `activo` (bit), `nombre`, `apellido` |
| `documento` | `idDocumento`, `nombreOriginal`, `urlPath`, `tipo`, `estado`, `motivoObservacion`, `fechaSubida` (datetime), `fechaRevision` (datetime), `id_familia`, `id_organizacion`, `id_solicitud_org`, `id_usuario_revisor` |
| `notificacion` (opcional) | `id_notificacion`, `id_usuario`, `tipo`, `id_referencia`, `fecha` (datetime 6), `leida` (bit) |

- `OPCIONALES = ("visitaObra", "notificacion")`.
- `MAPEO: dict[str, tuple[str, dict[str, str]]]`: nuestra tabla → (tabla de Java, {nuestra columna: columna de Java}),
  para las 13 tablas que salen de una sola tabla (todas menos `familias`), con los renombres de la sección 3.3 de la
  especificación. `etapas.numero` no está en el mapeo (sale del catálogo).

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_el_mapeo_cubre_nuestro_esquema():
    for tabla in TABLAS:
        if tabla == "familias":
            continue
        tabla_java, columnas = MAPEO[tabla]
        esperadas = [c for c in nombres(tabla) if not (tabla == "etapas" and c == "numero")]
        assert list(columnas) == esperadas, tabla
        assert set(columnas.values()) <= set(METADATA.tables[tabla_java].columns.keys()), tabla


def test_nunca_columnas_prohibidas():
    for tabla in METADATA.tables.values():
        assert columnas_prohibidas(tabla.columns.keys()) == [], tabla.name


def test_el_esquema_se_crea_en_sqlite():
    motor = sqlalchemy.create_engine("sqlite://")
    METADATA.create_all(motor)
    assert "visitaInicial" in sqlalchemy.inspect(motor).get_table_names()
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Describir las tablas de Java con sus nombres reales"`.

### Tarea 9: Origen MySQL

**Archivos:** `analitica/origenes/mysql.py`, `analitica/origenes/base.py` (`crear_origen`), `tests/test_origen_mysql.py`

**Interfaces:**
- `OrigenMySQL(url: str | None = None, ssl_ca: str | None = None, motor: Engine | None = None)`, `codigo = "mysql"`.
  Con `motor`, lo usa tal cual (pruebas); si no, `create_engine(url, connect_args={"ssl": {"ca": ssl_ca}} si hay
  ssl_ca, pool_pre_ping=True)`.
- `OrigenMySQL.verificar_esquema() -> list[str]`: con `sqlalchemy.inspect`, una línea por cada tabla o columna de
  `vivso3.METADATA` que falte, con la forma `"falta la tabla usuario"` o `"falta la columna usuario.nombre"` (las
  tablas de `OPCIONALES` pueden faltar enteras). Lista vacía = todo bien.
- `OrigenMySQL.leer() -> Lectura`: si `verificar_esquema()` no está vacía, lanza `LecturaInvalida("esquema de vivso3: "
  + "; ".join(problemas))`. Si no, en una conexión: un `select()` por tabla con las columnas del `MAPEO` y `.label()`
  con nuestro nombre; `familias` = `select(familia.id_familia, func.count(familiar.id_familiar).label("cantidad_familiares"),
  familia.antiguedadRancho.label("antiguedad_rancho")).outerjoin(...).group_by(...)`; las opcionales ausentes, vacías.
  Conversiones: `bytes` → `bool` (`b"\x01"` es verdadero), `Decimal` → `float`, `datetime` sin microsegundos, `numero`
  de cada etapa con `catalogo.etapa(codigo).numero`. Arma con `armar_tabla`, `leido_en = ahora_local()`, verifica con
  `verificar_lectura`.
- `crear_origen("mysql")` devuelve `OrigenMySQL(cfg.mysql_url, cfg.mysql_ssl_ca)`; sin `ANALITICA_MYSQL_URL`, lanza
  `ConfiguracionInvalida("falta ANALITICA_MYSQL_URL")`.

- [ ] **Paso 1: pruebas que fallan.** Fixture `base_java`: un motor SQLite en memoria con `vivso3.METADATA.create_all`
  y unas filas a mano (una organización, un usuario con `activo=b"\x01"`, una solicitud, su familia con 3 familiares, una
  etapa `ETAPA_2` con `fecha_envio` con microsegundos, una notificación).

```python
def test_lee_y_convierte(base_java):
    lectura = OrigenMySQL(motor=base_java).leer()
    verificar_lectura(lectura)
    assert lectura.tabla("usuarios")["activo"].tolist() == [True]
    assert lectura.tabla("familias")["cantidad_familiares"].tolist() == [3]
    etapa = lectura.tabla("etapas").iloc[0]
    assert etapa["numero"] == 2 and etapa["fecha_envio"].microsecond == 0
    assert len(lectura.tabla("notificaciones")) == 1


def test_una_columna_que_falta_da_un_error_claro():
    motor = sqlalchemy.create_engine("sqlite://")
    METADATA.create_all(motor)
    with motor.begin() as conexion:
        conexion.exec_driver_sql('ALTER TABLE usuario DROP COLUMN nombre')
    with pytest.raises(LecturaInvalida, match="usuario.nombre"):
        OrigenMySQL(motor=motor).leer()


def test_las_tablas_opcionales_pueden_faltar():
    motor = sqlalchemy.create_engine("sqlite://")
    METADATA.create_all(motor, tables=[t for n, t in METADATA.tables.items() if n not in OPCIONALES])
    lectura = OrigenMySQL(motor=motor).leer()
    assert lectura.tabla("notificaciones").empty and lectura.tabla("visitas_obra").empty


def test_crear_origen_mysql(monkeypatch):
    with pytest.raises(ConfiguracionInvalida, match="ANALITICA_MYSQL_URL"):
        crear_origen("mysql")
    monkeypatch.setenv("ANALITICA_MYSQL_URL", "mysql+pymysql://lector:secreta@servidor:3306/vivso3")
    assert isinstance(crear_origen("mysql"), OrigenMySQL)      # no se conecta hasta leer()
```

  Ajustar `test_crear_origen` de `tests/test_simulado.py`, que hoy espera `OrigenNoDisponible` para `mysql`.
- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Leer la base de Desarrollo con un usuario de solo lectura"`.
- [ ] **Paso 6 (cuando lleguen la contraseña y el certificado):** con el `.env` completo, correr
  `.venv\Scripts\python.exe -m analitica sincronizar` contra una base propia nueva y anotar en `docs/sincronizacion.md`
  la fecha y si `verificar_esquema()` encontró diferencias (es el criterio 7 de la Etapa 6). Si hay diferencias, se
  corrigen en `vivso3.py`.

### Tarea 10 (opcional): Origen API REST

Hacerla solo si Desarrollo despliega el backend antes del cierre. Si no, `crear_origen("api")` sigue lanzando
`OrigenNoDisponible` y se registra como limitación en `docs/sincronizacion.md`.

**Interfaces:** `OrigenApiRest(url: str, usuario: str, clave: str, cliente: httpx.Client | None = None)`,
`codigo = "api"`. `POST /auth/login` con el usuario `ADMIN` → token; después los recursos de la sección 6.3 de la
especificación con `Authorization: Bearer`. Cada recurso tiene su lista de campos (`CAMPOS_POR_RECURSO`) y su conversión
de valores de enum a nombres (con `catalogo`). Tiempo de espera de 20 s y un reintento.

- [ ] **Paso 1: prueba** con `httpx.MockTransport` que devuelva un login y dos recursos (`/usuario`, `/solicitud`) y
  verifique que `leer()` arma esas tablas y nunca guarda el token en la `Lectura`.
- [ ] **Pasos 2 a 5:** correr, implementar, suite en verde, commit `"Leer el sistema de Desarrollo por su API REST"`.

### Tarea 11: Experimento de frecuencia

**Archivos:** `analitica/origenes/simulado_proyeccion.py`, `analitica/evaluacion/__init__.py`,
`analitica/evaluacion/frecuencia.py`, `analitica/__main__.py`, `tests/test_frecuencia.py`

**Interfaces (las usa el Plan 7):**
- `proyectar_por_cortes(hechos, cortes: Sequence[datetime], *, tablas: tuple[str, ...] = ("etapas",),
  error_prorroga_java: bool = False, desfase_notificaciones_horas: int = 0) -> Iterator[tuple[datetime, dict[str, list[dict]]]]`:
  aplica los hechos una sola vez, en orden, y en cada corte entrega una copia de las filas de esas tablas (así 3000
  cortes no reprocesan todo).
- `FRECUENCIAS_MIN = (15, 30, 60, 360, 1440)`
- `experimento_frecuencia(semilla: int = 8, desde: datetime = datetime(2026, 9, 15), hasta: datetime = datetime(2026, 10, 15),
  frecuencias: Sequence[int] = FRECUENCIAS_MIN) -> pd.DataFrame` con columnas `fuente` (`versiones` o
  `notificaciones`), `frecuencia_min` (vacía para notificaciones), `tipo` (`envio`, `observacion`, `aprobacion`,
  `prorroga`, `rechazo`), `reales`, `detectados`, `exactos`, `pct_detectados`, `pct_exactos`.

Cómo se mide (criterio 2 de la Etapa 6):

```mermaid
flowchart LR
    V["verdad del simulado<br/>hechos de etapa en la ventana"] --> C["comparar por etapa,<br/>tipo y momento"]
    S["cortes cada f minutos<br/>proyectar_por_cortes"] --> D["derivar() entre cortes<br/>seguidos"] --> C
    N["lectura final<br/>eventos_de_notificaciones"] --> C
    C --> R["% detectados y<br/>% con el momento exacto"]
```

- Real = cada hecho `enviar_etapa`, `observar_etapa`, `aprobar_etapa`, `prorrogar_etapa`, `rechazar_etapa` con
  `momento_registrado` en `(desde, hasta]`, identificado por (etapa, tipo).
- Detectado = hay un evento derivado de ese tipo para esa etapa en la ventana entre el corte anterior y el siguiente al
  hecho. Exacto = además su `momento` es el `momento_registrado` truncado al minuto (las prórrogas nunca son exactas
  por versiones).
- Con notificaciones: los de `historia.eventos_de_notificaciones(lectura al corte hasta)`; exacto = mismo momento al
  segundo.

- [ ] **Paso 1: pruebas que fallan** (ventana corta para que sea rápida):

```python
def test_proyectar_por_cortes_da_lo_mismo_que_proyectar():
    origen = OrigenSimulado(8)
    cortes = [datetime(2026, 10, 1), datetime(2026, 10, 8)]
    for corte, filas in proyectar_por_cortes(origen.verdad(), cortes):
        assert filas["etapas"] == proyectar(origen.verdad(), corte)["etapas"]


def test_mas_frecuencia_recupera_mas_y_las_notificaciones_todo():
    tabla = experimento_frecuencia(desde=datetime(2026, 10, 5), hasta=datetime(2026, 10, 9), frecuencias=(15, 1440))
    envios = tabla[tabla["tipo"] == "envio"].set_index(["fuente", "frecuencia_min"])
    assert envios.loc[("versiones", 15), "pct_exactos"] >= envios.loc[("versiones", 1440), "pct_exactos"]
    notificaciones = tabla[tabla["fuente"] == "notificaciones"]
    assert (notificaciones["pct_exactos"] == 100).all()
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5:** comando `python -m analitica experimento-frecuencia [--semilla N] [--desde AAAA-MM-DD] [--hasta AAAA-MM-DD]`
  que corre el experimento completo (30 días), guarda `salidas/experimento_frecuencia.csv` e imprime la tabla en
  Markdown. Correrlo y **guardar la tabla**: va al documento de la Tarea 12 y a la Etapa 6.
- [ ] **Paso 6: commit** `"Medir cuánta historia se recupera según la frecuencia de sincronización"`.

### Tarea 12: Documento `docs/sincronizacion.md`

**Archivos:** `docs/sincronizacion.md`, `docs/README.md`, `README.md`, `tests/test_documentos.py`

Contenido, en lenguaje llano (para alguien que no programa), con un glosario corto al final:

1. Para qué sincronizamos: el sistema de Desarrollo pisa cada dato y nosotros queremos la historia.
2. Cómo funciona una corrida (el diagrama de secuencia de este plan) y qué guarda cada tabla propia.
3. De dónde sale la historia: notificaciones primero, versiones de respaldo (el diagrama de la Tarea 8 del Plan 1b), con
   la regla de elección y el desfase de la hora.
4. Las reglas de los eventos (las tablas de las Tareas 3 y 4, resumidas) con un diagrama de decisión.
5. El experimento de frecuencia: la tabla de la Tarea 11 y una lectura en dos oraciones (por ejemplo: «con
   notificaciones, la frecuencia deja de importar para las etapas; sigue importando para fotos borradas y textos»).
6. Lo que no se puede recuperar (sección 16 de la especificación).
7. Cómo correrlo (`sincronizar`, `--cada`, `vaciar`) y cómo conectarlo a la base real (`.env`, sin secretos).

- [ ] **Paso 1:** sumar `"docs/sincronizacion.md": 4` a `DIAGRAMAS_MINIMOS` y su nombre a la lista de documentos que
  exige `test_estan_los_documentos_de_desarrollo`; correr → falla.
- [ ] **Paso 2:** escribir el documento; sumarlo al índice `docs/README.md` y al README (estado y estructura).
- [ ] **Paso 3:** revisar los diagramas (vista previa de GitHub o VS Code); releer como alguien de afuera.
- [ ] **Paso 4:** suite completa en verde.
- [ ] **Paso 5: commit** `"Documentar cómo se recupera la historia y cada cuánto sincronizar"`.

---

## Decisiones para tomar al revisar

1. **Frecuencia de sincronización:** la especificación propone 30 minutos en horario hábil; el experimento dirá si con
   notificaciones alcanza con menos (recomendado: decidir con la tabla de la Tarea 11).
2. **Origen API REST:** dejarlo como opcional (recomendado mientras el backend no esté desplegado).
3. **Base propia en producción:** SQLite alcanza para el volumen de VIVSO (cientos de obras); MySQL o PostgreSQL solo si
   el área lo pide (cambia solo `ANALITICA_DB_URL`).
