# Microservicio de analítica de VIVSO (`vivso-analitica`)

**Componente de Ciencia de Datos · PP3 · versión 4, 04/11/2026**
**Historia del documento:** la versión 1 (15/10) definió el servicio; la 2 sumó la carpeta aparte como único
entregable, consumir el sistema Java al máximo, sin anonimizador, y el AFO por etapas; la 3 (19/10) sumó el esquema real
de la base y las respuestas de Desarrollo; los ajustes del 21/10 anclaron el mundo simulado a una fecha de referencia.
**La versión 4 (04/11/2026)** alinea todo con la rama `develop` de `PaganiB/vivso` al commit `5b36d7d` (03/11/2026):
estado `RECHAZADA`, prórroga desde `EN_REVISION`, envío de etapas prorrogadas (se arregló la etapa trabada), asignación
del técnico por el arquitecto antes de la visita inicial, la tabla nueva `notificacion` (fuente principal de la historia
de etapas), `usuario.nombre` y `usuario.apellido`, y el reparto del trabajo en ocho planes (sección 17).
**Documento interno del equipo:** vive en la carpeta de traspaso, junto a los planes. No se sube al repositorio
`vivso-analitica` (ver «Reglas» en `LEEME.md`).

**Lo ya construido:** el Plan 1 (21/10/2026: el dominio, el esquema de las tablas, el origen vacío, el origen simulado
completo y los documentos sobre el sistema de Desarrollo) y el Plan 1b (04/11/2026: la realineación con `develop`, la
tabla 14 y `analitica/historia.py`); 174 pruebas en verde. Lo que este documento describe y todavía no existe se
construye con los planes 2 a 8.

---

## 1. Qué queremos lograr

Cerrar PP3 con un servicio que funcione: lee todo lo que carga el sistema de Desarrollo (`PaganiB/vivso`) —solicitudes,
familias (solo conteos), viviendas, etapas de obra y sus fotos, visitas iniciales, organizaciones y su alta, integrantes,
usuarios y documentos—, conserva la historia que ese sistema no guarda, calcula indicadores y los publica en una API
JSON y en un tablero. Cubre las Etapas 6 a 9 de la guía de la cátedra.

Criterios de éxito:

1. Funciona de punta a punta **sin depender de Desarrollo**, con un origen simulado que reproduce sus tablas y sus reglas.
2. Se conecta a la base real cambiando variables de entorno, sin tocar código.
3. Ningún token ni contraseña sale del servicio, y ningún dato real llega al repositorio ni a la demo pública.
4. Los días hábiles se calculan igual que en Java (casos de referencia).
5. Si el sistema de Desarrollo se cae, el servicio sigue respondiendo con los últimos datos e informa su antigüedad.
6. La carpeta `vivso-analitica/` alcanza sola para entender, instalar, correr, probar, desplegar y evaluar el componente:
   es lo único que presentamos como parte nuestra en PP3.

Queda fuera: escribir en la base de Desarrollo, el importador de exports de VISOC, la app móvil, el frontend de
Desarrollo y lo administrativo (la entrega de la vivienda, la aprobación de los documentos de familia, los permisos).

## 2. Decisiones

| # | Decisión |
| ---: | :--- |
| 1 | Acceso: usuario MySQL de **solo lectura** `lector` sobre la base de pruebas en Aiven (sección 3.5). La API REST queda como respaldo, pero hoy no es una opción: el backend no está desplegado. Sin acceso, se trabaja con el simulador |
| 2 | Consumidores: nuestro tablero y el frontend de Desarrollo. En producción, el backend Java nos consulta de servidor a servidor; la clave en un navegador es solo para la demostración |
| 3 | Proyecto aparte `vivso-analitica/`, junto a `vivso` y `vivso-python` en la carpeta de trabajo, con su propio repositorio git. No importa ni modifica `vivso-python`, que queda como antecedente de PP1 y PP2 |
| 4 | **Consumir Java al máximo:** el dominio, los enums, las reglas, los nombres de columna y los datos salen del código y la base de Java. Lo propio se limita a lo que Java no tiene: la historia, el porcentaje y el plan del AFO, los indicadores, la API analítica y el simulador |
| 5 | **Sin anonimizador.** Se lee todo lo que Java carga, salvo tokens, contraseñas y sus hashes. De `familia` y `familiar`, solo conteos: cantidad de familiares y antigüedad del rancho. Los datos reales nunca van al repositorio ni a la demo pública: el simulador inventa nombres |
| 6 | **El simulador se apaga con una variable:** `ANALITICA_ORIGEN=vacio` deja todo vacío (las 14 tablas sin filas, los indicadores «sin datos», el tablero y la API con avisos). Lo que se muestra es siempre del origen configurado: datos de otro origen guardados en la base no aparecen ni se mezclan, y `python -m analitica vaciar` los borra |
| 7 | **El AFO es el de Desarrollo: las 6 etapas** (commit «Feature-18 \| AFO -> EtapaObra», 15/10/2026). El servicio lo toma tal cual (etapa actual, estado, etapas aprobadas) y le suma un porcentaje ponderado por días estimados y el plan contra lo real (sección 8). Los restos del AFO anterior (`visitaObra.afo`, `vivienda.avance_obra`) se leen pero no entran en ningún cálculo |
| 8 | **Visitas:** solo cuentan las visitas iniciales. `visita` no se lee: Desarrollo confirmó que quedó sin uso. `visitaObra` se lee mientras exista, pero Desarrollo la va a dejar sin efecto (el avance lo registran las organizaciones, no el técnico): no hay indicadores de visitas de obra ni tope de visitas |
| 9 | Historia: versiones propias desde la primera corrida, y pedido a Desarrollo de una tabla de eventos |
| 10 | Frecuencia de sincronización: cada 30 minutos en horario hábil, a confirmar con el experimento del Plan 2 |
| 11 | Semáforo: «por vencer» con 1 día hábil o menos para el límite; «revisión demorada» con más de 2 días hábiles esperando al técnico |
| 12 | El plazo sigue corriendo mientras la etapa espera al técnico: se muestra «vencida» y, entre paréntesis, los días hábiles que lleva esperando la revisión |
| 13 | **Feriados: se cuentan**, con la misma fuente que va a usar Desarrollo: la API de ArgentinaDatos (`GET https://api.argentinadatos.com/v1/feriados/{año}`), guardada en un archivo local para trabajar sin conexión. Se cuentan los tres tipos que devuelve (inamovible, trasladable y puente turístico). El límite oficial de cada etapa sigue siendo el que guarda Java, que todavía no cuenta feriados |
| 14 | Tope de prórroga (14 días): en días hábiles |
| 15 | Organizaciones en dos grupos, con la regla del informe OG/ONG (`informe_ong_og_2026/DECISIONES.md`, punto 2): **OG** (gubernamentales: Municipio y Comisión Municipal) y **ONG** (Asociación civil, Fundación, Cooperativa, Comisión de Fomento y el resto). En Java el tipo es texto libre, así que el grupo se deduce del tipo y, si no alcanza, del nombre |
| 16 | El tipo de vivienda (`URBANA`, `RURAL`, `ECONOMICA`) es el prototipo: define el presupuesto y los planos. No indica la ubicación |
| 17 | «Obra terminada» = `ETAPA_6` aprobada. Lo que sigue (la entrega) es un trámite administrativo, fuera del análisis |
| 18 | Fuera del análisis, por ser administrativo o del desarrollo: la aprobación de los documentos de familia y los permisos de rechazo, semiaprobación y expediente. El servicio lee esos estados tal cual, sin marcarlos como problema |
| 19 | Acuerdo con Desarrollo: «Python lee `vivso3` y expone indicadores; nunca escribe en `vivso3`» reemplaza a «Python no expone API» |
| 20 | Los problemas de seguridad del sistema de Desarrollo se les pasaron por privado (19/10); el pedido que queda en el repositorio menciona solo lo que hace falta para la analítica y para levantar el sistema |
| 21 | **Java de referencia: la rama `develop`** de `PaganiB/vivso`, revisada al commit `5b36d7d` (03/11/2026). El 04/11 la rama `main` quedó casi vacía (se borraron las carpetas a mano): no se usa como referencia hasta que Desarrollo diga cuál es la rama vigente |
| 22 | **`RECHAZADA`** (estado nuevo de `EstadoEtapa`): se lee tal cual y la obra queda «rechazada» (detenida: ninguna regla de Java la saca de ese estado). El simulador no la genera por defecto (`[E23]`), porque hoy nadie puede ejecutarla (sección 3.4, punto 13) |
| 23 | **Notificaciones como historia de etapas:** la tabla `notificacion` (una fila por destinatario, sin borrado) registra con hora exacta cada inicio de obra, envío, aprobación, observación, prórroga y rechazo. Se lee como tabla 14 (`notificaciones`, sin título ni mensaje) y es la **fuente principal** de la historia de etapas; los eventos que se deducen comparando versiones quedan de respaldo para lo anterior a la primera notificación y para las obras con notificaciones incompletas (sección 7.5) |
| 24 | De `usuario` se leen además `nombre` y `apellido` (el personal los tiene; los usuarios de integrantes los tienen vacíos). Nunca `push_token` ni `debe_cambiar_password` |
| 25 | No se leen `documento.id_familiar` ni las columnas nuevas de `familia` (`referencia_ubicacion`, `latitud`, `longitud`, `contacto`): de las familias, solo conteos (decisión 5) |
| 26 | El trabajo que falta se reparte en **ocho planes** (1b a 8) pensados para que cuatro personas avancen en paralelo con sus agentes (sección 17) |

## 3. El sistema de Desarrollo (verificado en su código y en su base el 19/10/2026; en el código de `develop`, el 04/11/2026)

Java 17, Spring Boot y JPA; MySQL `vivso3` en Aiven; esquema generado por Hibernate (`ddl-auto=update`, que agrega
columnas pero nunca las borra); enums guardados como texto (`EnumType.STRING`, columnas `enum` de MySQL); zona
`America/Argentina/Buenos_Aires`; auditoría de JPA activa (`@EnableJpaAuditing`).

### 3.1 Roles (`Security/SecurityConfig.java`)

| Rol | Qué hace |
| :--- | :--- |
| `ADMIN` | Todo, incluido borrar |
| `OPERADOR` | Asigna el expediente, aprueba, semiaprueba y rechaza solicitudes; aprueba, observa y rechaza altas de organización; marca documentos para corregir |
| `ARQUITECTO` | Completa los datos de la vivienda (`PUT /vivienda`); asigna el técnico de cada solicitud (`PUT /solicitud/{id}/asignar-tecnico/{idTecnico}`, también `ADMIN`), solo mientras la solicitud no tenga visita inicial |
| `TECNICO` | Visita inicial (solo el técnico asignado), inicia la obra, aprueba, observa, prorroga y rechaza etapas |
| `INTEGRANTE` | Registra solicitudes de familias, confirma correcciones, reemplaza documentos, sube las fotos de las etapas y las envía a revisión |

### 3.2 Flujos

- **Alta de organización** (`SolicitudOrganizacionService`). Formulario público → `solicitud_organizacion` en
  `Pendiente` (`fechaSolicitud` = hoy, token de edición) y los documentos de la organización en `PENDIENTE`.
  Observar → `SemiAprobada`, `motivo`, `campos_observados`, y los documentos marcados quedan `A_CORREGIR`; la
  organización corrige con el token → `Pendiente`. Aprobar → crea `organizacion`, el presidente y el tesorero como
  `integrante` activos, cada uno con un usuario `INTEGRANTE` (`username` = `nombre.apellido`), y aprueba los
  documentos; `fechaAprobacion` = hoy. Rechazar → documentos `RECHAZADO` (se borra el archivo, la fila queda),
  `motivo`, `fechaRechazo` = hoy.
- **Solicitud de vivienda** (`SolicitudService`). El integrante registra la familia, sus familiares y los documentos de
  la familia → `solicitud` en `Pendiente` (`fechaSolicitud` = hoy). El operador asigna `numExp` (obligatorio para
  aprobar). **El arquitecto asigna el técnico** (`asignarTecnico`, desde el 27/10/2026): el usuario tiene que tener rol
  `TECNICO` y la solicitud no puede tener visita inicial (después de la visita no se reasigna). Semiaprobar →
  `SemiAprobada`, `observacion`, `camposObservados`; el integrante confirma la corrección (sin documentos `A_CORREGIR`)
  → `Pendiente`. Aprobar → `Aprobada`, `fechaActivacion` = hoy, `revisor`, y se crea la `vivienda` **vacía** (solo
  familia y solicitud). Rechazar → `Rechazada` y `observacion`.
- **Vivienda.** El arquitecto completa departamento, localidad, dirección, barrio, superficie, tipo (el prototipo),
  dormitorios y coordenadas. `fechaInic` y `fechaFin` no las completa ningún flujo.
- **Visita inicial** (técnico, app con GPS). **Solo la registra el técnico asignado** a la solicitud
  (`VisitaInicialService`). La fecha la manda el celular (`fecha_visita`). Clasificación preliminar, coordenadas,
  localidad, departamento, observaciones, habitantes encontrados, prioridad social y fotos (`visita_fotos`).
- **Etapas** (`EtapaObraService`). Las organizaciones registran el avance y el técnico lo aprueba a distancia, con las
  fotos (Desarrollo, 19/10: los técnicos a veces no dan abasto para ir).
  - `iniciarObra` (técnico): exige la solicitud `Aprobada`; crea las 6 filas en `PENDIENTE` y activa `ETAPA_1`.
  - Activar una etapa: `EN_CURSO`, `fecha_inicio = hoy`, `fecha_limite = fecha_limite_original = sumarDiasHabiles(hoy, dias)`.
  - **Fotos de etapa** (`FotoEtapaService`): el integrante las sube con la etapa `EN_CURSO`, `OBSERVADA` o
    `PRORROGADA` (`fecha_subida` = hora del servidor) y puede borrarlas solo en `EN_CURSO` u `OBSERVADA`; el borrado
    no deja rastro. Desde el 22/10/2026 los archivos van a Cloudinary (`ruta_archivo` es una URL).
  - `enviarEtapa` (integrante): desde `EN_CURSO`, `OBSERVADA` o **`PRORROGADA`** (arreglado en `develop`) →
    `EN_REVISION`, `fecha_envio = ahora` truncado al minuto, `observacion_organizacion`, `id_usuario_envio`.
  - `aprobarEtapa` (técnico): solo desde `EN_REVISION` → `APROBADA`, `fecha_revision = ahora`; activa la siguiente en
    el mismo momento. La última observación del técnico no se borra.
  - `observarEtapa` (técnico): solo desde `EN_REVISION` → `OBSERVADA`, `observacion_tecnico`, `fecha_revision = ahora`.
  - `prorrogarEtapa` (técnico): desde `EN_CURSO`, `OBSERVADA`, **`EN_REVISION`** o `PRORROGADA`; el técnico elige la
    fecha nueva (`nuevaFechaLimite`), que tiene que ser posterior a la vigente y día hábil → `PRORROGADA`, sobrescribe
    `fecha_limite` y `motivo_prorroga`. El tope de 14 días hábiles está declarado y no se aplica. No registra quién
    prorrogó; el momento queda en la notificación `ETAPA_PRORROGADA`.
  - `rechazarEtapa` (técnico, nuevo): solo desde `EN_REVISION` → **`RECHAZADA`**, `observacion_tecnico` = motivo,
    `fecha_revision = ahora`, `id_revisor`. Ninguna regla saca una etapa de `RECHAZADA`, y la ruta
    `POST /etapa/{id}/rechazar` no figura en `SecurityConfig` (queda solo para `ADMIN`) mientras el servicio exige el
    técnico asignado: hoy nadie puede ejecutarla (sección 3.4, punto 13).
  - **Notificaciones** (`NotificacionService`, desde el 03/11/2026): cada acción de etapa guarda filas en `notificacion`
    (`tipo`, `id_referencia` = id de la solicitud, `fecha`, `id_usuario` destinatario, `leida`): `iniciarObra` →
    `ETAPA_HABILITADA`, `aprobarEtapa` → `ETAPA_APROBADA`, `observarEtapa` → `ETAPA_OBSERVADA`, `prorrogarEtapa` →
    `ETAPA_PRORROGADA` y `rechazarEtapa` → `ETAPA_RECHAZADA`, una fila por cada integrante **activo con usuario** de la
    organización; `enviarEtapa` → `ETAPA_ENVIADA`, una fila para el técnico asignado. La aprobación que habilita la
    etapa siguiente no genera `ETAPA_HABILITADA`. Las filas no se borran (solo se marcan como leídas). La `fecha` es
    `LocalDateTime.now()` sin zona: depende de la zona de la JVM (punto 12 de la sección 3.4). La notificación no dice
    de qué etapa se trata: se deduce contando las aprobaciones anteriores de la misma solicitud (sección 7.5).
  - Ninguna de estas acciones recibe la hora del celular: los DTO no traen fecha y el servidor pone la suya.
- **AFO.** Desde el 15/10/2026 el avance de obra son las 6 etapas: la etapa en curso y las aprobadas. La web de
  Desarrollo lo muestra como «Etapa N: nombre (estado)» y «Obra Finalizada» cuando las 6 están aprobadas
  (`SolicitudService.obtenerDashboardWebTecnico`). Java no calcula un porcentaje.
- **Visita de obra**: resto del AFO anterior (fecha del servidor, un `progreso` de 0 a 100 en la columna `afo`, fotos
  como `documento` `FOTO_OBRA`). El 30/10/2026 Desarrollo borró la entidad, el controlador y el servicio; la tabla puede
  seguir en la base (Hibernate no borra tablas) sin que nada la escriba.
- **Organizaciones:** `TipoOrganizacion` (nuevo enum) ofrece Asociación Civil, Fundación, Cooperativa, Comunidad
  Indígena, Institución Religiosa, Club Social y Deportivo, Mutual y Otra; la columna `tipo` sigue siendo texto libre.
  No hay opción para Municipio ni Comisión Municipal (OG): un municipio elegiría «Otra», y el grupo se deduce del nombre
  (decisión 15). Va como pregunta a Desarrollo.
- **Documentos** (`DocumentoService`). `PENDIENTE` → `A_CORREGIR` (operador: `motivoObservacion`, `fechaRevision`,
  `revisor`) → `PENDIENTE` al reemplazarlo (se borran el motivo y el revisor; `fechaRevision` queda). `APROBADO` y
  `RECHAZADO` solo existen en el alta de organizaciones. 19 tipos.
- **Días hábiles** (`CalendarioService`): `sumarDiasHabiles` no cuenta el día de inicio y cuenta solo de lunes a viernes;
  `diasHabilesRestantes` no cuenta hoy y cuenta el día del límite. Los feriados están previstos («TODO feriados:
  feriadoRepository.existsByFecha») con la API de ArgentinaDatos; para un feriado provincial, el técnico reasignaría la
  fecha límite sin que cuente como prórroga.

### 3.3 Las 14 tablas que se leen, con sus columnas reales

La tabla 14 (`notificacion` → `notificaciones`) y las columnas nuevas de `usuario` salen del código de `develop`
(04/11/2026), no del `SHOW CREATE TABLE`: hay que confirmarlas con `verificar_esquema()` en la primera lectura real.

Las columnas vienen del `SHOW CREATE TABLE` que mandó Desarrollo el 19/10, hecho con el usuario `lector`: muestra solo
lo que ese usuario puede leer (en `usuario` no aparece `password_hash`; en `solicitud_organizacion`, `token_edicion`
aparece en un índice pero no como columna; en `familia`, solo el id y la antigüedad). Cada tabla se renombra a nuestro
formato (sección 5.3); las columnas que en Java están en camelCase pasan a minúsculas con guion bajo.

| Tabla de Java | Nuestra tabla | Columnas de Java → nuestras (solo las que cambian de nombre) |
| :--- | :--- | :--- |
| `solicitud` | `solicitudes` | `numExp` → `num_exp`, `fechaSolicitud` → `fecha_solicitud`, `fechaActivacion` → `fecha_activacion`, `CUIT_org` → `cuit_org`, `camposObservados` → `campos_observados` |
| `familia` + `familiar` | `familias` | `id_familia`, `COUNT(familiar.id_familiar)` → `cantidad_familiares`, `antiguedadRancho` → `antiguedad_rancho`. De `familiar` solo se leen `id_familiar` e `id_familia` |
| `vivienda` | `viviendas` | `fechaInic` → `fecha_inicio`, `fechaFin` → `fecha_fin`, `tipoVivienda` → `tipo_vivienda`; `avance_obra` (resto del AFO anterior) se lee tal cual. La columna es `barrio`, sin espacio (en la entidad figura `"barrio "`) |
| `etapa_obra` | `etapas` | Sin cambios, más `numero` (del catálogo) |
| `foto_etapa` | `fotos_etapa` | `id_etapa` → `id_etapa_obra` |
| `visitaInicial` | `visitas_iniciales` | `id` → `id_visita`, `clasificacion_preliminar` → `clasificacion`, `cantHabitantesReal` → `habitantes`, `tienePrioridadSocial` → `prioridad_social` |
| `visita_fotos` | `fotos_visita` | Sin cambios (la tabla no tiene clave primaria) |
| `visitaObra` | `visitas_obra` | `id_usuario_tecnico` → `id_tecnico`. Opcional: si Desarrollo la borra, queda vacía sin error |
| `organizacion` | `organizaciones` | `CUIT` → `cuit` |
| `solicitud_organizacion` | `solicitudes_organizacion` | `domLegal` → `dom_legal`, `fechaVencimientoVigencia` → `fecha_vencimiento_vigencia`, `fechaUltimaAsamblea` → `fecha_ultima_asamblea`, `dniPresidente` → `dni_presidente` (y así los siete del presidente y los siete del tesorero), `fechaRechazo` → `fecha_rechazo`, `fechaAprobacion` → `fecha_aprobacion`, `fechaSolicitud` → `fecha_solicitud` |
| `integrante` | `integrantes` | `id_integranteOrganizacion` → `id_integrante`, `CUIT_org` → `cuit_org`, `DNI` → `dni` |
| `usuario` | `usuarios` | Sin cambios; se suman `nombre` y `apellido`. Nunca `password_hash`, `push_token` ni `debe_cambiar_password` |
| `documento` | `documentos` | `idDocumento` → `id_documento`, `nombreOriginal` → `nombre_original`, `urlPath` → `url`, `motivoObservacion` → `motivo_observacion`, `fechaSubida` → `fecha_subida`, `fechaRevision` → `fecha_revision`, `id_organizacion` → `cuit_org`, `id_usuario_revisor` → `id_revisor`. `id_familiar` no se lee (decisión 25) |
| `notificacion` | `notificaciones` | Sin cambios: `id_notificacion`, `id_usuario`, `tipo`, `id_referencia`, `fecha`, `leida` (`bit(1)` o `tinyint(1)`, a confirmar). `titulo` y `mensaje` no se leen (son textos armados con el tipo y el número de expediente) |

Detalles de la base que el origen MySQL tiene que respetar:

- Aiven corre en modo ANSI (el `SHOW CREATE TABLE` usa comillas dobles): los nombres van entre acentos graves y los
  textos entre comillas simples.
- `activo` y `tienePrioridadSocial` son `bit(1)`: el conector los devuelve como bytes y se pasan a `bool`.
- `superficie` es `decimal(10,2)` → `float`; `fecha_envio`, `fecha_revision` y `fecha_subida` son `datetime(6)`;
  `documento.fechaSubida` es `datetime`; `usuario.rol` es `tinytext`.

### 3.4 Errores y huecos que el servicio tiene que tolerar (y que informa `calidad_datos`)

1. ~~Una etapa `PRORROGADA` no puede enviarse: queda trabada.~~ **Arreglado en `develop`** (31/10/2026). Las obras
   trabadas que haya en la base de pruebas desde antes pueden enviarse ahora.
2. Aprobar `ETAPA_6` no cierra la solicitud.
3. ~~`tecnicoAsignado` no lo completa ningún flujo.~~ **Resuelto** (27/10/2026): el arquitecto lo asigna antes de la
   visita inicial. Sigue valiendo que `validarTecnicoAsignado` lo exige, sin excepción para `ADMIN`, para la visita
   inicial, para iniciar la obra y para aprobar, observar, prorrogar o rechazar etapas: las solicitudes de la base de
   pruebas cargadas antes del 27/10 sin técnico asignado no pueden avanzar.
4. `Vivienda.fechaInic` y `Vivienda.fechaFin` no se completan.
5. Los feriados todavía no se cuentan en Java.
6. Sin historial: cada ronda sobrescribe fechas y textos; las fotos borradas no dejan rastro; las fechas de
   semiaprobación y de rechazo de una solicitud no se guardan.
7. Modo sin conexión: `fecha_envio`, `foto_etapa.fecha_subida`, `fecha_revision` y la fecha de la visita de obra son la
   hora de llegada al servidor. La excepción es `visitaInicial.fecha_visita`, que la manda el celular.
8. (Del desarrollo, fuera del análisis: decisión 18.) Las reglas de seguridad no cubren el rechazo, la semiaprobación
   ni la asignación del expediente, que quedan solo para `ADMIN`. El simulador usa el flujo previsto: las hace un operador.
9. Restos del AFO anterior: `visitaObra.afo` y la columna `vivienda.avance_obra`, que la entidad ya no tiene pero
   Hibernate no borró.
10. `solicitud_organizacion.cuit` es único: una organización rechazada no puede volver a pedir el alta con el mismo CUIT
    (el servicio solo controla que no haya otra en trámite, y el alta fallaría en la base). Y `organizacion.fecha_vencimiento_vigencia`
    es obligatoria, pero en la solicitud no: aprobar un alta sin esa fecha fallaría.
11. ~~Para levantarlo: falta el `pom.xml`; dos `SecurityConfig`; `vivso.upload.dir` sin definir; `Mapper.java` no
    compila.~~ **Resuelto en `develop`** (04/11/2026: hay `pom.xml`, un solo `SecurityConfig`, `vivso.upload.dir` sale de
    la variable `uploaddir` y quedó solo `VivsoMapper.java`). No se verificó que compile.
12. Zona horaria (verificado el 21/10/2026): las etapas y las fotos usan `now(ZONA)` (Buenos Aires), pero los documentos
    (`fechaSubida` con `@CreatedDate`, `fechaRevision`), las solicitudes (`fechaSolicitud`, `fechaActivacion`) y las altas
    usan `LocalDateTime.now()` y `LocalDate.now()` sin zona, y la URL de conexión declara `serverTimezone=UTC`. Si la JVM
    corre en UTC, esas fechas quedan tres horas adelantadas y, después de las 21, con la fecha del día siguiente. Va como
    pregunta en el pedido a Desarrollo. **Las notificaciones tienen el mismo problema** (`fecha` = `LocalDateTime.now()`):
    el servicio estima el desfase comparando cada envío notificado con el `fecha_envio` de su etapa, que sí está en hora
    de Buenos Aires (sección 7.5).
13. **`RECHAZADA` (nuevo):** la etapa rechazada no tiene salida (ni enviar, ni prorrogar, ni aprobar la aceptan) y la
    obra queda detenida. Además, hoy nadie puede rechazar: la ruta queda solo para `ADMIN` y el servicio exige que quien
    rechaza sea el técnico asignado. Va como pregunta a Desarrollo: qué pasa después de un rechazo y quién puede hacerlo.
14. **Notificaciones incompletas:** las de la organización solo se guardan si tiene integrantes activos con usuario; la
    del envío, si la solicitud tiene técnico asignado. Una obra con notificaciones faltantes usa los eventos de versiones
    (sección 7.5).

### 3.5 Respuestas y datos de Desarrollo (19/10/2026, por chat)

| Tema | Respuesta | Qué cambia acá |
| :--- | :--- | :--- |
| Entorno de pruebas | Base desplegada en **Aiven** (MySQL administrado), funcionando. Backend y frontend sin desplegar. APK de la app móvil funcionando | El origen MySQL apunta a Aiven, con conexión cifrada |
| Conexión | Usuario `lector`, con el servidor y el puerto de Aiven (no se escriben acá: se pasan por privado junto con la contraseña). El certificado de la autoridad lo pasan aparte. La contraseña todavía no llegó | Todo va solo en el `.env` de cada máquina (sección 11), nunca en un repositorio |
| Esquema | `SHOW CREATE TABLE` de las 13 tablas (sección 3.3) | Nombres de columna reales en el origen MySQL y en el esquema de prueba |
| Porcentaje de la visita de obra | El `progreso` de 0 a 100 queda sin uso; lo van a borrar | No se usa (decisión 7) |
| Visitas de obra | `visitaObra` va a quedar sin efecto: el avance lo registran las organizaciones, no el técnico | Sin indicadores de visitas de obra ni tope de visitas (decisión 8); la evidencia del avance son las fotos (sección 8) |
| Tabla `visita` | Quedó de antes, no se usa (cuando vieron que había dos tipos de visita, la reemplazaron por `visitaInicial` y `visitaObra`) | No se lee (decisión 8) |
| Etapa 6 | Al aprobarla, la obra está finalizada; la entrega sigue con un trámite administrativo | Decisión 17 |
| Técnico asignado | Lo va a asignar el arquitecto; falta desarrollar esa vista | **Resuelto en `develop` el 27/10/2026** (`asignarTecnico`, antes de la visita inicial); el simulador lo hace igual (`[E19]`) |
| Feriados | API de ArgentinaDatos, por lo menos con los nacionales. Para un feriado provincial, el técnico reasignaría la fecha sin que cuente como prórroga | Decisión 13. Hace falta que esa reasignación quede distinguible (sección 7.3) |
| Hora de captura | Solo la visita inicial guarda la hora del celular (`fecha_visita`) | Se confirma el sesgo del modo sin conexión en envíos, fotos y revisiones (sección 3.4, punto 7) |

## 4. Arquitectura

```
vivso-analitica/                      # la carpeta del entregable, con su propio repositorio git
├── README.md                         # qué es, estado, instalación, uso, estructura
├── .gitignore, .env.example
├── requirements.txt                  # lo que hace falta para correrlo (crece con cada plan)
├── requirements-dev.txt              # requirements.txt + pytest
├── pytest.ini
├── Dockerfile, .dockerignore, docker-compose.yml          # Plan 6
├── analitica/
│   ├── __init__.py, __main__.py      # python -m analitica simular | sincronizar | salud | volcar | vaciar | feriados | informe | evaluar
│   ├── config.py                     # variables de entorno (sección 11)
│   ├── catalogo.py                   # etapas, estados, clasificaciones, tipos, grupos, roles y notificaciones, copiados de Java
│   ├── calendario.py                 # días hábiles como CalendarioService.java, con feriados
│   ├── feriados.py                   # feriados de ArgentinaDatos, con archivo local
│   ├── esquema.py                    # las 14 tablas: columnas, orden, tipos y claves
│   ├── historia.py                   # contrato de eventos, historia desde notificaciones, eventos de una lectura (Plan 1b)
│   ├── parametros_simulacion.py      # supuestos [E#] del simulador
│   ├── origenes/
│   │   ├── base.py                   # interfaz Origen + verificación del contrato + crear_origen
│   │   ├── vacio.py                  # las 14 tablas sin filas: el simulador apagado
│   │   ├── simulado.py               # orquesta el mundo simulado y lo proyecta como lo haría Java
│   │   ├── simulado_*.py             # una parte del mundo por archivo: organizaciones, solicitudes, obras
│   │   ├── vivso3.py                 # las tablas de Java con sus nombres reales (SQLAlchemy), para leer y para volcar
│   │   ├── mysql.py                  # consultas de solo lectura a vivso3, con los nombres reales de columna
│   │   └── api_rest.py               # login + llamadas, con lista de campos por recurso (opcional, Plan 2)
│   ├── base_de_datos.py              # motor, sesión y tablas propias: corrida, analitica_version, analitica_evento
│   ├── versiones.py                  # clave, normalización y huella de cada fila
│   ├── eventos.py                    # deriva eventos comparando versiones (función pura)
│   ├── sincronizador.py              # una corrida: leer, versionar, derivar, registrar
│   ├── instantanea.py                # el estado vigente en nuestra base: lectura, eventos y línea de tiempo
│   ├── afo.py                        # el AFO por etapas (sección 8)
│   ├── obras.py                      # la tabla de obras con su semáforo (la usan la API y el tablero)
│   ├── indicadores/                  # contexto, base.Indicador, un módulo por familia de indicadores y registro
│   ├── respuestas.py                 # el JSON de cada recurso, armado desde un Contexto (lo comparten API y tablero)
│   ├── salud.py                      # estado del servicio (frescura, corridas fallidas)
│   ├── api/                          # FastAPI /v1
│   ├── volcado.py                    # vuelca el simulado en un MySQL de prueba con el esquema real
│   ├── monitoreo.py                  # controles de la Etapa 9 e informe semanal
│   └── evaluacion/                   # mediciones de los criterios de la Etapa 6
├── tablero/                          # Streamlit: app.py, páginas y cliente (la API, o el origen en el mismo proceso)
├── tests/                            # pruebas automáticas, sin datos reales
└── docs/                             # documentación del entregable (sección 15)
```

```mermaid
flowchart LR
    subgraph origenes["Orígenes (uno por base)"]
        sim["simulado"]
        my["mysql<br/>usuario lector"]
        rest["api REST<br/>(opcional)"]
        vac["vacio"]
    end
    origenes --> sinc["sincronizador<br/>versiones y eventos"]
    sinc --> base[("base propia<br/>corrida, versión, evento")]
    base --> inst["instantánea<br/>lectura vigente y eventos"]
    inst --> hist["historia<br/>notificaciones + versiones"]
    hist --> ctx["Contexto"]
    ctx --> afo["AFO, semáforo<br/>e indicadores"]
    afo --> resp["respuestas JSON"]
    resp --> api["API /v1"]
    resp --> tab["tablero"]
    sim -. "sin base propia" .-> ctx
```

- **Independiente de `vivso-python`**: no lo importa ni lo modifica. De ahí se copia solo la lista de departamentos de
  Santiago del Estero con sus coordenadas (geografía para el simulador; Java guarda el departamento como texto libre).
- **Base propia** (`ANALITICA_DB_URL`; por defecto SQLite en `datos_locales/analitica.db`, fuera del repositorio). Las
  tablas se crean al arrancar (`crear_tablas`), sin Alembic: la base es nueva y solo la usa este servicio.
- **App FastAPI propia** (`analitica.api.app:app`).
- El núcleo (`catalogo`, `calendario`, `esquema`, `origenes.simulado`, `afo`, `indicadores`) **no importa fastapi ni
  streamlit**, y ninguna prueba usa la red.
- La API **responde desde nuestra base**; solo el sincronizador consulta el origen.
- La especificación y los planes quedan en la carpeta de traspaso, fuera del repositorio: la carpeta del entregable no
  lleva rastros del proceso de trabajo.

## 5. Dominio

### 5.1 Calendario (`analitica/calendario.py`) y feriados (`analitica/feriados.py`)

- `Calendario(feriados)`: la regla de `CalendarioService.java` con un conjunto de feriados.
  - `CALENDARIO_JAVA` no tiene feriados: es lo que hace Java hoy. Con él se calculan los límites en el simulador y se
    verifican los que guarda Java.
  - El de los indicadores usa los feriados de ArgentinaDatos (decisión 13).
- `es_habil(d)`: lunes a viernes que no sean feriado.
- `sumar_habiles(desde, n)`: el día `desde` no cuenta; se avanza de a un día y se descuentan solo los hábiles. `n < 0` → `ValueError`.
  Ejemplo de Java: habilitada un viernes con 2 días → vence el martes.
- `habiles_entre(a, b)` (H): cantidad de días hábiles `d` con `a < d ≤ b`; `0` si `a == b`; negativo (−H(b, a)) si `b < a`.
  Un `datetime` se pasa a fecha en hora de Buenos Aires antes de contar.
- `habiles_restantes(hoy, limite)`: igual que Java, `max(0, H(hoy, limite))`.
- Propiedad: `habiles_entre(a, sumar_habiles(a, n)) == n` para todo `a` y `n ≥ 0`, con y sin feriados.
- `feriados.cargar(años)`: lee `datos_locales/feriados/{año}.json`; si no está, lo baja de
  `ANALITICA_FERIADOS_URL` (`https://api.argentinadatos.com/v1/feriados/{año}`, que devuelve `fecha`, `tipo` y `nombre`)
  y lo guarda; si no hay conexión ni archivo, usa los feriados nacionales del paquete `holidays` y lo informa como
  advertencia. `python -m analitica feriados 2025 2026` deja los archivos listos. Las pruebas usan listas fijas.
- «Hoy» de los indicadores es la fecha de la última sincronización correcta, no el reloj, para que los números se puedan reproducir.

### 5.2 Catálogo (`analitica/catalogo.py`)

Copia de los enums de Java (coinciden con las columnas `enum` de la base), con el nombre como lo guarda la base y el
valor como lo devuelve la API:

- `ETAPAS` (`EtapaConstruccion`): 6 registros `Etapa(codigo, numero, nombre, dias_estimados)`; `TOTAL_DIAS_ESTIMADOS = 29`.
- `ESTADOS_ETAPA` (`EstadoEtapa`, 7 desde el 31/10: suma `RECHAZADA`), `ESTADOS_ACTIVOS` (`EN_CURSO`, `EN_REVISION`,
  `PRORROGADA`, `OBSERVADA`) y `ESTADOS_FINALES` (`APROBADA`, `RECHAZADA`).
- `TIPOS_NOTIFICACION`: `ETAPA_HABILITADA`, `ETAPA_ENVIADA`, `ETAPA_APROBADA`, `ETAPA_OBSERVADA`, `ETAPA_PRORROGADA`,
  `ETAPA_RECHAZADA` (los textos que usa `EtapaObraService`).
- `TIPOS_ORGANIZACION` (`TipoOrganizacion`): los 8 con su descripción; solo como referencia (la columna es texto libre).
- `ESTADOS_SOLICITUD` (`EstadoSolicitud`: `Pendiente`, `SemiAprobada`, `Aprobada`, `Rechazada`), que también usa el alta de organizaciones.
- `CLASIFICACIONES` (`ClasificacionPreeliminar`): `TIPO_1A → "1a"` … `TIPO_5D → "5d"`, `DERRUMBE → "DERRUMBE"`, `OTRA → "OTRA"`.
- `TIPOS_VIVIENDA` (`TipoVivienda`): `URBANA → "Urbana"`, `RURAL → "Rural"`, `ECONOMICA → "Económica"`: el prototipo (decisión 16).
- `GRUPOS_ORGANIZACION` y `grupo_organizacion(tipo, nombre)`: `OG` (Municipio, Comisión Municipal) u `ONG` (el resto),
  con la regla del informe OG/ONG (decisión 15): `COM. MUN.`, `COMISIÓN MUNICIPAL` o `COMISIONADO MUNICIPAL` → OG
  Comisión Municipal; `MUNICIPALIDAD`, `MUNICIPIO` o `INTENDENCIA` → OG Municipio; cualquier otra cosa (incluidas las
  Comisiones de Fomento, que en Santiago del Estero son organizaciones civiles) → ONG.
- `ESTADOS_DOCUMENTO` (`EstadoDocumento`): `PENDIENTE`, `A_CORREGIR`, `APROBADO`, `RECHAZADO`, con su valor.
- `TIPOS_DOCUMENTO` (`TipoDocumento`): los 19, con su valor y su ámbito (`organizacion`, `familia`, `obra`, `otro`).
- `TIPOS_CARGO` (`TipoCargo`): `PRESIDENTE`, `VICEPRESIDENTE`, `TESORERO`, `SECRETARIO`, `VOCAL`, `INTEGRANTE`.
- `ROLES`: `ADMIN`, `OPERADOR`, `ARQUITECTO`, `TECNICO`, `INTEGRANTE`.

Todo el servicio guarda el **nombre** del enum (como `vivso3`); el origen API REST convierte los valores a nombres.

### 5.3 Tablas normalizadas (`analitica/esquema.py`)

Todo origen devuelve una `Lectura` con 14 DataFrames de columnas fijas, en este orden. Tipos: `int`, `float`, `str`,
`bool`, `date` (sin hora) y `datetime` (en hora de Buenos Aires, sin zona); los vacíos se aceptan en cualquier columna.

| Tabla | Columnas |
| :--- | :--- |
| `solicitudes` | `id_solicitud` (int), `num_exp` (str), `estado` (str), `fecha_solicitud`, `fecha_activacion` (date), `cuit_org` (str), `id_familia` (int), `observacion`, `campos_observados` (str), `id_revisor`, `id_tecnico_asignado` (int) |
| `familias` | `id_familia`, `cantidad_familiares`, `antiguedad_rancho` (int) |
| `viviendas` | `id_vivienda`, `id_solicitud`, `id_familia` (int), `departamento`, `localidad`, `direccion`, `barrio` (str), `superficie` (float), `fecha_inicio`, `fecha_fin` (date), `latitud`, `longitud` (float), `tipo_vivienda` (str), `cant_dormitorios`, `avance_obra` (int) |
| `etapas` | `id_etapa_obra`, `id_solicitud` (int), `etapa` (str), `numero` (int), `estado` (str), `fecha_inicio`, `fecha_limite`, `fecha_limite_original` (date), `motivo_prorroga` (str), `fecha_envio` (datetime), `observacion_organizacion` (str), `id_usuario_envio`, `id_revisor` (int), `fecha_revision` (datetime), `observacion_tecnico` (str) |
| `fotos_etapa` | `id_foto`, `id_etapa_obra` (int), `nombre_archivo`, `ruta_archivo` (str), `fecha_subida` (datetime), `id_usuario_subida` (int) |
| `visitas_iniciales` | `id_visita`, `id_solicitud` (int), `fecha_visita` (datetime), `id_tecnico` (int), `clasificacion` (str), `latitud`, `longitud` (float), `localidad`, `departamento`, `observaciones` (str), `habitantes` (int), `prioridad_social` (bool) |
| `fotos_visita` | `id_visita` (int), `url_foto` (str) |
| `visitas_obra` | `id_visita_obra`, `id_vivienda` (int), `fecha` (date), `afo` (int), `observaciones` (str), `id_tecnico` (int) |
| `organizaciones` | `cuit`, `nombre`, `tipo`, `dom_legal`, `cpe` (str), `fecha_vencimiento_vigencia`, `fecha_ultima_asamblea` (date) |
| `solicitudes_organizacion` | `id_solicitud_org` (int), `cuit`, `nombre`, `tipo`, `dom_legal`, `cpe` (str), `fecha_vencimiento_vigencia`, `fecha_ultima_asamblea` (date), `dni_presidente`, `nombre_presidente`, `apellido_presidente`, `correo_presidente`, `telefono_presidente` (str), `fecha_alta_cargo_presidente` (date), `domicilio_presidente` (str), los mismos siete del tesorero, `estado`, `motivo` (str), `fecha_rechazo`, `fecha_aprobacion`, `fecha_solicitud` (date), `campos_observados` (str), `id_revisor` (int) |
| `integrantes` | `id_integrante` (int), `cuit_org`, `dni`, `nombre`, `apellido`, `correo`, `telefono`, `cargo`, `domicilio` (str), `activo` (bool), `fecha_alta_cargo` (date), `id_usuario` (int) |
| `usuarios` | `id_usuario` (int), `username`, `rol`, `email` (str), `activo` (bool), `nombre`, `apellido` (str) |
| `documentos` | `id_documento` (int), `nombre_original`, `url`, `tipo`, `estado`, `motivo_observacion` (str), `fecha_subida`, `fecha_revision` (datetime), `id_familia` (int), `cuit_org` (str), `id_solicitud_org`, `id_revisor` (int) |
| `notificaciones` | `id_notificacion`, `id_usuario` (int), `tipo` (str), `id_referencia` (int, la solicitud), `fecha` (datetime), `leida` (bool) |

Clave de cada tabla (`esquema.CLAVES`, la usan las versiones del sincronizador): `id_solicitud`, `id_familia`,
`id_vivienda`, `id_etapa_obra`, `id_foto`, `id_visita`, (`id_visita`, `url_foto`), `id_visita_obra`, `cuit`,
`id_solicitud_org`, `id_integrante`, `id_usuario`, `id_documento` e `id_notificacion`, en el orden de las tablas.

`Lectura` lleva además `origen` (`"simulado"`, `"mysql"`, `"api"`, `"vacio"`) y `leido_en` (datetime en hora de Buenos Aires).

### 5.4 Qué no sale nunca (`esquema.columnas_prohibidas`)

Ninguna columna cuyo nombre contenga `token`, `password`, `contrasena`, `clave`, `secret` o `hash` (así quedan afuera
`push_token` y `debe_cambiar_password` de `usuario`). De `familia` y
`familiar` solo salen las tres columnas de `familias`: el esquema es fijo y `armar_tabla` descarta todo lo demás. La
verificación del contrato lo controla en cada lectura, y una prueba de la API recorre todas sus respuestas. Además, el
propio usuario `lector` ya no puede leer `password_hash` ni `token_edicion`.

## 6. Orígenes

Interfaz (`origenes/base.py`):

```python
class Origen(Protocol):
    codigo: str                       # "simulado" | "mysql" | "api" | "vacio"
    def leer(self) -> Lectura: ...    # el estado completo actual, ya normalizado
```

`crear_origen(codigo)` arma el origen que dice `ANALITICA_ORIGEN`. `OrigenVacio` (`origenes/vacio.py`) devuelve las 14
tablas sin filas y con sus columnas: es la forma de apagar el simulador sin conectar nada.

`verificar_lectura(lectura)` comprueba columnas, orden, tipos y columnas prohibidas; la usan todos los orígenes y las
pruebas de contrato. Además, `verificar_relaciones(lectura)` informa (sin fallar) las filas huérfanas: etapas sin
solicitud, viviendas sin solicitud, documentos sin dueño, etc.

### 6.1 Simulado (`origenes/simulado.py` y `origenes/simulado_*.py`)

Dos capas:

1. **La verdad**: los hechos de cada flujo de la sección 3.2, con su momento real y su autor, generados con un
   `numpy.random.Generator` sembrado. Las personas son inventadas con Faker (`es_AR`) sembrado con la misma semilla.
   Para cada obra, la verdad incluye además el **avance físico real** (0 a 100 %) día por día, para evaluar el AFO.
2. **La proyección**: aplica los hechos, en el orden en que llegaron al servidor, sobre filas de las 14 tablas con las
   mismas reglas de Java (sobrescribiendo, borrando fotos, dejando la vivienda vacía hasta que la completa el
   arquitecto), hasta una fecha de corte.

`OrigenSimulado(semilla=8, hasta=date, parametros=PARAMETROS)`; `leer()` devuelve la proyección al corte y
`leer_al(corte)` la de otro corte sin volver a generar el mundo; `verdad()` devuelve los hechos reales y
`avance_real(id_solicitud, dia)` el avance físico (solo para pruebas, la evaluación y el experimento de frecuencia). El
mundo se genera alrededor de una fecha de referencia fija y no depende del corte: dos lecturas con cortes distintos ven
el mismo mundo en dos momentos.

Supuestos (`parametros_simulacion.py`, cada uno con su etiqueta):

| Etiqueta | Supuesto | Valor inicial |
| :--- | :--- | :--- |
| `[E1]` | Obras | 120, iniciadas a lo largo de los 8 meses previos a la fecha de referencia (15/10/2026), y el mismo ritmo (15 por mes) durante los 12 meses siguientes |
| `[E2]` | Organizaciones | 12 aprobadas, más 2 altas pendientes y 2 rechazadas; el 30 % de las altas pasa por una observación; revisión del alta en 3 a 20 días corridos; tipos «Municipio» y «Comisión Municipal» (OG), «Asociación civil» y «Cooperativa» (ONG), con un 30 % de OG |
| `[E3]` | Usuarios | 6 técnicos, 2 operadores, 1 arquitecto, 1 administrador; presidente y tesorero de cada organización con usuario `INTEGRANTE`; de 0 a 3 integrantes más sin usuario |
| `[E4]` | Duración real de una etapa | estimada × factor log-normal (mediana 1,2; desvío 0,35), en días hábiles, mínimo 1 |
| `[E5]` | Observación de un envío | 0,25 por ronda; a lo sumo 3 rondas |
| `[E6]` | Corrección tras una observación | 1 a 3 días hábiles |
| `[E7]` | Prórroga cuando la etapa se va a vencer | probabilidad 0,35; agrega 2 a 6 días hábiles; desde la prórroga se pueden seguir subiendo fotos (Java lo acepta en `PRORROGADA`) pero no borrarlas |
| `[E8]` | Espera de la revisión del técnico | 0 a 4 días hábiles |
| `[E9]` | Carga sin conexión | envíos y fotos de etapa que llegan con 1 a 5 días corridos de demora: 20 % en los departamentos urbanos (Capital y Banda) y 50 % en el resto. Se usa la ubicación y no el tipo de vivienda, que es el prototipo |
| `[E10]` | Error de Java en etapas prorrogadas | **desactivado desde la versión 4** (Java lo arregló el 31/10/2026). Activado reproduce el comportamiento anterior: una etapa prorrogada no puede enviarse y queda trabada |
| `[E11]` | Solicitudes que no llegan a obra | 25 %: pendientes, semiaprobadas, rechazadas o aprobadas sin iniciar |
| `[E12]` | Familias | 1 a 8 familiares; antigüedad del rancho de 5 a 40 años (la base exige 5 o más) |
| `[E13]` | Trámite de la solicitud | expediente a los 2 a 15 días; visita inicial a los 3 a 30 días del registro; el 30 % pasa por una semiaprobación (corrección en 3 a 20 días); aprobación 5 a 45 días después de la visita; obra iniciada 0 a 9 días después de aprobada |
| `[E14]` | Documentos de familia | 5 a 7 al registrar la solicitud; el 20 % se marca para corregir y se reemplaza en 2 a 10 días |
| `[E15]` | Vivienda | el arquitecto la completa en el 70 % de los casos, 1 a 20 días después de aprobada; tipo `URBANA` 45 %, `RURAL` 45 %, `ECONOMICA` 10 %; 40 a 60 m²; 2 o 3 dormitorios; `avance_obra` vacío |
| `[E16]` | Visitas de obra | ninguna, como en el flujo nuevo (decisión 8); un parámetro las activa si Desarrollo las vuelve a usar |
| `[E17]` | Fotos de etapa | 2 a 6 por ronda, subidas entre el inicio (o la observación) y el envío; en el 10 % de las rondas se borra una foto antes de enviar; en el 5 % de las rondas se envía sin fotos |
| `[E18]` | Avance físico real | dentro de cada etapa crece en línea recta con los días hábiles reales de esa etapa; al aprobarla vale lo acumulado por los días estimados (ponderado como en la sección 8) |
| `[E19]` | Técnico asignado | **como en `develop`:** el arquitecto lo asigna (`asignar_tecnico`) entre el registro y la visita inicial, 1 a 10 días después del registro y siempre antes de la visita; la visita la hace ese técnico. De las solicitudes que quedan pendientes sin visita, la mitad ya tiene técnico. Con el parámetro apagado no se asigna ninguno y ninguna solicitud puede tener visita ni obra |
| `[E20]` | Visita inicial | clasificación con pesos inventados (`PESOS_CLASIFICACION`); habitantes = familiares ± 1; prioridad social 30 %; 2 a 5 fotos |
| `[E21]` | Textos | motivos y observaciones de listas fijas inventadas; la organización deja un texto en el 50 % de los envíos |
| `[E22]` | Ubicación | el 70 % de las solicitudes en el departamento de la organización; coordenadas alrededor de la cabecera (desvío 0,05°) dentro de la provincia |
| `[E23]` | Rechazo de etapa | probabilidad 0 por cada revisión (hoy nadie puede rechazar en Java, sección 3.4 punto 13). Con un valor mayor, el técnico rechaza en vez de aprobar u observar y la obra se detiene ahí |
| `[E24]` | Notificaciones | se guardan como en Java: una fila por integrante activo con usuario (inicio, aprobación, observación, prórroga y rechazo) y una para el técnico asignado (envío), con la hora de llegada del hecho más un desfase de 0 horas (configurable, para probar una JVM en UTC con 3) y sin marcar como leídas |

Los departamentos, localidades, coordenadas y pesos son la tabla `[S15]` del simulador de PP2, copiada en
`parametros_simulacion.py` (no se importa ese módulo).

### 6.2 MySQL (`origenes/mysql.py`)

- Conexión: `ANALITICA_MYSQL_URL` (`mysql+pymysql://lector:clave@host:puerto/vivso3`) y `ANALITICA_MYSQL_SSL_CA` (el
  certificado de Aiven), en una sola transacción de lectura.
- `verificar_esquema()` compara las columnas esperadas (sección 3.3) contra `information_schema.columns`, que con el
  usuario `lector` muestra solo lo que puede leer. Si falta algo, la corrida termina en error con el detalle y no escribe
  nada. `visitaObra` y `feriado` son opcionales: si no existen, no es un error.
- Una consulta por tabla, con los nombres reales entre acentos graves (sección 3.3) y nunca `SELECT *`. `familias` sale
  de `SELECT f.id_familia, COUNT(fm.id_familiar), f.antiguedadRancho FROM familia f LEFT JOIN familiar fm … GROUP BY …`.
  `notificaciones` lee solo `id_notificacion`, `id_usuario`, `tipo`, `id_referencia`, `fecha` y `leida`, y es opcional
  como `visitaObra` (si la tabla no existe, queda vacía y la historia sale de las versiones).
- Las tablas de Java se describen una sola vez, con sus nombres reales, en `origenes/vivso3.py` (SQLAlchemy Core): de
  ahí salen las consultas, el esquema de prueba en SQLite para las pruebas y el MySQL de prueba de `docker-compose`.
- Conversión de tipos: `bit(1)` → `bool`, `decimal` → `float`, `datetime(6)` → `datetime` sin microsegundos.
- El esquema `vivso3` de prueba (para las pruebas contra SQLite y el MySQL de `docker-compose`) se escribe a partir de
  ese `SHOW CREATE TABLE`.

### 6.3 API REST (`origenes/api_rest.py`)

Para cuando el backend esté desplegado. `ANALITICA_VIVSO_API_URL`, `ANALITICA_VIVSO_API_USUARIO`,
`ANALITICA_VIVSO_API_CLAVE`; `httpx` con tiempo de espera y reintento simple. Recursos: `POST /auth/login`;
`GET /solicitud`, `/organizacion`, `/visita-inicial`, `/vivienda`, `/solicitud-organizacion`, `/usuario`, `/integrante`;
por cada solicitud, `/etapa/solicitud/{id}`; por cada etapa, `/etapa/{id}/fotos`; por cada organización y cada familia,
`/documento/organizacion/{cuit}` y `/documento/familia/{id}`; por cada familia, `/familia/{id}` y sus familiares, de los
que se guarda solo el conteo. Lista de campos por recurso. Limitaciones: hace falta un usuario `ADMIN` y son muchas
llamadas por corrida.

## 7. Sincronizador e historia

### 7.1 Tablas propias (`analitica/base_de_datos.py`)

- `corrida`: `id`, `origen`, `inicio`, `fin` (datetime), `resultado` (`en_curso`, `ok`, `sin_cambios`, `error`),
  `detalle` (JSON: filas leídas por tabla, versiones nuevas, eventos, advertencias, verificación de esquema y de relaciones).
- `analitica_version`: `id`, `entidad` (cada una de las 14 tablas), `clave` (str), `huella` (sha256 de la fila
  normalizada), `datos` (JSON), `vigente_desde` (datetime), `vigente_hasta` (datetime o nulo), `id_corrida`. Índice por
  (`entidad`, `clave`, `vigente_hasta`). Una fila que desaparece del origen cierra su versión y deja un evento de baja.
- `analitica_evento`: `id`, `entidad`, `clave`, `id_solicitud` (para la línea de tiempo; nulo si no corresponde),
  `tipo`, `ronda` (int o nulo), `momento` (datetime), `precision` (`exacto` | `intervalo`), `detectado_en`, `id_corrida`,
  `detalle` (JSON).

### 7.2 Una corrida (`sincronizador.sincronizar(origen, sesion, ahora)`)

0. **Un origen por base**: si la base ya tiene corridas de otro origen (por ejemplo, simuladas y ahora `mysql`), la
   corrida se detiene con el aviso «la base tiene datos de otro origen: correr `python -m analitica vaciar`». Así los
   datos simulados nunca se mezclan con los reales. `instantanea` y la API muestran solo lo del origen configurado; con
   `vacio`, nada.
1. **Candado**: si hay una corrida `en_curso` de hace menos de 10 minutos, se aborta con error «corrida en curso».
2. Se registra la corrida `en_curso`. Se llama a `origen.leer()`; si falla, la corrida termina en `error` y no se escribe nada más.
3. Para cada fila, si su huella difiere de la versión vigente: se cierra la vigente (`vigente_hasta = ahora`) y se
   inserta la nueva. Las filas que ya no están se cierran.
4. Se derivan eventos (7.3) comparando la versión anterior con la nueva.
5. La corrida termina `ok` (hubo cambios) o `sin_cambios`. Todo en una transacción: o queda la corrida completa o nada.

### 7.3 Derivación de eventos (`eventos.derivar(entidad, anterior, actual)`, función pura)

**Etapas** (por `id_solicitud` + `etapa`):

| Cambio detectado | Evento | Momento · precisión |
| :--- | :--- | :--- |
| La fila aparece por primera vez y ya tiene `fecha_inicio` | `inicio` | `fecha_inicio` · exacto |
| `fecha_inicio` pasa de vacía a fecha | `inicio` | `fecha_inicio` · exacto |
| `fecha_envio` cambia | `envio`, ronda + 1 | `fecha_envio` · exacto |
| `fecha_revision` cambia y el estado es `APROBADA` | `aprobacion` | `fecha_revision` · exacto |
| `fecha_revision` cambia y el estado es `RECHAZADA` | `rechazo` | `fecha_revision` · exacto |
| `fecha_revision` cambia y el estado no es `APROBADA` ni `RECHAZADA` | `observacion` | `fecha_revision` · exacto |
| `fecha_revision` cambió, el estado es `EN_REVISION` y `fecha_envio > fecha_revision` | `observacion` y luego `envio` (ronda + 1) | cada uno con su fecha · exacto |
| Cambia `observacion_tecnico` sin cambio de `fecha_revision` | `observacion` | entre las dos corridas · intervalo |
| Cambia `fecha_limite` | `prorroga` (detalle: límite anterior, nuevo, días hábiles agregados, motivo); `reasignacion_feriado` si Desarrollo la marca como tal (sección 3.2) | entre las dos corridas · intervalo |
| Cambia `estado` sin ninguna de las anteriores | `cambio_estado` | entre las dos corridas · intervalo |

**Resto de las entidades:**

| Entidad | Cambio | Evento | Momento · precisión |
| :--- | :--- | :--- | :--- |
| `solicitudes` | aparece | `solicitud_registrada` | `fecha_solicitud` · exacto (día) |
| | `num_exp` pasa de vacío a valor | `expediente_asignado` | intervalo |
| | estado → `SemiAprobada` / `Rechazada` | `solicitud_semiaprobada` / `solicitud_rechazada` | intervalo |
| | estado `SemiAprobada` → `Pendiente` | `correccion_confirmada` | intervalo |
| | estado → `Aprobada` | `solicitud_aprobada` | `fecha_activacion` · exacto (día) |
| | `id_tecnico_asignado` pasa de vacío a valor | `tecnico_asignado` | intervalo |
| `viviendas` | aparece | `vivienda_creada` | `fecha_activacion` de su solicitud · exacto (día) |
| | se completan datos (de vacío a valor) | `vivienda_completada` | intervalo |
| `fotos_etapa` | aparece / desaparece | `foto_subida` / `foto_borrada` | `fecha_subida` · exacto / intervalo |
| `visitas_iniciales`, `visitas_obra` | aparece | `visita_inicial` / `visita_obra` | su fecha · exacto |
| `documentos` | aparece | `documento_subido` | `fecha_subida` · exacto |
| | estado → `A_CORREGIR` | `documento_a_corregir` | `fecha_revision` · exacto |
| | estado `A_CORREGIR` → `PENDIENTE` | `documento_reemplazado` | intervalo |
| | estado → `APROBADO` / `RECHAZADO` | `documento_aprobado` / `documento_rechazado` | intervalo |
| `solicitudes_organizacion` | aparece | `alta_registrada` | `fecha_solicitud` · exacto (día) |
| | estado → `SemiAprobada` | `alta_observada` | intervalo |
| | estado `SemiAprobada` → `Pendiente` | `alta_corregida` | intervalo |
| | estado → `Aprobada` / `Rechazada` | `alta_aprobada` / `alta_rechazada` | `fecha_aprobacion` / `fecha_rechazo` · exacto (día) |
| `integrantes`, `usuarios` | `activo` cambia | `integrante_alta`, `integrante_baja`, `usuario_alta`, `usuario_baja` | intervalo |
| cualquiera | otros campos cambian | `cambio` (detalle: campos anteriores y nuevos) | intervalo |

- **Primera corrida (línea de base):** se guardan los eventos de `historia.eventos_de_la_lectura` (sección 7.5, punto
  6) más la prórroga neta si `fecha_limite ≠ fecha_limite_original`, con `detalle.linea_de_base = true`.
- **Lo que se pierde entre corridas** y queda declarado como limitación: rondas completas dentro de un mismo intervalo,
  prórrogas encadenadas (queda la neta), textos intermedios, quién prorrogó, fotos subidas y borradas entre dos corridas
  y toda la historia anterior a la primera corrida. **Con notificaciones, las rondas, las prórrogas y sus momentos se
  recuperan completos** (sección 7.5); siguen perdiéndose los textos intermedios, quién prorrogó y las fotos borradas.

### 7.4 Línea de tiempo del expediente

`instantanea.linea_de_tiempo(id_solicitud)`: todos los eventos ligados a la solicitud (por `id_solicitud` directo, o a
través de su familia, su vivienda, sus etapas y su organización), ordenados por momento, cada uno con su precisión y su
fuente. Es la base del AFO por etapas y de los indicadores de tiempos.

### 7.5 La historia de etapas desde las notificaciones (`analitica/historia.py`)

Todos los eventos del servicio comparten un contrato (`historia.COLUMNAS_EVENTOS`): `entidad`, `clave`,
`id_solicitud`, `numero` (de la etapa, o vacío), `tipo`, `ronda`, `momento`, `precision` (`exacto`, `dia` o
`intervalo`), `fuente` (`notificaciones`, `versiones` o `lectura`) y `detalle`.

1. **Deduplicar:** las filas de un mismo `tipo` e `id_referencia` con menos de 60 segundos entre sí son un solo evento
   (Java guarda una por destinatario); se toma la más temprana.
2. **Desfase:** `estimar_desfase_horas(lectura)` compara el último `ETAPA_ENVIADA` de cada etapa con su `fecha_envio`
   (que está en hora de Buenos Aires) y devuelve la mediana de las diferencias en horas enteras, entre −12 y 12; sin
   pares para comparar, 0. A cada `fecha` se le resta ese desfase.
3. **Etapa de cada evento:** por solicitud y en orden, `numero` = 1 + aprobaciones anteriores. Si las notificaciones
   empiezan con la obra ya avanzada, se suman las etapas que la tabla `etapas` muestra aprobadas antes de la primera
   notificación. Cada aprobación de la etapa k (k < 6) genera además el `inicio` de la k + 1 en el mismo momento.
4. **Tipos:** `ETAPA_HABILITADA` → `inicio`, `ETAPA_ENVIADA` → `envio` (con `ronda` 1, 2, …), `ETAPA_APROBADA` →
   `aprobacion`, `ETAPA_OBSERVADA` → `observacion`, `ETAPA_PRORROGADA` → `prorroga`, `ETAPA_RECHAZADA` → `rechazo`. Todos
   `exacto`, `fuente = notificaciones`.
5. **Elegir la historia de cada obra:** se usan las notificaciones desde la primera de esa solicitud, y los eventos de
   etapa de otra fuente solo antes de ese momento y si no coinciden con uno notificado (misma etapa y tipo, el mismo
   día si la precisión es `dia` o el mismo minuto si es `exacto`). Si una obra tiene notificaciones incompletas (menos
   aprobaciones notificadas que etapas aprobadas después de la primera notificación), se usan solo los eventos de otra
   fuente.
6. **Eventos de una lectura** (`eventos_de_la_lectura`): lo que una sola foto de las tablas permite fechar
   (registro y aprobación de la solicitud, vivienda creada, visita inicial, fotos y documentos subidos, documentos para
   corregir, altas registradas, aprobadas o rechazadas, y de cada etapa su inicio, su último envío y su última
   revisión), con `fuente = lectura`. Es la historia que tiene el tablero cuando calcula sin base propia, y la línea de
   base de la primera corrida del sincronizador.

```mermaid
flowchart TD
    N["notificaciones<br/>(una fila por destinatario)"] --> D["deduplicar y<br/>restar el desfase"]
    D --> E["número de etapa y tipo<br/>de cada evento"]
    V["eventos de versiones<br/>o de la lectura"] --> C{"¿la obra tiene<br/>notificaciones completas?"}
    E --> C
    C -->|sí| H1["notificaciones desde la primera<br/>+ lo anterior de otra fuente"]
    C -->|no| H2["solo versiones o lectura"]
    H1 --> H["historia de etapas de la obra"]
    H2 --> H
```

## 8. El AFO por etapas (`analitica/afo.py`)

El AFO es el de Desarrollo: las 6 etapas (sección 3.2). El servicio lo entrega como lo muestra Desarrollo (etapa
actual, su estado y etapas aprobadas) y le suma lo que Java no calcula: un porcentaje, el plan contra lo real y la
evidencia en fotos, que es con lo que el técnico aprueba a distancia. Para cada obra (solicitud con `ETAPA_1` iniciada)
y un día `d` (por defecto, «hoy»), una fila con:

| Campo | Definición | Fuente · precisión |
| :--- | :--- | :--- |
| `inicio_obra` | `fecha_inicio` de `ETAPA_1` | `etapa_obra` · exacto |
| `etapa_actual`, `estado_etapa`, `etapas_aprobadas` | Como en la web de Desarrollo: la etapa activa a `d` y su estado, o «Obra finalizada» con las 6 aprobadas; cuántas hay aprobadas (0 a 6) | historia de `etapa_obra` · exacto o intervalo |
| `afo` | 100 × Σ `dias_estimados` de las etapas aprobadas hasta `d` / 29. Se pondera por días porque las etapas no pesan lo mismo: la 3 son 10 de los 29 días y la 1, 2 | `fecha_revision` de las aprobadas · exacto |
| `afo_plan` | 100 × Σ `dias_estimados` de las etapas `k` con `fin_plan(k) ≤ d` / 29, donde `fin_plan(k) = sumar_habiles(inicio_obra, Σ_{j ≤ k} dias_estimados_j)` | calendario con feriados · exacto |
| `brecha_habiles` | días estimados del plan a `d` − días estimados aprobados a `d` (positivo: atrasada; negativo: adelantada) | derivado |
| `vencida_habiles` | H(`fecha_limite`, d) si la etapa actual está vencida, si no 0 | `etapa_obra` · exacto |
| `fotos_etapa_actual` | fotos subidas a la etapa actual en la ronda en curso | `foto_etapa` · exacto |
| `aprobadas_sin_fotos` | etapas aprobadas cuya última ronda se envió sin ninguna foto: el técnico aprobó sin evidencia | `foto_etapa` y la historia de etapas |

Reglas para que el dato sea robusto:

- Una obra con una etapa `RECHAZADA` queda «rechazada»: su `afo` se queda en las etapas aprobadas, no cuenta como
  atrasada sino como detenida, y `etapa_actual` es la rechazada.
- Si el mundo tiene etapas trabadas (con `[E10]` activado o datos viejos de la base de pruebas), una etapa
  `PRORROGADA` vencida cuenta como vencida desde su límite vigente.
- Las rondas (para `fotos_etapa_actual` y `aprobadas_sin_fotos`) salen de la historia de etapas (sección 7.5): la ronda
  en curso empieza en el último `inicio` u `observacion` de la etapa y termina en el envío siguiente.
- Cada campo lleva su fuente y su precisión; la confianza del indicador que lo use es la peor de las que use.
- Los restos del AFO anterior (`visitaObra.afo`, `vivienda.avance_obra`) no entran en ningún campo.
- Con el origen simulado, la evaluación compara `afo` contra el avance físico real `[E18]`, y separa cuánto del desvío
  viene del modo sin conexión (`[E9]`).

## 9. Indicadores

H = días hábiles (5.1); est = días estimados de la etapa; «obra» = solicitud con `ETAPA_1` iniciada; «hoy» = fecha de la
última corrida correcta. Cada indicador es un `analitica.indicadores.base.Indicador` (`codigo`, `pregunta`,
`explicacion`, `como_leerlo`, `confianza`, `capacidad`, `valor`; la explicación y el cómo leerlo son obligatorios).

| Código | Definición | Unidad |
| :--- | :--- | :--- |
| `duracion_etapa` | En etapas `APROBADA`: d = H(fecha_inicio, fecha_revision). Por etapa: mediana, p75 y máximo de d y de d − est; % con d ≤ est | días hábiles, % |
| `cumplimiento_plazo` | Universo: etapas con envío más las abiertas con hoy > fecha_limite. % cuyo primer envío fue ≤ fecha_limite_original; % cuyo último envío fue ≤ fecha_limite | % |
| `semaforo_obras` | Por obra: alguna etapa `RECHAZADA` → «rechazada»; las 6 aprobadas → «terminada»; si no, con la etapa activa: `EN_REVISION` → si hoy > fecha_limite, «vencida (N días hábiles esperando al técnico)» con N = H(fecha_envio, hoy); si no, «esperando técnico» («revisión demorada» si H(fecha_envio, hoy) > 2); `EN_CURSO`, `OBSERVADA` o `PRORROGADA` → hoy > fecha_limite → «vencida» («vencida con prórroga» si fecha_limite ≠ fecha_limite_original); H(hoy, fecha_limite) ≤ 1 → «por vencer»; resto → «en plazo». Códigos en `obras.SEMAFOROS` | obras |
| `afo` | Con la sección 8: obras por etapa actual y etapas aprobadas (como las muestra Desarrollo); distribución de `afo` contra `afo_plan` y de `brecha_habiles`; obras atrasadas más de 5 días hábiles; por organización, por grupo (OG u ONG), por tipo de vivienda y por departamento | obras, %, días hábiles |
| `prorrogas` | Prorrogada: fecha_limite ≠ fecha_limite_original. Días = H(original, límite). % sobre etapas iniciadas, mediana de días, por etapa y por organización; % que supera 14 días hábiles; motivos | %, días hábiles |
| `observaciones_rondas` | Rondas = eventos `envio` por etapa. Tasa = etapas con 2 rondas o más / etapas enviadas; cuello de botella = la etapa con mayor tasa | %, rondas |
| `tiempo_revision` | r = H(fecha_envio, fecha_revision) por ronda. Mediana y p90 por etapa y por técnico; cola actual: etapas `EN_REVISION` con su antigüedad | días hábiles |
| `evidencia_fotos` | Fotos por ronda y por etapa; rondas enviadas sin fotos; etapas aprobadas sin fotos (`aprobadas_sin_fotos`), por técnico; fotos borradas (eventos `foto_borrada`); subidas fuera de horario o en ráfaga (señal de carga sin conexión) | fotos, % |
| `quien_carga` | Envíos de etapa y fotos por cargo del integrante (presidente, tesorero, otros) y por organización; integrantes activos y con usuario por organización | envíos, % |
| `embudo_obras` | Solicitudes → con expediente → con visita inicial → aprobadas → vivienda completada → obras iniciadas → terminadas (`ETAPA_6` aprobada). Por cohorte mensual. Duración total H(inicio de ETAPA_1, aprobación de ETAPA_6) contra 29, separada en tiempo de la organización y de revisión | obras, %, días hábiles |
| `tramite_solicitudes` | Tiempos registro → expediente → visita inicial → aprobación (con su precisión), % semiaprobadas y tiempo de corrección, rechazos y sus motivos; por operador | días corridos, % |
| `circuito_documental` | Por tipo de documento: % marcados para corregir, días entre subida y revisión, días hasta el reemplazo; documentos pendientes por solicitud y por organización; rechazados cuyo archivo se borró; por operador. Los estados se muestran tal cual (decisión 18) | %, días |
| `alta_organizaciones` | Duración del alta (fecha_solicitud → aprobación o rechazo), % observadas, % rechazadas; organizaciones con vigencia vencida o por vencer en 60 días; última asamblea de hace más de 2 años (renovación bianual); por grupo (OG u ONG) | días, %, organizaciones |
| `desempeno_organizacion` | Por CUIT y por grupo (OG u ONG): % en plazo, % prorrogadas, tasa de observación, desvío mediano, brecha de avance, obras vencidas hoy, documentos a corregir, vigencia. «pocos datos» si tiene menos de 5 etapas | %, días hábiles |
| `carga_tecnicos` | Por técnico (`username`): revisiones en los últimos 30 días corridos, mediana de r, visitas iniciales, cola de etapas esperando su revisión | revisiones, días hábiles |
| `visitas_iniciales` | Distribución de clasificaciones, % con prioridad social, habitantes encontrados contra familiares declarados (`familias.cantidad_familiares`), antigüedad del rancho, por departamento; días corridos de solicitud a visita; tasa de aprobación por clasificación | %, personas, días |
| `calidad_datos` | Los huecos de la sección 3.4 contados sobre los datos (etapas rechazadas sin salida, etapas prorrogadas trabadas si las hay, obras terminadas sin cerrar, solicitudes aprobadas sin `tecnico_asignado`, viviendas sin completar, `fecha_inicio`/`fecha_fin` de vivienda vacías, límites que no coinciden con el calendario de Java); filas huérfanas; incoherencias (envío anterior al inicio, más de una etapa activa por obra); señales de carga sin conexión; obras con notificaciones incompletas y el desfase estimado de las notificaciones; % de obras con historia completa | conteos, % |

**Confianza:** `sin_confirmar` con el origen simulado; con los reales, `confirmado` si el indicador usa solo fechas que
pone el servidor y `inferido` si usa eventos de precisión `intervalo`. **Capacidad:** `disponible`, salvo los que usan
fotos con el origen API sin acceso a las fotos (`parcial`) y los que no tienen datos (`no_disponible`, con la
explicación de qué falta).

## 10. API `/v1` (`analitica/api/`)

| Ruta | Clave | Devuelve |
| :--- | :--- | :--- |
| `GET /v1/salud` | ninguna | `estado` (`ok`, `degradado` si pasaron más de 2 intervalos sin corrida correcta, `desactualizado` a las 24 h, `sin_datos`), `ultima_corrida`, `ultima_corrida_ok`, `origen`, `datos_al`. Nada de datos de obras |
| `GET /v1/obras` | lectura | cada obra: `id_solicitud`, `num_exp`, organización y su grupo, departamento y localidad, `etapa_actual`, `estado_etapa`, `fecha_limite`, `semaforo`, `dias_habiles_restantes`, `etapas_aprobadas`, `afo`, `brecha_habiles`. Filtros: `semaforo`, `cuit`, `grupo`, `departamento`, `tipo_vivienda` |
| `GET /v1/obras/{id}` | lectura | la ficha completa: solicitud, familia (conteos), vivienda, organización, etapas, fotos, visita inicial, documentos y el AFO por etapas |
| `GET /v1/obras/{id}/linea-de-tiempo` | lectura | la línea de tiempo (7.4) con la precisión de cada evento |
| `GET /v1/indicadores` y `/v1/indicadores/{codigo}` | lectura | los `Indicador` serializados |
| `GET /v1/etapas/resumen` | lectura | por etapa: obras en ella, vencidas, mediana de duración y de desvío, tasa de observación |
| `GET /v1/organizaciones` y `/v1/organizaciones/{cuit}` | lectura | `desempeno_organizacion`; la ficha con integrantes, documentos, alta y obras |
| `GET /v1/tecnicos` | lectura | `carga_tecnicos` |
| `GET /v1/documentos/resumen` | lectura | `circuito_documental` |
| `POST /v1/sincronizar` | administración | corre una sincronización y devuelve su resumen |

- Claves en el encabezado `X-API-Key`, comparadas con `secrets.compare_digest`: `ANALITICA_CLAVE_LECTURA` y
  `ANALITICA_CLAVE_ADMIN`. Sin clave → 401; clave de lectura en una ruta de administración → 403. Si no hay claves
  configuradas, la API rechaza todo salvo `/v1/salud`.
- CORS solo para los orígenes de `ANALITICA_CORS_ORIGENES` (separados por coma).
- Todas las respuestas llevan `datos_al`. La API nunca consulta el origen salvo en `POST /v1/sincronizar`.
- Corre con `uvicorn analitica.api.app:app`; `/docs` muestra la documentación automática.

## 11. Configuración

| Variable | Uso | Por defecto |
| :--- | :--- | :--- |
| `ANALITICA_ORIGEN` | `simulado`, `mysql`, `api` o `vacio` (apaga el simulador y deja todo vacío) | `simulado` |
| `ANALITICA_MYSQL_URL` | conexión de solo lectura a `vivso3` (usuario `lector` en Aiven) | — |
| `ANALITICA_MYSQL_SSL_CA` | ruta al certificado de la autoridad para la conexión cifrada (Aiven la exige) | vacío → sin cifrado |
| `ANALITICA_VIVSO_API_URL`, `_USUARIO`, `_CLAVE` | origen API REST | — |
| `ANALITICA_DB_URL` | nuestra base | `sqlite:///datos_locales/analitica.db` |
| `ANALITICA_INTERVALO_MIN` | minutos entre corridas (sincronizador continuo y estado de `/v1/salud`) | 30 |
| `ANALITICA_CLAVE_LECTURA`, `ANALITICA_CLAVE_ADMIN` | claves de la API | — |
| `ANALITICA_CORS_ORIGENES` | orígenes web permitidos | vacío |
| `ANALITICA_API_URL` | dónde encuentra el tablero a la API | vacío → el origen en el mismo proceso |
| `ANALITICA_FERIADOS_URL` | de dónde bajar los feriados | `https://api.argentinadatos.com/v1/feriados/{año}` |
| `ANALITICA_FERIADOS_TIPOS` | qué tipos cuentan como no hábiles | `inamovible,trasladable,puente` |

`.env.example` lista todas, sin valores reales. `.env` y el certificado nunca se suben (están en el `.gitignore`).

## 12. Tablero (`tablero/`)

Aplicación Streamlit propia, con `tablero/cliente.py`: si hay `ANALITICA_API_URL`, consume la API; si no, calcula con
el origen de `ANALITICA_ORIGEN` en el mismo proceso (con `analitica.respuestas`, las mismas funciones que arman el JSON
de la API, así los dos modos muestran lo mismo): con `simulado` lo rotula «datos simulados», y con `vacio` cada
página muestra su estado vacío («todavía no hay datos») en lugar de un error. Páginas: Resumen; Obras (semáforo, lista y
ficha con la línea de tiempo); Etapas; AFO; Evidencia en fotos; Visitas iniciales; Trámite y documentos;
Organizaciones; Técnicos; Calidad de datos. Cada indicador muestra su pregunta, su explicación y cómo leerlo. Se publica
en Streamlit Cloud **solo con el origen simulado**; con datos reales corre en un servidor con acceso restringido.

## 13. Despliegue (Etapa 8) y monitoreo (Etapa 9)

- `Dockerfile` (python:3.13-slim, usuario sin privilegios, instala `requirements.txt`), `.dockerignore` (`.env`,
  `.venv/`, `datos_locales/`, `salidas/`, `.git/`, certificados).
- `docker-compose.yml`: `api` (uvicorn, puerto 8000 solo en la red interna), `sincronizador`
  (`python -m analitica sincronizar --cada 30`) y `tablero`; perfil `prueba` con `mysql:8` cargado por
  `python -m analitica volcar` con el esquema real.
- Propuesta: el servicio corre en un servidor propio o junto al backend de Desarrollo cuando lo desplieguen, y lee la
  base de Aiven por conexión cifrada con el usuario `lector`; con 1 vCPU, 1 GB de memoria y 2 GB de disco alcanza.
  **Nuestra base guarda datos personales y sensibles** (nombres, DNI, domicilios, el tipo de documento «certificado de
  discapacidad»): tiene que tener el mismo control de acceso que `vivso3`, y la documentación lo registra entre las
  condiciones de la Ley 25.326.
- Monitoreo: frescura (`/v1/salud`); corridas fallidas (3 seguidas o esquema inválido → revisar credenciales, el
  certificado o el esquema con Desarrollo); volumen (caída de más del 20 % de filas leídas → sospechar de un problema
  de acceso); filas huérfanas en aumento; feriados del año siguiente cargados antes de diciembre; informe semanal al
  área (vencidas, revisiones demoradas, prórrogas de más de 14 días, etapas rechazadas, etapas aprobadas sin fotos,
  documentos sin revisar); obras con notificaciones incompletas y cambios en el desfase estimado. `python -m analitica salud` termina con código de error si el estado no es `ok`, para el
  Programador de tareas de Windows o `cron`.

## 14. Pruebas

En `tests/`, con TDD, datos armados a mano, sin datos reales y sin red:

- Calendario: casos de referencia (viernes + 2 → martes; sábado y domingo como inicio; n = 0; un feriado en el medio) y
  la propiedad sobre dos años, con y sin feriados. Feriados: lectura del formato de ArgentinaDatos, archivo local y
  respaldo sin conexión (con un cliente HTTP falso).
- Catálogo: igual a los enums de Java y a las columnas `enum` de la base (valores exactos, incluidos `"DERRUMBE"` y `"OTRA"`); grupos OG/ONG.
- Contrato de orígenes: parametrizada por origen; columnas, orden, tipos, columnas prohibidas y relaciones.
- Simulador: reproducible por semilla; cada regla de Java de la sección 3.2 (transiciones permitidas, vivienda vacía al
  aprobar, fotos solo en `EN_CURSO` u `OBSERVADA`, `[E10]`); momento registrado ≥ real.
- Eventos: un caso por regla de 7.3, incluidas «observación y reenvío entre dos corridas», «dos prórrogas → una neta» y «foto borrada».
- Historia (7.5): deduplicación por destinatario, desfase de 3 horas estimado y corregido, número de etapa deducido (con
  y sin notificaciones desde el inicio de la obra), obra con notificaciones incompletas que usa las versiones, rechazo.
- Sincronizador: dos corridas iguales → `sin_cambios`; candado; origen caído → `error` sin escritura; base con datos de otro origen.
- MySQL contra SQLite con el esquema real de prueba (nombres en camelCase, `bit(1)`, `visitaObra` ausente); API REST con `httpx.MockTransport`.
- AFO por etapas: obra recién iniciada, obra finalizada, obra rechazada, obra adelantada respecto del plan, etapa aprobada sin fotos, feriado dentro de una etapa.
- Indicadores: bordes (aprobada el mismo día, inicio en sábado, abiertas sin vencer, menos de 5 etapas).
- API: 401, 403, CORS permitido y rechazado; recorrido de credenciales sobre todas las respuestas (ningún campo con
  `token`, `password` o `hash`, y de las familias solo los conteos).
- Tablero: `AppTest` en los modos simulado, vacío y con API; importar el núcleo no carga fastapi ni streamlit (prueba en subproceso).

## 15. Documentación del entregable (`docs/`)

| Documento | Contenido | Plan |
| :--- | :--- | ---: |
| `README.md` (raíz) | Qué es, estado, instalación, uso, estructura, datos personales, antecedentes | 1, se completa en cada plan |
| `docs/README.md` | Índice de la documentación | 1 |
| `docs/sistema-de-desarrollo.md` | El sistema que leemos: roles, flujos, estados, plazos, las 14 tablas con sus columnas reales, lo que respondió Desarrollo y los huecos que afectan a la analítica | 1 |
| `docs/pedido-a-desarrollo.md` | Qué le pedimos a Desarrollo, qué respondió, qué queda pendiente y qué le ofrecemos (sin los puntos de seguridad: decisión 20) | 1 |
| `docs/simulacion.md` | Cómo funciona el origen simulado, cómo se apaga y sus supuestos `[E1]`–`[E19]` | 1 |
| `docs/sincronizacion.md` | Cómo se recupera la historia (notificaciones y versiones), las reglas de 7.3 y 7.5 y el experimento de frecuencia | 2 |
| `docs/afo.md` | El AFO por etapas de la sección 8: cómo lo maneja Desarrollo, qué le sumamos y ejemplos | 3 |
| `docs/indicadores.md` | Catálogo de indicadores: pregunta, definición, cómo leerlo, confianza (generado desde el registro) | 3 |
| `docs/api.md` | Rutas, claves y ejemplos, para el equipo de Desarrollo | 4 |
| `docs/tablero.md` | Qué muestra cada página y cómo leerla, para el área | 5 |
| `docs/etapa-8-despliegue.md` | Propuesta de implementación: entorno, recursos, acceso, datos personales, cómo corre, limitaciones | 6 |
| `docs/etapa-9-monitoreo.md` | Qué se observa, cada cuánto y qué acción dispara | 6 |
| `docs/etapa-6-evaluacion.md` | Criterios, cómo se midió cada uno, resultados, fortalezas, limitaciones y condiciones de validez | 7 |
| `docs/etapa-7-insights.md` | Resultados, interpretaciones y conclusiones, separados | 7 |
| `docs/glosario.md` | Los términos del proyecto en lenguaje llano (lo pide el profesor) | 8 |

**Criterios de la Etapa 6** (se fijan ahora, se miden en los Planes 2 a 7):

1. Fidelidad al sistema de Desarrollo: casos de referencia del calendario, igualdad con los enums y la base, y reglas de cada flujo (pruebas automáticas).
2. Recuperación de la historia: con el simulado como verdad, % de eventos reales que el sincronizador recupera con su
   fecha exacta, según la frecuencia de corrida (15 min, 30 min, 1 h, 6 h, 24 h), sin notificaciones y con ellas.
3. Exactitud del AFO: error de `afo` contra el avance físico real del simulado, separando el sesgo del modo sin conexión.
4. Exactitud de los indicadores: diferencia entre el indicador calculado desde nuestra base y el calculado desde la verdad del simulado.
5. Credenciales y datos reales: ninguna respuesta con tokens ni contraseñas (prueba); ningún dato real en el repositorio ni en la demo pública.
6. Continuidad: con el origen caído, la API sigue respondiendo e informa la antigüedad de los datos (prueba).
7. Lectura real: una sincronización contra la base de Aiven sin errores de esquema (cuando lleguen la contraseña y el certificado).
8. Utilidad: devolución del área y de Desarrollo sobre el tablero (cualitativa; si no llega a tiempo, queda como pendiente declarado).

**Etapa 7** distingue dos fuentes: lo que surge del análisis del sistema real (sección 3.4: lo que no guarda, el sesgo
del modo sin conexión, la etapa trabada que hubo hasta el 31/10, el rechazo sin salida, la hora sin zona de las notificaciones; y que el técnico aprueba a distancia, con las fotos
como única evidencia) y lo que muestran los resultados con datos simulados, que sirven para mostrar qué se podrá
concluir con datos reales, no para describir el programa.

Los documentos de las Etapas 6 a 9 se escriben para poder copiarse tal cual a la carpeta de la cátedra
(`VIVSO_PP_ITSE/01_Documentacion_Metodologia/Etapa_6_…` a `Etapa_9_…`), y `VIVSO_PP_ITSE/04_Notebooks_Codigo` enlaza al
repositorio de `vivso-analitica`.

## 16. Limitaciones que se registran

No hay historia anterior a la primera corrida ni a la primera notificación (las notificaciones existen desde el
03/11/2026); el sesgo del modo sin conexión en envíos, fotos y revisiones; mientras
Java no cuente feriados, sus límites y nuestros días hábiles pueden diferir cuando hay un feriado en el medio; los
feriados provinciales no están en ArgentinaDatos (Desarrollo los resolvería con una reasignación de la fecha); el
esquema puede cambiar sin aviso por `ddl-auto`; sin notificaciones, las prórrogas encadenadas entre corridas quedan
como una neta; la etapa de cada notificación se deduce (Java no la guarda) y su hora depende de la zona de la JVM; las fotos
subidas y borradas entre dos corridas no se ven; las fechas de semiaprobación y rechazo son aproximadas; el AFO mide lo
que se registra y se aprueba, no la obra física, y el técnico aprueba con las fotos; con la API REST hacen falta muchas
llamadas y un usuario `ADMIN`; nuestra base guarda datos personales y sensibles y depende de dónde y cómo se despliegue.

## 17. Orden de trabajo

| Plan | Qué entrega | Depende de | Archivo |
| :--- | :--- | :--- | :--- |
| 1 (hecho) | Esqueleto, dominio, esquema, origen vacío, simulador, documentos del sistema de Desarrollo | — | (ejecutado el 21/10/2026) |
| 1b | Realineación con `develop` y base común: `RECHAZADA`, técnico asignado, notificaciones (tabla 14), `historia.py` | 1 | `planes/plan-1b-realineacion-y-base-comun.md` |
| 2 | Base propia, versiones, eventos, sincronizador, línea de tiempo, origen MySQL, experimento de frecuencia | 1b | `planes/plan-2-sincronizador-e-historia.md` |
| 3 | AFO por etapas, tabla de obras y semáforo, los 17 indicadores y las respuestas JSON | 1b | `planes/plan-3-afo-e-indicadores.md` |
| 4 | API `/v1` con claves, CORS y salud | 2 y 3 | `planes/plan-4-api.md` |
| 5 | Tablero Streamlit (modo local y modo API) | 3 (y 4 para el modo API) | `planes/plan-5-tablero.md` |
| 6 | Docker, volcado a MySQL de prueba, salud, monitoreo, informe semanal, documentos de las Etapas 8 y 9 | 2, 4 y 5 | `planes/plan-6-despliegue-y-monitoreo.md` |
| 7 | Evaluación (criterios 1 a 8) e insights: código de medición y documentos de las Etapas 6 y 7 | 2 y 3 | `planes/plan-7-evaluacion-e-insights.md` |
| 8 | Cierre: glosario, carpeta de la cátedra, Ficha de PP3, presentación final a la entidad | todos | `planes/plan-8-cierre-y-presentacion.md` |

```mermaid
flowchart LR
    P1["1 · hecho"] --> P1b["1b"]
    P1b --> P2["2"]
    P1b --> P3["3"]
    P2 --> P4["4"]
    P3 --> P4
    P3 --> P5["5"]
    P4 --> P5
    P2 --> P6["6"]
    P4 --> P6
    P5 --> P6
    P2 --> P7["7"]
    P3 --> P7
    P6 --> P8["8"]
    P7 --> P8
```

Cada plan se trabaja con pruebas primero, termina con la suite completa en verde y, si se puede, con una revisión de
otra persona del equipo antes de unir la rama.
