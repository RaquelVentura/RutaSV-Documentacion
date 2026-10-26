# Guía de Arquitectura: RutaSV Backend (.NET 10 + PostgreSQL 18)

Este documento define los estándares de arquitectura de software, gestión centralizada de dependencias, configuración de arranque (`Program.cs`), búsqueda de texto completo en **PostgreSQL 18**, entorno de desarrollo local con contenedores y el flujo de Integración y Entrega Continua (CI/CD) para el backend del proyecto **RutaSV - Turismo y Experiencias**.

---

## 1. Arquitectura del Sistema: Clean Architecture + Monolito Modular

### 1.1 ¿Por qué usar Clean Architecture en nuestro equipo (5 integrantes)?

Para una aplicación empresarial en ASP.NET Core que debe entregarse en incrementos (MVPs) durante 12 semanas, **Clean Architecture (Arquitectura Limpia)** combinada con una organización por módulos de negocio (**Identidad, Catálogo, Reservas y Reseñas**) nos brinda las siguientes ventajas clave:

1. **Trabajo en paralelo sin bloqueos (Evitar conflictos en Git):** Al separar las responsabilidades en capas bien definidas y en carpetas por módulo, un desarrollador puede programar las reglas de negocio de *Reservas* mientras otro configura el almacenamiento de imágenes en *MinIO* o los endpoints de *Catálogo*, sin pisarse los mismos archivos.
2. **Alta Testabilidad (Requisito de CI con xUnit):** El núcleo del negocio (`Domain` y `Application`) no depende de PostgreSQL, ni de Entity Framework, ni de servidores de correo. Esto permite escribir pruebas unitarias con **xUnit** que se ejecutan en milisegundos dentro del pipeline de GitHub Actions sin necesidad de levantar una base de datos real.
3. **Independencia de Infraestructura:** Si en el futuro decidimos cambiar la librería de envío de correos o el proveedor de almacenamiento S3, solo modificamos la capa de `Infrastructure` sin tocar ni una sola línea de las reglas de negocio.
4. **Mantenibilidad y Orden:** Establece una estructura predecible. Cualquier miembro del equipo sabe exactamente en qué carpeta buscar una entidad, un DTO, una validación o una consulta a la base de datos.

### 1.2 ¿Cómo funciona Clean Architecture? (La Regla de Dependencia)

El principio fundamental de Clean Architecture es la **Regla de Dependencia**: *Las dependencias de código fuente solo pueden apuntar hacia adentro (hacia el centro)*. Las capas internas no saben absolutamente nada de las capas externas.

```text
+-------------------------------------------------------------------+
|                        RutaSV.Api (Presentación)                  |
|         Controladores, Endpoints, Middlewares, Program.cs         |
+---------------------------------+---------------------------------+
                                  | (Depende de Application e Infrastructure)
                                  v
+-------------------------------------------------------------------+
|                   RutaSV.Infrastructure (Datos/Externos)          |
|  EF Core, PostgreSQL 18 (Npgsql), MinIO (S3), SMTP (Correos), JWT |
+---------------------------------+---------------------------------+
                                  | (Implementa interfaces de Application)
                                  v
+-------------------------------------------------------------------+
|                  RutaSV.Application (Casos de Uso)                |
|        Servicios, DTOs, Validaciones, Interfaces (Contratos)      |
+---------------------------------+---------------------------------+
                                  | (Depende únicamente de Domain)
                                  v
+-------------------------------------------------------------------+
|                    RutaSV.Domain (Núcleo del Negocio)             |
|       Entidades, Value Objects, Enums, Excepciones de Dominio     |
+-------------------------------------------------------------------+
```

#### Detalle de cada capa:

1. **`RutaSV.Domain` (El Corazón):**
   * **Qué contiene:** Las entidades puras del negocio (ej. `Experience`, `Booking`, `Review`, `User`), enumeraciones (ej. `BookingStatus: Pending, Confirmed, Cancelled`) y reglas que nunca cambian.
   * **Dependencias:** **CERO.** No lleva paquetes NuGet de bases de datos ni de frameworks web. Es código C# puro.

2. **`RutaSV.Application` (El Orquestador):**
   * **Qué contiene:** Los casos de uso del sistema (ej. `CreateBookingService`, `SearchExperiencesQuery`), los **DTOs** (objetos de transferencia de datos para recibir y devolver JSON), las validaciones (con *FluentValidation*) y las **Interfaces** (ej. `IBookingRepository`, `IExperienceRepository`, `IFileStorageService`, `IEmailService`).
   * **Cómo funciona la Inversión de Dependencias:** Cuando `Application` necesita guardar una reserva en PostgreSQL o subir una foto a MinIO, no llama a PostgreSQL directamente; llama a la interfaz `IBookingRepository` o `IFileStorageService`.
   * **Dependencias:** Solo conoce a `RutaSV.Domain`.

3. **`RutaSV.Infrastructure` (Los Detalles Técnicos):**
   * **Qué contiene:** El código que habla con el mundo exterior. Aquí vive el `ApplicationDbContext` de **Entity Framework Core**, las migraciones e índices GIN de **PostgreSQL 18 (Npgsql)**, la implementación real que sube fotos a **MinIO**, la generación de tokens **JWT** y el cliente **SMTP** para enviar correos. Aquí se programan las clases que implementan las interfaces de `Application` (ej. `public class BookingRepository : IBookingRepository`).
   * **Dependencias:** Conoce a `RutaSV.Application` y `RutaSV.Domain`.

4. **`RutaSV.Api` (El Punto de Entrada):**
   * **Qué contiene:** Los Controladores (`Controllers`), configuración de autenticación/autorización, manejo global de errores (Middlewares), OpenAPI/Scalar y el archivo `Program.cs` donde se unen todas las piezas mediante **Inyección de Dependencias (DI)**.
   * **Dependencias:** Conoce a `Application` (para llamar a los casos de uso) y a `Infrastructure` (únicamente para registrar los servicios de base de datos y externos en el `Program.cs`).

---

## 2. Árbol de Directorios del Repositorio

A continuación se muestra la estructura completa del repositorio incluyendo los archivos de configuración global de .NET (`Directory.Build.props` y `Directory.Packages.props`), Docker y GitHub Actions:

```text
RutaSV-Backend/
├── .devcontainer/
│   └── devcontainer.json
├── .github/
│   └── workflows/
│       └── backend-ci-cd.yml             # Pipeline de CI/CD en GitHub Actions
├── docs/
│   └── Arquitectura_y_DevOps_RutaSV.md   # Este documento de referencia para el equipo
├── src/
│   ├── RutaSV.Domain/                    # Capa 1: Entidades y lógica pura
│   │   ├── Common/                       # Clases base (BaseEntity, AuditableEntity)
│   │   ├── Identity/                     # Entidades de Usuarios/Operadores
│   │   ├── Catalog/                      # Entidades de Experiencias, Fotos, Cupos
│   │   ├── Bookings/                     # Entidades de Reservas y Estados
│   │   ├── Reviews/                      # Entidades de Reseñas y Calificaciones
│   │   └── RutaSV.Domain.csproj
│   │
│   ├── RutaSV.Application/               # Capa 2: Casos de uso, DTOs e Interfaces
│   │   ├── Common/                       # Interfaces globales (IStorageService, IEmailService)
│   │   ├── Identity/                     # Login, Registro, DTOs de Auth
│   │   ├── Catalog/                      # Búsqueda, Filtros, Creación de Experiencias
│   │   ├── Bookings/                      # Lógica de reserva, validación de cupos
│   │   ├── Reviews/                      # Lógica para publicar reseñas verificadas
│   │   └── RutaSV.Application.csproj
│   │
│   ├── RutaSV.Infrastructure/            # Capa 3: PostgreSQL 18, EF Core, MinIO, SMTP
│   │   ├── Persistence/                  # DbContext, Configuraciones Fluent API, Migraciones
│   │   ├── Repositories/                 # Implementación de repositorios por módulo
│   │   ├── Storage/                      # Servicio de conexión con MinIO (S3)
│   │   ├── Email/                        # Servicio de envío de correos (MailKit/SMTP)
│   │   ├── Security/                     # Generador de JWT y Hashing de contraseñas
│   │   └── RutaSV.Infrastructure.csproj
│   │
│   └── RutaSV.Api/                       # Capa 4: Web API (ASP.NET Core)
│       ├── Controllers/                  # Endpoints divididos por módulo
│       ├── Middlewares/                  # Manejo global de excepciones
│       ├── appsettings.json              # Configuraciones base
│       ├── Program.cs                    # Inyección de dependencias, pipeline HTTP y arranque
│       └── RutaSV.Api.csproj
│
├── tests/
│   ├── RutaSV.UnitTests/                 # Pruebas unitarias con xUnit y Moq
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── RutaSV.UnitTests.csproj
│   └── RutaSV.IntegrationTests/          # Pruebas de integración de endpoints
│       └── RutaSV.IntegrationTests.csproj
│
├── .editorconfig                         # Reglas de estilo de código y Linter (IDE0005)
├── .env.example
├── .gitignore                            # Archivos ignorados por Git
├── Directory.Build.props                 # Propiedades globales de compilación (.NET 10)
├── Directory.Packages.props              # Gestión Centralizada de Paquetes NuGet (CPM)
├── Dockerfile                            # Construcción de imagen de producción (Multi-stage)
├── dotnet-tools.json
├── global.json
├── docker-compose.dev.yml                # Entorno de desarrollo (API + PostgreSQL 18 + MinIO + Mailpit)
└── RutaSV.sln                            # Solución de .NET que agrupa todos los proyectos
```

---
Nota: Es importante mencionar que cada módulo, en la capa de aplicación (y si es necesario, así se hará en otras), para no tener archivos con funciones o responsabilidades diferentes, también se organizan en base a sus responsabilidades, por ejemplo: /DTO, /Repository, /Service, /Validation, entre otros.