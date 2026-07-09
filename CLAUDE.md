# CLAUDE.md

Este fichero proporciona contexto a Claude Code (claude.ai/code) para trabajar con el código de este repositorio.

## Contexto

Este es el repositorio del microservicio **Clientes**, parte de un PoC de .NET Aspire en modo multirepo. La arquitectura global está documentada en `../CLAUDE.md` (carpeta padre). Este repo debe estar como carpeta hermana de `Orchestrator/` y `Pedidos/` porque `ClientesApi.csproj` tiene una `ProjectReference` a `../../../Orchestrator/src/MicroserviciosConAspire.ServiceDefaults/`.

Este servicio **no se ejecuta de forma autónoma** en el desarrollo habitual — lo lanza el AppHost de Aspire del repo `Orchestrator`, que inyecta la cadena de conexión a PostgreSQL y las credenciales de RabbitMQ como variables de entorno.

## Estructura de la solución

```
MicroservicioClientes.slnx
src/
  ClientesApi/      # ASP.NET Core Web API (.NET 10)
  ClientesData/     # Capa de datos EF Core (librería de clases, .NET 10)
```

## Comandos de compilación y ejecución

```bash
# Compilar la solución
dotnet build MicroservicioClientes.slnx

# Ejecutar de forma autónoma (requiere variables de entorno manuales; preferible usar el AppHost de Aspire)
dotnet run --project src/ClientesApi/ClientesApi.csproj
```

## Migraciones de EF Core

El proyecto de inicio para las migraciones es `ClientesApi`; el proyecto de migraciones es `ClientesData`.

```bash
# Añadir una nueva migración
dotnet ef migrations add <NombreMigración> --project src/ClientesData --startup-project src/ClientesApi

# Aplicar migraciones manualmente (se aplican automáticamente en Development al arrancar)
dotnet ef database update --project src/ClientesData --startup-project src/ClientesApi
```

Las migraciones están en `src/ClientesData/Migrations/`. En entorno `Development`, `db.Database.Migrate()` se llama automáticamente en `Program.cs` al arrancar.

## Puntos arquitectónicos clave

- **Ruta del controlador**: la clase se llama `ClienteController` (singular), por lo que `[Route("api/[controller]")]` resuelve a `api/Cliente`, no `api/Clientes`.
- **Publicador RabbitMQ** (`RabbitMqEventPublisher`): registrado como singleton; la conexión y el canal se crean de forma perezosa en la primera publicación y se reutilizan durante toda la vida de la app. El exchange se declara en ese primer uso.
- **Contrato de eventos** (`ClientesApi.Messaging.ClienteEventContract`): exchange `clientes-exchange` (topic), routing keys `cliente.actualizado` / `cliente.eliminado`, payload `ClienteEventEnvelope { EventType, IdCliente, Nombre }`. Este contrato está **duplicado** en el repo de Pedidos — ambas copias deben mantenerse sincronizadas manualmente si cambian el exchange, las routing keys o el payload.
- Los eventos solo se publican en `PUT` (actualización) y `DELETE` — no en `POST` (creación), porque Pedidos obtiene el nombre del cliente vía HTTP al crear un pedido.
- `builder.AddServiceDefaults()` y `app.MapDefaultEndpoints()` provienen del proyecto compartido `MicroserviciosConAspire.ServiceDefaults` del repo Orchestrator.
