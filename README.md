```markdown
# 🔨 SubastaYa - Plataforma de Subastas en Tiempo Real

**SubastaYa** es un sistema integral de subastas en línea en tiempo real desarrollado bajo una arquitectura limpia (Clean Architecture) y modular en **.NET 8** para el Backend y **React 19 + Vite** para el Frontend. Incorpora mecanismos de concurrencia optimista (*Optimistic Locking*), actualización de pujas en tiempo real vía **SignalR (WebSockets)**, sistema de billetera con transacciones auditables (*Ledger*) y reglas de negocio como *Anti-Sniping*.

---

## 📋 Tabla de Contenidos
1. [Arquitectura del Proyecto y Paquetes](#-arquitectura-del-proyecto-y-paquetes-instalados)
2. [Prerrequisitos](#-prerrequisitos)
3. [Guía Paso a Paso de Instalación y Ejecución](#-guía-paso-a-paso-de-instalación-y-ejecución)
   - [1. Configuración de la Base de Datos](#1-configuración-de-la-base-de-datos)
   - [2. Compilación del Proyecto](#2-compilación-del-proyecto)
   - [3. Ejecución de las Migraciones y Seed Data](#3-ejecución-de-las-migraciones)
   - [4. Lanzar la Aplicación Backend](#4-lanzar-la-aplicación-backend)
   - [5. Lanzar la Aplicación Frontend y Configuración de Puertos](#5-lanzar-la-aplicación-frontend-y-configuración-de-puertos)
4. [Demostración y Testing de Concurrencia Optimista](#-demostración-y-testing-de-concurrencia-optimista)
   - [¿Cómo funciona el Optimistic Locking?](#cómo-funciona-la-concurrencia-optimista-en-subastaya)
   - [Ejecución del Test de Concurrencia (`TestConcurrencia`)](#ejecución-del-test-de-concurrencia)
   - [Cómo probar con otra subasta en el test](#cómo-probar-con-otra-subasta-en-el-test)
5. [Cuentas de Prueba (Seed Data)](#-cuentas-de-prueba)
6. [Autores](#-autores)

---

## 🏗 Arquitectura del Proyecto y Paquetes Instalados

El proyecto está estructurado siguiendo los principios de **Clean Architecture**, distribuyendo las dependencias y paquetes de NuGet/NPM en cada capa:

### 📦 Backend (.NET 8)

- **`Domain/`**
  - Contiene las entidades principales (`Subasta`, `Puja`, `Billetera`, `Usuario`, `Categoria`, `Transaccion_Ledger`, `Auditoria_Log`) y excepciones de dominio.
  - *Paquetes instalados:* No requiere paquetes externos (C# puro y DataAnnotations).

- **`Application/`**
  - Contiene la lógica de negocio, casos de uso (Commands, Queries, Handlers), DTOs e interfaces de repositorios.
  - *Paquetes NuGet instalados:*
    - `BCrypt.Net-Next` (v4.2.0): Para el hasheo y verificación segura de contraseñas de usuarios.

- **`Infraestructure/`**
  - Contiene la capa de persistencia de datos, configuración de `SubastaDbContext`, mapeos de entidades y migraciones de Entity Framework Core.
  - *Paquetes NuGet instalados:*
    - `Microsoft.EntityFrameworkCore.SqlServer` (v8.0.0): Proveedor de base de datos SQL Server.
    - `Microsoft.EntityFrameworkCore.Tools` (v8.0.0): Herramientas para la gestión de migraciones en tiempo de diseño.

- **`SubastaYa/` (API)**
  - Capa de presentación y Web API con controladores ASP.NET Core, Middlewares de manejo global de excepciones y Hubs de comunicación en tiempo real.
  - *Paquetes NuGet instalados:*
    - `Microsoft.AspNetCore.SignalR.Common` (v8.0.0): Infraestructura de SignalR para WebSockets.
    - `Microsoft.EntityFrameworkCore.SqlServer` (v8.0.0): Integración del DbContext en el contenedor de dependencias.
    - `Microsoft.EntityFrameworkCore.Tools` (v8.0.0): Herramientas de EF Core.
    - `Swashbuckle.AspNetCore` (v6.6.2): Generación y documentación interactiva de endpoints con Swagger UI.

- **`TestConcurrencia/`**
  - Aplicación de consola diseñada para simular tráfico masivo simultáneo y comprobar el bloqueo optimista.

### 🌐 Frontend (React + Vite)

- **`Frontend/`**
  - Single Page Application (SPA) para la interfaz de usuario de subastas, pujas en vivo y administración de billetera.
  - *Paquetes instalados:*
    - `@microsoft/signalr` (^10.0.11): Cliente oficial de SignalR para recibir actualizaciones y nuevas pujas en tiempo real vía WebSockets.
    - `react` / `react-dom` (^19.2.8): Biblioteca para la construcción de interfaces interactivas.
    - `react-router-dom` (^7.18.3): Manejo de rutas y navegación en la aplicación.

---

## ⚙️ Prerrequisitos

Asegúrate de contar con el siguiente software instalado en tu equipo antes de comenzar:

- **[.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)** (v8.0 o superior)
- **[Node.js](https://nodejs.org/)** (v18.0 o superior) y **npm**
- **Microsoft SQL Server** (SQL Server Express o LocalDB)
- **Entity Framework Core CLI Tools** (opcional pero recomendado):
  ```bash
  dotnet tool install --global dotnet-ef
  ```

---

## 🚀 Guía Paso a Paso de Instalación y Ejecución

### 1. Configuración de la Base de Datos

La cadena de conexión está configurada por defecto para conectarse a una instancia local de **SQL Server Express** (`localhost\SQLEXPRESS`).

1. Abre el archivo de configuración `SubastaYa/appsettings.json`.
2. Verifica o ajusta la cadena según tu servidor local:
   ```json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=localhost\\SQLEXPRESS;Database=SubastaYaDb;Trusted_Connection=True;MultipleActiveResultSets=True;TrustServerCertificate=True"
     }
   }
   ```

---

### 2. Compilación del Proyecto

Desde la raíz de la solución, restaura los paquetes NuGet y compila todos los proyectos:

```bash
# Restaurar dependencias
dotnet restore

# Compilar la solución
dotnet build
```

---

### 3. Ejecución de las Migraciones

Aplica las migraciones pendientes con EF Core para crear la base de datos `SubastaYaDb` junto con sus tablas, índices y datos iniciales (*Seed Data*):

```bash
dotnet ef database update --project Infraestructure --startup-project SubastaYa
```

> **Nota:** El DbContext precarga de forma automática usuarios de prueba, billeteras con saldo disponible y subastas activas (por ejemplo, la subasta con `Id: 6`).

---

### 4. Lanzar la Aplicación Backend

Inicia la Web API de ASP.NET Core:

```bash
dotnet run --project SubastaYa
```

El servidor quedará en ejecución en:
- **HTTPS:** `https://localhost:7117`
- **HTTP:** `http://localhost:5100`
- **Documentación Swagger UI:** `https://localhost:7117/swagger`

---

### 5. Lanzar la Aplicación Frontend y Configuración de Puertos

1. Abre una nueva terminal y navega al directorio `Frontend`:
   ```bash
   cd Frontend
   ```
2. Instala las dependencias necesarias:
   ```bash
   npm install
   ```
3. Inicia el servidor de desarrollo:
   ```bash
   npm run dev
   ```
4. Abre tu navegador e ingresa a:
   - **URL Local:** `http://localhost:5173`

> ⚠️ **Importante sobre Puertos y Terminales:**
> - El Backend corre por defecto en `https://localhost:7117`. Si modificas los puertos en `SubastaYa/Properties/launchSettings.json` o en `SubastaYa/Program.cs` (CORS), asegúrate de actualizar la URL base en el Frontend y en el script de pruebas.
> - **Si tienes varias terminales abiertas o puertos bloqueados:** Cierra todas las terminales activas de Node y .NET, asegúrate de liberar los puertos `7117` y `5173`, y vuelve a arrancar tanto el Backend como el Frontend en terminales limpias.

---

## 🛡 Demostración y Testing de Concurrencia Optimista

### ¿Cómo funciona la Concurrencia Optimista en SubastaYa?

1. **Token de Concurrencia (`RowVersion` / `[Timestamp]`):**
   La entidad `Subasta` posee un atributo `[Timestamp]` configurado como fila de versión:
   ```csharp
   [Timestamp]
   public byte[] Version { get; set; }
   ```
2. **Detección de Colisión:**
   Cuando múltiples solicitudes intentan pujar por la misma subasta en el mismo milisegundo, solo la primera transacción que impacta la base de datos actualiza el `RowVersion`. Las demás transacciones fallan inmediatamente al detectar que la versión en memoria ya no coincide con la versión persistida, generando una excepción `DbUpdateConcurrencyException`.
3. **Respuesta HTTP:**
   - La capa de persistencia captura la excepción y la convierte en `ConflictException`.
   - Se registra el evento en `Auditorias_Log` con la acción `PUJA_RECHAZADA_CONCURRENCIA`.
   - El middleware global de excepciones devuelve al cliente el código de estado **`HTTP 409 Conflict`**.

---

### Ejecución del Test de Concurrencia

El repositorio cuenta con el proyecto `TestConcurrencia` que lanza **1000 peticiones HTTP simultáneas en el mismo milisegundo** contra el endpoint de pujas usando `Task.WhenAll`.

#### Pasos para ejecutar:

1. Asegúrate de que el Backend esté corriendo en `https://localhost:7117`.
2. Abre una nueva terminal y accede a la carpeta del test:
   ```bash
   cd TestConcurrencia
   ```
3. Ejecuta el proyecto de pruebas:
   ```bash
   dotnet run
   ```

*(Alternativamente, desde la raíz del proyecto puedes ejecutar: `dotnet run --project TestConcurrencia`)*

#### Código fuente del test (`TestConcurrencia/Program.cs`):
```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;

// URL del endpoint de pujas
string apiUrl = "https://localhost:7117/api/auctions/6/bids";

// Cantidad de peticiones simultáneas
int usuariosSimultaneos = 1000;
Console.WriteLine($"Iniciando ataque: {usuariosSimultaneos} usuarios pujando al mismo milisegundo...");

using var httpClient = new HttpClient();
var tareas = new List<Task<HttpResponseMessage>>();
var cronometro = Stopwatch.StartNew();

// Preparar las peticiones simultáneas
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

// Disparar todas en paralelo
var resultados = await Task.WhenAll(tareas);
cronometro.Stop();

// Evaluar resultados
int exitosas = resultados.Count(r => r.IsSuccessStatusCode);
int fallidas = resultados.Count(r => !r.IsSuccessStatusCode);

Console.WriteLine("\n=== RESULTADOS DE LA PRUEBA ===");
Console.WriteLine($"Tiempo de respuesta del server: {cronometro.ElapsedMilliseconds} ms");
Console.WriteLine($"Pujas que el servidor aceptó (HTTP 200/201): {exitosas}");
Console.WriteLine($"Pujas que el servidor rechazó (HTTP 409 Conflict): {fallidas}");

var falloDeEjemplo = resultados.FirstOrDefault(r => !r.IsSuccessStatusCode);
if (falloDeEjemplo != null)
{
    Console.WriteLine($"\n--- DETALLE DEL RECHAZO ---");
    Console.WriteLine($"Código HTTP: {(int)falloDeEjemplo.StatusCode} ({falloDeEjemplo.StatusCode})");
    string mensajeError = await falloDeEjemplo.Content.ReadAsStringAsync();
    Console.WriteLine($"Mensaje del backend: {mensajeError}");
}
```

#### Salida esperada en consola:
```text
Iniciando ataque: 1000 usuarios pujando al mismo milisegundo...

=== RESULTADOS DE LA PRUEBA ===
Tiempo de respuesta del server: 312 ms
Pujas que el servidor aceptó (HTTP 200/201): 1
Pujas que el servidor rechazó (HTTP 409 Conflict): 999

✅ INCREÍBLE: Entró solo 1. Tu sistema ya está manejando la concurrencia.

--- DETALLE DEL RECHAZO ---
Código HTTP: 409 (Conflict)
Mensaje del backend: {"message":"Otro usuario realizó una puja simultáneamente. Intente pujar con el nuevo valor."}
```

---

### Cómo probar con otra subasta en el test

Si deseas ejecutar el test de concurrencia sobre una subasta diferente, abre el archivo `TestConcurrencia/Program.cs` y actualiza:

1. La variable `apiUrl` con el ID de la subasta deseada:
   ```csharp
   string apiUrl = "https://localhost:7117/api/auctions/<ID_DE_LA_SUBASTA>/bids";
   ```
2. La variable `jsonPayload` con el ID correspondiente y un monto válido que supere el precio actual:
   ```csharp
   string jsonPayload = @"{
       ""subasta_Id"": <ID_DE_LA_SUBASTA>,
       ""comprador_Id"": 2,
       ""monto"": 100000.00
   }";
   ```

---

## 👥 Cuentas de Prueba

El sistema incluye los siguientes usuarios precargados en el seed de datos (contraseña de todos los usuarios: `123456`):

| ID | Nombre | Email | Rol / Estado | Saldo Disponible Inicial |
|:---|:---|:---|:---|:---|
| `1` | Vendedor | `vendedor@test.com` | Vendedor (creador de subastas 1 a 7) | $0.00 |
| `2` | Comprador Líder | `comprador1@test.com` | Comprador con fondos | $105,000.00 |
| `3` | Comprador Habilitado | `comprador2@test.com` | Comprador con fondos | $200,000.00 |
| `4` | Usuario Sin Fondos | `sinfondos@test.com` | Comprador sin saldo suficiente | $500.00 |

---

## ✍️ Autores

- **Tronando Tomas**
- **Carranza Braian**
```
