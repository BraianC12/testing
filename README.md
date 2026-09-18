# 🔨 SubastaYa - Plataforma de Subastas en Tiempo Real

**SubastaYa** es un sistema integral de subastas en línea en tiempo real desarrollado bajo una arquitectura limpia (Clean Architecture) y modular en **.NET 8** para el Backend y **React 19 + Vite** para el Frontend. Incorpora mecanismos de concurrencia optimista (*Optimistic Locking*), actualización de pujas en tiempo real vía **SignalR (WebSockets)**, sistema de billetera con transacciones auditables (*Ledger*) y reglas de negocio como *Anti-Sniping*.

---

## 📋 Tabla de Contenidos
1. [Arquitectura del Proyecto](#-arquitectura-del-proyecto)
2. [Prerrequisitos](#-prerrequisitos)
3. [Guía Paso a Paso de Instalación y Ejecución](#-guía-paso-a-paso-de-instalación-y-ejecución)
   - [1. Configuración de Base de Datos](#1-configuración-de-la-base-de-datos)
   - [2. Compilación del Proyecto](#2-compilación-del-proyecto)
   - [3. Ejecución de Migraciones y Seed Data](#3-ejecución-de-las-migraciones)
   - [4. Lanzar el Backend (API)](#4-lanzar-la-aplicación-backend)
   - [5. Lanzar el Frontend](#5-lanzar-la-aplicación-frontend)
4. [Demostración y Testing de Concurrencia Optimista](#-demostración-y-testing-de-concurrencia-optimista)
   - [¿Cómo funciona el Optimistic Locking?](#cómo-funciona-la-concurrencia-optimista-en-subastaya)
   - [Opción A: Proyecto de Consola .NET (`TestConcurrencia`)](#opción-a-ejecución-del-test-de-concurrencia-en-net)
   - [Opción B: Script Bash con `curl` en Paralelo](#opción-b-script-bash-con-curl-en-paralelo)
   - [Opción C: Prueba con Postman / JMeter](#opción-c-prueba-con-jmeter--postman-runner)
   - [Verificación en Base de Datos](#verificación-en-base-de-datos)
5. [Cuentas de Prueba (Seed Data)](#-cuentas-de-prueba)

---

## 🏗 Arquitectura del Proyecto

El repositorio está organizado siguiendo los principios de **Clean Architecture**:

- **`Domain/`**: Entidades centrales del negocio (`Subasta`, `Puja`, `Billetera`, `Usuario`, etc.) y excepciones de dominio.
- **`Application/`**: Casos de uso (Commands, Queries y Handlers), interfaces de repositorios, DTOs y lógica de negocio.
- **`Infraestructure/`**: Implementación de acceso a datos con **Entity Framework Core 8**, configuración de DbContext, mapeos y migraciones a SQL Server.
- **`SubastaYa/`**: Web API REST con controladores ASP.NET Core, Middlewares de manejo global de excepciones y Hub de SignalR.
- **`Frontend/`**: Aplicación Single Page Application (SPA) en React con Vite y conexión a SignalR.
- **`TestConcurrencia/`**: Herramienta de consola para pruebas de carga y colisión de concurrencia optimista.

---

## ⚙️ Prerrequisitos

Asegúrate de tener instaladas las siguientes herramientas en tu entorno de desarrollo:

- **[.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)** (v8.0 o superior)
- **[Node.js](https://nodejs.org/)** (v18.0 o superior) y **npm**
- **Microsoft SQL Server**:
  - Opción local: **SQL Server Express** o **LocalDB**.
  - Opción Docker: Contenedor oficial de SQL Server 2022.
- **Entity Framework Core CLI Tools** (opcional pero recomendado):
  ```bash
  dotnet tool install --global dotnet-ef
  ```

---

## 🚀 Guía Paso a Paso de Instalación y Ejecución

### 1. Configuración de la Base de Datos

Por defecto, la cadena de conexión apunta a una instancia local de SQL Server Express (`localhost\SQLEXPRESS`).

1. Abre el archivo de configuración `SubastaYa/appsettings.json`.
2. Verifica o actualiza la propiedad `ConnectionStrings:DefaultConnection`:
   ```json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=localhost\\SQLEXPRESS;Database=SubastaYaDb;Trusted_Connection=True;MultipleActiveResultSets=True;TrustServerCertificate=True"
     }
   }
   ```
   > **Tip si usas Docker**: Puedes levantar una instancia de SQL Server rápidamente con:
   > ```bash
   > docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=TuPasswordSeguro123!" -p 1433:1433 --name subastaya-sql -d mcr.microsoft.com/mssql/server:2022-latest
   > ```
   > En ese caso, actualiza la cadena a: `"Server=localhost,1433;Database=SubastaYaDb;User Id=sa;Password=TuPasswordSeguro123!;TrustServerCertificate=True"`

---

### 2. Compilación del Proyecto

Desde la raíz del repositorio, restaura los paquetes NuGet y compila la solución:

```bash
# Restaurar dependencias
dotnet restore

# Compilar la solución completa
dotnet build
```

---

### 3. Ejecución de las Migraciones

Aplica las migraciones pendientes para crear la base de datos `SubastaYaDb` junto con sus tablas, índices y datos iniciales (*Seed Data*):

```bash
dotnet ef database update --project Infraestructure --startup-project SubastaYa
```

> **Nota:** El DbContext incluye automáticamente datos semilla con usuarios precargados, billeteras con saldo disponible y subastas activas listas para probar (como la subasta con `Id: 6`).

---

### 4. Lanzar la Aplicación Backend

Inicia el servidor ASP.NET Core:

```bash
dotnet run --project SubastaYa
```

El servidor quedará a la escucha en:
- **HTTPS:** `https://localhost:7117`
- **HTTP:** `http://localhost:5100`
- **Documentación interactiva Swagger UI:** `https://localhost:7117/swagger`

---

### 5. Lanzar la Aplicación Frontend

En una nueva terminal, navega a la carpeta `Frontend`, instala las dependencias de Node.js e inicia el servidor de desarrollo:

```bash
cd Frontend
npm install
npm run dev
```

El cliente web estará disponible en:
- **URL Local:** `http://localhost:5173`

---

## 🛡 Demostración y Testing de Concurrencia Optimista

### ¿Cómo funciona la Concurrencia Optimista en SubastaYa?

1. **Token de Concurrencia (`RowVersion` / `[Timestamp]`):**
   La entidad `Subasta` posee una propiedad de versión mapeada con `IsRowVersion()` en Entity Framework:
   ```csharp
   [Timestamp]
   public byte[] Version { get; set; }
   ```
2. **Detección de Colisión:**
   Cuando dos transacciones leen la subasta en el mismo estado e intentan persistir una nueva puja simultáneamente, la primera en escribir actualiza el `RowVersion` en SQL Server. Cuando la segunda transacción intenta ejecutar el `UPDATE`, Entity Framework detecta que la versión en memoria ya no coincide con la de la base de datos y arroja una excepción `DbUpdateConcurrencyException`.
3. **Manejo y Auditoría:**
   - La capa de persistencia (`UnitOfWork`) captura `DbUpdateConcurrencyException` y la transforma en una excepción de dominio `ConflictException`.
   - `CreateBidCommandHandler` registra un evento de auditoría en la tabla `Auditorias_Log` con la acción `PUJA_RECHAZADA_CONCURRENCIA`.
   - El middleware global `ExceptionMiddleware` captura la excepción y responde al cliente con código de estado **`HTTP 409 Conflict`**.

---

### Opción A: Ejecución del Test de Concurrencia en .NET

El repositorio incluye un proyecto de prueba específico en la carpeta `TestConcurrencia/` que dispara **1000 solicitudes HTTP concurrentes** exactamente en el mismo instante utilizando `Task.WhenAll`.

#### Ejecutar la prueba:
Con el Backend en ejecución, abre otra terminal y ejecuta:

```bash
dotnet run --project TestConcurrencia
```

#### Código fuente de la prueba (`TestConcurrencia/Program.cs`):
```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;

string apiUrl = "https://localhost:7117/api/auctions/6/bids";
int usuariosSimultaneos = 1000;

Console.WriteLine($"Iniciando ataque: {usuariosSimultaneos} usuarios pujando al mismo milisegundo...");

using var httpClient = new HttpClient();
var tareas = new List<Task<HttpResponseMessage>>();
var cronometro = Stopwatch.StartNew();

for (int i = 1; i <= usuariosSimultaneos; i++)
{
    string jsonPayload = @"{
        ""subasta_Id"": 6,
        ""comprador_Id"": 2,
        ""monto"": 100000.00
    }";

    var contenido = new StringContent(jsonPayload, Encoding.UTF8, "application/json");
    tareas.Add(httpClient.PostAsync(apiUrl, contenido));
}

var resultados = await Task.WhenAll(tareas);
cronometro.Stop();

int exitosas = resultados.Count(r => r.IsSuccessStatusCode);
int fallidas = resultados.Count(r => !r.IsSuccessStatusCode);

Console.WriteLine("\n=== RESULTADOS DE CONCURRENCIA ===");
Console.WriteLine($"Tiempo total: {cronometro.ElapsedMilliseconds} ms");
Console.WriteLine($"Pujas aceptadas (HTTP 201 Created): {exitosas}");
Console.WriteLine($"Pujas rechazadas por colisión (HTTP 409 Conflict): {fallidas}");
```

#### Salida esperada en consola:
```text
Iniciando ataque: 1000 usuarios pujando al mismo milisegundo...

=== RESULTADOS DE CONCURRENCIA ===
Tiempo de respuesta del server: 312 ms
Pujas que el servidor aceptó (HTTP 200/201): 1
Pujas que el servidor rechazó: 999

✅ INCREÍBLE: Entró solo 1. Tu sistema ya está manejando la concurrencia.

--- DETALLE DEL RECHAZO ---
Código HTTP: 409 (Conflict)
Mensaje del backend: {"message":"Otro usuario realizó una puja simultáneamente. Intente pujar con el nuevo valor."}
```

---

### Opción B: Script Bash con `curl` en Paralelo

Para probar el escenario exacto de **dos peticiones de puja idénticas enviadas en el mismo milisegundo**:

```bash
#!/usr/bin/env bash

URL="https://localhost:7117/api/auctions/6/bids"
PAYLOAD='{"subasta_Id": 6, "comprador_Id": 2, "monto": 90000.00}'

echo "Enviando 2 peticiones idénticas de puja en paralelo..."

# Lanzar ambas peticiones en segundo plano simultáneamente
curl -k -s -w "\n[Petición 1] Código HTTP: %{http_code}\n" -X POST "$URL" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" &

curl -k -s -w "\n[Petición 2] Código HTTP: %{http_code}\n" -X POST "$URL" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" &

# Esperar a que ambas peticiones finalicen
wait
echo "Prueba finalizada."
```

#### Respuesta obtenida:
```json
[Petición 1]
HTTP Status: 201 Created
{
  "mensaje": "Puja creada exitosamente",
  "result": {
    "pujaId": 4,
    "subastaId": 6,
    "monto": 90000.00,
    "saldoDisponibleRestante": 15000.00
  }
}

[Petición 2]
HTTP Status: 409 Conflict
{
  "message": "Otro usuario realizó una puja simultáneamente. Intente pujar con el nuevo valor."
}
```

---

### Opción C: Prueba con JMeter / Postman Runner

#### En Postman:
1. Crea una petición `POST` a `https://localhost:7117/api/auctions/6/bids` con el header `Content-Type: application/json` y el body:
   ```json
   {
     "subasta_Id": 6,
     "comprador_Id": 3,
     "monto": 95000.00
   }
   ```
2. Abre el **Collection Runner** o la pestaña de **Performance Testing**.
3. Configura **10 Virtual Users** con un tiempo de rampa (*Ramp-up*) de `0 segundos`.
4. Observa cómo 1 solicitud obtiene `201 Created` y las 9 restantes devuelven `409 Conflict`.

#### En Apache JMeter:
- **Thread Group:** 50 hilos (threads), Ramp-up period = 0 segundos, Loop Count = 1.
- **HTTP Request:** Método `POST`, Path `/api/auctions/6/bids`, Body con el JSON de la puja.
- **View Results Tree:** Comprueba que solo la primera respuesta registrada por la base de datos es exitosa (`Response Code: 201`) y el resto son colisiones rechazadas (`Response Code: 409`).

---

### Verificación en Base de Datos

Puedes corroborar en SQL Server Management Studio (SSMS) o Azure Data Studio que solo una puja fue registrada y que las colisiones fueron auditadas:

```sql
-- 1. Verificar que únicamente existe una puja registrada para ese monto
SELECT Id, Subasta_Id, Comprador_Id, Monto, Fecha_Puja 
FROM Pujas 
WHERE Subasta_Id = 6;

-- 2. Consultar el log de auditoría de colisiones por concurrencia
SELECT Id, Entidad, Entidad_Id, Accion, Detalle, Fecha, Usuario_Id
FROM Auditorias_Log
WHERE Accion = 'PUJA_RECHAZADA_CONCURRENCIA';
```

---

## 👥 Cuentas de Prueba

El sistema incluye los siguientes usuarios precargados en el seed de datos (contraseña de todos los usuarios: `123456`):

| ID | Nombre | Email | Rol / Estado | Saldo Inicial |
|:---|:---|:---|:---|:---|
| `1` | Vendedor | `vendedor@test.com` | Vendedor (creador de subastas 1 a 7) | $0.00 |
| `2` | Comprador Líder | `comprador1@test.com` | Comprador con fondos | $105,000.00 disp. |
| `3` | Comprador Habilitado | `comprador2@test.com` | Comprador con fondos | $200,000.00 disp. |
| `4` | Usuario Sin Fondos | `sinfondos@test.com` | Comprador sin saldo suficiente | $500.00 disp. |

---

## 📄 Licencia

Proyecto desarrollado con fines académicos. Distribuido bajo la licencia correspondiente al plan de estudios de la cátedra.
