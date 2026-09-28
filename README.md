# Gimnasio API — Código base (Semana 5)

API REST en NestJS para el gimnasio: `Clases`, `Horarios`, `Miembros` e `Inscripciones`, cada
módulo con dominio, DTOs e infraestructura separados (patrón repositorio + inyección por token).
Los repositorios usan Prisma y MySQL.

Este proyecto es el punto de partida de la Práctica 8 (Prisma) y la Práctica 9 (Blindar la API).

## Cómo correrlo

```bash
npm install
npm run start:dev
```

El servidor levanta en `http://localhost:3000`. En `peticiones.http` está la batería completa de
pruebas (requiere la extensión "REST Client" de VS Code).

## Estructura

```
src/
  clases/        CRUD de clases del gimnasio
  horarios/      CRUD de horarios (día, hora, cupo, entrenador)
  miembros/      CRUD de miembros del gimnasio
  inscripciones/ inscribir a un miembro a un horario, con reglas de cupo y duplicados
  prisma/        cliente Prisma compartido y global
```

Cada módulo sigue la misma forma: `dominio/` (entidades + interfaz del repositorio), `dto/`,
`infra/` (repositorio Prisma) y el token de inyección en `<módulo>.tokens.ts`.

## Práctica 9 — Conexión y blindaje de la API

### Prisma

`PrismaModule` es global y proporciona `PrismaService`, que configura el adaptador
`@prisma/adapter-mariadb` a partir de `DATABASE_URL`. Los cuatro módulos conservan sus servicios
y sus contratos de repositorio: únicamente cambió el `useClass` de cada token hacia su repositorio
Prisma. **No cambió ninguna línea del service ni del controller de Clases**; la abstracción
`ClaseRepository` permite sustituir la infraestructura sin afectar las capas superiores.

Por el mismo motivo, `InscripcionesService` no cambia sus reglas de cupo y duplicados: depende de
la interfaz `InscripcionRepository`, cuyas operaciones tienen el mismo significado tanto con un
arreglo como con MySQL. El service sigue contando las confirmadas y buscando la inscripción activa
antes de guardar.

Ejecute las peticiones de `peticiones.http` en orden después de migrar y arrancar la API. Las dos
primeras inscripciones responden `201`, la tercera responde `409` por cupo lleno, el duplicado
responde `409`, los recursos ausentes responden `404` y la cancelación libera el lugar para el
último `201`.

### Validación

Los DTO de creación y actualización son clases decoradas con `class-validator`. La tubería global
usa `transform`, `whitelist`, `forbidNonWhitelisted` y `forbidUnknownValues`. `whitelist` es la
opción indispensable para el caso de campos desconocidos: `forbidNonWhitelisted` solo puede
detectar y rechazar propiedades que `whitelist` marcó como ajenas; sin ella ese campo no se elimina
ni genera aviso.

Una interfaz de TypeScript no puede validar en tiempo de ejecución porque desaparece durante la
compilación y no conserva decoradores ni metadatos. Por ejemplo, enviar
`{ "horarioId": "hola" }` a `POST /inscripciones` responde `400` con un cuerpo que incluye
`statusCode`, `message` (el arreglo de restricciones) y `error`. Enviar un campo que no pertenece
al DTO también responde `400`; `forbidNonWhitelisted` convierte ese campo en un error explícito
en vez de ignorarlo silenciosamente.

### Errores y CORS

El filtro `ErrorDominioFilter` captura la clase base `ErrorDominio`: los errores de horario o
miembro inexistente se traducen a `404`, y los de cupo lleno o inscripción duplicada a `409`.
Todas esas respuestas tienen `statusCode`, `message`, `path` y `timestamp`. Al retirar la
validación manual y el bloque `try/catch`, el método `crear` del controlador de inscripciones pasó
de 28 a 8 líneas: **20 líneas menos**. Los conflictos continúan respondiendo `409`.

CORS solo acepta `http://localhost:4200` y `http://localhost:5173`, y expone `Location` y
`X-Request-Id`. Una petición HTTP puede recibir respuesta desde ambos orígenes, pero el navegador
es quien bloquea que JavaScript lea una respuesta que no cumple CORS; CORS protege al usuario y
sus credenciales/sesión frente a sitios de otro origen, no al servidor frente a solicitudes hechas
con herramientas como REST Client o curl.

## Prisma y MySQL (Práctica 8)

La configuración de Prisma está en `prisma/schema.prisma` y toma la URL de conexión desde
`DATABASE_URL`. Copia `.env.example` a `.env` y reemplaza `TU_CONTRASENA` con la contraseña local
antes de aplicar migraciones. El archivo `.env` queda excluido por Git.

```bash
npm install
cp .env.example .env
npm run prisma:generate
npm run prisma:migrate -- --name inicial_clase
npm run prisma:migrate -- --name agregar_descripcion_clase
npm run prisma:migrate -- --name agregar_horario
npm run prisma:migrate -- --name agregar_miembro
npm run prisma:migrate -- --name agregar_inscripcion
```

Las cinco migraciones versionadas están incluidas en `prisma/migrations`: crean `clases`, agregan
su descripción, y después crean `horarios`, `miembros` e `inscripciones`.

### Respuestas de la práctica

- **¿Por qué `@prisma/adapter-mariadb` si se usa MySQL?** MariaDB implementa el protocolo de
  conexión de MySQL; el adaptador sirve para ambos motores compatibles. El proveedor y el dialecto
  que Prisma usa para el esquema siguen siendo `mysql`.
- **¿Cambiar `schema.prisma` altera la base antes de migrar?** No. El archivo es solo la
  descripción declarativa hasta ejecutar `prisma migrate dev` (o desplegar una migración).
- **¿Las migraciones son una foto o un historial?** Son un historial ordenado de cambios. Cada
  directorio conserva el SQL necesario para evolucionar desde el estado anterior.
- **¿Por qué `Horario.clase` crea columna y `Clase.horarios` no?** La relación que declara
  `fields: [claseId]` es el lado que posee la llave foránea. `Clase.horarios` solo expresa la
  navegación inversa de uno a muchos y no necesita una columna adicional.
- **¿De dónde sale la relación muchos-a-muchos entre `Miembro` y `Horario`?** Se deriva de
  `Inscripcion`: cada fila enlaza un miembro con un horario. La restricción única
  `(horarioId, miembroId)` evita repetir el mismo vínculo.
