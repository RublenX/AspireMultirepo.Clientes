# AspireMultirepo.Clientes

Microservicio de **gestión de Clientes**, implementado como prueba de concepto para validar la orquestación de **.NET Aspire en modo multirepo**. Forma parte de una solución compuesta por cuatro repositorios independientes: este microservicio, [AspireMultirepo.Pedidos](https://github.com/RublenX/AspireMultirepo.Pedidos), [AspireMultirepo.Orchestrator](https://github.com/RublenX/AspireMultirepo.Orchestrator) (AppHost de Aspire) y [AspireMultirepo.Portal](https://github.com/RublenX/AspireMultirepo.Portal) (frontend React).

> Este repositorio no se ejecuta de forma aislada en el flujo normal: es orquestado por `AspireMultirepo.Orchestrator`, que además de este microservicio levanta PostgreSQL, RabbitMQ y el microservicio de Pedidos. Para ejecutar la solución completa, ver el README de ese repositorio.

## Stack técnico

- **.NET 10** / ASP.NET Core Web API
- **Entity Framework Core** + **Npgsql** (PostgreSQL)
- **RabbitMQ.Client** para publicación de eventos
- **.NET Aspire ServiceDefaults** (service discovery, resiliencia de `HttpClient`, health checks, OpenTelemetry), aportados por `MicroserviciosConAspire.ServiceDefaults` del repositorio Orchestrator
- Swagger / OpenAPI (en entorno de desarrollo)

## Estructura del repositorio

```
src/
├── ClientesApi/       Web API: controlador, mensajería (publicador de eventos) y arranque
└── ClientesData/      Acceso a datos: DbContext de EF Core, entidad Cliente, repositorio y migraciones
```

## Modelo de datos

`Cliente`: `Id`, `Nombre`, `Vip`.

## Endpoints (`api/Cliente`)

| Verbo | Ruta | Descripción |
|---|---|---|
| GET | `/api/cliente` | Lista todos los clientes |
| GET | `/api/cliente/{id}` | Obtiene un cliente por Id |
| POST | `/api/cliente` | Crea un cliente |
| PUT | `/api/cliente/{id}` | Actualiza un cliente y publica el evento `cliente.actualizado` |
| DELETE | `/api/cliente/{id}` | Elimina un cliente y publica el evento `cliente.eliminado` |

## Integración con Pedidos

Al actualizar o eliminar un cliente, `ClienteController` publica un evento en RabbitMQ a través de `RabbitMqEventPublisher`:

- **Exchange**: `clientes-exchange` (tipo *topic*)
- **Routing keys**: `cliente.actualizado`, `cliente.eliminado`
- **Payload**: `ClienteEventEnvelope` (JSON) con `EventType`, `IdCliente` y, en la actualización, `Nombre`

El microservicio de Pedidos consume estos eventos para mantener sincronizado el nombre de cliente desnormalizado en sus pedidos, y también consulta esta API vía HTTP (con *service discovery* de Aspire) al crear o actualizar un pedido.

## Configuración

La cadena de conexión a PostgreSQL se resuelve desde `ConnectionStrings:DefaultConnection` y la configuración de RabbitMQ desde la sección `RabbitMq` (`Host`, `Port`, `User`, `Password`); ambas son inyectadas como variables de entorno por el AppHost cuando el servicio se ejecuta orquestado.

