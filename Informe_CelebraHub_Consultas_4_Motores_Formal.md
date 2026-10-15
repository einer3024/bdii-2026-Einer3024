# Informe de práctica de bases de datos: consultas SQL en cuatro motores
## Caso de estudio: CelebraHub — Centro de eventos

| | |
|---|---|
| **Universidad** | Universidad de La Guajira |
| **Programa** | Ingeniería de Sistemas |
| **Asignatura** | Base de datos II |
| **Estudiante** | Einer Coa Jimenez |
| **Docente** | Jaider Quintero Mendoza |
| **Motores utilizados** | MySQL 8.x · PostgreSQL 12+ · Microsoft SQL Server 2017+ · Oracle Database 12c+ |
| **Herramienta cliente** | DBeaver |
| **Fecha** | Octubre de 2026 |

---

## Tabla de contenido

1. [Introducción](#1-introducción)
2. [Objetivos](#2-objetivos)
3. [Descripción del caso CelebraHub](#3-descripción-del-caso-celebrahub)
4. [Entorno de trabajo](#4-entorno-de-trabajo)
5. [Modelo de datos](#5-modelo-de-datos)
6. [Datos de prueba](#6-datos-de-prueba)
7. [Procedimiento para obtener la evidencia fotográfica](#7-procedimiento-para-obtener-la-evidencia-fotográfica)
8. [Diferencias de sintaxis entre los motores](#8-diferencias-de-sintaxis-entre-los-motores)
9. [Desarrollo: consultas por tema](#9-desarrollo-consultas-por-tema)
    - [0. Verificación de la carga de datos](#0-verificación-de-la-carga-de-datos)
    - [1. LIKE — búsqueda por patrones](#1-like--búsqueda-por-patrones)
    - [2. Subconsultas](#2-subconsultas)
    - [3. GROUP BY — agrupación y agregación](#3-group-by--agrupación-y-agregación)
    - [4. HAVING — filtro sobre grupos](#4-having--filtro-sobre-grupos)
    - [5. ORDER BY — ordenamiento](#5-order-by--ordenamiento)
    - [6. JOIN — combinación de tablas](#6-join--combinación-de-tablas)
    - [7. WHERE — filtros por fila](#7-where--filtros-por-fila)
10. [Matriz de verificación de evidencias](#10-matriz-de-verificación-de-evidencias)
11. [Análisis comparativo entre motores](#11-análisis-comparativo-entre-motores)
12. [Conclusiones](#12-conclusiones)
13. [Anexos](#13-anexos)

---

## 1. Introducción

El presente informe documenta la ejecución de un conjunto de consultas SQL sobre la base de datos de **CelebraHub**, un centro de eventos ficticio que alquila salones para celebraciones y eventos corporativos. La base de datos gestiona clientes, salones, servicios y proveedores, reservas con bloqueo temporal de fechas, eventos, servicios contratados por evento, contratos con control de versiones, pagos, cambios de contrato y cancelaciones con penalidad, además de un modelo transversal de autenticación y autorización basado en roles (RBAC).

Cada consulta se formula a partir de una **situación empresarial concreta**: un área del centro de eventos (comercial, operaciones, cartera, gerencia, entre otras) plantea una necesidad de información y la consulta constituye su respuesta técnica. De este modo se evidencia no solo la escritura de la sentencia SQL, sino el propósito y la utilidad de cada cláusula.

El mismo conjunto de consultas se ejecuta en **cuatro sistemas gestores de bases de datos** (MySQL, PostgreSQL, SQL Server y Oracle) con el fin de comparar las diferencias de sintaxis y comprobar que, con las adaptaciones necesarias, los resultados son equivalentes. La evidencia de cada ejecución se registra mediante capturas de pantalla tomadas en DBeaver.

## 2. Objetivos

### 2.1 Objetivo general

Aplicar consultas SQL de selección sobre una base de datos con más de 200 registros ficticios, justificando cada una mediante una situación del negocio y verificando su ejecución en cuatro motores de bases de datos distintos.

### 2.2 Objetivos específicos

- Construir consultas con los operadores `LIKE`, subconsultas, `GROUP BY`, `HAVING`, `ORDER BY`, `JOIN` y `WHERE`.
- Relacionar cada consulta con una necesidad real de un área de la organización.
- Ejecutar el mismo conjunto de consultas en MySQL, PostgreSQL, SQL Server y Oracle, adaptando la sintaxis cuando el motor lo exija.
- Documentar con evidencia fotográfica el resultado de cada ejecución.
- Identificar y explicar las diferencias de dialecto entre los motores (limitación de filas, concatenación, funciones de fecha y sensibilidad a mayúsculas).

## 3. Descripción del caso CelebraHub

CelebraHub administra un centro de eventos con varios salones de distinta capacidad. El sistema debe registrar reservas sin solapamientos de agenda, convertirlas en eventos, contratar los servicios complementarios (catering, mobiliario, decoración, sonido, entre otros) con proveedores externos, formalizar el acuerdo mediante un contrato y controlar los pagos del cliente. Las modificaciones y cancelaciones quedan registradas como trazabilidad.

### 3.1 Áreas de la organización que formulan las solicitudes

| Área | Qué necesita saber | Tablas que consulta con más frecuencia |
|---|---|---|
| Comercial | Segmentar clientes por correo o tipo de evento, recuperar clientes inactivos, agenda de reservas | `clientes`, `reservas`, `salones` |
| Operaciones | Servicios pendientes, eventos planeados, cruces de agenda, carga por proveedor | `eventos`, `evento_servicios`, `proveedores` |
| Cartera y Tesorería | Recaudo, pagos pendientes, saldos, métodos de pago, penalidades | `pagos`, `contratos`, `cancelaciones` |
| Proveedores | Servicios asignados y confirmados por proveedor | `proveedores`, `evento_servicios`, `servicios` |
| Gerencia | Demanda por salón, ingresos mensuales, servicios más vendidos | todas |
| Seguridad / Auditoría | Qué rol puede ejecutar qué operación (RBAC) | `users`, `roles`, `resources` |

### 3.2 Reglas de negocio relevantes

- Un salón no puede tener dos reservas activas cuyos horarios se crucen.
- Una reserva puede estar en estado `PRE_RESERVA` (con fecha de expiración del bloqueo), `CONFIRMADA`, `CANCELADA` o `FINALIZADA`.
- Una reserva tiene como máximo un evento (`UNIQUE` sobre `reserva_id`) y un evento tiene exactamente un contrato (`UNIQUE` sobre `evento_id`).
- Todo contrato exige un porcentaje de anticipo (30 % por defecto) y conserva un número de versión; los cambios aprobados generan una nueva versión.
- Los pagos pueden ser de tipo `ANTICIPO`, `ABONO`, `SALDO` o `PENALIDAD`.
- Las cancelaciones se conservan como trazabilidad: no se elimina información, se modifica su estado.

## 4. Entorno de trabajo

La práctica se desarrolló sobre cuatro sistemas gestores de bases de datos: **MySQL 8.x**, **PostgreSQL 12+**, **Microsoft SQL Server 2017+** y **Oracle Database 12c+**. Las consultas se ejecutaron desde **DBeaver**, un cliente universal que permite administrar las cuatro conexiones desde una misma interfaz.

En cada motor se creó la base de datos con el script `celebrahub_<motor>.sql` (esquema y datos semilla) y se amplió con `celebrahub_datos_<motor>.sql` (datos de prueba), de manera que los resultados fueran comparables. En PostgreSQL todo se crea dentro del esquema `celebrahub`; en Oracle se utilizó el esquema propio del usuario, evitando el usuario `SYS`.

| Motor | Versión | Cliente | Observación |
|---|---|---|---|
| MySQL | 8.x | DBeaver | Juego de caracteres `utf8mb4` |
| PostgreSQL | 12+ | DBeaver | Esquema `celebrahub`; ejecutar `SET search_path TO celebrahub;` |
| SQL Server | 2017+ | DBeaver | Base de datos `CelebraHub` |
| Oracle | 12c+ | DBeaver | Esquema propio del usuario |



![Diagrama entidad-relación](evidencias/diagrama_er.png)

*Figura 1. Diagrama entidad-relación del negocio.*

### 5.2 Relaciones del negocio

- **Cliente 1:N Reserva** y **Salón 1:N Reserva**.
- **Reserva 0..1:1 Evento** (restricción `uq_eventos_reserva`).
- **Evento N:M Servicio**, resuelta mediante la tabla asociativa `evento_servicios`, que además referencia al **Proveedor** que ejecuta cada servicio.
- **Evento 1:1 Contrato** (restricción `uq_contratos_evento`) y **Cliente 1:N Contrato**.
- **Contrato 1:N Pago**, **Contrato 1:N Cambio de contrato** y **Contrato 1:N Cancelación**.

### 5.3 Estructura de las tablas

| Tabla | Columnas principales | Relaciones (FK) |
|---|---|---|
| `clientes` | `id`, `tipo_documento`, `numero_documento`, `nombre`, `telefono`, `email`, `is_active` | — |
| `salones` | `id`, `nombre`, `descripcion`, `capacidad`, `precio_base`, `is_active`, `created_at`, `updated_at` | — |
| `servicios` | `id`, `nombre`, `descripcion`, `precio_base`, `is_active`, `created_at`, `updated_at` | — |
| `proveedores` | `id`, `nit`, `razon_social`, `contacto`, `telefono`, `email`, `is_active` | — |
| `reservas` | `id`, `cliente_id`, `salon_id`, `fecha_inicio`, `fecha_fin`, `estado`, `bloqueo_expira_at`, `observaciones` | `cliente_id` → `clientes`, `salon_id` → `salones` |
| `eventos` | `id`, `reserva_id`, `tipo`, `fecha`, `cantidad`, `observaciones`, `estado` | `reserva_id` → `reservas` |
| `evento_servicios` | `id`, `evento_id`, `servicio_id`, `proveedor_id`, `tipo`, `fecha`, `cantidad`, `precio_unitario`, `observaciones`, `estado` | `evento_id` → `eventos`, `servicio_id` → `servicios`, `proveedor_id` → `proveedores` |
| `contratos` | `id`, `cliente_id`, `evento_id`, `numero`, `fecha_inicio`, `fecha_fin`, `valor`, `porcentaje_anticipo`, `version_actual`, `estado` | `cliente_id` → `clientes`, `evento_id` → `eventos` |
| `pagos` | `id`, `contrato_id`, `referencia_tipo`, `referencia_id`, `tipo_pago`, `metodo`, `monto`, `fecha`, `estado` | `contrato_id` → `contratos` |
| `cambios_contrato` | `id`, `contrato_id`, `version`, `nombre`, `descripcion`, `valor_anterior`, `valor_nuevo`, `estado`, `is_active`, `created_at`, `updated_at` | `contrato_id` → `contratos` |
| `cancelaciones` | `id`, `contrato_id`, `nombre`, `descripcion`, `porcentaje_penalidad`, `valor_penalidad`, `estado`, `is_active`, `created_at`, `updated_at` | `contrato_id` → `contratos` |

Dominios de estado: reservas (`PRE_RESERVA`, `CONFIRMADA`, `CANCELADA`, `FINALIZADA`), eventos (`PLANEADO`, `EN_CURSO`, `REALIZADO`, `CANCELADO`), servicios del evento (`PENDIENTE`, `CONFIRMADO`, `CANCELADO`), contratos (`BORRADOR`, `VIGENTE`, `MODIFICADO`, `CANCELADO`, `FINALIZADO`), pagos (`PENDIENTE`, `APLICADO`, `ANULADO`), cambios (`SOLICITADO`, `APROBADO`, `RECHAZADO`) y cancelaciones (`SOLICITADA`, `APLICADA`, `ANULADA`).

### 5.4 Modelo RBAC

El control de acceso se resuelve con seis tablas: `users`, `roles`, `role_users` (usuarios N:M roles), `resources` (cada recurso se identifica por la pareja `path` y `method`), `resource_roles` (roles N:M recursos) y `refresh_tokens` (sesiones por dispositivo). Una asignación con estado `INACTIVE` no concede privilegios. Los roles iniciales son ADMIN, COMERCIAL, OPERACIONES, PROVEEDOR y CARTERA, y los recursos protegidos son `POST /reservas`, `POST /contratos`, `POST /cambios-contrato` y `POST /cancelaciones`.

## 6. Datos de prueba

Los datos son **100 % ficticios**: nombres, documentos y correos fueron generados de forma pseudoaleatoria y no corresponden a personas reales. Se cargan con el archivo `celebrahub_datos_<motor>.sql`, que contiene únicamente sentencias `INSERT` en el orden que respeta las llaves foráneas y que se ejecuta **después** del script de esquema. Se suman a los datos semilla originales (2 clientes, 3 salones, 4 servicios y 2 proveedores).

| Tabla | Registros |
|---|---:|
| clientes | 60 |
| salones | 6 |
| servicios | 8 |
| proveedores | 8 |
| reservas | 120 |
| eventos | 92 |
| evento_servicios | 247 |
| contratos | 87 |
| pagos | 200 |
| cambios_contrato | 16 |
| cancelaciones | 5 |
| **Total (tablas del negocio)** | **849** |

Con 60 clientes, 120 reservas y 849 registros en las tablas del negocio se cumple ampliamente el requisito de más de 200 datos. La consulta **Q00** lo demuestra con evidencia.

### 6.1 Distribución de los datos

- Los clientes son personas naturales (`CC`) y empresas (`NIT`); predominan las cuentas de Gmail, con presencia de Hotmail, Outlook y Yahoo, y algunos clientes inactivos o sin teléfono.
- Hay seis salones (uno de ellos inactivo, `Salon Cielo`, sin reservas), ocho servicios y ocho proveedores (uno sin servicios asignados).
- Las reservas se distribuyen entre enero y diciembre de 2026 sin cruces de horario dentro de un mismo salón; las anteriores al 15 de octubre de 2026 están `FINALIZADA` o `CANCELADA`, y las posteriores `CONFIRMADA`, `PRE_RESERVA` o `CANCELADA`.
- Cada reserva confirmada o finalizada tiene un evento; cada evento tiene entre 1 y 4 servicios contratados y, en la mayoría de los casos, un contrato con su anticipo.
- Los pagos se encuentran en estado `APLICADO` o `PENDIENTE` y se registran por efectivo, tarjeta o transferencia; los contratos cancelados tienen una penalidad.

## 7. Procedimiento para obtener la evidencia fotográfica

La evidencia de cada ejecución consiste en una captura de pantalla tomada en DBeaver. Para cada consulta y para cada motor se siguió el procedimiento descrito a continuación:

1. Abrir un editor SQL asociado a la conexión del motor correspondiente (MySQL, PostgreSQL, SQL Server u Oracle). En PostgreSQL ejecutar antes `SET search_path TO celebrahub;`.
2. Pegar la sentencia del informe (o del archivo `consultas_dbeaver_<motor>.sql`) y ejecutarla con `Ctrl + Enter`.
3. Verificar que la grilla de resultados sea visible junto con el código ejecutado, el nombre de la conexión activa, el número de filas devueltas y el tiempo de ejecución.
4. Tomar la captura de pantalla y guardarla con el nombre `<CÓDIGO>_<motor>.png` (por ejemplo, `L01_mysql.png`) en la carpeta `evidencias/`.
5. Insertar la imagen en el informe bajo la sentencia respectiva, con su pie de figura numerado.
6. Registrar en la tabla de ejecución el número de filas obtenidas, el tiempo y las observaciones.

En total se requieren **188 capturas** (47 consultas × 4 motores), además del diagrama entidad-relación del modelo.

## 8. Diferencias de sintaxis entre los motores

Las consultas son equivalentes en los cuatro motores, aunque algunas debieron reescribirse. El siguiente cuadro resume las diferencias que aparecen en el informe:

| Aspecto | MySQL | PostgreSQL | SQL Server | Oracle |
|---|---|---|---|---|
| Limitar filas | `LIMIT n` | `LIMIT n` | `SELECT TOP n` | `FETCH FIRST n ROWS ONLY` (12c+) |
| Concatenar texto | `CONCAT(a, ' ', b)` | `a \|\| ' ' \|\| b` | `a + ' ' + b` | `a \|\| ' ' \|\| b` |
| Año-mes de una fecha | `DATE_FORMAT(f, '%Y-%m')` | `TO_CHAR(f, 'YYYY-MM')` | `CONVERT(CHAR(7), f, 120)` | `TO_CHAR(f, 'YYYY-MM')` |
| Literal de fecha | `DATE '2026-10-15'` | `DATE '2026-10-15'` | `'2026-10-15'` | `DATE '2026-10-15'` |
| `LIKE` y mayúsculas | No distingue (collation por defecto) | **Distingue** | No distingue (collation por defecto) | **Distingue** |
| Alias de tabla con `AS` | Permitido | Permitido | Permitido | **No permitido** (se omite `AS`) |
| Columnas en `GROUP BY` | Flexible si hay clave primaria | Todas las no agregadas | Todas las no agregadas | Todas las no agregadas |
| Columna autoincremental | `AUTO_INCREMENT` | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | `GENERATED BY DEFAULT AS IDENTITY` |
| Dominios de estado | `ENUM` | `VARCHAR` + `CHECK` | `VARCHAR` + `CHECK` | `VARCHAR2` + `CHECK` |
| Texto | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR2` |
| Fecha y hora | `DATETIME` | `TIMESTAMP` | `DATETIME2` | `TIMESTAMP` |
| Consulta sin tabla | `SELECT 1` | `SELECT 1` | `SELECT 1` | `SELECT 1 FROM dual` |

---

## 9. Desarrollo: consultas por tema

Cada consulta se presenta con la siguiente estructura:

1. **Situación de negocio:** área que solicita la información y motivo de la solicitud.
2. **Concepto de SQL:** cláusula o técnica que se demuestra.
3. **Resultado esperado:** cómo interpretar la grilla de resultados.
4. **Ejecución en los cuatro motores:** código, evidencia fotográfica, registro de resultados y nota de sintaxis.

---

### 0. Verificación de la carga de datos

Antes de ejecutar cualquier consulta se comprueba que la base contiene los datos ficticios. Esta única consulta sirve como evidencia de que el volumen supera los 200 registros pedidos.

#### Q00 — Conteo de registros por tabla

**Situación de negocio.** Antes de responder las solicitudes de las áreas de CelebraHub, la coordinación de sistemas pidió una prueba de que la base quedó cargada con volumen suficiente (más de 200 registros). Sin esa verificación, cualquier consulta posterior podría devolver resultados vacíos por una carga incompleta y no por un error de la consulta.

**Concepto de SQL.** Se usa `COUNT(*)` sobre cada tabla del negocio y se unen los resultados con `UNION ALL`, de modo que toda la verificación aparece en una sola grilla. `UNION ALL` conserva todas las filas sin eliminar duplicados, que es lo que se necesita aquí.

**Resultado esperado.** La grilla muestra una fila por tabla del negocio y una fila final `TOTAL negocio` (849 con los datos cargados). Valores esperados: clientes 60, reservas 120, pagos 200.

##### Q00 · MySQL

```sql
SELECT 'clientes' AS tabla, COUNT(*) AS registros FROM clientes
UNION ALL
SELECT 'salones' AS tabla, COUNT(*) AS registros FROM salones
UNION ALL
SELECT 'servicios' AS tabla, COUNT(*) AS registros FROM servicios
UNION ALL
SELECT 'proveedores' AS tabla, COUNT(*) AS registros FROM proveedores
UNION ALL
SELECT 'reservas' AS tabla, COUNT(*) AS registros FROM reservas
UNION ALL
SELECT 'eventos' AS tabla, COUNT(*) AS registros FROM eventos
UNION ALL
SELECT 'evento_servicios' AS tabla, COUNT(*) AS registros FROM evento_servicios
UNION ALL
SELECT 'contratos' AS tabla, COUNT(*) AS registros FROM contratos
UNION ALL
SELECT 'pagos' AS tabla, COUNT(*) AS registros FROM pagos
UNION ALL
SELECT 'cambios_contrato' AS tabla, COUNT(*) AS registros FROM cambios_contrato
UNION ALL
SELECT 'cancelaciones' AS tabla, COUNT(*) AS registros FROM cancelaciones
UNION ALL
SELECT 'TOTAL negocio' AS tabla, (SELECT COUNT(*) FROM clientes) + (SELECT COUNT(*) FROM salones) + (SELECT COUNT(*) FROM servicios) + (SELECT COUNT(*) FROM proveedores) + (SELECT COUNT(*) FROM reservas) + (SELECT COUNT(*) FROM eventos) + (SELECT COUNT(*) FROM evento_servicios) + (SELECT COUNT(*) FROM contratos) + (SELECT COUNT(*) FROM pagos) + (SELECT COUNT(*) FROM cambios_contrato) + (SELECT COUNT(*) FROM cancelaciones);
```

![Evidencia Q00 en MySQL](evidencias/Q00_mysql.png)

*Figura 2. Resultado de la consulta Q00 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Este motor ejecuta la versión estándar de la sentencia, sin funciones propias del dialecto.

##### Q00 · PostgreSQL

```sql
SELECT 'clientes' AS tabla, COUNT(*) AS registros FROM clientes
UNION ALL
SELECT 'salones' AS tabla, COUNT(*) AS registros FROM salones
UNION ALL
SELECT 'servicios' AS tabla, COUNT(*) AS registros FROM servicios
UNION ALL
SELECT 'proveedores' AS tabla, COUNT(*) AS registros FROM proveedores
UNION ALL
SELECT 'reservas' AS tabla, COUNT(*) AS registros FROM reservas
UNION ALL
SELECT 'eventos' AS tabla, COUNT(*) AS registros FROM eventos
UNION ALL
SELECT 'evento_servicios' AS tabla, COUNT(*) AS registros FROM evento_servicios
UNION ALL
SELECT 'contratos' AS tabla, COUNT(*) AS registros FROM contratos
UNION ALL
SELECT 'pagos' AS tabla, COUNT(*) AS registros FROM pagos
UNION ALL
SELECT 'cambios_contrato' AS tabla, COUNT(*) AS registros FROM cambios_contrato
UNION ALL
SELECT 'cancelaciones' AS tabla, COUNT(*) AS registros FROM cancelaciones
UNION ALL
SELECT 'TOTAL negocio' AS tabla, (SELECT COUNT(*) FROM clientes) + (SELECT COUNT(*) FROM salones) + (SELECT COUNT(*) FROM servicios) + (SELECT COUNT(*) FROM proveedores) + (SELECT COUNT(*) FROM reservas) + (SELECT COUNT(*) FROM eventos) + (SELECT COUNT(*) FROM evento_servicios) + (SELECT COUNT(*) FROM contratos) + (SELECT COUNT(*) FROM pagos) + (SELECT COUNT(*) FROM cambios_contrato) + (SELECT COUNT(*) FROM cancelaciones);
```

![Evidencia Q00 en PostgreSQL](evidencias/Q00_postgresql.png)

*Figura 3. Resultado de la consulta Q00 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Este motor ejecuta la versión estándar de la sentencia, sin funciones propias del dialecto.

##### Q00 · SQL Server

```sql
SELECT 'clientes' AS tabla, COUNT(*) AS registros FROM clientes
UNION ALL
SELECT 'salones' AS tabla, COUNT(*) AS registros FROM salones
UNION ALL
SELECT 'servicios' AS tabla, COUNT(*) AS registros FROM servicios
UNION ALL
SELECT 'proveedores' AS tabla, COUNT(*) AS registros FROM proveedores
UNION ALL
SELECT 'reservas' AS tabla, COUNT(*) AS registros FROM reservas
UNION ALL
SELECT 'eventos' AS tabla, COUNT(*) AS registros FROM eventos
UNION ALL
SELECT 'evento_servicios' AS tabla, COUNT(*) AS registros FROM evento_servicios
UNION ALL
SELECT 'contratos' AS tabla, COUNT(*) AS registros FROM contratos
UNION ALL
SELECT 'pagos' AS tabla, COUNT(*) AS registros FROM pagos
UNION ALL
SELECT 'cambios_contrato' AS tabla, COUNT(*) AS registros FROM cambios_contrato
UNION ALL
SELECT 'cancelaciones' AS tabla, COUNT(*) AS registros FROM cancelaciones
UNION ALL
SELECT 'TOTAL negocio' AS tabla, (SELECT COUNT(*) FROM clientes) + (SELECT COUNT(*) FROM salones) + (SELECT COUNT(*) FROM servicios) + (SELECT COUNT(*) FROM proveedores) + (SELECT COUNT(*) FROM reservas) + (SELECT COUNT(*) FROM eventos) + (SELECT COUNT(*) FROM evento_servicios) + (SELECT COUNT(*) FROM contratos) + (SELECT COUNT(*) FROM pagos) + (SELECT COUNT(*) FROM cambios_contrato) + (SELECT COUNT(*) FROM cancelaciones);
```

![Evidencia Q00 en SQL Server](evidencias/Q00_sqlserver.png)

*Figura 4. Resultado de la consulta Q00 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Este motor ejecuta la versión estándar de la sentencia, sin funciones propias del dialecto.

##### Q00 · Oracle

```sql
SELECT 'clientes' AS tabla, COUNT(*) AS registros FROM clientes
UNION ALL
SELECT 'salones' AS tabla, COUNT(*) AS registros FROM salones
UNION ALL
SELECT 'servicios' AS tabla, COUNT(*) AS registros FROM servicios
UNION ALL
SELECT 'proveedores' AS tabla, COUNT(*) AS registros FROM proveedores
UNION ALL
SELECT 'reservas' AS tabla, COUNT(*) AS registros FROM reservas
UNION ALL
SELECT 'eventos' AS tabla, COUNT(*) AS registros FROM eventos
UNION ALL
SELECT 'evento_servicios' AS tabla, COUNT(*) AS registros FROM evento_servicios
UNION ALL
SELECT 'contratos' AS tabla, COUNT(*) AS registros FROM contratos
UNION ALL
SELECT 'pagos' AS tabla, COUNT(*) AS registros FROM pagos
UNION ALL
SELECT 'cambios_contrato' AS tabla, COUNT(*) AS registros FROM cambios_contrato
UNION ALL
SELECT 'cancelaciones' AS tabla, COUNT(*) AS registros FROM cancelaciones
UNION ALL
SELECT 'TOTAL negocio' AS tabla, (SELECT COUNT(*) FROM clientes) + (SELECT COUNT(*) FROM salones) + (SELECT COUNT(*) FROM servicios) + (SELECT COUNT(*) FROM proveedores) + (SELECT COUNT(*) FROM reservas) + (SELECT COUNT(*) FROM eventos) + (SELECT COUNT(*) FROM evento_servicios) + (SELECT COUNT(*) FROM contratos) + (SELECT COUNT(*) FROM pagos) + (SELECT COUNT(*) FROM cambios_contrato) + (SELECT COUNT(*) FROM cancelaciones) FROM dual;
```

![Evidencia Q00 en Oracle](evidencias/Q00_oracle.png)

*Figura 5. Resultado de la consulta Q00 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Este motor ejecuta la versión estándar de la sentencia, sin funciones propias del dialecto.

---

### 1. LIKE — búsqueda por patrones

El operador `LIKE` compara texto contra un patrón. Dos comodines: `%` (cualquier cantidad de caracteres) y `_` (exactamente un carácter). En esta sección se muestran patrones de prefijo, sufijo, contenido, un solo carácter y negación con `NOT LIKE`.

#### L01 — Correos que empiezan por la letra l

**Situación de negocio.** El área Comercial planea una campaña de contacto y necesita depurar la lista de correos de los clientes. Como primer paso se pidió aislar a quienes tienen un correo que empieza por la letra l, para revisar errores de digitación o direcciones repetidas.

**Concepto de SQL.** `LIKE 'l%'` filtra por **prefijo**: el símbolo `%` representa cualquier cantidad de caracteres. En MySQL y SQL Server la comparación no distingue mayúsculas con la configuración por defecto; en PostgreSQL y Oracle sí, por lo que se aplica `LOWER()` para que el resultado sea idéntico en los cuatro motores.

**Resultado esperado.** Todas las filas devueltas deben tener un correo cuya primera letra sea l.

##### L01 · MySQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.email) LIKE 'l%'
ORDER BY c.email
LIMIT 20;
```

![Evidencia L01 en MySQL](evidencias/L01_mysql.png)

*Figura 6. Resultado de la consulta L01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L01 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.email) LIKE 'l%'
ORDER BY c.email
LIMIT 20;
```

![Evidencia L01 en PostgreSQL](evidencias/L01_postgresql.png)

*Figura 7. Resultado de la consulta L01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L01 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.email) LIKE 'l%'
ORDER BY c.email;
```

![Evidencia L01 en SQL Server](evidencias/L01_sqlserver.png)

*Figura 8. Resultado de la consulta L01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### L01 · Oracle

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.email) LIKE 'l%'
ORDER BY c.email
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia L01 en Oracle](evidencias/L01_oracle.png)

*Figura 9. Resultado de la consulta L01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### L02 — Clientes empresariales cuyo nombre termina en SAS

**Situación de negocio.** Cartera quiere identificar a los clientes constituidos como sociedad por acciones simplificadas (SAS) para preparar la facturación empresarial con sus datos fiscales.

**Concepto de SQL.** `LIKE '%SAS'` filtra por **sufijo** (el patrón empieza con `%`). Es el caso típico de buscar por la terminación de un texto.

**Resultado esperado.** Todas las filas devueltas deben terminar en SAS. Los clientes personas naturales no aparecen.

##### L02 · MySQL

```sql
SELECT c.id, c.tipo_documento, c.numero_documento, c.nombre
FROM clientes c
WHERE c.nombre LIKE '%SAS'
ORDER BY c.nombre;
```

![Evidencia L02 en MySQL](evidencias/L02_mysql.png)

*Figura 10. Resultado de la consulta L02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L02 · PostgreSQL

```sql
SELECT c.id, c.tipo_documento, c.numero_documento, c.nombre
FROM clientes c
WHERE c.nombre LIKE '%SAS'
ORDER BY c.nombre;
```

![Evidencia L02 en PostgreSQL](evidencias/L02_postgresql.png)

*Figura 11. Resultado de la consulta L02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L02 · SQL Server

```sql
SELECT c.id, c.tipo_documento, c.numero_documento, c.nombre
FROM clientes c
WHERE c.nombre LIKE '%SAS'
ORDER BY c.nombre;
```

![Evidencia L02 en SQL Server](evidencias/L02_sqlserver.png)

*Figura 12. Resultado de la consulta L02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L02 · Oracle

```sql
SELECT c.id, c.tipo_documento, c.numero_documento, c.nombre
FROM clientes c
WHERE c.nombre LIKE '%SAS'
ORDER BY c.nombre;
```

![Evidencia L02 en Oracle](evidencias/L02_oracle.png)

*Figura 13. Resultado de la consulta L02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### L03 — Nombres que terminan en 'ez'

**Situación de negocio.** El área de archivo quiere organizar las carpetas físicas de los clientes por terminación del apellido, empezando por los muy comunes que terminan en «ez» (Gómez, Pérez, Martínez, Rodríguez...).

**Concepto de SQL.** `LIKE '%ez'` filtra por **terminación**. Un `%` al inicio impide usar un índice de forma eficiente, limitación de rendimiento a mencionar en tablas muy grandes. Se aplica `LOWER()` para igualar el comportamiento entre motores.

**Resultado esperado.** El nombre completo de cada fila debe terminar en «ez» (último apellido).

##### L03 · MySQL

```sql
SELECT c.id, c.tipo_documento, c.nombre
FROM clientes c
WHERE LOWER(c.nombre) LIKE '%ez'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia L03 en MySQL](evidencias/L03_mysql.png)

*Figura 14. Resultado de la consulta L03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L03 · PostgreSQL

```sql
SELECT c.id, c.tipo_documento, c.nombre
FROM clientes c
WHERE LOWER(c.nombre) LIKE '%ez'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia L03 en PostgreSQL](evidencias/L03_postgresql.png)

*Figura 15. Resultado de la consulta L03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L03 · SQL Server

```sql
SELECT TOP 20 c.id, c.tipo_documento, c.nombre
FROM clientes c
WHERE LOWER(c.nombre) LIKE '%ez'
ORDER BY c.nombre;
```

![Evidencia L03 en SQL Server](evidencias/L03_sqlserver.png)

*Figura 16. Resultado de la consulta L03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### L03 · Oracle

```sql
SELECT c.id, c.tipo_documento, c.nombre
FROM clientes c
WHERE LOWER(c.nombre) LIKE '%ez'
ORDER BY c.nombre
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia L03 en Oracle](evidencias/L03_oracle.png)

*Figura 17. Resultado de la consulta L03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### L04 — Comodín _ (un solo carácter): nombres cuya segunda letra es 'a'

**Situación de negocio.** Durante la revisión de calidad de datos se pidió una demostración del comodín de un solo carácter: encontrar los nombres cuya segunda letra es «a» (Camila, Mateo, Daniela, Laura, Mariana...).

**Concepto de SQL.** `_` reemplaza **exactamente un carácter**, a diferencia de `%`. El patrón `'_a%'` significa «un carácter cualquiera, luego una a, luego lo que sea».

**Resultado esperado.** La segunda letra de cada nombre devuelto debe ser «a».

##### L04 · MySQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.nombre) LIKE '_a%'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia L04 en MySQL](evidencias/L04_mysql.png)

*Figura 18. Resultado de la consulta L04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L04 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.nombre) LIKE '_a%'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia L04 en PostgreSQL](evidencias/L04_postgresql.png)

*Figura 19. Resultado de la consulta L04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L04 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.nombre) LIKE '_a%'
ORDER BY c.nombre;
```

![Evidencia L04 en SQL Server](evidencias/L04_sqlserver.png)

*Figura 20. Resultado de la consulta L04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### L04 · Oracle

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE LOWER(c.nombre) LIKE '_a%'
ORDER BY c.nombre
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia L04 en Oracle](evidencias/L04_oracle.png)

*Figura 21. Resultado de la consulta L04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### L05 — NOT LIKE: clientes que no usan Gmail ni Hotmail

**Situación de negocio.** Comercial quiere abrir un canal alternativo para los clientes que **no** usan Gmail ni Hotmail, es decir, quienes usan Outlook, Yahoo o correo corporativo. La pregunta fue: «¿a quiénes no llegamos con la campaña masiva de Gmail?».

**Concepto de SQL.** `NOT LIKE` invierte la condición del patrón. Se combinan dos condiciones con `AND`, porque el cliente debe cumplir ambas negaciones a la vez.

**Resultado esperado.** En la columna `email` no debe aparecer ninguna dirección que contenga gmail ni hotmail.

##### L05 · MySQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.email NOT LIKE '%gmail.com' AND c.email NOT LIKE '%hotmail.com'
ORDER BY c.email
LIMIT 20;
```

![Evidencia L05 en MySQL](evidencias/L05_mysql.png)

*Figura 22. Resultado de la consulta L05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L05 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.email NOT LIKE '%gmail.com' AND c.email NOT LIKE '%hotmail.com'
ORDER BY c.email
LIMIT 20;
```

![Evidencia L05 en PostgreSQL](evidencias/L05_postgresql.png)

*Figura 23. Resultado de la consulta L05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L05 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.email
FROM clientes c
WHERE c.email NOT LIKE '%gmail.com' AND c.email NOT LIKE '%hotmail.com'
ORDER BY c.email;
```

![Evidencia L05 en SQL Server](evidencias/L05_sqlserver.png)

*Figura 24. Resultado de la consulta L05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### L05 · Oracle

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.email NOT LIKE '%gmail.com' AND c.email NOT LIKE '%hotmail.com'
ORDER BY c.email
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia L05 en Oracle](evidencias/L05_oracle.png)

*Figura 25. Resultado de la consulta L05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### L06 — Eventos de boda o quinceaños (LIKE + OR)

**Situación de negocio.** Operaciones planea contratar personal adicional de protocolo y necesita ver los eventos sociales de mayor exigencia: bodas y quinceaños.

**Concepto de SQL.** Se combina `LIKE` con `OR` para aceptar cualquiera de los dos tipos de evento.

**Resultado esperado.** Cada fila debe mostrar un tipo que empieza por Boda o por Quince.

##### L06 · MySQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.tipo LIKE 'Boda%' OR e.tipo LIKE 'Quince%'
ORDER BY e.fecha
LIMIT 20;
```

![Evidencia L06 en MySQL](evidencias/L06_mysql.png)

*Figura 26. Resultado de la consulta L06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L06 · PostgreSQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.tipo LIKE 'Boda%' OR e.tipo LIKE 'Quince%'
ORDER BY e.fecha
LIMIT 20;
```

![Evidencia L06 en PostgreSQL](evidencias/L06_postgresql.png)

*Figura 27. Resultado de la consulta L06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### L06 · SQL Server

```sql
SELECT TOP 20 e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.tipo LIKE 'Boda%' OR e.tipo LIKE 'Quince%'
ORDER BY e.fecha;
```

![Evidencia L06 en SQL Server](evidencias/L06_sqlserver.png)

*Figura 28. Resultado de la consulta L06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### L06 · Oracle

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.tipo LIKE 'Boda%' OR e.tipo LIKE 'Quince%'
ORDER BY e.fecha
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia L06 en Oracle](evidencias/L06_oracle.png)

*Figura 29. Resultado de la consulta L06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### L07 — Servicios por patrón de nombre

**Situación de negocio.** El área comercial quiere listar los servicios de ambientación y de sonido del catálogo. No se sabe si el nombre se digitó con tilde, por lo que se pidió un patrón tolerante.

**Concepto de SQL.** Se mezclan los comodines en un mismo patrón: `'%decorac_on%'` tolera cualquier carácter en la posición de la «i» o «ó». Con `LOWER()` se evita depender de la capitalización.

**Resultado esperado.** Se esperan únicamente los servicios Decoracion y Sonido e iluminacion.

##### L07 · MySQL

```sql
SELECT sv.id, sv.nombre, sv.precio_base
FROM servicios sv
WHERE LOWER(sv.nombre) LIKE '%decorac_on%' OR LOWER(sv.nombre) LIKE 'sonido%'
ORDER BY sv.nombre;
```

![Evidencia L07 en MySQL](evidencias/L07_mysql.png)

*Figura 30. Resultado de la consulta L07 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L07 · PostgreSQL

```sql
SELECT sv.id, sv.nombre, sv.precio_base
FROM servicios sv
WHERE LOWER(sv.nombre) LIKE '%decorac_on%' OR LOWER(sv.nombre) LIKE 'sonido%'
ORDER BY sv.nombre;
```

![Evidencia L07 en PostgreSQL](evidencias/L07_postgresql.png)

*Figura 31. Resultado de la consulta L07 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L07 · SQL Server

```sql
SELECT sv.id, sv.nombre, sv.precio_base
FROM servicios sv
WHERE LOWER(sv.nombre) LIKE '%decorac_on%' OR LOWER(sv.nombre) LIKE 'sonido%'
ORDER BY sv.nombre;
```

![Evidencia L07 en SQL Server](evidencias/L07_sqlserver.png)

*Figura 32. Resultado de la consulta L07 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### L07 · Oracle

```sql
SELECT sv.id, sv.nombre, sv.precio_base
FROM servicios sv
WHERE LOWER(sv.nombre) LIKE '%decorac_on%' OR LOWER(sv.nombre) LIKE 'sonido%'
ORDER BY sv.nombre;
```

![Evidencia L07 en Oracle](evidencias/L07_oracle.png)

*Figura 33. Resultado de la consulta L07 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

---

### 2. Subconsultas

Una subconsulta es una consulta dentro de otra. Se presentan seis variantes: escalar, con `IN`, con `NOT IN`, con `EXISTS`, en la cláusula `FROM` (tabla derivada) y correlacionada.

#### S01 — Subconsulta escalar: pagos superiores al promedio

**Situación de negocio.** Cartera quiere identificar los **pagos de alto valor**, entendidos como aquellos que superan el promedio de todos los pagos aplicados. No se puede escribir el promedio a mano porque cambia cada vez que se registran nuevos pagos.

**Concepto de SQL.** Una **subconsulta escalar** (devuelve un solo valor) dentro del `WHERE` calcula el promedio de los pagos en estado `APLICADO`; la consulta externa compara cada pago contra ese valor.

**Resultado esperado.** Todos los montos mostrados deben ser mayores que el promedio. Se comprueba con `SELECT AVG(monto) FROM pagos WHERE estado = 'APLICADO'`.

##### S01 · MySQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
  AND p.monto > (SELECT AVG(p2.monto) FROM pagos p2 WHERE p2.estado = 'APLICADO')
ORDER BY p.monto DESC, p.fecha DESC
LIMIT 15;
```

![Evidencia S01 en MySQL](evidencias/S01_mysql.png)

*Figura 34. Resultado de la consulta S01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S01 · PostgreSQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
  AND p.monto > (SELECT AVG(p2.monto) FROM pagos p2 WHERE p2.estado = 'APLICADO')
ORDER BY p.monto DESC, p.fecha DESC
LIMIT 15;
```

![Evidencia S01 en PostgreSQL](evidencias/S01_postgresql.png)

*Figura 35. Resultado de la consulta S01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S01 · SQL Server

```sql
SELECT TOP 15 p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
  AND p.monto > (SELECT AVG(p2.monto) FROM pagos p2 WHERE p2.estado = 'APLICADO')
ORDER BY p.monto DESC, p.fecha DESC;
```

![Evidencia S01 en SQL Server](evidencias/S01_sqlserver.png)

*Figura 36. Resultado de la consulta S01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### S01 · Oracle

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
  AND p.monto > (SELECT AVG(p2.monto) FROM pagos p2 WHERE p2.estado = 'APLICADO')
ORDER BY p.monto DESC, p.fecha DESC
FETCH FIRST 15 ROWS ONLY;
```

![Evidencia S01 en Oracle](evidencias/S01_oracle.png)

*Figura 37. Resultado de la consulta S01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### S02 — Subconsulta con IN: clientes con reserva confirmada en salones grandes

**Situación de negocio.** Operaciones necesita saber qué clientes tienen una reserva confirmada en salones de capacidad igual o superior a 120 personas, porque a ellos hay que asignarles montaje reforzado y personal adicional.

**Concepto de SQL.** `IN` con subconsultas anidadas: la consulta interna devuelve los salones grandes, la intermedia las reservas confirmadas en ellos y la externa los clientes. Equivale a un `JOIN`, pero deja claro el razonamiento «pertenece a este conjunto».

**Resultado esperado.** Cada cliente devuelto debe tener al menos una reserva `CONFIRMADA` en un salón con capacidad >= 120.

##### S02 · MySQL

```sql
SELECT c.id, c.nombre, CONCAT(c.tipo_documento, ' ', c.numero_documento) AS documento
FROM clientes c
WHERE c.id IN (SELECT r.cliente_id FROM reservas r
               WHERE r.estado = 'CONFIRMADA'
                 AND r.salon_id IN (SELECT s.id FROM salones s WHERE s.capacidad >= 120))
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia S02 en MySQL](evidencias/S02_mysql.png)

*Figura 38. Resultado de la consulta S02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; concatenación con `CONCAT()`.

##### S02 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.tipo_documento || ' ' || c.numero_documento AS documento
FROM clientes c
WHERE c.id IN (SELECT r.cliente_id FROM reservas r
               WHERE r.estado = 'CONFIRMADA'
                 AND r.salon_id IN (SELECT s.id FROM salones s WHERE s.capacidad >= 120))
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia S02 en PostgreSQL](evidencias/S02_postgresql.png)

*Figura 39. Resultado de la consulta S02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; concatenación con `||`.

##### S02 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.tipo_documento + ' ' + c.numero_documento AS documento
FROM clientes c
WHERE c.id IN (SELECT r.cliente_id FROM reservas r
               WHERE r.estado = 'CONFIRMADA'
                 AND r.salon_id IN (SELECT s.id FROM salones s WHERE s.capacidad >= 120))
ORDER BY c.nombre;
```

![Evidencia S02 en SQL Server](evidencias/S02_sqlserver.png)

*Figura 40. Resultado de la consulta S02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; concatenación con `+`.

##### S02 · Oracle

```sql
SELECT c.id, c.nombre, c.tipo_documento || ' ' || c.numero_documento AS documento
FROM clientes c
WHERE c.id IN (SELECT r.cliente_id FROM reservas r
               WHERE r.estado = 'CONFIRMADA'
                 AND r.salon_id IN (SELECT s.id FROM salones s WHERE s.capacidad >= 120))
ORDER BY c.nombre
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia S02 en Oracle](evidencias/S02_oracle.png)

*Figura 41. Resultado de la consulta S02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; concatenación con `||`.

#### S03 — Subconsulta con NOT IN: clientes sin reservas desde julio de 2026

**Situación de negocio.** Comercial quiere contactar a los clientes que no han reservado ningún salón desde el 1 de julio de 2026 para ofrecerles las fechas disponibles de fin de año.

**Concepto de SQL.** `NOT IN` con subconsulta devuelve los clientes cuyo id **no** aparece entre las reservas recientes. La columna de la subconsulta no debe contener `NULL` (aquí `cliente_id` es `NOT NULL`); de lo contrario habría que usar `NOT EXISTS`.

**Resultado esperado.** Ninguno de los clientes listados debe tener reservas con fecha de inicio desde el 2026-07-01.

##### S03 · MySQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.id NOT IN (SELECT r.cliente_id FROM reservas r WHERE r.fecha_inicio >= DATE '2026-07-01')
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia S03 en MySQL](evidencias/S03_mysql.png)

*Figura 42. Resultado de la consulta S03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### S03 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.id NOT IN (SELECT r.cliente_id FROM reservas r WHERE r.fecha_inicio >= DATE '2026-07-01')
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia S03 en PostgreSQL](evidencias/S03_postgresql.png)

*Figura 43. Resultado de la consulta S03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### S03 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.email
FROM clientes c
WHERE c.id NOT IN (SELECT r.cliente_id FROM reservas r WHERE r.fecha_inicio >= '2026-07-01')
ORDER BY c.nombre;
```

![Evidencia S03 en SQL Server](evidencias/S03_sqlserver.png)

*Figura 44. Resultado de la consulta S03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; literal de fecha como cadena `'AAAA-MM-DD'`.

##### S03 · Oracle

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
WHERE c.id NOT IN (SELECT r.cliente_id FROM reservas r WHERE r.fecha_inicio >= DATE '2026-07-01')
ORDER BY c.nombre
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia S03 en Oracle](evidencias/S03_oracle.png)

*Figura 45. Resultado de la consulta S03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

#### S04 — Subconsulta con EXISTS: reservas con servicios pendientes de confirmar

**Situación de negocio.** Operaciones necesita saber qué reservas tienen al menos un servicio contratado en estado pendiente, porque esos eventos no pueden darse por listos hasta confirmar con el proveedor.

**Concepto de SQL.** `EXISTS` con subconsulta correlacionada: para cada reserva se pregunta si existe al menos un servicio pendiente ligado a su evento. Se detiene en cuanto encuentra el primero.

**Resultado esperado.** Cada reserva listada debe tener al menos un registro de `evento_servicios` con estado `PENDIENTE`.

##### S04 · MySQL

```sql
SELECT r.id AS reserva, c.nombre AS cliente, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
WHERE EXISTS (SELECT 1 FROM eventos e
              JOIN evento_servicios es ON es.evento_id = e.id
              WHERE e.reserva_id = r.id AND es.estado = 'PENDIENTE')
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia S04 en MySQL](evidencias/S04_mysql.png)

*Figura 46. Resultado de la consulta S04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S04 · PostgreSQL

```sql
SELECT r.id AS reserva, c.nombre AS cliente, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
WHERE EXISTS (SELECT 1 FROM eventos e
              JOIN evento_servicios es ON es.evento_id = e.id
              WHERE e.reserva_id = r.id AND es.estado = 'PENDIENTE')
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia S04 en PostgreSQL](evidencias/S04_postgresql.png)

*Figura 47. Resultado de la consulta S04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S04 · SQL Server

```sql
SELECT TOP 20 r.id AS reserva, c.nombre AS cliente, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
WHERE EXISTS (SELECT 1 FROM eventos e
              JOIN evento_servicios es ON es.evento_id = e.id
              WHERE e.reserva_id = r.id AND es.estado = 'PENDIENTE')
ORDER BY r.fecha_inicio;
```

![Evidencia S04 en SQL Server](evidencias/S04_sqlserver.png)

*Figura 48. Resultado de la consulta S04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### S04 · Oracle

```sql
SELECT r.id AS reserva, c.nombre AS cliente, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
WHERE EXISTS (SELECT 1 FROM eventos e
              JOIN evento_servicios es ON es.evento_id = e.id
              WHERE e.reserva_id = r.id AND es.estado = 'PENDIENTE')
ORDER BY r.fecha_inicio
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia S04 en Oracle](evidencias/S04_oracle.png)

*Figura 49. Resultado de la consulta S04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### S05 — Subconsulta en FROM (tabla derivada): proveedores sobre el promedio de servicios

**Situación de negocio.** Operaciones quiere reconocer a los proveedores que prestan más servicios que el promedio. El promedio no es un dato fijo: depende de cuántos servicios se han contratado y de cuántos proveedores participan.

**Concepto de SQL.** Combina una **tabla derivada** en el `FROM` (servicios por proveedor) con una subconsulta escalar que calcula el promedio de esos conteos. Una subconsulta puede vivir en `FROM` y en `WHERE` dentro de la misma sentencia.

**Resultado esperado.** Solo aparecen proveedores con un número de servicios estrictamente mayor al promedio.

##### S05 · MySQL

```sql
SELECT t.proveedor, t.servicios
FROM (SELECT p.id, p.razon_social AS proveedor, COUNT(es.id) AS servicios
      FROM proveedores p
      JOIN evento_servicios es ON es.proveedor_id = p.id
      WHERE es.estado <> 'CANCELADO'
      GROUP BY p.id, p.razon_social) t
WHERE t.servicios > (SELECT AVG(x.n * 1.0)
                     FROM (SELECT COUNT(*) AS n FROM evento_servicios es2
                           WHERE es2.estado <> 'CANCELADO' GROUP BY es2.proveedor_id) x)
ORDER BY t.servicios DESC;
```

![Evidencia S05 en MySQL](evidencias/S05_mysql.png)

*Figura 50. Resultado de la consulta S05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### S05 · PostgreSQL

```sql
SELECT t.proveedor, t.servicios
FROM (SELECT p.id, p.razon_social AS proveedor, COUNT(es.id) AS servicios
      FROM proveedores p
      JOIN evento_servicios es ON es.proveedor_id = p.id
      WHERE es.estado <> 'CANCELADO'
      GROUP BY p.id, p.razon_social) t
WHERE t.servicios > (SELECT AVG(x.n * 1.0)
                     FROM (SELECT COUNT(*) AS n FROM evento_servicios es2
                           WHERE es2.estado <> 'CANCELADO' GROUP BY es2.proveedor_id) x)
ORDER BY t.servicios DESC;
```

![Evidencia S05 en PostgreSQL](evidencias/S05_postgresql.png)

*Figura 51. Resultado de la consulta S05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### S05 · SQL Server

```sql
SELECT t.proveedor, t.servicios
FROM (SELECT p.id, p.razon_social AS proveedor, COUNT(es.id) AS servicios
      FROM proveedores p
      JOIN evento_servicios es ON es.proveedor_id = p.id
      WHERE es.estado <> 'CANCELADO'
      GROUP BY p.id, p.razon_social) t
WHERE t.servicios > (SELECT AVG(x.n * 1.0)
                     FROM (SELECT COUNT(*) AS n FROM evento_servicios es2
                           WHERE es2.estado <> 'CANCELADO' GROUP BY es2.proveedor_id) x)
ORDER BY t.servicios DESC;
```

![Evidencia S05 en SQL Server](evidencias/S05_sqlserver.png)

*Figura 52. Resultado de la consulta S05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### S05 · Oracle

```sql
SELECT t.proveedor, t.servicios
FROM (SELECT p.id, p.razon_social AS proveedor, COUNT(es.id) AS servicios
      FROM proveedores p
      JOIN evento_servicios es ON es.proveedor_id = p.id
      WHERE es.estado <> 'CANCELADO'
      GROUP BY p.id, p.razon_social) t
WHERE t.servicios > (SELECT AVG(x.n * 1.0)
                     FROM (SELECT COUNT(*) AS n FROM evento_servicios es2
                           WHERE es2.estado <> 'CANCELADO' GROUP BY es2.proveedor_id) x)
ORDER BY t.servicios DESC;
```

![Evidencia S05 en Oracle](evidencias/S05_oracle.png)

*Figura 53. Resultado de la consulta S05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### S06 — Subconsulta correlacionada: último pago registrado de cada contrato

**Situación de negocio.** Cartera quiere hacer seguimiento de cobro: para cada contrato necesita ver únicamente su pago más reciente, no todo el historial, y con eso decidir a quién llamar primero.

**Concepto de SQL.** **Subconsulta correlacionada**: por cada fila del `pagos` externo se evalúa `MAX(fecha)` de los pagos de ese mismo contrato. La referencia `p.contrato_id` conecta la consulta interna con la externa.

**Resultado esperado.** Cada `contrato_id` aparece con el pago de su fecha más reciente. Si un contrato tuvo dos pagos el mismo día, podrían salir ambos.

##### S06 · MySQL

```sql
SELECT p.contrato_id, p.id AS pago, p.tipo_pago, p.fecha, p.monto, p.estado
FROM pagos p
WHERE p.fecha = (SELECT MAX(p2.fecha) FROM pagos p2 WHERE p2.contrato_id = p.contrato_id)
ORDER BY p.fecha DESC, p.contrato_id
LIMIT 15;
```

![Evidencia S06 en MySQL](evidencias/S06_mysql.png)

*Figura 54. Resultado de la consulta S06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S06 · PostgreSQL

```sql
SELECT p.contrato_id, p.id AS pago, p.tipo_pago, p.fecha, p.monto, p.estado
FROM pagos p
WHERE p.fecha = (SELECT MAX(p2.fecha) FROM pagos p2 WHERE p2.contrato_id = p.contrato_id)
ORDER BY p.fecha DESC, p.contrato_id
LIMIT 15;
```

![Evidencia S06 en PostgreSQL](evidencias/S06_postgresql.png)

*Figura 55. Resultado de la consulta S06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### S06 · SQL Server

```sql
SELECT TOP 15 p.contrato_id, p.id AS pago, p.tipo_pago, p.fecha, p.monto, p.estado
FROM pagos p
WHERE p.fecha = (SELECT MAX(p2.fecha) FROM pagos p2 WHERE p2.contrato_id = p.contrato_id)
ORDER BY p.fecha DESC, p.contrato_id;
```

![Evidencia S06 en SQL Server](evidencias/S06_sqlserver.png)

*Figura 56. Resultado de la consulta S06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### S06 · Oracle

```sql
SELECT p.contrato_id, p.id AS pago, p.tipo_pago, p.fecha, p.monto, p.estado
FROM pagos p
WHERE p.fecha = (SELECT MAX(p2.fecha) FROM pagos p2 WHERE p2.contrato_id = p.contrato_id)
ORDER BY p.fecha DESC, p.contrato_id
FETCH FIRST 15 ROWS ONLY;
```

![Evidencia S06 en Oracle](evidencias/S06_oracle.png)

*Figura 57. Resultado de la consulta S06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

---

### 3. GROUP BY — agrupación y agregación

`GROUP BY` agrupa las filas que comparten valores y permite calcular totales con `COUNT`, `SUM`, `AVG`, `MIN` y `MAX`. Cada fila del resultado representa un grupo.

#### G01 — Reservas por salón y estado

**Situación de negocio.** La gerencia quiere saber cuántas reservas tiene cada salón según su estado (pre-reserva, confirmada, cancelada o finalizada) para decidir qué espacios promocionar y cuáles revisar por cancelaciones.

**Concepto de SQL.** `GROUP BY` con dos columnas (salón y estado) y `COUNT()` como función de agregación. Cada combinación única genera una fila.

**Resultado esperado.** La suma de la columna de conteo debe ser 120, el total de reservas cargadas.

##### G01 · MySQL

```sql
SELECT s.nombre AS salon, r.estado, COUNT(r.id) AS total
FROM reservas r
JOIN salones s ON s.id = r.salon_id
GROUP BY s.id, s.nombre, r.estado
ORDER BY s.nombre, r.estado;
```

![Evidencia G01 en MySQL](evidencias/G01_mysql.png)

*Figura 58. Resultado de la consulta G01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G01 · PostgreSQL

```sql
SELECT s.nombre AS salon, r.estado, COUNT(r.id) AS total
FROM reservas r
JOIN salones s ON s.id = r.salon_id
GROUP BY s.id, s.nombre, r.estado
ORDER BY s.nombre, r.estado;
```

![Evidencia G01 en PostgreSQL](evidencias/G01_postgresql.png)

*Figura 59. Resultado de la consulta G01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G01 · SQL Server

```sql
SELECT s.nombre AS salon, r.estado, COUNT(r.id) AS total
FROM reservas r
JOIN salones s ON s.id = r.salon_id
GROUP BY s.id, s.nombre, r.estado
ORDER BY s.nombre, r.estado;
```

![Evidencia G01 en SQL Server](evidencias/G01_sqlserver.png)

*Figura 60. Resultado de la consulta G01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G01 · Oracle

```sql
SELECT s.nombre AS salon, r.estado, COUNT(r.id) AS total
FROM reservas r
JOIN salones s ON s.id = r.salon_id
GROUP BY s.id, s.nombre, r.estado
ORDER BY s.nombre, r.estado;
```

![Evidencia G01 en Oracle](evidencias/G01_oracle.png)

*Figura 61. Resultado de la consulta G01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### G02 — Recaudo por método de pago

**Situación de negocio.** Cartera y Tesorería necesitan conocer cuánto dinero ingresa por cada medio de pago (efectivo, tarjeta y transferencia) para negociar comisiones con el banco y decidir si conviene incentivar alguno.

**Concepto de SQL.** Varias funciones de agregación en una sola consulta: `SUM`, `COUNT`, `AVG`, `MIN` y `MAX` agrupadas por método. Solo se consideran los pagos en estado `APLICADO`.

**Resultado esperado.** Una fila por método de pago. El mínimo nunca puede ser mayor que el promedio ni el promedio mayor que el máximo.

##### G02 · MySQL

```sql
SELECT p.metodo, SUM(p.monto) AS total, COUNT(p.id) AS cantidad, AVG(p.monto) AS promedio,
       MIN(p.monto) AS minimo, MAX(p.monto) AS maximo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY p.metodo
ORDER BY total DESC;
```

![Evidencia G02 en MySQL](evidencias/G02_mysql.png)

*Figura 62. Resultado de la consulta G02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G02 · PostgreSQL

```sql
SELECT p.metodo, SUM(p.monto) AS total, COUNT(p.id) AS cantidad, AVG(p.monto) AS promedio,
       MIN(p.monto) AS minimo, MAX(p.monto) AS maximo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY p.metodo
ORDER BY total DESC;
```

![Evidencia G02 en PostgreSQL](evidencias/G02_postgresql.png)

*Figura 63. Resultado de la consulta G02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G02 · SQL Server

```sql
SELECT p.metodo, SUM(p.monto) AS total, COUNT(p.id) AS cantidad, AVG(p.monto) AS promedio,
       MIN(p.monto) AS minimo, MAX(p.monto) AS maximo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY p.metodo
ORDER BY total DESC;
```

![Evidencia G02 en SQL Server](evidencias/G02_sqlserver.png)

*Figura 64. Resultado de la consulta G02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G02 · Oracle

```sql
SELECT p.metodo, SUM(p.monto) AS total, COUNT(p.id) AS cantidad, AVG(p.monto) AS promedio,
       MIN(p.monto) AS minimo, MAX(p.monto) AS maximo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY p.metodo
ORDER BY total DESC;
```

![Evidencia G02 en Oracle](evidencias/G02_oracle.png)

*Figura 65. Resultado de la consulta G02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### G03 — Recaudo mensual

**Situación de negocio.** La gerencia financiera quiere proyectar el flujo de caja y para eso necesita los ingresos mes a mes. El informe se presenta ante un comité que revisa si hay meses de baja recaudación.

**Concepto de SQL.** Se agrupa por el **año y mes** de la fecha (formato `AAAA-MM`). La función cambia entre motores (`DATE_FORMAT` en MySQL, `TO_CHAR` en PostgreSQL y Oracle, `CONVERT(CHAR(7), fecha, 120)` en SQL Server), lo que hace de esta consulta un buen ejemplo de diferencias de dialecto.

**Resultado esperado.** Una fila por mes con pagos aplicados, ordenada cronológicamente.

##### G03 · MySQL

```sql
SELECT DATE_FORMAT(p.fecha, '%Y-%m') AS mes, COUNT(p.id) AS pagos, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY DATE_FORMAT(p.fecha, '%Y-%m')
ORDER BY mes;
```

![Evidencia G03 en MySQL](evidencias/G03_mysql.png)

*Figura 66. Resultado de la consulta G03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `DATE_FORMAT()` para dar formato a fechas.

##### G03 · PostgreSQL

```sql
SELECT TO_CHAR(p.fecha, 'YYYY-MM') AS mes, COUNT(p.id) AS pagos, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY TO_CHAR(p.fecha, 'YYYY-MM')
ORDER BY mes;
```

![Evidencia G03 en PostgreSQL](evidencias/G03_postgresql.png)

*Figura 67. Resultado de la consulta G03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `TO_CHAR()` para dar formato a fechas.

##### G03 · SQL Server

```sql
SELECT CONVERT(CHAR(7), p.fecha, 120) AS mes, COUNT(p.id) AS pagos, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY CONVERT(CHAR(7), p.fecha, 120)
ORDER BY mes;
```

![Evidencia G03 en SQL Server](evidencias/G03_sqlserver.png)

*Figura 68. Resultado de la consulta G03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `CONVERT()` para dar formato a fechas.

##### G03 · Oracle

```sql
SELECT TO_CHAR(p.fecha, 'YYYY-MM') AS mes, COUNT(p.id) AS pagos, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY TO_CHAR(p.fecha, 'YYYY-MM')
ORDER BY mes;
```

![Evidencia G03 en Oracle](evidencias/G03_oracle.png)

*Figura 69. Resultado de la consulta G03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `TO_CHAR()` para dar formato a fechas.

#### G04 — Eventos por tipo y estado

**Situación de negocio.** Operaciones necesita ver la carga de trabajo por tipo de evento: cuántos se han realizado, cuántos están planeados y cuántos se cancelaron, junto con el total de asistentes esperados, para planificar personal.

**Concepto de SQL.** `GROUP BY` sobre tipo y estado del evento con `COUNT()` y `SUM()`.

**Resultado esperado.** Un mismo tipo puede aparecer en varias filas (una por estado). La suma de eventos debe coincidir con el total de eventos cargados.

##### G04 · MySQL

```sql
SELECT e.tipo, e.estado, COUNT(e.id) AS eventos, SUM(e.cantidad) AS asistentes
FROM eventos e
GROUP BY e.tipo, e.estado
ORDER BY e.tipo, e.estado;
```

![Evidencia G04 en MySQL](evidencias/G04_mysql.png)

*Figura 70. Resultado de la consulta G04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G04 · PostgreSQL

```sql
SELECT e.tipo, e.estado, COUNT(e.id) AS eventos, SUM(e.cantidad) AS asistentes
FROM eventos e
GROUP BY e.tipo, e.estado
ORDER BY e.tipo, e.estado;
```

![Evidencia G04 en PostgreSQL](evidencias/G04_postgresql.png)

*Figura 71. Resultado de la consulta G04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G04 · SQL Server

```sql
SELECT e.tipo, e.estado, COUNT(e.id) AS eventos, SUM(e.cantidad) AS asistentes
FROM eventos e
GROUP BY e.tipo, e.estado
ORDER BY e.tipo, e.estado;
```

![Evidencia G04 en SQL Server](evidencias/G04_sqlserver.png)

*Figura 72. Resultado de la consulta G04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G04 · Oracle

```sql
SELECT e.tipo, e.estado, COUNT(e.id) AS eventos, SUM(e.cantidad) AS asistentes
FROM eventos e
GROUP BY e.tipo, e.estado
ORDER BY e.tipo, e.estado;
```

![Evidencia G04 en Oracle](evidencias/G04_oracle.png)

*Figura 73. Resultado de la consulta G04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### G05 — Clientes por tipo de documento y actividad

**Situación de negocio.** Comercial quiere medir cuántos clientes son personas naturales (CC) y cuántos empresas (NIT), separando activos de inactivos, porque los inactivos pueden depurarse del sistema.

**Concepto de SQL.** `GROUP BY` con dos columnas sobre una sola tabla, sin `JOIN`. Sirve como tabla de contingencia sencilla.

**Resultado esperado.** La suma de la columna de conteo debe ser 60, el total de clientes.

##### G05 · MySQL

```sql
SELECT c.tipo_documento, c.is_active, COUNT(*) AS clientes
FROM clientes c
GROUP BY c.tipo_documento, c.is_active
ORDER BY c.tipo_documento, c.is_active;
```

![Evidencia G05 en MySQL](evidencias/G05_mysql.png)

*Figura 74. Resultado de la consulta G05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G05 · PostgreSQL

```sql
SELECT c.tipo_documento, c.is_active, COUNT(*) AS clientes
FROM clientes c
GROUP BY c.tipo_documento, c.is_active
ORDER BY c.tipo_documento, c.is_active;
```

![Evidencia G05 en PostgreSQL](evidencias/G05_postgresql.png)

*Figura 75. Resultado de la consulta G05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G05 · SQL Server

```sql
SELECT c.tipo_documento, c.is_active, COUNT(*) AS clientes
FROM clientes c
GROUP BY c.tipo_documento, c.is_active
ORDER BY c.tipo_documento, c.is_active;
```

![Evidencia G05 en SQL Server](evidencias/G05_sqlserver.png)

*Figura 76. Resultado de la consulta G05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G05 · Oracle

```sql
SELECT c.tipo_documento, c.is_active, COUNT(*) AS clientes
FROM clientes c
GROUP BY c.tipo_documento, c.is_active
ORDER BY c.tipo_documento, c.is_active;
```

![Evidencia G05 en Oracle](evidencias/G05_oracle.png)

*Figura 77. Resultado de la consulta G05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### G06 — Servicios más contratados y valor facturado

**Situación de negocio.** La gerencia desea conocer cuántas veces se ha contratado cada servicio y cuánto dinero representa, para negociar mejores tarifas con los proveedores de los servicios más vendidos.

**Concepto de SQL.** `GROUP BY` sobre el servicio con `COUNT()` y `SUM()` de una expresión (`cantidad * precio_unitario`), uniendo `evento_servicios` con `servicios`.

**Resultado esperado.** Una fila por servicio, ordenadas del mayor al menor valor. Los servicios cancelados no se cuentan.

##### G06 · MySQL

```sql
SELECT sv.nombre AS servicio, COUNT(es.id) AS contrataciones, SUM(es.cantidad * es.precio_unitario) AS valor_total
FROM evento_servicios es
JOIN servicios sv ON sv.id = es.servicio_id
WHERE es.estado <> 'CANCELADO'
GROUP BY sv.id, sv.nombre
ORDER BY valor_total DESC, sv.nombre;
```

![Evidencia G06 en MySQL](evidencias/G06_mysql.png)

*Figura 78. Resultado de la consulta G06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G06 · PostgreSQL

```sql
SELECT sv.nombre AS servicio, COUNT(es.id) AS contrataciones, SUM(es.cantidad * es.precio_unitario) AS valor_total
FROM evento_servicios es
JOIN servicios sv ON sv.id = es.servicio_id
WHERE es.estado <> 'CANCELADO'
GROUP BY sv.id, sv.nombre
ORDER BY valor_total DESC, sv.nombre;
```

![Evidencia G06 en PostgreSQL](evidencias/G06_postgresql.png)

*Figura 79. Resultado de la consulta G06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G06 · SQL Server

```sql
SELECT sv.nombre AS servicio, COUNT(es.id) AS contrataciones, SUM(es.cantidad * es.precio_unitario) AS valor_total
FROM evento_servicios es
JOIN servicios sv ON sv.id = es.servicio_id
WHERE es.estado <> 'CANCELADO'
GROUP BY sv.id, sv.nombre
ORDER BY valor_total DESC, sv.nombre;
```

![Evidencia G06 en SQL Server](evidencias/G06_sqlserver.png)

*Figura 80. Resultado de la consulta G06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### G06 · Oracle

```sql
SELECT sv.nombre AS servicio, COUNT(es.id) AS contrataciones, SUM(es.cantidad * es.precio_unitario) AS valor_total
FROM evento_servicios es
JOIN servicios sv ON sv.id = es.servicio_id
WHERE es.estado <> 'CANCELADO'
GROUP BY sv.id, sv.nombre
ORDER BY valor_total DESC, sv.nombre;
```

![Evidencia G06 en Oracle](evidencias/G06_oracle.png)

*Figura 81. Resultado de la consulta G06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

---

### 4. HAVING — filtro sobre grupos

`HAVING` es el `WHERE` de los grupos: filtra después de agrupar, y por eso puede usar funciones de agregación.

#### H01 — Salones con más de 24 reservas

**Situación de negocio.** La gerencia quiere identificar los salones más demandados para priorizar su mantenimiento y ampliar su disponibilidad, pero solo le interesan los que superan un umbral de reservas, no todo el catálogo.

**Concepto de SQL.** `HAVING` filtra **después** de agrupar, a diferencia de `WHERE`, que filtra antes. Aquí se conservan solo los grupos cuyo `COUNT` supera el umbral; un `WHERE` no puede usar funciones de agregación.

**Resultado esperado.** Todos los salones mostrados deben tener un conteo mayor a 24. Si se quita el `HAVING` aparecen también los demás.

##### H01 · MySQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
GROUP BY s.id, s.nombre
HAVING COUNT(r.id) > 24
ORDER BY reservas DESC;
```

![Evidencia H01 en MySQL](evidencias/H01_mysql.png)

*Figura 82. Resultado de la consulta H01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H01 · PostgreSQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
GROUP BY s.id, s.nombre
HAVING COUNT(r.id) > 24
ORDER BY reservas DESC;
```

![Evidencia H01 en PostgreSQL](evidencias/H01_postgresql.png)

*Figura 83. Resultado de la consulta H01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H01 · SQL Server

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
GROUP BY s.id, s.nombre
HAVING COUNT(r.id) > 24
ORDER BY reservas DESC;
```

![Evidencia H01 en SQL Server](evidencias/H01_sqlserver.png)

*Figura 84. Resultado de la consulta H01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H01 · Oracle

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
GROUP BY s.id, s.nombre
HAVING COUNT(r.id) > 24
ORDER BY reservas DESC;
```

![Evidencia H01 en Oracle](evidencias/H01_oracle.png)

*Figura 85. Resultado de la consulta H01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### H02 — Clientes con 3 o más reservas

**Situación de negocio.** Fidelización quiere premiar con un descuento a los clientes que han reservado varias veces, y necesita la lista de quienes tienen tres o más reservas.

**Concepto de SQL.** `GROUP BY` por cliente y `HAVING COUNT(*) >= 3`. Es la forma estándar de encontrar «recurrentes» en una tabla relacionada.

**Resultado esperado.** Cada cliente mostrado debe tener un conteo de 3 o más. Ordenar por conteo descendente muestra primero a los más fieles.

##### H02 · MySQL

```sql
SELECT c.nombre AS cliente, COUNT(r.id) AS reservas
FROM clientes c
JOIN reservas r ON r.cliente_id = c.id
GROUP BY c.id, c.nombre
HAVING COUNT(r.id) >= 3
ORDER BY reservas DESC, cliente
LIMIT 20;
```

![Evidencia H02 en MySQL](evidencias/H02_mysql.png)

*Figura 86. Resultado de la consulta H02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### H02 · PostgreSQL

```sql
SELECT c.nombre AS cliente, COUNT(r.id) AS reservas
FROM clientes c
JOIN reservas r ON r.cliente_id = c.id
GROUP BY c.id, c.nombre
HAVING COUNT(r.id) >= 3
ORDER BY reservas DESC, cliente
LIMIT 20;
```

![Evidencia H02 en PostgreSQL](evidencias/H02_postgresql.png)

*Figura 87. Resultado de la consulta H02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### H02 · SQL Server

```sql
SELECT TOP 20 c.nombre AS cliente, COUNT(r.id) AS reservas
FROM clientes c
JOIN reservas r ON r.cliente_id = c.id
GROUP BY c.id, c.nombre
HAVING COUNT(r.id) >= 3
ORDER BY reservas DESC, cliente;
```

![Evidencia H02 en SQL Server](evidencias/H02_sqlserver.png)

*Figura 88. Resultado de la consulta H02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### H02 · Oracle

```sql
SELECT c.nombre AS cliente, COUNT(r.id) AS reservas
FROM clientes c
JOIN reservas r ON r.cliente_id = c.id
GROUP BY c.id, c.nombre
HAVING COUNT(r.id) >= 3
ORDER BY reservas DESC, cliente
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia H02 en Oracle](evidencias/H02_oracle.png)

*Figura 89. Resultado de la consulta H02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### H03 — Proveedores con al menos 32 servicios confirmados

**Situación de negocio.** Operaciones quiere un cuadro de honor con los proveedores que más servicios han confirmado, para reconocerlos y priorizarlos en nuevos eventos.

**Concepto de SQL.** `WHERE` y `HAVING` conviven: `WHERE` deja solo los servicios confirmados (filtro por fila) y `HAVING` conserva los proveedores con el mínimo exigido (filtro por grupo).

**Resultado esperado.** Los conteos mostrados corresponden solo a servicios `CONFIRMADO`; los pendientes y cancelados no suman.

##### H03 · MySQL

```sql
SELECT p.razon_social AS proveedor, COUNT(es.id) AS confirmados
FROM proveedores p
JOIN evento_servicios es ON es.proveedor_id = p.id
WHERE es.estado = 'CONFIRMADO'
GROUP BY p.id, p.razon_social
HAVING COUNT(es.id) >= 32
ORDER BY confirmados DESC, proveedor;
```

![Evidencia H03 en MySQL](evidencias/H03_mysql.png)

*Figura 90. Resultado de la consulta H03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H03 · PostgreSQL

```sql
SELECT p.razon_social AS proveedor, COUNT(es.id) AS confirmados
FROM proveedores p
JOIN evento_servicios es ON es.proveedor_id = p.id
WHERE es.estado = 'CONFIRMADO'
GROUP BY p.id, p.razon_social
HAVING COUNT(es.id) >= 32
ORDER BY confirmados DESC, proveedor;
```

![Evidencia H03 en PostgreSQL](evidencias/H03_postgresql.png)

*Figura 91. Resultado de la consulta H03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H03 · SQL Server

```sql
SELECT p.razon_social AS proveedor, COUNT(es.id) AS confirmados
FROM proveedores p
JOIN evento_servicios es ON es.proveedor_id = p.id
WHERE es.estado = 'CONFIRMADO'
GROUP BY p.id, p.razon_social
HAVING COUNT(es.id) >= 32
ORDER BY confirmados DESC, proveedor;
```

![Evidencia H03 en SQL Server](evidencias/H03_sqlserver.png)

*Figura 92. Resultado de la consulta H03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H03 · Oracle

```sql
SELECT p.razon_social AS proveedor, COUNT(es.id) AS confirmados
FROM proveedores p
JOIN evento_servicios es ON es.proveedor_id = p.id
WHERE es.estado = 'CONFIRMADO'
GROUP BY p.id, p.razon_social
HAVING COUNT(es.id) >= 32
ORDER BY confirmados DESC, proveedor;
```

![Evidencia H03 en Oracle](evidencias/H03_oracle.png)

*Figura 93. Resultado de la consulta H03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### H04 — Contratos con pagos pendientes superiores a 1.000.000

**Situación de negocio.** Cartera necesita identificar los contratos que presentan pagos en estado pendiente y el monto adeudado, con el fin de priorizar las gestiones de cobro.

**Concepto de SQL.** `HAVING` aplicado sobre una función de agregación (`SUM(p.monto)`) tras un `JOIN` de tres tablas. El `WHERE` filtra primero los pagos pendientes (filas) y el `HAVING` filtra después los grupos cuyo monto supera el umbral.

**Resultado esperado.** Cada fila corresponde a un contrato cuyo monto pendiente acumulado es superior a $1.000.000, ordenados de mayor a menor deuda.

##### H04 · MySQL

```sql
SELECT ct.numero AS contrato, c.nombre AS cliente,
       COUNT(p.id) AS pagos_pendientes, SUM(p.monto) AS monto_pendiente
FROM contratos ct
JOIN clientes c ON c.id = ct.cliente_id
JOIN pagos p ON p.contrato_id = ct.id
WHERE p.estado = 'PENDIENTE'
GROUP BY ct.id, ct.numero, c.nombre
HAVING SUM(p.monto) > 1000000
ORDER BY monto_pendiente DESC, ct.numero
LIMIT 20;
```

![Evidencia H04 en MySQL](evidencias/H04_mysql.png)

*Figura 94. Resultado de la consulta H04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### H04 · PostgreSQL

```sql
SELECT ct.numero AS contrato, c.nombre AS cliente,
       COUNT(p.id) AS pagos_pendientes, SUM(p.monto) AS monto_pendiente
FROM contratos ct
JOIN clientes c ON c.id = ct.cliente_id
JOIN pagos p ON p.contrato_id = ct.id
WHERE p.estado = 'PENDIENTE'
GROUP BY ct.id, ct.numero, c.nombre
HAVING SUM(p.monto) > 1000000
ORDER BY monto_pendiente DESC, ct.numero
LIMIT 20;
```

![Evidencia H04 en PostgreSQL](evidencias/H04_postgresql.png)

*Figura 95. Resultado de la consulta H04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### H04 · SQL Server

```sql
SELECT TOP 20 ct.numero AS contrato, c.nombre AS cliente,
       COUNT(p.id) AS pagos_pendientes, SUM(p.monto) AS monto_pendiente
FROM contratos ct
JOIN clientes c ON c.id = ct.cliente_id
JOIN pagos p ON p.contrato_id = ct.id
WHERE p.estado = 'PENDIENTE'
GROUP BY ct.id, ct.numero, c.nombre
HAVING SUM(p.monto) > 1000000
ORDER BY monto_pendiente DESC, ct.numero;
```

![Evidencia H04 en SQL Server](evidencias/H04_sqlserver.png)

*Figura 96. Resultado de la consulta H04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### H04 · Oracle

```sql
SELECT ct.numero AS contrato, c.nombre AS cliente,
       COUNT(p.id) AS pagos_pendientes, SUM(p.monto) AS monto_pendiente
FROM contratos ct
JOIN clientes c ON c.id = ct.cliente_id
JOIN pagos p ON p.contrato_id = ct.id
WHERE p.estado = 'PENDIENTE'
GROUP BY ct.id, ct.numero, c.nombre
HAVING SUM(p.monto) > 1000000
ORDER BY monto_pendiente DESC, ct.numero
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia H04 en Oracle](evidencias/H04_oracle.png)

*Figura 97. Resultado de la consulta H04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### H05 — Eventos con 3 o más servicios contratados

**Situación de negocio.** Operaciones desea identificar los eventos más complejos, es decir, aquellos con tres o más servicios contratados, para asignarles un coordinador de tiempo completo.

**Concepto de SQL.** `HAVING COUNT(...) >= 3` filtra los grupos (eventos) después de agrupar; un `WHERE` no podría usar la función de agregación.

**Resultado esperado.** Todos los eventos listados tienen tres o más servicios, ordenados de mayor a menor.

##### H05 · MySQL

```sql
SELECT e.id AS evento, e.tipo, e.fecha, COUNT(es.id) AS servicios
FROM eventos e
JOIN evento_servicios es ON es.evento_id = e.id
GROUP BY e.id, e.tipo, e.fecha
HAVING COUNT(es.id) >= 3
ORDER BY servicios DESC, e.fecha;
```

![Evidencia H05 en MySQL](evidencias/H05_mysql.png)

*Figura 98. Resultado de la consulta H05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H05 · PostgreSQL

```sql
SELECT e.id AS evento, e.tipo, e.fecha, COUNT(es.id) AS servicios
FROM eventos e
JOIN evento_servicios es ON es.evento_id = e.id
GROUP BY e.id, e.tipo, e.fecha
HAVING COUNT(es.id) >= 3
ORDER BY servicios DESC, e.fecha;
```

![Evidencia H05 en PostgreSQL](evidencias/H05_postgresql.png)

*Figura 99. Resultado de la consulta H05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H05 · SQL Server

```sql
SELECT e.id AS evento, e.tipo, e.fecha, COUNT(es.id) AS servicios
FROM eventos e
JOIN evento_servicios es ON es.evento_id = e.id
GROUP BY e.id, e.tipo, e.fecha
HAVING COUNT(es.id) >= 3
ORDER BY servicios DESC, e.fecha;
```

![Evidencia H05 en SQL Server](evidencias/H05_sqlserver.png)

*Figura 100. Resultado de la consulta H05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### H05 · Oracle

```sql
SELECT e.id AS evento, e.tipo, e.fecha, COUNT(es.id) AS servicios
FROM eventos e
JOIN evento_servicios es ON es.evento_id = e.id
GROUP BY e.id, e.tipo, e.fecha
HAVING COUNT(es.id) >= 3
ORDER BY servicios DESC, e.fecha;
```

![Evidencia H05 en Oracle](evidencias/H05_oracle.png)

*Figura 101. Resultado de la consulta H05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### H06 — Meses con recaudo superior a 75.000.000

**Situación de negocio.** La gerencia financiera quiere resaltar los meses de mayor ingreso, para cruzarlos con las campañas comerciales realizadas en esas fechas.

**Concepto de SQL.** Se agrupa por año-mes y se filtra con `HAVING SUM(monto) > 75000000`. Es la versión mensual del filtro por umbral de la consulta H01.

**Resultado esperado.** Solo aparecen los meses con recaudo superior al umbral, ordenados del mayor al menor.

##### H06 · MySQL

```sql
SELECT DATE_FORMAT(p.fecha, '%Y-%m') AS mes, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY DATE_FORMAT(p.fecha, '%Y-%m')
HAVING SUM(p.monto) > 75000000
ORDER BY recaudo DESC;
```

![Evidencia H06 en MySQL](evidencias/H06_mysql.png)

*Figura 102. Resultado de la consulta H06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `DATE_FORMAT()` para dar formato a fechas.

##### H06 · PostgreSQL

```sql
SELECT TO_CHAR(p.fecha, 'YYYY-MM') AS mes, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY TO_CHAR(p.fecha, 'YYYY-MM')
HAVING SUM(p.monto) > 75000000
ORDER BY recaudo DESC;
```

![Evidencia H06 en PostgreSQL](evidencias/H06_postgresql.png)

*Figura 103. Resultado de la consulta H06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `TO_CHAR()` para dar formato a fechas.

##### H06 · SQL Server

```sql
SELECT CONVERT(CHAR(7), p.fecha, 120) AS mes, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY CONVERT(CHAR(7), p.fecha, 120)
HAVING SUM(p.monto) > 75000000
ORDER BY recaudo DESC;
```

![Evidencia H06 en SQL Server](evidencias/H06_sqlserver.png)

*Figura 104. Resultado de la consulta H06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `CONVERT()` para dar formato a fechas.

##### H06 · Oracle

```sql
SELECT TO_CHAR(p.fecha, 'YYYY-MM') AS mes, SUM(p.monto) AS recaudo
FROM pagos p
WHERE p.estado = 'APLICADO'
GROUP BY TO_CHAR(p.fecha, 'YYYY-MM')
HAVING SUM(p.monto) > 75000000
ORDER BY recaudo DESC;
```

![Evidencia H06 en Oracle](evidencias/H06_oracle.png)

*Figura 105. Resultado de la consulta H06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `TO_CHAR()` para dar formato a fechas.

---

### 5. ORDER BY — ordenamiento

`ORDER BY` define el orden de las filas. Admite varios criterios, orden ascendente y descendente y posición de columna. Aquí también aparece la mayor diferencia entre motores: cómo limitar el número de filas.

#### O01 — Clientes ordenados por nombre (ASC)

**Situación de negocio.** El área Comercial necesita entregar el listado oficial de clientes en orden alfabético, primero por tipo de documento y, dentro de cada tipo, por nombre.

**Concepto de SQL.** `ORDER BY` con dos criterios; el segundo solo actúa como desempate del primero. `ASC` es el orden por defecto y se escribe de forma explícita por claridad.

**Resultado esperado.** Los tipos de documento aparecen agrupados (CC, NIT) y, dentro de cada uno, los nombres de la A a la Z.

##### O01 · MySQL

```sql
SELECT c.id, c.tipo_documento, c.nombre, c.email
FROM clientes c
ORDER BY c.tipo_documento ASC, c.nombre ASC
LIMIT 20;
```

![Evidencia O01 en MySQL](evidencias/O01_mysql.png)

*Figura 106. Resultado de la consulta O01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O01 · PostgreSQL

```sql
SELECT c.id, c.tipo_documento, c.nombre, c.email
FROM clientes c
ORDER BY c.tipo_documento ASC, c.nombre ASC
LIMIT 20;
```

![Evidencia O01 en PostgreSQL](evidencias/O01_postgresql.png)

*Figura 107. Resultado de la consulta O01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O01 · SQL Server

```sql
SELECT TOP 20 c.id, c.tipo_documento, c.nombre, c.email
FROM clientes c
ORDER BY c.tipo_documento ASC, c.nombre ASC;
```

![Evidencia O01 en SQL Server](evidencias/O01_sqlserver.png)

*Figura 108. Resultado de la consulta O01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### O01 · Oracle

```sql
SELECT c.id, c.tipo_documento, c.nombre, c.email
FROM clientes c
ORDER BY c.tipo_documento ASC, c.nombre ASC
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia O01 en Oracle](evidencias/O01_oracle.png)

*Figura 109. Resultado de la consulta O01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### O02 — Los 10 pagos más altos (DESC con desempate)

**Situación de negocio.** Tesorería quiere conocer los diez mayores ingresos individuales y, si hay empate en el monto, mostrar primero el más reciente. Es información para el reporte de auditoría trimestral.

**Concepto de SQL.** `ORDER BY monto DESC, fecha DESC` más una cláusula para limitar filas. Limitar filas es **la** diferencia más visible entre motores: `LIMIT` (MySQL, PostgreSQL), `TOP` (SQL Server) y `FETCH FIRST` (Oracle).

**Resultado esperado.** Deben aparecer exactamente 10 filas con montos decrecientes. Con igual monto, la fecha más reciente va primero.

##### O02 · MySQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
ORDER BY p.monto DESC, p.fecha DESC
LIMIT 10;
```

![Evidencia O02 en MySQL](evidencias/O02_mysql.png)

*Figura 110. Resultado de la consulta O02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O02 · PostgreSQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
ORDER BY p.monto DESC, p.fecha DESC
LIMIT 10;
```

![Evidencia O02 en PostgreSQL](evidencias/O02_postgresql.png)

*Figura 111. Resultado de la consulta O02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O02 · SQL Server

```sql
SELECT TOP 10 p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
ORDER BY p.monto DESC, p.fecha DESC;
```

![Evidencia O02 en SQL Server](evidencias/O02_sqlserver.png)

*Figura 112. Resultado de la consulta O02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### O02 · Oracle

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.estado = 'APLICADO'
ORDER BY p.monto DESC, p.fecha DESC
FETCH FIRST 10 ROWS ONLY;
```

![Evidencia O02 en Oracle](evidencias/O02_oracle.png)

*Figura 113. Resultado de la consulta O02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### O03 — Reservas con fecha más lejana primero

**Situación de negocio.** Comercial quiere revisar las reservas con la fecha de evento más adelantada en la agenda, para confirmar que sus datos y anticipos estén completos, viendo primero las más lejanas.

**Concepto de SQL.** `ORDER BY fecha_inicio DESC` más dos `JOIN` para mostrar nombres en lugar de ids. Ordenar de forma descendente y limitar filas es la manera típica de obtener «lo último».

**Resultado esperado.** La primera fila debe tener la fecha de inicio más lejana de toda la tabla de reservas.

##### O03 · MySQL

```sql
SELECT r.id, c.nombre AS cliente, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
LIMIT 15;
```

![Evidencia O03 en MySQL](evidencias/O03_mysql.png)

*Figura 114. Resultado de la consulta O03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O03 · PostgreSQL

```sql
SELECT r.id, c.nombre AS cliente, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
LIMIT 15;
```

![Evidencia O03 en PostgreSQL](evidencias/O03_postgresql.png)

*Figura 115. Resultado de la consulta O03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### O03 · SQL Server

```sql
SELECT TOP 15 r.id, c.nombre AS cliente, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC;
```

![Evidencia O03 en SQL Server](evidencias/O03_sqlserver.png)

*Figura 116. Resultado de la consulta O03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### O03 · Oracle

```sql
SELECT r.id, c.nombre AS cliente, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
FETCH FIRST 15 ROWS ONLY;
```

![Evidencia O03 en Oracle](evidencias/O03_oracle.png)

*Figura 117. Resultado de la consulta O03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### O04 — Salones activos ordenados por capacidad

**Situación de negocio.** La gerencia quiere presentar el portafolio de salones activos comenzando por los de mayor capacidad, con el precio base como criterio de desempate.

**Concepto de SQL.** `ORDER BY` sobre columnas numéricas en orden descendente y filtro previo de salones activos.

**Resultado esperado.** Los salones se presentan del de mayor al de menor capacidad; el salón inactivo no aparece.

##### O04 · MySQL

```sql
SELECT s.id, s.nombre, s.capacidad, s.precio_base
FROM salones s
WHERE s.is_active = 'ACTIVE'
ORDER BY s.capacidad DESC, s.precio_base DESC;
```

![Evidencia O04 en MySQL](evidencias/O04_mysql.png)

*Figura 118. Resultado de la consulta O04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O04 · PostgreSQL

```sql
SELECT s.id, s.nombre, s.capacidad, s.precio_base
FROM salones s
WHERE s.is_active = 'ACTIVE'
ORDER BY s.capacidad DESC, s.precio_base DESC;
```

![Evidencia O04 en PostgreSQL](evidencias/O04_postgresql.png)

*Figura 119. Resultado de la consulta O04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O04 · SQL Server

```sql
SELECT s.id, s.nombre, s.capacidad, s.precio_base
FROM salones s
WHERE s.is_active = 'ACTIVE'
ORDER BY s.capacidad DESC, s.precio_base DESC;
```

![Evidencia O04 en SQL Server](evidencias/O04_sqlserver.png)

*Figura 120. Resultado de la consulta O04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O04 · Oracle

```sql
SELECT s.id, s.nombre, s.capacidad, s.precio_base
FROM salones s
WHERE s.is_active = 'ACTIVE'
ORDER BY s.capacidad DESC, s.precio_base DESC;
```

![Evidencia O04 en Oracle](evidencias/O04_oracle.png)

*Figura 121. Resultado de la consulta O04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### O05 — ORDER BY por posición de columna

**Situación de negocio.** Comercial desea conocer la demanda vigente de cada salón (número de reservas no canceladas), de mayor a menor, para decidir dónde concentrar la inversión publicitaria.

**Concepto de SQL.** `ORDER BY 2 DESC, 1` usa la **posición** de la columna en el `SELECT` en lugar de su nombre. Es válido en los cuatro motores, aunque se recomienda usar alias por legibilidad.

**Resultado esperado.** Los salones se presentan de mayor a menor número de reservas; los empates se ordenan por nombre. Reemplazar la posición `2` por el alias `reservas` produce el mismo orden.

##### O05 · MySQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
WHERE r.estado <> 'CANCELADA'
GROUP BY s.id, s.nombre
ORDER BY 2 DESC, 1;
```

![Evidencia O05 en MySQL](evidencias/O05_mysql.png)

*Figura 122. Resultado de la consulta O05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O05 · PostgreSQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
WHERE r.estado <> 'CANCELADA'
GROUP BY s.id, s.nombre
ORDER BY 2 DESC, 1;
```

![Evidencia O05 en PostgreSQL](evidencias/O05_postgresql.png)

*Figura 123. Resultado de la consulta O05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O05 · SQL Server

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
WHERE r.estado <> 'CANCELADA'
GROUP BY s.id, s.nombre
ORDER BY 2 DESC, 1;
```

![Evidencia O05 en SQL Server](evidencias/O05_sqlserver.png)

*Figura 124. Resultado de la consulta O05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### O05 · Oracle

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS reservas
FROM salones s
JOIN reservas r ON r.salon_id = s.id
WHERE r.estado <> 'CANCELADA'
GROUP BY s.id, s.nombre
ORDER BY 2 DESC, 1;
```

![Evidencia O05 en Oracle](evidencias/O05_oracle.png)

*Figura 125. Resultado de la consulta O05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### O06 — Próximas reservas confirmadas (ASC por fecha)

**Situación de negocio.** Operaciones quiere la agenda inmediata de eventos: las próximas reservas confirmadas desde el 15 de octubre de 2026, en orden cronológico, para avisar a los proveedores.

**Concepto de SQL.** Filtro por fecha con `WHERE` y `ORDER BY fecha_inicio ASC`. El literal de fecha se escribe distinto según el motor (`DATE '2026-10-15'` o `'2026-10-15'`).

**Resultado esperado.** Todas las reservas mostradas tienen fecha igual o posterior al 15 de octubre de 2026 y estado `CONFIRMADA`, de la más cercana a la más lejana.

##### O06 · MySQL

```sql
SELECT r.id, r.fecha_inicio, c.nombre AS cliente, s.nombre AS salon
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
WHERE r.estado = 'CONFIRMADA' AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio ASC
LIMIT 15;
```

![Evidencia O06 en MySQL](evidencias/O06_mysql.png)

*Figura 126. Resultado de la consulta O06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### O06 · PostgreSQL

```sql
SELECT r.id, r.fecha_inicio, c.nombre AS cliente, s.nombre AS salon
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
WHERE r.estado = 'CONFIRMADA' AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio ASC
LIMIT 15;
```

![Evidencia O06 en PostgreSQL](evidencias/O06_postgresql.png)

*Figura 127. Resultado de la consulta O06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### O06 · SQL Server

```sql
SELECT TOP 15 r.id, r.fecha_inicio, c.nombre AS cliente, s.nombre AS salon
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
WHERE r.estado = 'CONFIRMADA' AND r.fecha_inicio >= '2026-10-15'
ORDER BY r.fecha_inicio ASC;
```

![Evidencia O06 en SQL Server](evidencias/O06_sqlserver.png)

*Figura 128. Resultado de la consulta O06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; literal de fecha como cadena `'AAAA-MM-DD'`.

##### O06 · Oracle

```sql
SELECT r.id, r.fecha_inicio, c.nombre AS cliente, s.nombre AS salon
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
WHERE r.estado = 'CONFIRMADA' AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio ASC
FETCH FIRST 15 ROWS ONLY;
```

![Evidencia O06 en Oracle](evidencias/O06_oracle.png)

*Figura 129. Resultado de la consulta O06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

---

### 6. JOIN — combinación de tablas

Los `JOIN` combinan filas de varias tablas mediante llaves foráneas. Se muestran `INNER`, `LEFT`, `RIGHT`, uniones de siete tablas y el *self join* para detectar cruces de agenda.

#### J01 — INNER JOIN de 3 tablas: reserva + cliente + salón

**Situación de negocio.** El área Comercial necesita un reporte que muestre en una sola tabla qué cliente reservó qué salón y en qué fecha, en lugar de consultar tres tablas por separado.

**Concepto de SQL.** `INNER JOIN` de tres tablas (reservas, clientes y salones) enlazadas por llaves foráneas. Solo aparecen las filas que tienen correspondencia en todas las tablas. Se muestra además la concatenación de tipo y número de documento.

**Resultado esperado.** No debe haber celdas vacías en cliente ni en salón: el `INNER JOIN` descarta lo que no tiene correspondencia.

##### J01 · MySQL

```sql
SELECT r.id, c.nombre AS cliente, CONCAT(c.tipo_documento, ' ', c.numero_documento) AS documento, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
LIMIT 20;
```

![Evidencia J01 en MySQL](evidencias/J01_mysql.png)

*Figura 130. Resultado de la consulta J01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; concatenación con `CONCAT()`.

##### J01 · PostgreSQL

```sql
SELECT r.id, c.nombre AS cliente, c.tipo_documento || ' ' || c.numero_documento AS documento, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
LIMIT 20;
```

![Evidencia J01 en PostgreSQL](evidencias/J01_postgresql.png)

*Figura 131. Resultado de la consulta J01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; concatenación con `||`.

##### J01 · SQL Server

```sql
SELECT TOP 20 r.id, c.nombre AS cliente, c.tipo_documento + ' ' + c.numero_documento AS documento, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC;
```

![Evidencia J01 en SQL Server](evidencias/J01_sqlserver.png)

*Figura 132. Resultado de la consulta J01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; concatenación con `+`.

##### J01 · Oracle

```sql
SELECT r.id, c.nombre AS cliente, c.tipo_documento || ' ' || c.numero_documento AS documento, s.nombre AS salon, r.fecha_inicio, r.estado
FROM reservas r
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
ORDER BY r.fecha_inicio DESC, r.id DESC
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia J01 en Oracle](evidencias/J01_oracle.png)

*Figura 133. Resultado de la consulta J01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; concatenación con `||`.

#### J02 — LEFT JOIN ... IS NULL: clientes que nunca han reservado

**Situación de negocio.** Comercial quiere recuperar posibles clientes: los que se registraron en el sistema pero jamás hicieron una reserva.

**Concepto de SQL.** `LEFT JOIN ... WHERE r.id IS NULL` es el patrón clásico de «anti-join»: conserva todos los clientes de la tabla izquierda y deja solo aquellos sin pareja en reservas.

**Resultado esperado.** Los clientes listados no tienen ninguna reserva.

##### J02 · MySQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
LEFT JOIN reservas r ON r.cliente_id = c.id
WHERE r.id IS NULL
ORDER BY c.nombre;
```

![Evidencia J02 en MySQL](evidencias/J02_mysql.png)

*Figura 134. Resultado de la consulta J02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J02 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
LEFT JOIN reservas r ON r.cliente_id = c.id
WHERE r.id IS NULL
ORDER BY c.nombre;
```

![Evidencia J02 en PostgreSQL](evidencias/J02_postgresql.png)

*Figura 135. Resultado de la consulta J02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J02 · SQL Server

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
LEFT JOIN reservas r ON r.cliente_id = c.id
WHERE r.id IS NULL
ORDER BY c.nombre;
```

![Evidencia J02 en SQL Server](evidencias/J02_sqlserver.png)

*Figura 136. Resultado de la consulta J02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J02 · Oracle

```sql
SELECT c.id, c.nombre, c.email
FROM clientes c
LEFT JOIN reservas r ON r.cliente_id = c.id
WHERE r.id IS NULL
ORDER BY c.nombre;
```

![Evidencia J02 en Oracle](evidencias/J02_oracle.png)

*Figura 137. Resultado de la consulta J02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### J03 — RIGHT JOIN: reservas confirmadas por salón (incluye salones sin reservas)

**Situación de negocio.** La gerencia quiere ver el catálogo completo de salones con el número de reservas confirmadas, **incluyendo los que no tienen ninguna**, porque un salón con cero reservas también es información.

**Concepto de SQL.** `RIGHT JOIN` conserva todos los salones (tabla derecha) aunque no tengan reservas. Escribir el filtro de estado en la condición del `ON` y no en el `WHERE` es lo que mantiene los salones con cero.

**Resultado esperado.** Aparecen los 6 salones del catálogo. Un salón sin reservas confirmadas (como Salon Cielo) se muestra con 0, no desaparece.

##### J03 · MySQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS confirmadas
FROM reservas r
RIGHT JOIN salones s ON s.id = r.salon_id AND r.estado = 'CONFIRMADA'
GROUP BY s.id, s.nombre
ORDER BY confirmadas DESC, s.nombre;
```

![Evidencia J03 en MySQL](evidencias/J03_mysql.png)

*Figura 138. Resultado de la consulta J03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J03 · PostgreSQL

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS confirmadas
FROM reservas r
RIGHT JOIN salones s ON s.id = r.salon_id AND r.estado = 'CONFIRMADA'
GROUP BY s.id, s.nombre
ORDER BY confirmadas DESC, s.nombre;
```

![Evidencia J03 en PostgreSQL](evidencias/J03_postgresql.png)

*Figura 139. Resultado de la consulta J03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J03 · SQL Server

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS confirmadas
FROM reservas r
RIGHT JOIN salones s ON s.id = r.salon_id AND r.estado = 'CONFIRMADA'
GROUP BY s.id, s.nombre
ORDER BY confirmadas DESC, s.nombre;
```

![Evidencia J03 en SQL Server](evidencias/J03_sqlserver.png)

*Figura 140. Resultado de la consulta J03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J03 · Oracle

```sql
SELECT s.nombre AS salon, COUNT(r.id) AS confirmadas
FROM reservas r
RIGHT JOIN salones s ON s.id = r.salon_id AND r.estado = 'CONFIRMADA'
GROUP BY s.id, s.nombre
ORDER BY confirmadas DESC, s.nombre;
```

![Evidencia J03 en Oracle](evidencias/J03_oracle.png)

*Figura 141. Resultado de la consulta J03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### J04 — LEFT JOIN: proveedores activos sin servicios asignados

**Situación de negocio.** Operaciones quiere saber qué proveedores activos no han prestado ningún servicio, para decidir si se les ofrece participar en los próximos eventos o se depuran del catálogo.

**Concepto de SQL.** `LEFT JOIN` con la condición de estado dentro del `ON` y `WHERE es.id IS NULL` para quedarse con los proveedores sin servicios. Es otro uso del anti-join.

**Resultado esperado.** Los proveedores listados no tienen servicios vigentes asignados (debe aparecer Logistica Sin Contratos SAS).

##### J04 · MySQL

```sql
SELECT p.id, p.razon_social, p.contacto, p.telefono
FROM proveedores p
LEFT JOIN evento_servicios es ON es.proveedor_id = p.id AND es.estado <> 'CANCELADO'
WHERE p.is_active = 'ACTIVE' AND es.id IS NULL
ORDER BY p.razon_social;
```

![Evidencia J04 en MySQL](evidencias/J04_mysql.png)

*Figura 142. Resultado de la consulta J04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J04 · PostgreSQL

```sql
SELECT p.id, p.razon_social, p.contacto, p.telefono
FROM proveedores p
LEFT JOIN evento_servicios es ON es.proveedor_id = p.id AND es.estado <> 'CANCELADO'
WHERE p.is_active = 'ACTIVE' AND es.id IS NULL
ORDER BY p.razon_social;
```

![Evidencia J04 en PostgreSQL](evidencias/J04_postgresql.png)

*Figura 143. Resultado de la consulta J04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J04 · SQL Server

```sql
SELECT p.id, p.razon_social, p.contacto, p.telefono
FROM proveedores p
LEFT JOIN evento_servicios es ON es.proveedor_id = p.id AND es.estado <> 'CANCELADO'
WHERE p.is_active = 'ACTIVE' AND es.id IS NULL
ORDER BY p.razon_social;
```

![Evidencia J04 en SQL Server](evidencias/J04_sqlserver.png)

*Figura 144. Resultado de la consulta J04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J04 · Oracle

```sql
SELECT p.id, p.razon_social, p.contacto, p.telefono
FROM proveedores p
LEFT JOIN evento_servicios es ON es.proveedor_id = p.id AND es.estado <> 'CANCELADO'
WHERE p.is_active = 'ACTIVE' AND es.id IS NULL
ORDER BY p.razon_social;
```

![Evidencia J04 en Oracle](evidencias/J04_oracle.png)

*Figura 145. Resultado de la consulta J04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### J05 — JOIN de 7 tablas: detalle de servicios por evento

**Situación de negocio.** Cartera y Operaciones necesitan el detalle completo de cada servicio contratado: qué cliente, en qué salón, qué tipo de evento, qué servicio y qué proveedor, para validar las órdenes de servicio contra el sistema.

**Concepto de SQL.** `JOIN` de siete tablas: evento_servicios, eventos, reservas, clientes, salones, servicios y proveedores. Muestra cómo se recorren varias relaciones 1:N hasta reunir toda la información en una fila.

**Resultado esperado.** Cada fila contiene cliente, salón, evento, servicio y proveedor, sin valores nulos en los nombres.

##### J05 · MySQL

```sql
SELECT es.id, c.nombre AS cliente, s.nombre AS salon, e.tipo AS evento, sv.nombre AS servicio, p.razon_social AS proveedor, es.fecha, es.estado
FROM evento_servicios es
JOIN eventos e ON e.id = es.evento_id
JOIN reservas r ON r.id = e.reserva_id
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
JOIN servicios sv ON sv.id = es.servicio_id
JOIN proveedores p ON p.id = es.proveedor_id
ORDER BY es.fecha DESC, es.id
LIMIT 20;
```

![Evidencia J05 en MySQL](evidencias/J05_mysql.png)

*Figura 146. Resultado de la consulta J05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### J05 · PostgreSQL

```sql
SELECT es.id, c.nombre AS cliente, s.nombre AS salon, e.tipo AS evento, sv.nombre AS servicio, p.razon_social AS proveedor, es.fecha, es.estado
FROM evento_servicios es
JOIN eventos e ON e.id = es.evento_id
JOIN reservas r ON r.id = e.reserva_id
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
JOIN servicios sv ON sv.id = es.servicio_id
JOIN proveedores p ON p.id = es.proveedor_id
ORDER BY es.fecha DESC, es.id
LIMIT 20;
```

![Evidencia J05 en PostgreSQL](evidencias/J05_postgresql.png)

*Figura 147. Resultado de la consulta J05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### J05 · SQL Server

```sql
SELECT TOP 20 es.id, c.nombre AS cliente, s.nombre AS salon, e.tipo AS evento, sv.nombre AS servicio, p.razon_social AS proveedor, es.fecha, es.estado
FROM evento_servicios es
JOIN eventos e ON e.id = es.evento_id
JOIN reservas r ON r.id = e.reserva_id
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
JOIN servicios sv ON sv.id = es.servicio_id
JOIN proveedores p ON p.id = es.proveedor_id
ORDER BY es.fecha DESC, es.id;
```

![Evidencia J05 en SQL Server](evidencias/J05_sqlserver.png)

*Figura 148. Resultado de la consulta J05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### J05 · Oracle

```sql
SELECT es.id, c.nombre AS cliente, s.nombre AS salon, e.tipo AS evento, sv.nombre AS servicio, p.razon_social AS proveedor, es.fecha, es.estado
FROM evento_servicios es
JOIN eventos e ON e.id = es.evento_id
JOIN reservas r ON r.id = e.reserva_id
JOIN clientes c ON c.id = r.cliente_id
JOIN salones s ON s.id = r.salon_id
JOIN servicios sv ON sv.id = es.servicio_id
JOIN proveedores p ON p.id = es.proveedor_id
ORDER BY es.fecha DESC, es.id
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia J05 en Oracle](evidencias/J05_oracle.png)

*Figura 149. Resultado de la consulta J05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### J06 — Self JOIN: reservas solapadas en el mismo salón

**Situación de negocio.** Una regla del negocio de CelebraHub dice que **un salón no puede tener dos reservas activas que se crucen en el tiempo**. Se pidió verificar que los datos cumplan esa regla.

**Concepto de SQL.** **Self join**: la tabla `reservas` se une consigo misma con alias `a` y `b`. La condición `a.id < b.id` evita comparar una reserva consigo misma y duplicar pares, y el cruce se detecta con `a.fecha_inicio < b.fecha_fin AND b.fecha_inicio < a.fecha_fin`.

**Resultado esperado.** Cada fila sería un par de reservas del mismo salón cuyos horarios se cruzan. Si no devuelve filas, la agenda cumple la regla (resultado esperado con los datos cargados).

##### J06 · MySQL

```sql
SELECT a.id AS reserva_a, b.id AS reserva_b, s.nombre AS salon,
       a.fecha_inicio AS inicio_a, a.fecha_fin AS fin_a, b.fecha_inicio AS inicio_b, b.fecha_fin AS fin_b
FROM reservas a
JOIN reservas b ON a.salon_id = b.salon_id AND a.id < b.id
               AND a.fecha_inicio < b.fecha_fin AND b.fecha_inicio < a.fecha_fin
JOIN salones s ON s.id = a.salon_id
WHERE a.estado <> 'CANCELADA' AND b.estado <> 'CANCELADA'
ORDER BY a.fecha_inicio;
```

![Evidencia J06 en MySQL](evidencias/J06_mysql.png)

*Figura 150. Resultado de la consulta J06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J06 · PostgreSQL

```sql
SELECT a.id AS reserva_a, b.id AS reserva_b, s.nombre AS salon,
       a.fecha_inicio AS inicio_a, a.fecha_fin AS fin_a, b.fecha_inicio AS inicio_b, b.fecha_fin AS fin_b
FROM reservas a
JOIN reservas b ON a.salon_id = b.salon_id AND a.id < b.id
               AND a.fecha_inicio < b.fecha_fin AND b.fecha_inicio < a.fecha_fin
JOIN salones s ON s.id = a.salon_id
WHERE a.estado <> 'CANCELADA' AND b.estado <> 'CANCELADA'
ORDER BY a.fecha_inicio;
```

![Evidencia J06 en PostgreSQL](evidencias/J06_postgresql.png)

*Figura 151. Resultado de la consulta J06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J06 · SQL Server

```sql
SELECT a.id AS reserva_a, b.id AS reserva_b, s.nombre AS salon,
       a.fecha_inicio AS inicio_a, a.fecha_fin AS fin_a, b.fecha_inicio AS inicio_b, b.fecha_fin AS fin_b
FROM reservas a
JOIN reservas b ON a.salon_id = b.salon_id AND a.id < b.id
               AND a.fecha_inicio < b.fecha_fin AND b.fecha_inicio < a.fecha_fin
JOIN salones s ON s.id = a.salon_id
WHERE a.estado <> 'CANCELADA' AND b.estado <> 'CANCELADA'
ORDER BY a.fecha_inicio;
```

![Evidencia J06 en SQL Server](evidencias/J06_sqlserver.png)

*Figura 152. Resultado de la consulta J06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

##### J06 · Oracle

```sql
SELECT a.id AS reserva_a, b.id AS reserva_b, s.nombre AS salon,
       a.fecha_inicio AS inicio_a, a.fecha_fin AS fin_a, b.fecha_inicio AS inicio_b, b.fecha_fin AS fin_b
FROM reservas a
JOIN reservas b ON a.salon_id = b.salon_id AND a.id < b.id
               AND a.fecha_inicio < b.fecha_fin AND b.fecha_inicio < a.fecha_fin
JOIN salones s ON s.id = a.salon_id
WHERE a.estado <> 'CANCELADA' AND b.estado <> 'CANCELADA'
ORDER BY a.fecha_inicio;
```

![Evidencia J06 en Oracle](evidencias/J06_oracle.png)

*Figura 153. Resultado de la consulta J06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* La sentencia es idéntica en los cuatro motores y no requirió adaptaciones de sintaxis.

#### J07 — Self JOIN: proveedor asignado a dos eventos el mismo día

**Situación de negocio.** Igual que con los salones, un proveedor puede no tener capacidad para atender dos eventos distintos el mismo día. Operaciones solicitó detectar esos cruces para reforzar la logística.

**Concepto de SQL.** Mismo patrón de self join de la consulta anterior, cambiando la columna de enlace a `proveedor_id` y comparando solo la parte de fecha con `CAST(... AS DATE)`; se exige además que los eventos sean distintos.

**Resultado esperado.** Cada fila identifica un par de servicios de un mismo proveedor en eventos distintos que ocurren el mismo día.

##### J07 · MySQL

```sql
SELECT a.id AS servicio_a, b.id AS servicio_b, p.razon_social AS proveedor,
       a.evento_id AS evento_a, b.evento_id AS evento_b, a.fecha
FROM evento_servicios a
JOIN evento_servicios b ON a.proveedor_id = b.proveedor_id AND a.id < b.id
                        AND a.evento_id <> b.evento_id
                        AND CAST(a.fecha AS DATE) = CAST(b.fecha AS DATE)
JOIN proveedores p ON p.id = a.proveedor_id
WHERE a.estado <> 'CANCELADO' AND b.estado <> 'CANCELADO'
ORDER BY a.fecha
LIMIT 20;
```

![Evidencia J07 en MySQL](evidencias/J07_mysql.png)

*Figura 154. Resultado de la consulta J07 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### J07 · PostgreSQL

```sql
SELECT a.id AS servicio_a, b.id AS servicio_b, p.razon_social AS proveedor,
       a.evento_id AS evento_a, b.evento_id AS evento_b, a.fecha
FROM evento_servicios a
JOIN evento_servicios b ON a.proveedor_id = b.proveedor_id AND a.id < b.id
                        AND a.evento_id <> b.evento_id
                        AND CAST(a.fecha AS DATE) = CAST(b.fecha AS DATE)
JOIN proveedores p ON p.id = a.proveedor_id
WHERE a.estado <> 'CANCELADO' AND b.estado <> 'CANCELADO'
ORDER BY a.fecha
LIMIT 20;
```

![Evidencia J07 en PostgreSQL](evidencias/J07_postgresql.png)

*Figura 155. Resultado de la consulta J07 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### J07 · SQL Server

```sql
SELECT TOP 20 a.id AS servicio_a, b.id AS servicio_b, p.razon_social AS proveedor,
       a.evento_id AS evento_a, b.evento_id AS evento_b, a.fecha
FROM evento_servicios a
JOIN evento_servicios b ON a.proveedor_id = b.proveedor_id AND a.id < b.id
                        AND a.evento_id <> b.evento_id
                        AND CAST(a.fecha AS DATE) = CAST(b.fecha AS DATE)
JOIN proveedores p ON p.id = a.proveedor_id
WHERE a.estado <> 'CANCELADO' AND b.estado <> 'CANCELADO'
ORDER BY a.fecha;
```

![Evidencia J07 en SQL Server](evidencias/J07_sqlserver.png)

*Figura 156. Resultado de la consulta J07 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### J07 · Oracle

```sql
SELECT a.id AS servicio_a, b.id AS servicio_b, p.razon_social AS proveedor,
       a.evento_id AS evento_a, b.evento_id AS evento_b, a.fecha
FROM evento_servicios a
JOIN evento_servicios b ON a.proveedor_id = b.proveedor_id AND a.id < b.id
                        AND a.evento_id <> b.evento_id
                        AND CAST(a.fecha AS DATE) = CAST(b.fecha AS DATE)
JOIN proveedores p ON p.id = a.proveedor_id
WHERE a.estado <> 'CANCELADO' AND b.estado <> 'CANCELADO'
ORDER BY a.fecha
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia J07 en Oracle](evidencias/J07_oracle.png)

*Figura 157. Resultado de la consulta J07 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

---

### 7. WHERE — filtros por fila

`WHERE` filtra filas individuales antes de agrupar. Se muestran comparaciones, `AND/OR/NOT`, `BETWEEN`, `IN`, `IS NULL` / `IS NOT NULL` y comparaciones de fechas.

#### W01 — Comparación + AND: reservas confirmadas futuras del Salón Perla

**Situación de negocio.** Comercial requiere las reservas confirmadas del Salón Perla (id 1) con fecha desde el 15 de octubre de 2026, para preparar el seguimiento de los eventos próximos.

**Concepto de SQL.** `WHERE` con varias condiciones unidas por `AND`: comparación de texto (estado), de número (salón) y de fecha (`>=`).

**Resultado esperado.** Todas las filas deben cumplir las tres condiciones. Quitar una de ellas debe aumentar el número de filas.

##### W01 · MySQL

```sql
SELECT r.id, r.cliente_id, r.salon_id, r.fecha_inicio, r.estado
FROM reservas r
WHERE r.estado = 'CONFIRMADA' AND r.salon_id = 1 AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia W01 en MySQL](evidencias/W01_mysql.png)

*Figura 158. Resultado de la consulta W01 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### W01 · PostgreSQL

```sql
SELECT r.id, r.cliente_id, r.salon_id, r.fecha_inicio, r.estado
FROM reservas r
WHERE r.estado = 'CONFIRMADA' AND r.salon_id = 1 AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia W01 en PostgreSQL](evidencias/W01_postgresql.png)

*Figura 159. Resultado de la consulta W01 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### W01 · SQL Server

```sql
SELECT TOP 20 r.id, r.cliente_id, r.salon_id, r.fecha_inicio, r.estado
FROM reservas r
WHERE r.estado = 'CONFIRMADA' AND r.salon_id = 1 AND r.fecha_inicio >= '2026-10-15'
ORDER BY r.fecha_inicio;
```

![Evidencia W01 en SQL Server](evidencias/W01_sqlserver.png)

*Figura 160. Resultado de la consulta W01 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; literal de fecha como cadena `'AAAA-MM-DD'`.

##### W01 · Oracle

```sql
SELECT r.id, r.cliente_id, r.salon_id, r.fecha_inicio, r.estado
FROM reservas r
WHERE r.estado = 'CONFIRMADA' AND r.salon_id = 1 AND r.fecha_inicio >= DATE '2026-10-15'
ORDER BY r.fecha_inicio
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W01 en Oracle](evidencias/W01_oracle.png)

*Figura 161. Resultado de la consulta W01 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

#### W02 — BETWEEN: pagos de mayo a julio con montos entre 1.000.000 y 3.000.000

**Situación de negocio.** Cartera audita los pagos de tamaño medio realizados entre mayo y julio de 2026, cuyo monto está entre $1.000.000 y $3.000.000, para comprobar que se hayan emitido sus recibos.

**Concepto de SQL.** `BETWEEN` es inclusivo en ambos extremos y se puede usar con números y con fechas. Aquí se aplica dos veces: sobre la fecha y sobre el monto.

**Resultado esperado.** Todos los montos están entre 1.000.000 y 3.000.000 (incluidos) y todas las fechas caen entre el 1 de mayo y el 31 de julio de 2026.

##### W02 · MySQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.fecha BETWEEN DATE '2026-05-01' AND DATE '2026-07-31'
  AND p.monto BETWEEN 1000000 AND 3000000
ORDER BY p.fecha
LIMIT 20;
```

![Evidencia W02 en MySQL](evidencias/W02_mysql.png)

*Figura 162. Resultado de la consulta W02 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### W02 · PostgreSQL

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.fecha BETWEEN DATE '2026-05-01' AND DATE '2026-07-31'
  AND p.monto BETWEEN 1000000 AND 3000000
ORDER BY p.fecha
LIMIT 20;
```

![Evidencia W02 en PostgreSQL](evidencias/W02_postgresql.png)

*Figura 163. Resultado de la consulta W02 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

##### W02 · SQL Server

```sql
SELECT TOP 20 p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.fecha BETWEEN '2026-05-01' AND '2026-07-31'
  AND p.monto BETWEEN 1000000 AND 3000000
ORDER BY p.fecha;
```

![Evidencia W02 en SQL Server](evidencias/W02_sqlserver.png)

*Figura 164. Resultado de la consulta W02 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas; literal de fecha como cadena `'AAAA-MM-DD'`.

##### W02 · Oracle

```sql
SELECT p.id, p.contrato_id, p.tipo_pago, p.metodo, p.monto, p.fecha
FROM pagos p
WHERE p.fecha BETWEEN DATE '2026-05-01' AND DATE '2026-07-31'
  AND p.monto BETWEEN 1000000 AND 3000000
ORDER BY p.fecha
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W02 en Oracle](evidencias/W02_oracle.png)

*Figura 165. Resultado de la consulta W02 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas; literal de fecha `DATE 'AAAA-MM-DD'`.

#### W03 — IN: pagos pendientes por tarjeta o transferencia

**Situación de negocio.** Tesorería concilia los pagos electrónicos —tarjeta y transferencia— pendientes de verificar contra el extracto bancario, y deja por fuera los pagos en efectivo.

**Concepto de SQL.** `IN (...)` reemplaza varias comparaciones `OR` sobre la misma columna. Es más corto y más fácil de leer cuando la lista crece.

**Resultado esperado.** La columna `metodo` solo contiene `TARJETA` o `TRANSFERENCIA`; nunca `EFECTIVO`. El estado es siempre `PENDIENTE`.

##### W03 · MySQL

```sql
SELECT p.id, p.contrato_id, p.metodo, p.monto, p.estado
FROM pagos p
WHERE p.metodo IN ('TARJETA', 'TRANSFERENCIA') AND p.estado = 'PENDIENTE'
ORDER BY p.fecha
LIMIT 20;
```

![Evidencia W03 en MySQL](evidencias/W03_mysql.png)

*Figura 166. Resultado de la consulta W03 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W03 · PostgreSQL

```sql
SELECT p.id, p.contrato_id, p.metodo, p.monto, p.estado
FROM pagos p
WHERE p.metodo IN ('TARJETA', 'TRANSFERENCIA') AND p.estado = 'PENDIENTE'
ORDER BY p.fecha
LIMIT 20;
```

![Evidencia W03 en PostgreSQL](evidencias/W03_postgresql.png)

*Figura 167. Resultado de la consulta W03 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W03 · SQL Server

```sql
SELECT TOP 20 p.id, p.contrato_id, p.metodo, p.monto, p.estado
FROM pagos p
WHERE p.metodo IN ('TARJETA', 'TRANSFERENCIA') AND p.estado = 'PENDIENTE'
ORDER BY p.fecha;
```

![Evidencia W03 en SQL Server](evidencias/W03_sqlserver.png)

*Figura 168. Resultado de la consulta W03 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### W03 · Oracle

```sql
SELECT p.id, p.contrato_id, p.metodo, p.monto, p.estado
FROM pagos p
WHERE p.metodo IN ('TARJETA', 'TRANSFERENCIA') AND p.estado = 'PENDIENTE'
ORDER BY p.fecha
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W03 en Oracle](evidencias/W03_oracle.png)

*Figura 169. Resultado de la consulta W03 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### W04 — OR con paréntesis: clientes inactivos o sin teléfono con Gmail

**Situación de negocio.** Comercial debe depurar la base: clientes inactivos o sin teléfono registrado, pero solo los que tienen correo Gmail, porque son los únicos con los que se puede avisar por ese canal.

**Concepto de SQL.** Mezcla de `OR` y `AND`. Los **paréntesis** son decisivos: sin ellos, `AND` tendría prioridad sobre `OR` y la consulta devolvería resultados distintos. Se usa además `IS NULL` para el teléfono.

**Resultado esperado.** Cada fila cumple (inactivo **o** sin teléfono) y además tiene correo Gmail.

##### W04 · MySQL

```sql
SELECT c.id, c.nombre, c.email, c.telefono, c.is_active
FROM clientes c
WHERE (c.is_active = 'INACTIVE' OR c.telefono IS NULL)
  AND c.email LIKE '%gmail.com'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia W04 en MySQL](evidencias/W04_mysql.png)

*Figura 170. Resultado de la consulta W04 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W04 · PostgreSQL

```sql
SELECT c.id, c.nombre, c.email, c.telefono, c.is_active
FROM clientes c
WHERE (c.is_active = 'INACTIVE' OR c.telefono IS NULL)
  AND c.email LIKE '%gmail.com'
ORDER BY c.nombre
LIMIT 20;
```

![Evidencia W04 en PostgreSQL](evidencias/W04_postgresql.png)

*Figura 171. Resultado de la consulta W04 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W04 · SQL Server

```sql
SELECT TOP 20 c.id, c.nombre, c.email, c.telefono, c.is_active
FROM clientes c
WHERE (c.is_active = 'INACTIVE' OR c.telefono IS NULL)
  AND c.email LIKE '%gmail.com'
ORDER BY c.nombre;
```

![Evidencia W04 en SQL Server](evidencias/W04_sqlserver.png)

*Figura 172. Resultado de la consulta W04 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### W04 · Oracle

```sql
SELECT c.id, c.nombre, c.email, c.telefono, c.is_active
FROM clientes c
WHERE (c.is_active = 'INACTIVE' OR c.telefono IS NULL)
  AND c.email LIKE '%gmail.com'
ORDER BY c.nombre
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W04 en Oracle](evidencias/W04_oracle.png)

*Figura 173. Resultado de la consulta W04 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### W05 — IS NULL: eventos planeados sin observaciones

**Situación de negocio.** Operaciones quiere ver los eventos planeados que aún no tienen ninguna observación registrada, para llamar a los clientes y completar los requerimientos especiales.

**Concepto de SQL.** `IS NULL` es la única forma correcta de buscar valores nulos: `= NULL` nunca es verdadero porque `NULL` no es igual a nada, ni siquiera a otro `NULL`.

**Resultado esperado.** La columna `observaciones` debe estar vacía en todas las filas y el estado debe ser `PLANEADO`.

##### W05 · MySQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.observaciones IS NULL AND e.estado = 'PLANEADO'
ORDER BY e.fecha
LIMIT 20;
```

![Evidencia W05 en MySQL](evidencias/W05_mysql.png)

*Figura 174. Resultado de la consulta W05 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W05 · PostgreSQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.observaciones IS NULL AND e.estado = 'PLANEADO'
ORDER BY e.fecha
LIMIT 20;
```

![Evidencia W05 en PostgreSQL](evidencias/W05_postgresql.png)

*Figura 175. Resultado de la consulta W05 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W05 · SQL Server

```sql
SELECT TOP 20 e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.observaciones IS NULL AND e.estado = 'PLANEADO'
ORDER BY e.fecha;
```

![Evidencia W05 en SQL Server](evidencias/W05_sqlserver.png)

*Figura 176. Resultado de la consulta W05 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### W05 · Oracle

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.observaciones IS NULL AND e.estado = 'PLANEADO'
ORDER BY e.fecha
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W05 en Oracle](evidencias/W05_oracle.png)

*Figura 177. Resultado de la consulta W05 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### W06 — IS NOT NULL: reservas con observaciones

**Situación de negocio.** Operaciones revisa las reservas que tienen alguna observación registrada (montaje anticipado, accesibilidad, visita técnica) para llevar una estadística de requerimientos especiales.

**Concepto de SQL.** `IS NOT NULL` filtra las filas donde el campo opcional sí tiene contenido.

**Resultado esperado.** La columna de observaciones no debe tener valores vacíos en ninguna fila.

##### W06 · MySQL

```sql
SELECT r.id, r.fecha_inicio, r.estado, r.observaciones
FROM reservas r
WHERE r.observaciones IS NOT NULL
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia W06 en MySQL](evidencias/W06_mysql.png)

*Figura 178. Resultado de la consulta W06 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W06 · PostgreSQL

```sql
SELECT r.id, r.fecha_inicio, r.estado, r.observaciones
FROM reservas r
WHERE r.observaciones IS NOT NULL
ORDER BY r.fecha_inicio
LIMIT 20;
```

![Evidencia W06 en PostgreSQL](evidencias/W06_postgresql.png)

*Figura 179. Resultado de la consulta W06 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W06 · SQL Server

```sql
SELECT TOP 20 r.id, r.fecha_inicio, r.estado, r.observaciones
FROM reservas r
WHERE r.observaciones IS NOT NULL
ORDER BY r.fecha_inicio;
```

![Evidencia W06 en SQL Server](evidencias/W06_sqlserver.png)

*Figura 180. Resultado de la consulta W06 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### W06 · Oracle

```sql
SELECT r.id, r.fecha_inicio, r.estado, r.observaciones
FROM reservas r
WHERE r.observaciones IS NOT NULL
ORDER BY r.fecha_inicio
FETCH FIRST 20 ROWS ONLY;
```

![Evidencia W06 en Oracle](evidencias/W06_oracle.png)

*Figura 181. Resultado de la consulta W06 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

#### W07 — Comparación de fechas: eventos realizados antes de julio de 2026

**Situación de negocio.** La gerencia requiere revisar los eventos realizados antes del 1 de julio de 2026 para consolidar el cierre del primer semestre.

**Concepto de SQL.** Comparación de fechas con el operador `<` sobre una columna de fecha y hora. El literal de fecha se escribe de forma distinta según el motor.

**Resultado esperado.** Todos los eventos listados tienen estado `REALIZADO` y fecha anterior al 1 de julio de 2026, en orden cronológico.

##### W07 · MySQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.estado = 'REALIZADO' AND e.fecha < DATE '2026-07-01'
ORDER BY e.fecha;
```

![Evidencia W07 en MySQL](evidencias/W07_mysql.png)

*Figura 182. Resultado de la consulta W07 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: literal de fecha `DATE 'AAAA-MM-DD'`.

##### W07 · PostgreSQL

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.estado = 'REALIZADO' AND e.fecha < DATE '2026-07-01'
ORDER BY e.fecha;
```

![Evidencia W07 en PostgreSQL](evidencias/W07_postgresql.png)

*Figura 183. Resultado de la consulta W07 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: literal de fecha `DATE 'AAAA-MM-DD'`.

##### W07 · SQL Server

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.estado = 'REALIZADO' AND e.fecha < '2026-07-01'
ORDER BY e.fecha;
```

![Evidencia W07 en SQL Server](evidencias/W07_sqlserver.png)

*Figura 184. Resultado de la consulta W07 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: literal de fecha como cadena `'AAAA-MM-DD'`.

##### W07 · Oracle

```sql
SELECT e.id, e.tipo, e.fecha, e.cantidad, e.estado
FROM eventos e
WHERE e.estado = 'REALIZADO' AND e.fecha < DATE '2026-07-01'
ORDER BY e.fecha;
```

![Evidencia W07 en Oracle](evidencias/W07_oracle.png)

*Figura 185. Resultado de la consulta W07 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: literal de fecha `DATE 'AAAA-MM-DD'`.

#### W08 — NOT: contratos que no están cancelados ni finalizados

**Situación de negocio.** Cartera solo gestiona cobro sobre contratos en curso, por lo que debe excluir los que ya están cancelados o finalizados.

**Concepto de SQL.** `NOT` aplicado a una condición compuesta (`NOT (... OR ...)`) para excluir un conjunto de valores. Equivale a listar todos los demás estados.

**Resultado esperado.** Solo aparecen contratos cuyo estado no es `CANCELADO` ni `FINALIZADO`, es decir, borrador, vigente o modificado.

##### W08 · MySQL

```sql
SELECT ct.id, ct.numero, ct.cliente_id, ct.valor, ct.estado
FROM contratos ct
WHERE NOT (ct.estado = 'CANCELADO' OR ct.estado = 'FINALIZADO')
ORDER BY ct.id
LIMIT 15;
```

![Evidencia W08 en MySQL](evidencias/W08_mysql.png)

*Figura 186. Resultado de la consulta W08 en MySQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (MySQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W08 · PostgreSQL

```sql
SELECT ct.id, ct.numero, ct.cliente_id, ct.valor, ct.estado
FROM contratos ct
WHERE NOT (ct.estado = 'CANCELADO' OR ct.estado = 'FINALIZADO')
ORDER BY ct.id
LIMIT 15;
```

![Evidencia W08 en PostgreSQL](evidencias/W08_postgresql.png)

*Figura 187. Resultado de la consulta W08 en PostgreSQL.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (PostgreSQL):* Particularidades de sintaxis de este motor en esta consulta: `LIMIT n` para limitar filas.

##### W08 · SQL Server

```sql
SELECT TOP 15 ct.id, ct.numero, ct.cliente_id, ct.valor, ct.estado
FROM contratos ct
WHERE NOT (ct.estado = 'CANCELADO' OR ct.estado = 'FINALIZADO')
ORDER BY ct.id;
```

![Evidencia W08 en SQL Server](evidencias/W08_sqlserver.png)

*Figura 188. Resultado de la consulta W08 en SQL Server.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (SQL Server):* Particularidades de sintaxis de este motor en esta consulta: `SELECT TOP n` para limitar filas.

##### W08 · Oracle

```sql
SELECT ct.id, ct.numero, ct.cliente_id, ct.valor, ct.estado
FROM contratos ct
WHERE NOT (ct.estado = 'CANCELADO' OR ct.estado = 'FINALIZADO')
ORDER BY ct.id
FETCH FIRST 15 ROWS ONLY;
```

![Evidencia W08 en Oracle](evidencias/W08_oracle.png)

*Figura 189. Resultado de la consulta W08 en Oracle.*

| Registro de ejecución | |
|---|---|
| Filas obtenidas | |
| Tiempo de ejecución (ms) | |
| Observaciones | |

*Nota de sintaxis (Oracle):* Particularidades de sintaxis de este motor en esta consulta: `FETCH FIRST n ROWS ONLY` para limitar filas.

---

## 10. Matriz de verificación de evidencias

La siguiente matriz permite verificar el registro de la evidencia fotográfica de cada consulta.

| Código | Tema | MySQL | PostgreSQL | SQL Server | Oracle |
|---|---|:---:|:---:|:---:|:---:|
| Q00 | 0. Verificación de la carga de datos | ☐ | ☐ | ☐ | ☐ |
| L01 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L02 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L03 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L04 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L05 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L06 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| L07 | 1. LIKE — búsqueda por patrones | ☐ | ☐ | ☐ | ☐ |
| S01 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| S02 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| S03 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| S04 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| S05 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| S06 | 2. Subconsultas | ☐ | ☐ | ☐ | ☐ |
| G01 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| G02 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| G03 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| G04 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| G05 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| G06 | 3. GROUP BY — agrupación y agregación | ☐ | ☐ | ☐ | ☐ |
| H01 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| H02 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| H03 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| H04 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| H05 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| H06 | 4. HAVING — filtro sobre grupos | ☐ | ☐ | ☐ | ☐ |
| O01 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| O02 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| O03 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| O04 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| O05 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| O06 | 5. ORDER BY — ordenamiento | ☐ | ☐ | ☐ | ☐ |
| J01 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J02 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J03 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J04 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J05 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J06 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| J07 | 6. JOIN — combinación de tablas | ☐ | ☐ | ☐ | ☐ |
| W01 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W02 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W03 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W04 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W05 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W06 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W07 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |
| W08 | 7. WHERE — filtros por fila | ☐ | ☐ | ☐ | ☐ |

**Total de capturas requeridas:** 47 consultas × 4 motores = **188** imágenes, más el diagrama entidad-relación del modelo.

## 11. Análisis comparativo entre motores

### 11.1 Resultados

Las consultas basadas en el estándar SQL (`SELECT`, `JOIN`, `GROUP BY`, `HAVING`, `ORDER BY` y subconsultas) devuelven resultados equivalentes en los cuatro motores. Las únicas diferencias esperables en el número de filas provienen de la sensibilidad a mayúsculas en `LIKE` (PostgreSQL y Oracle), del orden de los empates en `ORDER BY` y del formato de las fechas.

### 11.2 Sintaxis

- **Limitación de filas:** `LIMIT n` (MySQL y PostgreSQL), `SELECT TOP n` (SQL Server) y `FETCH FIRST n ROWS ONLY` (Oracle 12c o superior).
- **Concatenación:** `CONCAT()` en MySQL, `||` en PostgreSQL y Oracle, y `+` en SQL Server.
- **Fechas:** la agrupación por año y mes emplea `DATE_FORMAT`, `TO_CHAR` o `CONVERT` según el motor.
- **Alias de tabla:** Oracle no admite la palabra `AS` en alias de tabla.
- **`GROUP BY`:** PostgreSQL, SQL Server y Oracle exigen incluir todas las columnas no agregadas; MySQL es más flexible cuando se agrupa por la clave primaria.

### 11.3 Rendimiento

Con el volumen de datos utilizado (849 registros en el negocio) las diferencias de tiempo son mínimas y poco significativas. A continuación se registran los tiempos de ejecución de algunas consultas con `JOIN` de varias tablas.

| Consulta | MySQL (ms) | PostgreSQL (ms) | SQL Server (ms) | Oracle (ms) |
|---|---:|---:|---:|---:|
| J05 | | | | |
| S05 | | | | |
| G06 | | | | |
| H04 | | | | |
| J06 | | | | |

### 11.4 Observaciones generales

- El **estándar SQL** se comporta de forma equivalente en los cuatro motores.
- Las diferencias se concentran en las **funciones auxiliares**: limitación de filas, concatenación, fechas y manejo de mayúsculas.
- Oracle y SQL Server resultaron los más estrictos con la sintaxis (por ejemplo, alias de tabla sin `AS` en Oracle).
- PostgreSQL y Oracle distinguen mayúsculas en `LIKE`, por lo que conviene normalizar el texto con `LOWER()`.

## 12. Conclusiones

1. Las **situaciones de negocio** permiten comprender la utilidad de cada cláusula: `LIKE` para segmentar, `GROUP BY` y `HAVING` para indicadores, `JOIN` para integrar información dispersa y las subconsultas para comparar contra valores calculados.
2. La diferencia entre `WHERE` y `HAVING` es la distinción conceptual más relevante: el primero filtra filas antes de agrupar y el segundo filtra grupos.
3. Las construcciones `LEFT JOIN ... IS NULL` y `NOT EXISTS` resuelven problemas de ausencia (clientes sin reservas, proveedores sin servicios) que un `INNER JOIN` no puede resolver.
4. El *self join* permitió validar una regla de negocio (no solapar reservas en un mismo salón) directamente en SQL.
5. Aunque SQL es un estándar, **cada motor tiene su propio dialecto**; conocer las diferencias evita errores al migrar consultas entre sistemas.
6. Disponer de más de 200 datos ficticios hizo que los resultados fueran significativos y no triviales.

## 13. Anexos

### 13.1 Problemas frecuentes por motor

| Motor | Problema | Solución |
|---|---|---|
| MySQL | Error de conexión con Docker o WSL2 | Verifique que el contenedor está levantado y que el puerto expuesto es el configurado en DBeaver |
| MySQL | Caracteres con tilde se ven mal | Use `utf8mb4` en la base y en la conexión |
| PostgreSQL | `relation "clientes" does not exist` | Ejecute `SET search_path TO celebrahub;` en la sesión del editor |
| PostgreSQL | `LIKE` no encuentra resultados | Revise las mayúsculas o use `LOWER()` / `ILIKE` |
| SQL Server | `Invalid object name` | Seleccione la base `CelebraHub` en la conexión o ejecute `USE CelebraHub;` |
| SQL Server | `CREATE PROCEDURE` o `CREATE TRIGGER` marca error | Debe ser la única sentencia del lote; ejecútelo seleccionando solo ese bloque |
| Oracle | `ORA-00933` por usar `AS` en alias de tabla | Elimine el `AS` en alias de tablas |
| Oracle | No se puede crear tablas | Conéctese con el usuario/esquema propio, no con `SYS`; otorgue los permisos `CREATE TABLE` |
| Oracle | Los datos no aparecen desde otra sesión | Ejecute `COMMIT;` (el archivo de datos ya lo incluye al final) |
| Todos | Violación de clave única o de llave foránea al cargar datos | El archivo de datos solo se ejecuta una vez y sobre tablas con los datos semilla; si falla, vuelva a crear el esquema y ejecútelo de nuevo |

---

*Fin del informe.*
