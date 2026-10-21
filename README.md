# Fase-2---Implementaci-n-Sembrado-de-Datos-y-Consultas-CRUD
from pathlib import Path

readme = r'''# SportBooking API — Fase 2: Implementación, sembrado de datos y consultas CRUD

> **Estado de la entrega:** este repositorio contiene la documentación y los scripts para implementar la base de datos. La ejecución real en MongoDB Atlas, las capturas y la publicación en GitHub deben completarse y verificarse antes de entregar.

## 1. Descripción del proyecto

**SportBooking API** es un sistema de gestión y reserva de complejos deportivos. Busca centralizar la publicación de complejos y canchas, la consulta de disponibilidad, la administración de tarifas y la gestión de reservas.

### Problema que resuelve

La administración de turnos deportivos mediante mensajes y planillas puede generar reservas duplicadas, falta de información actualizada sobre las canchas y dificultades para controlar señas, pagos y cancelaciones.

### Rol de MongoDB

MongoDB almacena los datos en documentos BSON, lo que permite representar objetos anidados y arreglos de forma natural. Para este proyecto se utilizan tres colecciones principales:

- `users`: clientes y administradores.
- `complexes`: complejos deportivos, ubicación, horarios, amenidades y canchas.
- `reservations`: reservas, horarios, precios, estado y datos históricos del momento de la reserva.

La base de datos se llama `sportbooking`. Si la consigna o el docente exige el nombre `sports_booking_db`, hay que usar ese nombre de manera consistente en los comandos, el script y este README.

## 2. Estructura del repositorio

La estructura esperada es:

```text
sportbooking/
├── README.md
└── scripts/
    ├── fase2_mql.js
    └── data/
        ├── users.json
        ├── complexes.json
        └── reservations.json
```

Los archivos JSON contienen datos ficticios para pruebas académicas. No deben utilizarse contraseñas reales ni datos personales reales.

## 3. Modelo de datos

### 3.1. Colección `users`

Almacena información de los usuarios:

- `first_name`, `last_name`, `email`, `password_hash`, `phone`
- `role`: `CUSTOMER`, `COMPLEX_ADMIN` o `SYSTEM_ADMIN`
- `status`: `ACTIVE`, `INACTIVE` o `SUSPENDED`
- `preferences.favorite_sports`: arreglo de deportes preferidos
- `preferences.notifications_enabled`: preferencia de notificaciones
- `created_at`, `updated_at`

Algunos documentos de prueba pueden incluir `loyalty_points`, `vip_tier` o `admin_metadata`, según el tipo de usuario.

### 3.2. Colección `complexes`

Contiene:

- `name`, `slug`, `owner_id`, `contact`
- `location`: dirección, ciudad, país y coordenadas GeoJSON
- `amenities`: arreglo de servicios disponibles
- `opening_hours`: arreglo de horarios por día
- `fields`: arreglo de canchas con deporte, superficie, iluminación, cubierta, precio y estado
- `status`, `created_at`

Algunas canchas pueden incluir propiedades adicionales como `dimensions`, cuando corresponda a su disciplina.

### 3.3. Colección `reservations`

Contiene:

- `user_id` y `complex_id`: referencias a los documentos relacionados
- `field_id`: identificador de la cancha dentro del complejo
- `customer_snapshot` y `complex_snapshot`: copias de datos relevantes al momento de reservar
- `schedule`: fecha, hora de inicio, hora de fin y duración
- `pricing`: importe total, seña y moneda
- `status`: estado de la reserva
- `payment_details`: información opcional del pago
- `created_at`, `updated_at`

Los estados utilizados en el modelo son `PENDING_PAYMENT`, `CONFIRMED`, `CANCELLED`, `COMPLETED` y `NO_SHOW`.

## 4. Fundamentación del modelo documental

### Documentos embebidos

Las canchas se almacenan dentro de `complexes.fields`, ya que pertenecen a un complejo y normalmente se consultan junto con sus datos. Los horarios, la ubicación y las amenidades también se embeben porque describen al complejo.

En `reservations`, los objetos `customer_snapshot` y `complex_snapshot` conservan información histórica. Así, una modificación posterior del teléfono del cliente o del nombre de una cancha no cambia los datos guardados en una reserva anterior.

### Referencias

Las reservas mantienen `user_id` y `complex_id` como referencias. No se guardan todas las reservas dentro del documento del complejo porque su cantidad podría crecer continuamente.

## 5. Creación de la base de datos en MongoDB Atlas

1. Crear un clúster gratuito en MongoDB Atlas.
2. Crear un usuario de base de datos y guardar la contraseña de forma segura.
3. En **Network Access**, autorizar la IP pública actual desde la que se realizará la conexión.
4. En **Database → Clusters → Connect**, obtener la cadena de conexión.
5. No publicar la contraseña ni subir la cadena de conexión con credenciales a GitHub.

MongoDB crea la base y las colecciones cuando se insertan datos, salvo que se creen explícitamente antes. Para esta práctica, se recomienda crearlas explícitamente desde `mongosh` o MongoDB Compass.

Conectarse al clúster con `mongosh` utilizando la cadena de conexión proporcionada por Atlas. Una vez conectado, ejecutar:

```javascript
use("sportbooking");

db.createCollection("users");
db.createCollection("complexes");
db.createCollection("reservations");

show collections;
```

Si las colecciones ya existen, no volver a ejecutar `createCollection` sin verificarlo, porque MongoDB informará que ya existen.

## 6. Carga inicial de datos (seeding)

Cada colección principal debe quedar con **al menos 10 documentos**. Los archivos de datos se encuentran en `scripts/data/`.

### Opción A: importar con `mongoimport`

Instalar MongoDB Database Tools si el comando `mongoimport` no está disponible. Desde la raíz del repositorio, importar cada archivo. La cadena de conexión debe ser la del clúster de Atlas y no debe compartirse públicamente.

En PowerShell, definir la cadena de conexión en una variable de entorno para evitar repetirla en los comandos:

```powershell
$env:MONGODB_URI = "PEGAR_AQUI_LA_CADENA_DE_CONEXION_DE_ATLAS"
```

Después, ejecutar:

```powershell
mongoimport --uri "$env:MONGODB_URI" --db sportbooking --collection users --file .\scripts\data\users.json --jsonArray
mongoimport --uri "$env:MONGODB_URI" --db sportbooking --collection complexes --file .\scripts\data\complexes.json --jsonArray
mongoimport --uri "$env:MONGODB_URI" --db sportbooking --collection reservations --file .\scripts\data\reservations.json --jsonArray
```

Los archivos deben estar en formato JSON válido y, si contienen varios documentos en un único arreglo, se debe usar `--jsonArray`. Si se vuelve a importar el mismo conjunto, pueden producirse duplicados o errores por índices únicos; revisar la colección antes de repetir la carga.

### Opción B: importar desde MongoDB Compass

1. Conectarse al clúster.
2. Abrir la base `sportbooking`.
3. Entrar en cada colección y seleccionar **Add Data → Import JSON or CSV File**.
4. Seleccionar el JSON correspondiente y elegir formato JSON.
5. Repetir para `users`, `complexes` y `reservations`.

### Verificar cantidades

Después de importar los datos, ejecutar:

```javascript
use("sportbooking");

db.users.countDocuments();
db.complexes.countDocuments();
db.reservations.countDocuments();
```

Cada resultado debe ser igual o superior a `10`. Si se importaron exactamente los archivos de prueba una sola vez, se espera contar 10 documentos en cada colección.

## 7. Variaciones justificadas del esquema (schema-less)

Los documentos pueden tener campos opcionales diferentes sin que todos deban compartir exactamente la misma estructura. Para demostrarlo, los datos de prueba incluyen estas variaciones:

1. **Fidelización en usuarios:** algunos usuarios pueden tener `loyalty_points` o `vip_tier`. Son datos útiles para beneficios y promociones, pero no son necesarios para todos los clientes.
2. **Metadatos de administradores:** los usuarios con rol `COMPLEX_ADMIN` pueden tener `admin_metadata`, con información administrativa que no corresponde a un cliente común.
3. **Atributos particulares de las canchas:** algunas canchas pueden incluir `dimensions`, por ejemplo medidas de una cancha de fútbol. Ese atributo no necesariamente aplica a pádel o tenis.

Estas variaciones son válidas cuando responden a necesidades del dominio. No significan que deban omitirse las validaciones importantes de datos.

## 8. Consultas MQL de lectura (Read)

Ejecutar las consultas en `mongosh` después de importar los datos. Las consultas se deben revisar contra los valores reales de los JSON si se modificó el dataset.

### Consulta 1 — Coincidencia exacta

**Caso de negocio:** listar los usuarios clientes que están activos para campañas informativas.

```javascript
db.users.find({
  role: "CUSTOMER",
  status: "ACTIVE"
});
```

**Qué demuestra:** filtrado mediante coincidencia exacta en más de un campo.

### Consulta 2 — Operador de comparación (`$lte`)

**Caso de negocio:** encontrar canchas con precio por hora de hasta ARS 22.000.

```javascript
db.complexes.find({
  fields: {
    $elemMatch: {
      "price_per_hour.amount": { $lte: 22000 },
      "price_per_hour.currency": "ARS"
    }
  }
});
```

**Qué demuestra:** uso de `$lte` para comparar valores numéricos y `$elemMatch` para exigir que las condiciones se cumplan en el mismo elemento del arreglo `fields`.

### Consulta 3 — Notación de punto (campo anidado)

**Caso de negocio:** encontrar complejos ubicados en Mendoza.

```javascript
db.complexes.find({
  "location.city": "Mendoza"
});
```

**Qué demuestra:** acceso a un campo dentro del objeto anidado `location`.

### Consulta 4 — Proyección excluyendo `_id`

**Caso de negocio:** obtener un reporte compacto de las reservas confirmadas con nombre del cliente, fecha y hora, sin mostrar el identificador interno.

```javascript
db.reservations.find(
  { status: "CONFIRMED" },
  {
    _id: 0,
    "customer_snapshot.full_name": 1,
    "schedule.date": 1,
    "schedule.start_time": 1,
    status: 1
  }
);
```

**Qué demuestra:** selección de campos de salida y exclusión explícita de `_id`. Cuando se incluyen campos específicos, `_id` se incluye por defecto salvo que se lo excluya expresamente.

### Consulta 5 — Filtro de arreglos (`$all`)

**Caso de negocio:** buscar usuarios cuyas preferencias incluyan tanto fútbol 5 como pádel.

```javascript
db.users.find({
  "preferences.favorite_sports": {
    $all: ["FOOTBALL_5", "PADEL"]
  }
});
```

**Qué demuestra:** `$all` exige que el arreglo contenga todos los valores indicados, aunque pueda contener otros adicionales.

## 9. Operaciones de escritura (Update y Delete)

> **Precaución:** ejecutar las operaciones de escritura solamente después de comprobar que se está conectado a la base de pruebas `sportbooking`. Las operaciones siguientes modifican o eliminan datos.

### Escritura 1 — Actualizar campos y agregar una propiedad (`$set`)

**Caso de negocio:** cancelar una reserva pendiente de pago y guardar el motivo de cancelación.

```javascript
db.reservations.updateOne(
  {
    status: "PENDING_PAYMENT",
    "customer_snapshot.full_name": "Mariana Ruiz"
  },
  {
    $set: {
      status: "CANCELLED",
      cancellation_reason: "Falta de pago dentro del plazo",
      updated_at: new Date()
    }
  }
);
```

**Qué demuestra:** `$set` modifica campos existentes y crea `cancellation_reason` si no existía. Verificar previamente que haya una reserva que coincida con el filtro; si `matchedCount` es `0`, no se modificó ningún documento.

### Escritura 2 — Incrementar un contador (`$inc`)

**Caso de negocio:** sumar 50 puntos de fidelización a una cliente después de completar una reserva.

```javascript
db.users.updateOne(
  { email: "carla.perez@example.com" },
  {
    $inc: { loyalty_points: 50 },
    $set: { updated_at: new Date() }
  }
);
```

**Qué demuestra:** `$inc` incrementa un valor numérico de forma atómica. Si `loyalty_points` no existe, MongoDB lo crea con el incremento aplicado. Confirmar que el correo exista en el dataset antes de ejecutar.

### Escritura 3 — Eliminación segura de un documento (`deleteOne`)

**Caso de negocio:** retirar del dataset una reserva de prueba cancelada que ya no se necesita.

```javascript
db.reservations.deleteOne({
  _id: ObjectId("66a000000000000000000204"),
  status: "CANCELLED"
});
```

**Qué demuestra:** `deleteOne` elimina como máximo un documento y el filtro combina un identificador exacto con un estado. El `_id` debe corresponder a una reserva de prueba existente. Si el identificador no existe en el JSON importado, la operación no eliminará documentos; reemplazarlo por el `_id` real de una reserva de prueba cancelada, comprobándolo antes con `find`.

Para comprobar el resultado antes de borrar, ejecutar:

```javascript
db.reservations.find({
  _id: ObjectId("66a000000000000000000204"),
  status: "CANCELLED"
});
```

No usar `deleteMany({ status: "CANCELLED" })` sobre datos que se quieran conservar: podría borrar todas las reservas canceladas de la colección.

## 10. Verificación de resultados

Después de ejecutar las consultas y operaciones, guardar evidencia de resultados reales. Para verificar la operación de actualización, se pueden consultar los documentos con los mismos filtros usados en el `updateOne`.

```javascript
db.users.find({ role: "CUSTOMER", status: "ACTIVE" });
db.complexes.find({ "location.city": "Mendoza" });
db.reservations.find(
  { status: "CONFIRMED" },
  { _id: 0, "customer_snapshot.full_name": 1, "schedule.date": 1, "schedule.start_time": 1, status: 1 }
);
```

También se pueden comprobar las cantidades finales:

```javascript
db.users.countDocuments();
db.complexes.countDocuments();
db.reservations.countDocuments();
```

Las cantidades de reservas pueden cambiar después de ejecutar la eliminación. Registrar las cantidades antes y después si se necesita demostrar el efecto de `deleteOne`.

## 11. Capturas de pantalla para la entrega

Guardar capturas propias en `docs/capturas/` o en la ubicación indicada por la consigna. Se recomienda incluir:

1. El clúster de Atlas creado y conectado, sin mostrar credenciales.
2. Las colecciones `users`, `complexes` y `reservations`.
3. El conteo de documentos de las tres colecciones.
4. La ejecución y resultados de las cinco consultas de lectura.
5. El resultado de `updateOne` y la comprobación del documento actualizado.
6. El resultado de `$inc` y el valor actualizado de `loyalty_points`.
7. La comprobación previa y el resultado de `deleteOne`.

Las capturas deben mostrar resultados obtenidos realmente en MongoDB Compass o `mongosh`. No usar capturas simuladas ni marcar como ejecutada una operación que todavía no se haya probado.

## 12. Publicación en GitHub

Desde la terminal, en la carpeta raíz del proyecto:

```bash
git status
git add README.md scripts/
git commit -m "Completa Fase 2 de MongoDB: datos y consultas CRUD"
git push
```

Si el repositorio remoto todavía no está configurado, crear primero el repositorio en GitHub y seguir las instrucciones que GitHub indique para vincularlo. No subir contraseñas, URI de Atlas con credenciales ni archivos `.env`.

## 13. Lista final de control

- [ ] Clúster de MongoDB Atlas creado y accesible.
- [ ] Base `sportbooking` creada.
- [ ] Colecciones `users`, `complexes` y `reservations` creadas.
- [ ] Al menos 10 documentos cargados en cada colección.
- [ ] Variaciones del esquema documentadas y justificadas.
- [ ] Cinco consultas de lectura ejecutadas y verificadas.
- [ ] Tres operaciones de escritura ejecutadas y verificadas.
- [ ] Capturas reales guardadas.
- [ ] `README.md`, JSON y script MQL subidos al repositorio.
- [ ] No hay credenciales ni secretos publicados.

---

**Nota académica:** este README documenta el diseño y los pasos de ejecución. La entrega queda verificada cuando se completen los pasos pendientes y se incorporen las capturas reales de la base de datos funcionando.
'''
out = Path("/mnt/data/README_SportBooking_Fase2.md")
out.write_text(readme, encoding="utf-8")
print(f"Creado: {out} ({len(readme)} caracteres)")
