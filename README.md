# Instructor
Para vincular Google Forms para la toma de asistencia y registrar automáticamente las respuestas al escanear un código QR, puedes seguir los siguientes pasos:

1. **Crear un Google Form**:
   - Diseña un formulario en Google Forms que contenga los campos necesarios para la asistencia, como el nombre, la fecha, y cualquier otro campo relevante.
   - Publica el formulario y obtén la URL del formulario.

2. **Generar un código QR que apunte al formulario**:
   - Usa una herramienta de generación de códigos QR (puedes usar la función `GenerarQR` que ya tienes) para crear un código QR que redirija a la URL del formulario de Google.

3. **Modificar tu aplicación para registrar la asistencia**:
   - Al escanear el código QR, obtén los datos necesarios del formulario y envíalos a tu aplicación.
   - Puedes usar AJAX para enviar estos datos a una acción en tu controlador que se encargue de registrar la asistencia en tu base de datos.

A continuación, se muestra un ejemplo de cómo puedes modificar tu vista y controlador para lograr esto:

### Paso 1: Modificar la vista para escanear el QR y enviar los datos del formulario

 `RegistrarAsistencia.cshtml`
@{
    ViewBag.Title = "Registrar Asistencia";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<script src="https://cdn.jsdelivr.net/npm/html5-qrcode@2.0.0-beta.8/dist/html5-qrcode.min.js"></script>

<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-6">
            <div class="card">
                <div class="card-body text-center">
                    <h5 class="card-title">Registrar Asistencia</h5>
                    <p class="card-text">Escanea el código QR para registrar tu asistencia.</p>
                    <div id="qr-reader" style="width: 100%;"></div>
                    <div id="qr-reader-results" class="mt-3"></div>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
    function docReady(fn) {
        if (document.readyState === "complete" || document.readyState === "interactive") {
            setTimeout(fn, 1);
        } else {
            document.addEventListener("DOMContentLoaded", fn);
        }
    }

    docReady(function () {
        var resultContainer = document.getElementById('qr-reader-results');
        var lastResult, countResults = 0;

        function onScanSuccess(qrMessage) {
            if (qrMessage !== lastResult) {
                ++countResults;
                lastResult = qrMessage;
                resultContainer.innerHTML = `<div>[${countResults}] - ${qrMessage}</div>`;
                
                // Enviar el código QR al servidor para registrar la asistencia
                $.post('@Url.Action("RegistrarAsistencia", "Instructor")', { qrData: qrMessage }, function (response) {
                    alert(response.message);
                });
            }
        }

        var html5QrcodeScanner = new Html5QrcodeScanner(
            "qr-reader", { fps: 10, qrbox: 250 });
        html5QrcodeScanner.render(onScanSuccess);
    });
</script>

```

### Paso 2: Controlador para registrar la asistencia
InstructorController.cs

 using System;
using System.Web.Mvc;
using System.Data.SqlClient;
using System.Web;
using QRCoder;
using System.Drawing;
using System.IO;

namespace CronosControl.Controllers
{
    public class InstructorController : Controller
    {
        // Otras acciones...

        [HttpPost]
        public JsonResult RegistrarAsistencia(string qrData)
        {
            try
            {
                // Aquí puedes procesar los datos del QR. Por ejemplo, podrías esperar que qrData tenga el formato de una URL de Google Forms con parámetros.
                var parametros = HttpUtility.ParseQueryString(new Uri(qrData).Query);
                string correo = parametros["correo"];
                string nombreCompleto = parametros["nombreCompleto"];
                string tipoDocumento = parametros["tipoDocumento"];
                string numeroDocumento = parametros["numeroDocumento"];
                string vinculacion = parametros["vinculacion"];
                string numeroFicha = parametros["numeroFicha"];
                string nombrePrograma = parametros["nombrePrograma"];
                string fecha = parametros["fecha"];

                // Conexión a la base de datos y registro de asistencia
                string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    string query = "INSERT INTO Asistencia (Correo, NombreCompleto, TipoDocumento, NumeroDocumento, Vinculacion, NumeroFicha, NombrePrograma, Fecha) VALUES (@Correo, @NombreCompleto, @TipoDocumento, @NumeroDocumento, @Vinculacion, @NumeroFicha, @NombrePrograma, @Fecha)";
                    SqlCommand command = new SqlCommand(query, connection);
                    command.Parameters.AddWithValue("@Correo", correo);
                    command.Parameters.AddWithValue("@NombreCompleto", nombreCompleto);
                    command.Parameters.AddWithValue("@TipoDocumento", tipoDocumento);
                    command.Parameters.AddWithValue("@NumeroDocumento", numeroDocumento);
                    command.Parameters.AddWithValue("@Vinculacion", vinculacion);
                    command.Parameters.AddWithValue("@NumeroFicha", numeroFicha);
                    command.Parameters.AddWithValue("@NombrePrograma", nombrePrograma);
                    command.Parameters.AddWithValue("@Fecha", fecha);
                    connection.Open();
                    command.ExecuteNonQuery();
                }

                return Json(new { message = "Asistencia registrada con éxito" });
            }
            catch (Exception ex)
            {
                return Json(new { message = "Error al registrar la asistencia: " + ex.Message });
            }
        }
    }
}

```

Paso 3: Generar el código QR

Asegúrate de que tu función para generar el código QR apunte a la URL del formulario de Google con los parámetros necesarios:

```csharp
public ActionResult GenerarQR(string data)
{
    // Aquí puedes construir la URL de Google Forms con los parámetros necesarios
    string googleFormUrl = $"https://docs.google.com/forms/d/e/1FAIpQLSeflLjsNf-xyz/viewform?usp=pp_url&entry.1234567890={data}";

    // Generar código QR
    QRCodeGenerator qrGenerator = new QRCodeGenerator();
    QRCodeData qrCodeData = qrGenerator.CreateQrCode(googleFormUrl, QRCodeGenerator.ECCLevel.Q);
    QRCode qrCode = new QRCode(qrCodeData);

    // Convertir código QR a bitmap
    Bitmap qrCodeImage = qrCode.GetGraphic(10);

    // Convertir bitmap a arreglo de bytes
    using (MemoryStream stream = new MemoryStream())
    {
        qrCodeImage.Save(stream, System.Drawing.Imaging.ImageFormat.Png);
        byte[] imageBytes = stream.ToArray();

        // Devolver la imagen del código QR como FileResult
        return File(imageBytes, "image/png");
    }
}
```

### Resumen

1. **Crear y publicar un formulario de Google Forms** para capturar la asistencia.
2. **Generar códigos QR** que apunten a este formulario con los parámetros necesarios.
3. **Modificar tu aplicación** para escanear el código QR, extraer la información y enviarla a tu servidor para registrar la asistencia en tu base de datos.

Esta integración permite que al escanear el código QR, los datos se registren tanto en el Google Form como en tu base de datos.

____________________________________

Para crear una vista en ASP.NET MVC que muestre los programas de formación a los cuales está asociado un instructor, sigue estos pasos:

1. **Modificar el controlador**: Agrega una acción para obtener los programas de formación asociados al instructor.
2. **Crear una vista**: Diseña una vista que muestre estos programas.
3. **Modificar el modelo (si es necesario)**: Asegúrate de tener los modelos necesarios para representar los programas de formación y su relación con los instructores.

 Paso 1: Modificar el controlador

En el controlador `InstructorController`, agrega una acción que obtenga los programas de formación para el instructor actual. Supongamos que tienes un modelo `ProgramaFormacion` y una base de datos que contiene esta información.

`InstructorController.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Data.SqlClient;
using System.Web.Mvc;
using CronosControl.Models;

namespace CronosControl.Controllers
{
    public class InstructorController : Controller
    {
        // GET: Instructor
        public ActionResult Index()
        {
            int ReporteInasistencias;
            string Conexion = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = "SELECT COUNT(*) FROM reporte WHERE tipo_reporte = 'Inasistencia'";

                SqlCommand comando = new SqlCommand(query, connection);
                connection.Open();

                ReporteInasistencias = (int)comando.ExecuteScalar();
            }

            ViewBag.ReporteInasistencias = ReporteInasistencias;
            return View();
        }

        public ActionResult Fichas()
        {
            return View();
        }

        public ActionResult ProgramasFormacion()
        {
            // Obtener los programas de formación asociados al instructor
            string instructorId = "1043968399"; // Reemplaza con la lógica para obtener el ID del instructor actual
            List<ProgramaFormacion> programas = ObtenerProgramasFormacion(instructorId);

            return View(programas);
        }

        public ActionResult GenerarReporteInasistencias()
        {
            return View();
        }

        public ActionResult GenerarQR(string data)
        {
            // Generar código QR
            QRCodeGenerator qrGenerator = new QRCodeGenerator();
            QRCodeData qrCodeData = qrGenerator.CreateQrCode(data, QRCodeGenerator.ECCLevel.Q);
            QRCode qrCode = new QRCode(qrCodeData);

            // Convertir código QR a bitmap
            Bitmap qrCodeImage = qrCode.GetGraphic(10);

            // Convertir bitmap a arreglo de bytes
            using (MemoryStream stream = new MemoryStream())
            {
                qrCodeImage.Save(stream, System.Drawing.Imaging.ImageFormat.Png);
                byte[] imageBytes = stream.ToArray();

                // Devolver la imagen del código QR como FileResult
                return File(imageBytes, "image/png");
            }
        }

        public ActionResult CerrarSesion()
        {
            Session.Abandon();
            return RedirectToAction("Index", "Instructor");
        }

        private List<ProgramaFormacion> ObtenerProgramasFormacion(string instructorId)
        {
            List<ProgramaFormacion> programas = new List<ProgramaFormacion>();
            string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(connectionString))
            {
                string query = "SELECT * FROM ProgramasFormacion WHERE InstructorId = @InstructorId";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@InstructorId", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    ProgramaFormacion programa = new ProgramaFormacion
                    {
                        Id = (int)reader["Id"],
                        Nombre = reader["Nombre"].ToString(),
                        Codigo = reader["Codigo"].ToString(),
                        FechaInicio = (DateTime)reader["FechaInicio"]
                    };
                    programas.Add(programa);
                }
            }

            return programas;
        }
    }
}
```

### Paso 2: Crear una vista

Crea una vista llamada `ProgramasFormacion.cshtml` dentro de la carpeta `Views/Instructor`. Esta vista mostrará la lista de programas de formación.

`ProgramasFormacion.cshtml`

```html
@model IEnumerable<CronosControl.Models.ProgramaFormacion>

@{
    ViewBag.Title = "Programas de Formación";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<div class="container mt-5">
    <div class="row">
        <div class="col-lg-12">
            <h2 class="text-center">Programas de Formación</h2>
            <div class="table-responsive">
                <table class="table table-bordered table-hover">
                    <thead class="thead-dark">
                        <tr>
                            <th>Nombre del Programa</th>
                            <th>Código</th>
                            <th>Fecha de Inicio</th>
                        </tr>
                    </thead>
                    <tbody>
                        @foreach (var programa in Model)
                        {
                            <tr>
                                <td>@programa.Nombre</td>
                                <td>@programa.Codigo</td>
                                <td>@programa.FechaInicio.ToShortDateString()</td>
                            </tr>
                        }
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
```

### Paso 3: Modelo de datos (si es necesario)

Asegúrate de tener un modelo que represente los programas de formación. Puedes colocarlo en una carpeta `Models`.

`ProgramaFormacion.cs`

```csharp
namespace CronosControl.Models
{
    public class ProgramaFormacion
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public string Codigo { get; set; }
        public DateTime FechaInicio { get; set; }
    }
}
```

### Resumen

1. **Modificar el controlador**: Se agregó la lógica para obtener los programas de formación asociados a un instructor.
2. **Crear la vista**: Se diseñó una vista que muestra la lista de programas de formación.
3. **Modelo de datos**: Se definió un modelo `ProgramaFormacion` para representar los programas de formación.

Con estos pasos, habrás creado una vista que muestra los programas de formación asociados al instructor actual.

____________________________________

using System;
using System.Collections.Generic;
using System.Data.SqlClient;
using System.Web.Mvc;
using CronosControl.Models;

namespace CronosControl.Controllers
{
    public class InstructorController : Controller
    {
        // Otras acciones...

        public ActionResult Fichas()
        {
            // Obtener las fichas asociadas al instructor
            string instructorId = "123"; // Reemplaza con la lógica para obtener el ID del instructor actual
            List<Ficha> fichas = ObtenerFichas(instructorId);

            // Pasar la lista de fichas como modelo a la vista
            return View(fichas);
        }

        // Método privado para obtener las fichas asociadas al instructor
        private List<Ficha> ObtenerFichas(string instructorId)
        {
            List<Ficha> fichas = new List<Ficha>();
            string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(connectionString))
            {
                string query = "SELECT * FROM Fichas WHERE InstructorId = @InstructorId";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@InstructorId", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    Ficha ficha = new Ficha
                    {
                        Id = (int)reader["Id"],
                        Nombre = reader["Nombre"].ToString(),
                        Descripcion = reader["Descripcion"].ToString(),
                        // Agrega más propiedades según la estructura de tu tabla Fichas en la base de datos
                    };
                    fichas.Add(ficha);
                }
            }

            return fichas;
        }
    }
}


@model List<CronosControl.Models.Ficha>

@{
    ViewBag.Title = "Fichas";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<h2>Fichas Asociadas</h2>

<div class="container">
    <div class="row">
        <div class="col-md-12">
            <table class="table">
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Nombre</th>
                        <th>Descripción</th>
                        <!-- Agrega más columnas según la estructura de tu tabla Fichas en la base de datos -->
                    </tr>
                </thead>
                <tbody>
                    @foreach (var ficha in Model)
                    {
                        <tr>
                            <td>@ficha.Id</td>
                            <td>@ficha.Nombre</td>
                            <td>@ficha.Descripcion</td>
                            <!-- Agrega más celdas según la estructura de tu tabla Fichas en la base de datos -->
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    </div>
</div>


namespace CronosControl.Models
{
    public class Ficha
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public string Descripcion { get; set; }
        // Agrega más propiedades según la estructura de tu tabla Fichas en la base de datos
    }
}

____________________________________

Para crear una tabla de `Inasistencia` en SQL Server que almacene los datos necesarios, puedes usar una consulta SQL que defina la estructura de la tabla con todos los campos requeridos. Aquí tienes un ejemplo de cómo puedes crear esta tabla:

 1. Consulta SQL para Crear la Tabla `Inasistencia`
```sql
CREATE TABLE Inasistencia (
    Id INT PRIMARY KEY IDENTITY(1,1),
    NombreCompleto VARCHAR(100) NOT NULL,
    CorreoElectronico VARCHAR(100) NOT NULL,
    TipoDocumento VARCHAR(50) NOT NULL,
    NumeroDocumento VARCHAR(50) NOT NULL,
    Vinculacion VARCHAR(50) NOT NULL,
    NumeroFicha VARCHAR(50) NOT NULL,
    NombrePrograma VARCHAR(100) NOT NULL,
    FechaInasistencia DATE NOT NULL,
    Motivo VARCHAR(255)
);
```

 2. Explicación de los Campos

- `Id`: Un campo de identificación único para cada registro de inasistencia. Es una clave primaria con auto-incremento.
- `NombreCompleto`: El nombre completo del estudiante.
- `CorreoElectronico`: El correo electrónico del estudiante.
- `TipoDocumento`: El tipo de documento (por ejemplo, DNI, Pasaporte).
- `NumeroDocumento`: El número de documento del estudiante.
- `Vinculacion`: La vinculación del estudiante (por ejemplo, estudiante regular, visitante, etc.).
- `NumeroFicha`: El número de la ficha asociada al estudiante.
- `NombrePrograma`: El nombre del programa de formación asociado al estudiante.
- `FechaInasistencia`: La fecha en la que se registró la inasistencia.
- `Motivo`: El motivo de la inasistencia (puede ser nulo).

3. Ejecutar la Consulta en SQL Server Management Studio (SSMS)

1. Abre SQL Server Management Studio (SSMS) y conéctate a tu servidor de base de datos.
2. Abre una nueva consulta.
3. Copia y pega la consulta SQL anterior en la ventana de consulta.
4. Ejecuta la consulta para crear la tabla `Inasistencia`.

4. Ingresar Datos en la Tabla `Inasistencia`

Una vez creada la tabla, puedes ingresar datos manualmente o mediante un formulario en tu aplicación web. Aquí tienes un ejemplo de cómo insertar datos en la tabla usando una consulta SQL:

```sql
INSERT INTO Inasistencia (NombreCompleto, CorreoElectronico, TipoDocumento, NumeroDocumento, Vinculacion, NumeroFicha, NombrePrograma, FechaInasistencia, Motivo)
VALUES ('Juan Perez', 'juan.perez@example.com', 'DNI', '12345678', 'Estudiante Regular', '1234', 'Ingeniería de Sistemas', '2024-05-15', 'Enfermedad');
```

5. Integración en el Controlador de ASP.NET MVC

A continuación, se muestra cómo puedes modificar tu controlador `InstructorController` para registrar inasistencias en esta tabla. 

```csharp
using System;
using System.Collections.Generic;
using System.Data.SqlClient;
using System.Web.Mvc;
using CronosControl.Models;

namespace CronosControl.Controllers
{
    public class InstructorController : Controller
    {
        // Otras acciones...

        [HttpPost]
        public JsonResult RegistrarAsistencia(string qrData)
        {
            try
            {
                // Procesar los datos del QR
                var parametros = HttpUtility.ParseQueryString(new Uri(qrData).Query);
                string nombreCompleto = parametros["nombreCompleto"];
                string correoElectronico = parametros["correoElectronico"];
                string tipoDocumento = parametros["tipoDocumento"];
                string numeroDocumento = parametros["numeroDocumento"];
                string vinculacion = parametros["vinculacion"];
                string numeroFicha = parametros["numeroFicha"];
                string nombrePrograma = parametros["nombrePrograma"];
                string fechaInasistencia = parametros["fechaInasistencia"];
                string motivo = parametros["motivo"];

                // Conexión a la base de datos y registro de inasistencia
                string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    string query = "INSERT INTO Inasistencia (NombreCompleto, CorreoElectronico, TipoDocumento, NumeroDocumento, Vinculacion, NumeroFicha, NombrePrograma, FechaInasistencia, Motivo) " +
                                   "VALUES (@NombreCompleto, @CorreoElectronico, @TipoDocumento, @NumeroDocumento, @Vinculacion, @NumeroFicha, @NombrePrograma, @FechaInasistencia, @Motivo)";
                    SqlCommand command = new SqlCommand(query, connection);
                    command.Parameters.AddWithValue("@NombreCompleto", nombreCompleto);
                    command.Parameters.AddWithValue("@CorreoElectronico", correoElectronico);
                    command.Parameters.AddWithValue("@TipoDocumento", tipoDocumento);
                    command.Parameters.AddWithValue("@NumeroDocumento", numeroDocumento);
                    command.Parameters.AddWithValue("@Vinculacion", vinculacion);
                    command.Parameters.AddWithValue("@NumeroFicha", numeroFicha);
                    command.Parameters.AddWithValue("@NombrePrograma", nombrePrograma);
                    command.Parameters.AddWithValue("@FechaInasistencia", fechaInasistencia);
                    command.Parameters.AddWithValue("@Motivo", motivo);

                    connection.Open();
                    command.ExecuteNonQuery();
                }

                return Json(new { message = "Inasistencia registrada con éxito" });
            }
            catch (Exception ex)
            {
                return Json(new { message = "Error al registrar la inasistencia: " + ex.Message });
            }
        }
    }
}
```

 6. Crear una Vista para Registrar la Asistencia

A continuación, se muestra un ejemplo de cómo podrías crear una vista en ASP.NET MVC para registrar la asistencia utilizando un escáner de código QR.

Crear la Vista `RegistrarAsistencia.cshtml`
```html
@{
    ViewBag.Title = "Registrar Asistencia";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<script src="https://cdn.jsdelivr.net/npm/html5-qrcode@2.0.0-beta.8/dist/html5-qrcode.min.js"></script>

<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-6">
            <div class="card">
                <div class="card-body text-center">
                    <h5 class="card-title">Registrar Asistencia</h5>
                    <p class="card-text">Escanea el código QR para registrar tu asistencia.</p>
                    <div id="qr-reader" style="width: 100%;"></div>
                    <div id="qr-reader-results"></div>
                </div>
            </div>
        </div>
    </div>
</div>

<script src="https://unpkg.com/html5-qrcode/minified/html5-qrcode.min.js"></script>
<script>
    function docReady(fn) {
        if (document.readyState === "complete" || document.readyState === "interactive") {
            setTimeout(fn, 1);
        } else {
            document.addEventListener("DOMContentLoaded", fn);
        }
    }

    docReady(function () {
        var resultContainer = document.getElementById('qr-reader-results');
        var lastResult, countResults = 0;

        function onScanSuccess(qrMessage) {
            if (qrMessage !== lastResult) {
                ++countResults;
                lastResult = qrMessage;
                resultContainer.innerHTML += `<div>[${countResults}] - ${qrMessage}</div>`;
                
                // Enviar el código QR al servidor para registrar la asistencia
                $.post('@Url.Action("RegistrarAsistencia", "Instructor")', { qrData: qrMessage }, function (response) {
                    alert(response.message);
                });
            }
        }

        var html5QrcodeScanner = new Html5QrcodeScanner(
            "qr-reader", { fps: 10, qrbox: 250 });
        html5QrcodeScanner.render(onScanSuccess);
    });
</script>
```

Con estos pasos, deberías poder crear una tabla para registrar inasistencias y una vista en tu aplicación ASP.NET MVC para escanear códigos QR y registrar inasistencias en la base de datos.


____________________________________

Para crear un informe mensual de inasistencias utilizando Crystal Reports en ASP.NET MVC, seguirás estos pasos:

1. **Instalar Crystal Reports**:
   - Asegúrate de tener instalado Crystal Reports Developer Edition para Visual Studio.
   - Puedes descargar e instalar Crystal Reports desde el sitio web de SAP.

2. **Crear el Informe en Crystal Reports**:
   - Abre Visual Studio y crea un nuevo informe de Crystal Reports.
   - Diseña el informe para mostrar los datos de la tabla de inasistencias.

3. **Configurar el Proyecto**:
   - Añade el informe de Crystal Reports a tu proyecto ASP.NET MVC.
   - Añade las referencias necesarias para Crystal Reports.

4. **Crear el Controlador y la Vista para Mostrar el Informe**:
   - Implementa el controlador para cargar los datos y mostrar el informe.
   - Crea la vista para renderizar el informe.

A continuación se detallan estos pasos:

1. Instalar Crystal Reports

Descarga e instala Crystal Reports Developer Edition para Visual Studio desde el [sitio web de SAP](https://www.sap.com/products/crystal-reports.html).

 2. Crear el Informe en Crystal Reports

1. **Añadir un nuevo informe**:
   - En tu proyecto ASP.NET MVC, haz clic derecho en la carpeta donde deseas añadir el informe (por ejemplo, `Reports`) y selecciona `Add` > `New Item`.
   - Selecciona `Crystal Report` y dale un nombre, por ejemplo, `InasistenciasMensuales.rpt`.

2. **Diseñar el informe**:
   - Conecta el informe a tu base de datos.
   - Diseña el informe para mostrar los campos de la tabla `Inasistencia`.
   - Añade parámetros para filtrar los datos por mes y año si es necesario.

 3. Configurar el Proyecto

Asegúrate de tener instaladas las siguientes referencias de NuGet en tu proyecto:
- `CrystalDecisions.CrystalReports.Engine`
- `CrystalDecisions.Shared`
- `CrystalDecisions.Web`

Puedes instalar estas bibliotecas desde el administrador de paquetes NuGet.

4. Crear el Controlador y la Vista

 Controlador
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Mvc;
using System.Data.SqlClient;
using CronosControl.Models;
using CrystalDecisions.CrystalReports.Engine;
using CrystalDecisions.Shared;
using System.IO;

namespace CronosControl.Controllers
{
    public class ReportController : Controller
    {
        public ActionResult InasistenciasMensuales(int month, int year)
        {
            ReportDocument rd = new ReportDocument();
            rd.Load(Path.Combine(Server.MapPath("~/Reports"), "InasistenciasMensuales.rpt"));

            // Conexión a la base de datos
            string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";
            SqlConnection connection = new SqlConnection(connectionString);
            SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Inasistencia WHERE MONTH(FechaInasistencia) = @Month AND YEAR(FechaInasistencia) = @Year", connection);
            da.SelectCommand.Parameters.AddWithValue("@Month", month);
            da.SelectCommand.Parameters.AddWithValue("@Year", year);

            DataSet ds = new DataSet();
            da.Fill(ds, "Inasistencia");

            rd.SetDataSource(ds);

            Response.Buffer = false;
            Response.ClearContent();
            Response.ClearHeaders();

            Stream stream = rd.ExportToStream(CrystalDecisions.Shared.ExportFormatType.PortableDocFormat);
            stream.Seek(0, SeekOrigin.Begin);

            return File(stream, "application/pdf", "InasistenciasMensuales.pdf");
        }
    }
}
```

Vista
Crea una vista para seleccionar el mes y año, y mostrar el informe.

```html
@{
    ViewBag.Title = "Reporte de Inasistencias Mensuales";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<div class="container mt-5">
    <h2>Reporte de Inasistencias Mensuales</h2>
    <form action="@Url.Action("InasistenciasMensuales", "Report")" method="get">
        <div class="form-group">
            <label for="month">Mes:</label>
            <select class="form-control" id="month" name="month">
                @for (int i = 1; i <= 12; i++)
                {
                    <option value="@i">@i</option>
                }
            </select>
        </div>
        <div class="form-group">
            <label for="year">Año:</label>
            <input type="number" class="form-control" id="year" name="year" value="@DateTime.Now.Year">
        </div>
        <button type="submit" class="btn btn-primary">Generar Reporte</button>
    </form>
</div>
```

5. Ejecutar el Proyecto

- Inicia tu proyecto en Visual Studio.
- Navega a la vista que creaste para seleccionar el mes y año.
- Selecciona el mes y año y haz clic en "Generar Reporte" para ver el informe de inasistencias en formato PDF.

Con estos pasos, habrás creado un informe mensual de inasistencias utilizando Crystal Reports en tu proyecto ASP.NET MVC.

____________________________________

Para crear una página de inicio más formal y funcional para el instructor en un proyecto ASP.NET MVC, es importante incluir elementos esenciales que proporcionen una buena experiencia de usuario y faciliten la navegación por el sistema. A continuación, te proporciono un ejemplo de cómo podrías estructurar la página de inicio del instructor, incluyendo una barra de navegación y secciones destacadas.

### Paso 1: Modificar el Controlador
Primero, asegúrate de que el controlador `InstructorController` tenga la lógica necesaria para pasar los datos a la vista.

```csharp
using System.Data.SqlClient;
using System.Web.Mvc;
using CronosControl.Models;

namespace CronosControl.Controllers
{
    public class InstructorController : Controller
    {
        public ActionResult Index()
        {
            int ReporteInasistencias;
            string Conexion = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = "SELECT COUNT(*) FROM reporte WHERE tipo_reporte = 'Inasistencia'";

                SqlCommand comando = new SqlCommand(query, connection);
                connection.Open();

                ReporteInasistencias = (int)comando.ExecuteScalar();
            }

            ViewBag.ReporteInasistencias = ReporteInasistencias;
            return View();
        }

        public ActionResult ProgramasFormacion()
        {
            // Obtener los programas de formación asociados al instructor
            string instructorId = "123"; // Reemplaza con la lógica para obtener el ID del instructor actual
            List<ProgramaFormacion> programas = ObtenerProgramasFormacion(instructorId);

            return View(programas);
        }

        public ActionResult Fichas()
        {
            // Obtener las fichas asociadas al instructor
            string instructorId = "123"; // Reemplaza con la lógica para obtener el ID del instructor actual
            List<Ficha> fichas = ObtenerFichas(instructorId);

            return View(fichas);
        }

        private List<ProgramaFormacion> ObtenerProgramasFormacion(string instructorId)
        {
            List<ProgramaFormacion> programas = new List<ProgramaFormacion>();
            string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(connectionString))
            {
                string query = "SELECT * FROM ProgramasFormacion WHERE InstructorId = @InstructorId";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@InstructorId", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    ProgramaFormacion programa = new ProgramaFormacion
                    {
                        Id = (int)reader["Id"],
                        Nombre = reader["Nombre"].ToString(),
                        Codigo = reader["Codigo"].ToString(),
                        FechaInicio = (DateTime)reader["FechaInicio"]
                    };
                    programas.Add(programa);
                }
            }

            return programas;
        }

        private List<Ficha> ObtenerFichas(string instructorId)
        {
            List<Ficha> fichas = new List<Ficha>();
            string connectionString = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

            using (SqlConnection connection = new SqlConnection(connectionString))
            {
                string query = "SELECT * FROM Fichas WHERE InstructorId = @InstructorId";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@InstructorId", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    Ficha ficha = new Ficha
                    {
                        Id = (int)reader["Id"],
                        NumeroFicha = reader["NumeroFicha"].ToString(),
                        NombrePrograma = reader["NombrePrograma"].ToString()
                    };
                    fichas.Add(ficha);
                }
            }

            return fichas;
        }
    }
}
```

### Paso 2: Crear la Vista `Index.cshtml`

Esta vista proporcionará un panel de control inicial para el instructor, con acceso a diferentes secciones del sistema.

```html
@{
    ViewBag.Title = "Inicio Instructor";
    Layout = "~/Views/Shared/_Layout.cshtml";
}

<div class="container mt-5">
    <div class="jumbotron">
        <h1 class="display-4">Bienvenido, Instructor</h1>
        <p class="lead">Este es tu panel de control, donde puedes gestionar tus actividades y reportes.</p>
        <hr class="my-4">
        <p>Aquí encontrarás accesos rápidos a tus fichas, programas de formación y reportes de inasistencias.</p>
    </div>

    <div class="row">
        <div class="col-md-4">
            <div class="card mb-4 shadow-sm">
                <div class="card-header">
                    <h4 class="my-0 font-weight-normal">Programas de Formación</h4>
                </div>
                <div class="card-body">
                    <h1 class="card-title pricing-card-title">Programas</h1>
                    <ul class="list-unstyled mt-3 mb-4">
                        <li>Consulta y gestiona tus programas de formación</li>
                    </ul>
                    <a href="@Url.Action("ProgramasFormacion", "Instructor")" class="btn btn-lg btn-block btn-primary">Ver Programas</a>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card mb-4 shadow-sm">
                <div class="card-header">
                    <h4 class="my-0 font-weight-normal">Fichas</h4>
                </div>
                <div class="card-body">
                    <h1 class="card-title pricing-card-title">Fichas</h1>
                    <ul class="list-unstyled mt-3 mb-4">
                        <li>Accede a las fichas asociadas a tus programas</li>
                    </ul>
                    <a href="@Url.Action("Fichas", "Instructor")" class="btn btn-lg btn-block btn-primary">Ver Fichas</a>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card mb-4 shadow-sm">
                <div class="card-header">
                    <h4 class="my-0 font-weight-normal">Reportes de Inasistencias</h4>
                </div>
                <div class="card-body">
                    <h1 class="card-title pricing-card-title">Reportes</h1>
                    <ul class="list-unstyled mt-3 mb-4">
                        <li>Genera y consulta reportes de inasistencias</li>
                        <li><strong>@ViewBag.ReporteInasistencias</strong> Inasistencias registradas</li>
                    </ul>
                    <a href="@Url.Action("GenerarReporteInasistencias", "Instructor")" class="btn btn-lg btn-block btn-primary">Ver Reportes</a>
                </div>
            </div>
        </div>
    </div>
</div>
```

### Paso 3: Crear los Modelos `ProgramaFormacion` y `Ficha`

#### ProgramaFormacion.cs
```csharp
namespace CronosControl.Models
{
    public class ProgramaFormacion
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public string Codigo { get; set; }
        public DateTime FechaInicio { get; set; }
    }
}
```

#### Ficha.cs
```csharp
namespace CronosControl.Models
{
    public class Ficha
    {
        public int Id { get; set; }
        public string NumeroFicha { get; set; }
        public string NombrePrograma { get; set; }
    }
}
```

### Paso 4: Asegurarse de que la Base de Datos y las Tablas Estén Configuradas

#### Crear la tabla `Fichas`
```sql
CREATE TABLE Fichas (
    Id INT PRIMARY KEY IDENTITY,
    NumeroFicha NVARCHAR(50),
    NombrePrograma NVARCHAR(100),
    InstructorId NVARCHAR(50)
);
```

#### Crear la tabla `ProgramasFormacion`
```sql
CREATE TABLE ProgramasFormacion (
    Id INT PRIMARY KEY IDENTITY,
    Nombre NVARCHAR(100),
    Codigo NVARCHAR(50),
    FechaInicio DATE,
    InstructorId NVARCHAR(50)
);
```

Con estos pasos, tendrás una página de inicio más formal y funcional para el instructor, con accesos rápidos a las secciones más importantes y una visualización clara de la información esencial.
