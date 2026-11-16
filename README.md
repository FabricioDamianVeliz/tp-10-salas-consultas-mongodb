# Trabajo Práctico 10 - Consultas de reservas paso a paso

## 1. Origen del Proyecto y Comparación con el TP 09

- **Origen de la copia**: Este proyecto toma como punto de partida una copia funcional del Trabajo Práctico 09 (`tp-09-salas-crud-mongodb`), renombrado a `tp-10-salas-consultas-mongodb`.
- **Comparación con el TP 09**: En el TP 09 se completó el CRUD (POST, GET, PUT, PATCH, DELETE). En este TP 10, se agregan consultas avanzadas al listado de la API (GET `/api/reservas`) incorporando filtrado exacto, ordenamiento, paginación y proyección de campos, sin modificar el esquema de documentos ni alterar el funcionamiento del HTML y `/estado`[cite: 5].

## 2. Entorno y Versiones

- **Base de datos**: MongoDB Atlas (Clúster Free M0), base de datos asignada: `reservas_salas_u10`, colección: `reservas`[cite: 5].
- **Dependencias principales**:
  - `express` (^4.19.2)
  - `mongoose` (^8.0.0)
  - `ejs` (^3.1.10)
  - `express-ejs-layouts` (^2.5.1)
  - `morgan` (^1.10.0)

## 3. Preparación de Atlas, Entorno y DNS

1. Clonar el repositorio y ejecutar `npm ci` para instalar las dependencias manteniendo estrictamente el lockfile[cite: 5].
2. Copiar el archivo `.env.example` y renombrarlo a `.env`. Configurar las variables `PORT`, `NODE_ENV` y `MONGODB_URI` apuntando a la base de datos `reservas_salas_u10`[cite: 5].
3. **Configuración DNS**: Se conserva la configuración de resolutores públicos de Google y Cloudflare en la conexión para evitar errores `ENOTFOUND` con Atlas[cite: 5].

## 4. Comandos de Verificación e Inicio

Antes de iniciar el servidor, ejecutar la verificación de calidad y conexión[cite: 5]:

```bash
npm run check
node --env-file=.env scripts/comprobar-conexion.js
node --env-file=.env src/index.js
```

## 5. Guía de Postman y Endpoints (Consultas Avanzadas)

La API puede probarse utilizando la colección de Postman "TP 10 - Reservas". Las peticiones GET ahora soportan parámetros por query string (Params) para consultas avanzadas. En las peticiones `POST`, `PUT` y `PATCH` se debe usar la pestaña Body, opción raw, formato JSON y comprobar el header `Content-Type: application/json`.

### Endpoints Disponibles

- **GET `/api/reservas`**: Listado general con soporte de consultas. Retorna un objeto con paginación en lugar de un arreglo plano. Admite combinaciones de:
  - `sala` (ej. `?sala=Sala Norte`): Filtrado exacto.
  - `orden` (ej. `?orden=personas-asc`): `estudiante`, `personas-asc` o `personas-desc`.
  - `pagina` y `limite` (ej. `?pagina=1&limite=2`): Paginación numérica.
- **GET `/api/reservas/:id`**: Consulta el detalle de una reserva por ID (Mantiene su contrato original).
- **POST, PUT, PATCH, DELETE**: Mantienen el contrato y límites del TP 09 para el CRUD individual.
- **GET `/estado`**: Retorna metadatos del servicio y la cantidad global de registros (total de base de datos).

### Estructura de Respuesta del Listado (GET con Paginación)

```json
{
  "datos": [
    {
      "id": "654a1b...",
      "estudiante": "Ana Ejemplo",
      "email": "ana@example.com",
      "sala": "Sala Norte",
      "fecha": "2026-10-28",
      "turno": "Mañana",
      "personas": 2,
      "estado": "pendiente"
    }
  ],
  "pagina": 1,
  "limite": 2,
  "total": 6,
  "paginas": 3
}
```

## 6. Arquitectura de Consultas (Filtros, Orden, Páginas y Select)

- **Filtros (Paso 1):** El filtro por sala evalúa igualdad exacta. Si se envía una sala inexistente, retorna `200` con datos vacío. Parámetros repetidos o desconocidos arrojan `400`.
- **Ordenamiento y Desempate (Paso 2):** El orden predeterminado es estudiante. Las opciones `personas-asc` y `personas-desc` incluyen internamente un desempate por `_id` ascendente para evitar inconsistencias en registros con igual valor.
- **Paginación Estricta (Paso 3):** Los parámetros de paginación exigen números enteros (`pagina` de 1 a 1000, `limite` de 1 a 20). Si se supera el límite de páginas, retorna `200` con datos vacío (no fuerza la última página ni arroja `404`).
- **Proyección de Campos (Paso 4):** Las consultas implementan `select` para obtener exclusivamente los campos públicos requeridos (estudiante, email, sala, fecha, turno, personas, estado) y el `_id` para mapear la salida, ignorando datos internos.

## 7. Reflexión de Diseño: Modelado de Salas

**Justificación conceptual:** Si incorporáramos información detallada de las salas (como capacidad máxima, proyector, ubicación física), la sala pasaría a ser una entidad independiente y la reserva sería una transacción que hace referencia a ella.

- **Datos de la sala:** Nombre, capacidad, equipamiento (deberían estar en una colección `salas`).
- **Datos de la reserva:** Estudiante, fecha, turno, cantidad de personas, estado (deberían mantener la colección `reservas`).

Tendría sentido utilizar **referencias** (vinculando el ID de la sala en el documento de la reserva) porque los atributos físicos de una sala cambian rara vez, mientras que las reservas son dinámicas. Embeber los datos completos de la sala dentro de cada reserva generaría redundancia de datos innecesaria.

## 8. Matriz de Pruebas: Datos Iniciales y Consultas

Para validar el funcionamiento, se cargaron inicialmente 6 reservas de ejercicio mediante POST. El total global de `/estado` arroja 6.

**Casos comprobados mediante Query Params:**

1. **Sin Params:** Retorna `datos` con 2 reservas, `pagina: 1`, `limite: 2`, `total: 6`, `paginas: 3`.
2. **`?sala=Sala Norte&pagina=3&limite=2`:** Retorna `datos` vacío `[]`, `total: 4`, `paginas: 2`, estado `200`.
3. **`?sala=Sala inexistente`:** Retorna `datos` vacío `[]`, `total: 0`, `paginas: 0`, estado `200`.
4. **Validaciones `400 Bad Request`:** Se comprobó el rechazo exitoso al enviar `pagina=0`, `pagina=1.5`, `limite=21`, dos filas de página activas, orden desconocido o intentar inyectar selecciones de campo desde el cliente.
