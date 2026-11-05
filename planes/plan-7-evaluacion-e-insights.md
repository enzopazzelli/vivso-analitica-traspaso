# Plan 7 · Evaluación e insights (Etapas 6 y 7)

> **Para quien lo ejecuta (persona o agente):** seguí las tareas en orden y marcá cada casilla `- [ ]`. Leé antes `../LEEME.md`, la sección 15 de
> `../especificacion.md` (criterios de la Etapa 6) y las Etapas 6 y 7 de `pp2 devs/pp3/Guia_Detallada_PP3_v1.0.docx`.

**Objetivo:** medir en qué grado el servicio cumple su objetivo con los 8 criterios fijados en la especificación, y
convertir esos resultados (y lo que encontramos en el sistema real) en interpretaciones y recomendaciones para el área y
para Desarrollo, con resultados, interpretaciones y conclusiones separados, como pide la guía.

**Arquitectura:** el paquete `analitica/evaluacion/` reúne las mediciones (cada una es una función que devuelve un
`DataFrame` reproducible con la semilla 8) y un comando `python -m analitica evaluar` que las corre todas y deja tablas
en `salidas/evaluacion/` y un `resultados.md`. Los documentos de las Etapas 6 y 7 se escriben a partir de esas
salidas: los números salen del código, no se copian a mano de otro lado.

**Tecnología:** pandas, el origen simulado como «verdad», el sincronizador sobre SQLite en memoria; pytest con ventanas
cortas para que las pruebas sean rápidas.

**Especificación:** `../especificacion.md`, sección 15 (criterios 1 a 8 de la Etapa 6 y alcance de la Etapa 7), 16
(limitaciones) y 3.4 (huecos del sistema real).

**Depende de:** el Plan 2 (sincronizador, `OrigenFijo`, `proyectar_por_cortes`, experimento de frecuencia) y el Plan 3
(AFO, indicadores). Los criterios 5 y 6 usan pruebas del Plan 4. Los documentos se terminan cuando esos planes están.

## Restricciones globales

- Python 3.13 con `.venv`; comandos de PowerShell desde `vivso-analitica/`; todo en español.
- Todo resultado es reproducible: semilla 8, ventanas y frecuencias fijas, y el comando que lo produce queda escrito
  en el documento junto a la tabla.
- **Con el simulado no se describe el programa real:** los resultados simulados sirven para mostrar qué se podrá
  concluir con datos reales y cuánto se puede confiar en el método (sección 15 de la especificación). Cada conclusión
  dice de qué fuente sale.
- Lenguaje llano y sin «no se puede» tajante: decir qué adaptación o qué dato haría falta (preferencia del grupo y
  devolución del profesor).
- Documentos sin «Plan N», sin nombres de herramientas de IA ni el host de Aiven; diagramas Mermaid 11.
- Commits en español, sin coautoría ni menciones a IA; suite completa en verde antes de cada commit.

## Foco de revisión

1. **Conclusiones que exceden la evidencia** (criterio de calidad de la guía): cada conclusión de la Etapa 7 cita el
   resultado y su fuente (sistema real o simulado).
2. **Un resultado que cambia con la semilla:** el documento de la Etapa 6 informa además el rango con 5 semillas (8 a
   12) para las métricas principales (error del AFO, % de historia recuperada).
3. **Criterios sin medir** (7: lectura real; 8: utilidad): se declaran «pendientes» con la causa y cómo se medirían, no
   se omiten.
4. **Una medición lenta:** `evaluar --rapido` (ventanas cortas) tiene que terminar en menos de 5 minutos para poder
   repetirla antes de la presentación.
5. **Números distintos entre documentos y salidas:** los documentos citan `salidas/evaluacion/resultados.md` con su
   fecha; si se vuelve a correr, se actualizan los dos.

## Mapa

```mermaid
flowchart TD
    T1["1 · AFO contra el avance real"] --> T5["5 · Comando evaluar"]
    T2["2 · Indicadores con y sin<br/>notificaciones"] --> T5
    T3["3 · Revisión de secretos<br/>y datos reales"] --> T5
    T4["4 · Pruebas por criterio"] --> T5
    F["Experimento de frecuencia<br/>(sincronización)"] --> T5
    T5 --> T6["6 · Documento de la Etapa 6"]
    T6 --> T7["7 · Documento de la Etapa 7"]
    T7 --> T8["8 · Devolución del área<br/>y de Desarrollo"]
```

Cómo se usa el simulado como verdad:

```mermaid
flowchart LR
    V["Verdad del simulado<br/>hechos con su momento real<br/>y avance físico día por día"] --> M["Mediciones"]
    P["Lo que ve el servicio<br/>(lecturas al corte, sincronizaciones)"] --> M
    M --> E1["¿Cuánto historia recupera?<br/>criterio 2"]
    M --> E2["¿Cuánto se aleja el AFO<br/>de la obra real? criterio 3"]
    M --> E3["¿Cuánto cambian los indicadores<br/>sin notificaciones? criterio 4"]
```

## Archivos

| Archivo | Responsabilidad |
| :--- | :--- |
| `analitica/evaluacion/afo.py` | AFO contra el avance físico real |
| `analitica/evaluacion/indicadores.py` | Indicadores con historia completa contra solo versiones |
| `analitica/evaluacion/secretos.py` | Revisión del repositorio |
| `analitica/evaluacion/pruebas.py` | Resultados de las pruebas agrupados por criterio |
| `analitica/evaluacion/informe.py` | `evaluar()`: corre todo y escribe las salidas |
| `analitica/__main__.py` | Comando `evaluar` |
| `docs/etapa-6-evaluacion.md`, `docs/etapa-7-insights.md` | Documentos del entregable |

---

### Tarea 1: AFO contra el avance real (criterio 3)

**Archivos:** `analitica/evaluacion/afo.py`, `tests/test_evaluacion_afo.py`

**Interfaces:**
- `afo_contra_real(semilla: int = 8, desde: date = date(2026, 3, 2), hasta: date = date(2026, 10, 15), cada_dias: int = 7,
  parametros: ParametrosSimulacion = PARAMETROS) -> pd.DataFrame` con columnas `dia`, `id_solicitud`, `etapa_actual`,
  `afo`, `real` (`OrigenSimulado.avance_real`), `error` (`afo − real`). Para cada día de corte: `leer_al(dia)`,
  `contexto_desde_lectura(..., Calendario())` y `calcular_afo(ctx)`.
- `resumen_afo(tabla: pd.DataFrame) -> dict`: `obras_dia` (filas), `error_medio_absoluto`, `sesgo_medio` (media de
  `error`; negativo = el AFO va detrás de la obra real porque la aprobación llega después), `p90_error_absoluto`, y
  `por_etapa` (lista de `{etapa_actual, error_medio_absoluto, sesgo_medio}`).
- `sesgo_sin_conexion(semilla: int = 8, **kwargs) -> dict`: corre `afo_contra_real` con `PARAMETROS` y con
  `replace(PARAMETROS, prob_sin_conexion_urbano=0.0, prob_sin_conexion_resto=0.0)` y devuelve
  `{"sesgo_con": …, "sesgo_sin": …, "atribuible_a_sin_conexion": sesgo_con − sesgo_sin}`. (El generador saca el mismo
  número al azar con o sin conexión, así el resto del mundo es el mismo.)

- [ ] **Paso 1: pruebas que fallan** (ventana corta):

```python
def test_el_afo_nunca_supera_a_la_obra_real_por_mucho():
    tabla = afo_contra_real(desde=date(2026, 9, 1), hasta=date(2026, 10, 15), cada_dias=14)
    assert not tabla.empty and set(tabla.columns) >= {"dia", "id_solicitud", "afo", "real", "error"}
    # el AFO cuenta etapas aprobadas: puede ir atrás de la obra, pero no adelante más que una etapa chica
    assert tabla["error"].max() <= 100 * 2 / 29 + 0.1
    assert resumen_afo(tabla)["sesgo_medio"] < 0


def test_sin_conexion_el_sesgo_no_empeora():
    resultado = sesgo_sin_conexion(desde=date(2026, 9, 1), hasta=date(2026, 10, 15), cada_dias=14)
    assert resultado["atribuible_a_sin_conexion"] <= 0
```

  Si la primera aserción del máximo falla, investigar antes de relajarla: puede ser un error del AFO (por ejemplo,
  contar una aprobación del futuro) o un caso real del simulado (una etapa aprobada antes de su fin físico); en ese caso,
  documentarlo y ajustar el umbral con su explicación.
- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Medir cuánto se aleja el AFO del avance real de la obra"`.

### Tarea 2: Indicadores con historia completa contra solo versiones (criterio 4)

**Archivos:** `analitica/evaluacion/indicadores.py`, `tests/test_evaluacion_indicadores.py`

**Interfaces:**
- `CLAVES_COMPARADAS = {"observaciones_rondas": ("tasa",), "tiempo_revision": ("mediana", "p90"),
  "prorrogas": ("pct_prorrogadas", "prorrogas_notificadas"), "cumplimiento_plazo": ("pct_primer_envio_en_plazo_original",),
  "evidencia_fotos": ("rondas_sin_fotos", "aprobadas_sin_fotos")}`
- `indicadores_segun_historia(semilla: int = 8, frecuencia_min: int = 30, desde: datetime = datetime(2026, 9, 15),
  hasta: datetime = datetime(2026, 10, 15)) -> pd.DataFrame` con columnas `codigo`, `clave`, `con_notificaciones`,
  `solo_versiones`, `diferencia`. «Con notificaciones» = `contexto_desde_lectura(leer_al(hasta), Calendario())`. «Solo
  versiones» = sincronizar en una base SQLite en memoria con `OrigenFijo` sobre cada corte cada `frecuencia_min`
  minutos entre `desde` y `hasta` (las lecturas salen de `proyectar_por_cortes` con todas las tablas y la tabla
  `notificaciones` vacía), y después `contexto_desde_lectura(lectura sin notificaciones, Calendario(),
  eventos_guardados=instantanea.eventos_guardados(...))`.

- [ ] **Paso 1: prueba que falla** (ventana de dos días, cada 6 horas):

```python
def test_sin_notificaciones_se_pierden_rondas():
    tabla = indicadores_segun_historia(frecuencia_min=360, desde=datetime(2026, 10, 12), hasta=datetime(2026, 10, 14))
    assert set(tabla["codigo"]) == set(CLAVES_COMPARADAS)
    prorrogas = tabla.set_index(["codigo", "clave"]).loc[("prorrogas", "prorrogas_notificadas")]
    assert prorrogas["solo_versiones"] in (None, 0) or prorrogas["solo_versiones"] <= prorrogas["con_notificaciones"]
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Comparar los indicadores con historia completa y con solo versiones"`.

### Tarea 3: Revisión de secretos y datos reales (criterio 5)

**Archivos:** `analitica/evaluacion/secretos.py`, `tests/test_evaluacion_secretos.py`

**Interfaces:**
- `PATRONES = {"host de Aiven": r"\.aivencloud\.com", "certificado o clave privada": r"-----BEGIN [A-Z ]+-----",
  "URL con contraseña": r"mysql\+pymysql://[^:/\s]+:(?!CLAVE@|\*\*\*@)[^@\s]+@", "clave de la API":
  r"ANALITICA_CLAVE_(LECTURA|ADMIN)=\S+"}`
- `revisar(archivos: Iterable[tuple[str, str]]) -> list[str]`: por cada `(ruta, texto)`, una línea
  `"ruta: patrón"` por cada patrón que aparece. Los archivos `.env`, `*.pem`, `*.crt` o `*.key` en la lista son un
  hallazgo por sí mismos, con la línea `"ruta: archivo de secretos"`.
- `archivos_del_repositorio(raiz: Path) -> list[tuple[str, str]]`: los de `git ls-files` (texto UTF-8; los binarios se
  saltean) **salvo los de `tests/`**, que tienen ejemplos inventados de estos patrones a propósito.

- [ ] **Paso 1: pruebas que fallan:**

```python
def test_encuentra_lo_que_no_tiene_que_estar():
    hallazgos = revisar([
        ("docs/a.md", "conectar a vivsosql-x.h.aivencloud.com"),
        ("analitica/b.py", 'URL = "mysql+pymysql://lector:secreta@host/vivso3"'),
        (".env.example", "ANALITICA_MYSQL_URL=mysql+pymysql://lector:CLAVE@HOST:PUERTO/vivso3\nANALITICA_CLAVE_LECTURA="),
        (".env", ""),
    ])
    assert hallazgos == ["docs/a.md: host de Aiven", "analitica/b.py: URL con contraseña", ".env: archivo de secretos"]


def test_el_repositorio_esta_limpio():
    assert revisar(archivos_del_repositorio(RAIZ)) == []
```

- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Revisar que el repositorio no tenga secretos ni datos de conexión"`.

### Tarea 4: Pruebas agrupadas por criterio (criterios 1, 5 y 6)

**Archivos:** `analitica/evaluacion/pruebas.py`, `tests/test_evaluacion_pruebas.py`

**Interfaces:**
- `CRITERIOS_POR_ARCHIVO = {1: ("test_catalogo.py", "test_calendario.py", "test_esquema.py", "test_simulado_proyeccion.py",
  "test_simulado_obras.py", "test_origen_mysql.py", "test_vivso3.py", "test_volcado.py"), 5: ("test_api_seguridad.py",
  "test_config.py", "test_documentos.py", "test_evaluacion_secretos.py"), 6: ("test_api.py",)}`
- `resumen_de_pruebas(junit_xml: Path) -> pd.DataFrame` con columnas `criterio`, `archivo`, `pruebas`, `pasaron`,
  `fallaron` (lee el XML de `pytest --junitxml`).

- [ ] **Paso 1: prueba** con un XML de ejemplo escrito a mano en `tmp_path` (dos casos de `test_catalogo.py`, uno
  fallado) → `criterio 1, test_catalogo.py, 2, 1, 1`.
- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5: commit** `"Agrupar los resultados de las pruebas por criterio de evaluación"`.

### Tarea 5: Comando `evaluar`

**Archivos:** `analitica/evaluacion/informe.py`, `analitica/__main__.py`, `tests/test_evaluacion_informe.py`

**Interfaces:**
- `evaluar(salida: Path, rapido: bool = False, semillas: Sequence[int] = (8,)) -> Path`: corre el experimento de
  frecuencia (Plan 2), `afo_contra_real` + `resumen_afo` + `sesgo_sin_conexion`, `indicadores_segun_historia`,
  `revisar(archivos_del_repositorio(...))` y `pytest --junitxml` (en un subproceso) con `resumen_de_pruebas`; escribe un
  CSV por medición en `salida/` y `salida/resultados.md` (una sección por criterio, con la tabla, el comando y la fecha).
  Con `rapido=True`, ventanas de 7 días y cortes cada 14 días. Con varias semillas, agrega el mínimo y el máximo de las
  métricas principales. Devuelve la ruta de `resultados.md`.
- `python -m analitica evaluar [--salida salidas/evaluacion] [--rapido] [--semillas 8 9 10 11 12]`.

- [ ] **Paso 1: prueba que falla:** `evaluar(tmp_path, rapido=True)` (reemplazando con `monkeypatch` la corrida de
  pytest por un XML fijo, para no correr la suite dentro de la suite) crea `resultados.md` con las secciones
  «Criterio 1» a «Criterio 8», y los criterios 7 y 8 dicen «Pendiente».
- [ ] **Pasos 2 a 4:** correr, implementar, suite en verde.
- [ ] **Paso 5:** correr la evaluación completa con las cinco semillas
  (`.venv\Scripts\python.exe -m analitica evaluar --semillas 8 9 10 11 12`), anotar cuánto tardó y **guardar
  `salidas/evaluacion/`** para los documentos (no se sube: `salidas/` está en el `.gitignore`; las tablas que importan
  se copian al documento).
- [ ] **Paso 6: commit** `"Agregar el comando que corre toda la evaluación"`.

### Tarea 6: Documento de la Etapa 6 (`docs/etapa-6-evaluacion.md`)

Responde **«¿En qué medida la solución desarrollada cumple con el objetivo planteado?»** con los elementos obligatorios de
la guía: evaluación con criterios definidos, fortalezas y limitaciones, condiciones de validez.

1. **Qué evaluamos y por qué estos criterios:** el objetivo (sección 1 de la especificación) y, para cada uno de los 8
   criterios, por qué es pertinente.
2. **Cómo evaluamos:** el simulado como verdad (qué es y por qué sirve), las mediciones y los comandos para repetirlas.
3. **Resultados por criterio:** una subsección por criterio con su tabla (copiada de `resultados.md`), una lectura en
   una o dos oraciones y si se cumple (cumple / cumple con condiciones / pendiente). Criterio 7: si llegó el acceso a
   Aiven, el resultado de la primera corrida real; si no, «pendiente» con la causa y que el `docker-compose` de prueba ya
   ejercitó las consultas contra MySQL. Criterio 8: la devolución recogida en la Tarea 8 o «pendiente».
4. **Fortalezas** (por ejemplo: reproducible; no depende de Desarrollo para funcionar; la historia completa con
   notificaciones; los bloqueos del sistema real que el simulador encontró antes que nadie: el técnico sin asignar y la
   etapa prorrogada trabada, informados el 21/10 y arreglados por Desarrollo).
5. **Limitaciones** (sección 16 de la especificación, al día).
6. **Condiciones de validez:** los resultados valen para el sistema Java de `develop` al 03/11/2026, para el mundo
   simulado con los supuestos `[E1]` a `[E24]` y la semilla 8 (con el rango de 5 semillas), y para los datos que el
   servicio puede leer; qué cambiaría con datos reales.

Diagramas: el «simulado como verdad» de este plan y uno que resuma los 8 criterios con su resultado (`flowchart` con un
nodo por criterio).

- [ ] **Paso 1:** sumar `"docs/etapa-6-evaluacion.md": 2` a `DIAGRAMAS_MINIMOS`; correr → falla.
- [ ] **Paso 2:** escribir con los números de `salidas/evaluacion/resultados.md`; sumarlo al índice y al README.
- [ ] **Paso 3:** releer con los criterios de calidad de la guía (¿la evaluación está respaldada por evidencia? ¿las
  conclusiones están dentro de lo que permiten los resultados?); suite en verde.
- [ ] **Paso 4: commit** `"Escribir la evaluación de la Etapa 6"`.

### Tarea 7: Documento de la Etapa 7 (`docs/etapa-7-insights.md`)

Responde **«¿Qué significan los resultados obtenidos para el problema que estamos abordando?»**, distinguiendo
**resultados**, **interpretaciones** y **conclusiones** (elemento obligatorio de la guía). Dos fuentes, siempre
nombradas: lo que muestra el **sistema real** (su código, su esquema y su comportamiento) y lo que muestran los
**resultados con datos simulados**.

Cada hallazgo, con el mismo formato: *Resultado* (qué se midió o se vio, con su fuente) → *Interpretación* (qué
significa para el programa) → *Conclusión o recomendación* (para el área o para Desarrollo, como «hoy pasa X → con Y
pasaría Z», el formato que pidió el profesor en el Hito 2). Hallazgos candidatos (confirmar cada uno con los datos):

| # | Hallazgo | Fuente | Recomendación |
| ---: | :--- | :--- | :--- |
| 1 | El sistema pisa cada dato; desde el 03/11 las notificaciones guardan la historia de las etapas, pero sin decir de qué etapa ni quién | Sistema real | Guardar `id_etapa_obra` y el autor en cada notificación |
| 2 | La hora de envíos, fotos y revisiones es la de llegada al servidor: una demora de señal parece una demora de obra | Sistema real + experimento (`[E9]`, `sesgo_sin_conexion`) | Guardar la hora de captura del celular |
| 3 | El técnico aprueba a distancia con las fotos como única evidencia; hay etapas aprobadas sin fotos (el código de Java tiene pendiente «exigir al menos una foto») | Sistema real + `evidencia_fotos` | Exigir al menos una foto por envío |
| 4 | Una etapa rechazada no tiene salida y hoy nadie puede rechazar | Sistema real | Definir con el área qué pasa después de un rechazo |
| 5 | Java no cuenta feriados ni aplica el tope de prórroga | Sistema real + `calidad_datos` | Cargar los feriados y aplicar el tope |
| 6 | Con notificaciones, sincronizar cada 30 minutos alcanza (o lo que diga el experimento) | Simulado (criterio 2) | Frecuencia de la Etapa 8 |
| 7 | El AFO por etapas va detrás de la obra real (cuánto, por qué) | Simulado (criterio 3) | Leer el AFO junto con el semáforo y la cola de revisión |
| 8 | El simulador encontró dos bloqueos antes de que hubiera datos reales | Sistema real + simulado | Mantener el simulador como prueba de cada cambio de Java |

Cierre: **3 o 4 hallazgos priorizados** para la presentación (la lección de la devolución del Hito 2: no sobrecargar
de técnica) y qué se podrá concluir cuando haya datos reales (qué indicadores, con qué confianza).

Diagrama: un `flowchart` de resultado → interpretación → conclusión para el hallazgo principal, y uno que ubique los
hallazgos en el circuito de la obra (dónde ocurre cada uno).

- [ ] **Paso 1:** sumar `"docs/etapa-7-insights.md": 2` a `DIAGRAMAS_MINIMOS`; correr → falla.
- [ ] **Paso 2:** escribir; sumarlo al índice y al README.
- [ ] **Paso 3:** releer: ¿cada conclusión cita su resultado y su fuente? ¿alguna dice más de lo que muestran los datos?;
  suite en verde.
- [ ] **Paso 4: commit** `"Escribir los insights de la Etapa 7"`.

### Tarea 8: Devolución del área y de Desarrollo (criterio 8)

**Archivos:** `docs/etapa-6-evaluacion.md` (anexo «Devolución»)

- [ ] **Paso 1:** preparar una guía corta (5 a 7 preguntas) para mostrar el tablero al responsable del área y a
  Desarrollo: ¿entiende qué dice el semáforo sin explicación?, ¿qué decisión tomaría con la página de obras?, ¿qué
  indicador le sirve más y cuál sobra?, ¿qué le falta?, ¿la API le sirve a su web tal como está?
- [ ] **Paso 2:** hacer la consulta (en persona, por videollamada o por escrito) y anotar las respuestas textuales en
  el anexo, con la fecha y quién respondió (rol, no nombre si no quiere figurar).
- [ ] **Paso 3:** actualizar el criterio 8 en el documento (cumple / cumple con condiciones) y, si hay pedidos chicos,
  sumarlos como mejoras; si no llega a tiempo, dejarlo «pendiente» con la guía preparada.
- [ ] **Paso 4: commit** `"Registrar la devolución del área y de Desarrollo sobre el tablero"`.

---

## Decisiones para tomar al revisar

1. **Cuántas semillas:** 5 (recomendado, de 8 a 12) o solo la 8.
2. **Hallazgos para la presentación:** recomendados el 1, 2, 3 y 8 (son del sistema real y tienen una recomendación
   concreta).
3. **Figuras:** tablas en los documentos (recomendado) y, si se quiere, gráficos simples con `xychart-beta` de Mermaid o
   capturas del tablero.
