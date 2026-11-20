# CRM Backend — Muñoz & Puente

API REST de un **CRM (Customer Relationship Management) de nivel intermedio, inspirado en HubSpot**, desarrollada con **Spring Boot**, **Spring Data JPA** y **PostgreSQL**.

**Integrantes:** Esteban Muñoz · Kevin Puente

---

## 1. ¿De qué trata el proyecto?

Un CRM es el sistema que usa un equipo comercial para registrar **con quién habla** (contactos y empresas), **qué está intentando vender** (negocios) y **qué tiene que hacer después** (tareas y notas).

Este backend replica el núcleo de HubSpot CRM:

- **Contactos y empresas:** registro de personas y organizaciones, con un vendedor responsable (propietario).
- **Ciclo de vida del contacto:** cada contacto avanza por etapas: `SUSCRIPTOR → LEAD → LEAD_CALIFICADO → OPORTUNIDAD → CLIENTE`.
- **Pipeline de ventas:** los negocios (deals) se mueven por etapas, desde `CALIFICACION` hasta `GANADO` o `PERDIDO`, y se puede consultar cuántos hay en cada etapa.
- **Tareas:** llamadas, emails, reuniones y seguimientos asignados a un vendedor, con prioridad y fecha de vencimiento (incluye consulta de tareas vencidas).
- **Notas:** historial de anotaciones sobre un contacto o un negocio.
- **Usuarios:** administradores y vendedores que son dueños de los registros.

---

## 2. Tecnologías

| Herramienta | Uso |
|---|---|
| Java 27 | Lenguaje |
| Spring Boot 4.1.2 (Web MVC) | API REST |
| Spring Data JPA / Hibernate | Persistencia y ORM |
| PostgreSQL | Base de datos relacional |
| Lombok | Getters, setters y constructores (`@Data`) |
| Maven (wrapper `mvnw`) | Build y dependencias |

---

## 3. Modelo de datos

```text
Usuario  1 ──< N  Empresa     (propietario)
Usuario  1 ──< N  Contacto    (propietario)
Usuario  1 ──< N  Negocio     (propietario)
Usuario  1 ──< N  Tarea       (asignadoA)
Usuario  1 ──< N  Nota        (autor)

Empresa  1 ──< N  Contacto
Empresa  1 ──< N  Negocio
Contacto 1 ──< N  Negocio
Contacto 1 ──< N  Tarea / Nota
Negocio  1 ──< N  Tarea / Nota
```

| Entidad | Tabla | Campos principales | Relaciones (N:1) |
|---|---|---|---|
| `Usuario` | `usuarios` | nombre, email (único), rol, activo | — |
| `Empresa` | `empresas` | nombre, dominio, industria, teléfono, ciudad, país, numeroEmpleados | propietario → `Usuario` |
| `Contacto` | `contactos` | nombre, apellido, email (único), teléfono, cargo, estado | empresa → `Empresa`, propietario → `Usuario` |
| `Negocio` | `negocios` | nombre, monto, etapa, fechaCierreEstimada | contacto, empresa, propietario |
| `Tarea` | `tareas` | título, descripción, tipo, estado, prioridad, fechaVencimiento | asignadoA → `Usuario`, contacto, negocio |
| `Nota` | `notas` | contenido | autor → `Usuario`, contacto, negocio |

Todas las entidades tienen `id`, `fechaCreacion` y `fechaActualizacion` (automáticas).

### Enumeraciones

| Enum | Valores |
|---|---|
| `Rol` | `ADMIN`, `VENDEDOR` |
| `EstadoContacto` | `SUSCRIPTOR`, `LEAD`, `LEAD_CALIFICADO`, `OPORTUNIDAD`, `CLIENTE`, `INACTIVO` |
| `EtapaNegocio` | `CALIFICACION`, `REUNION_AGENDADA`, `PROPUESTA_ENVIADA`, `NEGOCIACION`, `GANADO`, `PERDIDO` |
| `TipoTarea` | `LLAMADA`, `EMAIL`, `REUNION`, `SEGUIMIENTO` |
| `EstadoTarea` | `PENDIENTE`, `EN_PROGRESO`, `COMPLETADA` |
| `Prioridad` | `BAJA`, `MEDIA`, `ALTA` |

---

## 4. Estructura del proyecto

Arquitectura por capas: **Controller → Service → Repository → Model**.

```text
ProyectoBackEnd-munoz-puente/
├── pom.xml
├── mvnw / mvnw.cmd
├── README.md
└── src/main/
    ├── resources/
    │   └── application.properties        # Conexión a PostgreSQL y puerto
    └── java/com/munozpuente/crm/
        ├── CrmApplication.java           # Punto de entrada
        ├── model/                        # Entidades JPA
        │   ├── Usuario.java
        │   ├── Empresa.java
        │   ├── Contacto.java
        │   ├── Negocio.java
        │   ├── Tarea.java
        │   ├── Nota.java
        │   └── enums/
        │       ├── Rol.java
        │       ├── EstadoContacto.java
        │       ├── EtapaNegocio.java
        │       ├── TipoTarea.java
        │       ├── EstadoTarea.java
        │       └── Prioridad.java
        ├── repository/                   # Acceso a datos (JpaRepository)
        │   ├── UsuarioRepository.java
        │   ├── EmpresaRepository.java
        │   ├── ContactoRepository.java
        │   ├── NegocioRepository.java
        │   ├── TareaRepository.java
        │   └── NotaRepository.java
        ├── service/                      # Lógica de negocio
        │   ├── UsuarioService.java
        │   ├── EmpresaService.java
        │   ├── ContactoService.java
        │   ├── NegocioService.java
        │   ├── TareaService.java
        │   └── NotaService.java
        └── controller/                   # Endpoints REST
            ├── UsuarioController.java
            ├── EmpresaController.java
            ├── ContactoController.java
            ├── NegocioController.java
            ├── TareaController.java
            └── NotaController.java
```

| Capa | Responsabilidad |
|---|---|
| `model` | Define las tablas y relaciones con anotaciones JPA. |
| `repository` | Interfaces que extienden `JpaRepository`, con consultas derivadas (`findByEtapa`, `findByEmpresaId`...). |
| `service` | Reglas de negocio: CRUD, cambio de etapa, resumen del pipeline, tareas vencidas. |
| `controller` | Expone la API REST y devuelve los códigos HTTP adecuados (200, 201, 204, 404). |

---

## 5. Endpoints

Puerto por defecto: `9090`. Todas las entidades tienen CRUD completo:

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/api/{recurso}` | Listar todos |
| `GET` | `/api/{recurso}/{id}` | Obtener uno (404 si no existe) |
| `POST` | `/api/{recurso}` | Crear (201) |
| `PUT` | `/api/{recurso}/{id}` | Editar (404 si no existe) |
| `DELETE` | `/api/{recurso}/{id}` | Eliminar (204 / 404) |

Recursos: `usuarios`, `empresas`, `contactos`, `negocios`, `tareas`, `notas`.

### Endpoints especiales

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/api/usuarios/activos` | Usuarios activos |
| `GET` | `/api/empresas/buscar?nombre=acme` | Buscar empresas por nombre |
| `GET` | `/api/contactos/estado/{estado}` | Contactos en una etapa del ciclo de vida |
| `GET` | `/api/contactos/empresa/{empresaId}` | Contactos de una empresa |
| `GET` | `/api/negocios/etapa/{etapa}` | Negocios en una etapa del pipeline |
| `PATCH` | `/api/negocios/{id}/etapa/{etapa}` | Mover un negocio a otra etapa |
| `GET` | `/api/negocios/pipeline` | Cantidad de negocios por etapa |
| `GET` | `/api/tareas/usuario/{usuarioId}` | Tareas asignadas a un usuario |
| `GET` | `/api/tareas/vencidas` | Tareas vencidas sin completar |
| `GET` | `/api/notas/contacto/{contactoId}` | Historial de notas de un contacto |
| `GET` | `/api/notas/negocio/{negocioId}` | Historial de notas de un negocio |

Ejemplo para crear un negocio:

```json
POST /api/negocios
{
  "nombre": "Licencias anuales Acme",
  "monto": 4500.00,
  "etapa": "CALIFICACION",
  "fechaCierreEstimada": "2026-12-15",
  "contacto":    { "id": 1 },
  "empresa":     { "id": 1 },
  "propietario": { "id": 1 }
}
```

---

## 6. Cómo ejecutarlo

1. Crear la base de datos en PostgreSQL:

   ```sql
   CREATE DATABASE crm;
   ```

2. Definir las credenciales como variables de entorno (no se guardan en el repositorio):

   ```bash
   export DB_URL=jdbc:postgresql://localhost:5432/crm
   export DB_USER=tu_usuario
   export DB_PASSWORD=tu_clave
   ```

3. Levantar la aplicación (Hibernate crea las tablas automáticamente):

   ```bash
   ./mvnw spring-boot:run
   ```

4. Probar en `http://localhost:9090/api/contactos`.

---

## 7. Próximos pasos

- [ ] DTOs y validaciones (`@Valid`, `@NotBlank`, `@Email`).
- [ ] Manejo global de errores con `@RestControllerAdvice`.
- [ ] Autenticación con Spring Security + JWT (roles `ADMIN` / `VENDEDOR`).
- [ ] Paginación y filtros en los listados.
- [ ] Script SQL con datos de prueba y colección de Postman.
- [ ] Tests unitarios y de integración.
