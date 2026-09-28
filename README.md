# Gimnasio API — Código base (Semana 5)

API REST en NestJS para el gimnasio: `Clases`, `Horarios`, `Miembros` e `Inscripciones`, cada
módulo con dominio, DTOs e infraestructura separados (patrón repositorio + inyección por token).
Los datos viven en memoria — ningún repositorio se conecta todavía a una base de datos real.

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
  datos/         datos de arranque (seed) que usan Horarios y Miembros
```

Cada módulo sigue la misma forma: `dominio/` (entidades + interfaz del repositorio), `dto/`,
`infra/` (repositorio en memoria) y el token de inyección en `<módulo>.tokens.ts`.

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
