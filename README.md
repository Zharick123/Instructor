To add a function that allows the instructor to set a 3-day deadline for students to submit a medical excuse or justification support after being absent, you need to make several modifications. First, you need a new method for setting this deadline and storing it. Then, you should update the absence report logic to respect this deadline. Here is how you can integrate these features:

1. **Update the database schema (if needed)**: Ensure your database has a way to store this deadline setting. For simplicity, let's assume each instructor can set their own deadline.

2. **Add methods to the `InstructorController`**: You need methods to set the deadline and to enforce it when checking absences.

### 1. Updating the Database Schema
Assuming the deadline setting is stored per instructor, you need a table or a column in your `usuarios` table (or a relevant table) to store this information. Here's an example SQL command to add a column to the `usuarios` table:

```sql
ALTER TABLE usuarios ADD dias_para_excusa INT DEFAULT 3;
```

### 2. Update the `InstructorController` Class

Here is the updated `InstructorController` class with new methods:

```csharp
using System;
using System.Collections.Generic;
using System.Web.Mvc;
using System.IO;
using QRCoder;
using System.Data.SqlClient;
using System.Drawing;
using CronosControl_14.Model;
using CrystalDecisions.CrystalReports.Engine;
using CrystalDecisions.Shared;
using CrystalDecisions.Web;
using System.Data;

namespace CronosControl_14.Controllers
{
    public class InstructorController : Controller
    {
        private static string currentQRData;
        private static DateTime lastQRUpdateTime;
        private static bool qrEnabled = true;
        private static TimeSpan? classStartTime;
        private static TimeSpan? classEndTime;
        private static int diasParaExcusa = 3; // Default value

        string Conexion = "data source=CNGAPRCIPFSD068\\SQLEXPRESS01;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";
        
        string Conexion = "data source=DESKTOP-HPRJHVG;initial catalog=proyectoASIS;integrated security=True;MultipleActiveResultSets=True;";

        // GET: Instructor
        public ActionResult Index()
        {
            int ReporteInasistencias;

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
            int instructorId = ObtenerIdInstructor();
            List<Ficha> fichas = ObtenerFichas(instructorId);

            return View(fichas);
        }

        public ActionResult ProgramasFormacion()
        {
            int instructorId = ObtenerIdInstructor();
            List<ProgramaFormacion> programas = ObtenerProgramasFormacion(instructorId);

            return View(programas);
        }

        private List<ProgramaFormacion> ObtenerProgramasFormacion(int instructorId)
        {
            List<ProgramaFormacion> programas = new List<ProgramaFormacion>();

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = "SELECT id_programa_formacion, tipo_programa, nombre_programa, fecha_registro, numero_ficha, id_competencia, id_usuario ";
                query += "FROM programa_formacion WHERE id_usuario = @InstructorId";

                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@InstructorId", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    ProgramaFormacion programa = new ProgramaFormacion
                    {
                        id_programa_formacion = (int)reader["id_programa_formacion"],
                        tipo_programa = reader["tipo_programa"].ToString(),
                        nombre_programa = reader["nombre_programa"].ToString(),
                        fecha_registro = (DateTime)reader["fecha_registro"],
                        numero_ficha = (int)reader["numero_ficha"],
                        id_competencia = (int)reader["id_competencia"],
                        id_usuario = (int)reader["id_usuario"]
                    };
                    programas.Add(programa);
                }
            }

            return programas;
        }

        public ActionResult EstablecerDuracionQR()
        {
            return View();
        }
        
        public ActionResult GenerarReporteInasistencias()
        {
            return View();
        }

        private List<ReporteInasistencias> ObtenerReporteInasistencias(DateTime fechaInicio, DateTime fechaFin)
        {
            List<ReporteInasistencias> reporte = new List<ReporteInasistencias>();

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = @"
                SELECT id_reporte, tipo_reporte, COUNT(*) AS cantidad_reporte, fecha_registro, codigo_asistencia, id_usuario
                FROM reporte
                WHERE fecha_registro >= @FechaInicio AND fecha_registro <= @FechaFin
                GROUP BY id_reporte, fecha_registro, tipo_reporte, codigo_asistencia, id_usuario";

                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@FechaInicio", fechaInicio);
                command.Parameters.AddWithValue("@FechaFin", fechaFin);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    ReporteInasistencias item = new ReporteInasistencias
                    {
                        id_reporte = (int)reader["id_reporte"],
                        tipo_reporte = reader["tipo_reporte"].ToString(),
                        Cantidad_reporte = (int)reader["cantidad_reporte"],
                        Fecha_registro = (DateTime)reader["fecha_registro"],
                        Codigo_asistencia = (int)reader["codigo_asistencia"],
                        Id_usuario = (int)reader["id_usuario"],
                    };
                    reporte.Add(item);
                }
            }

            return reporte;
        }

        public ActionResult VerReporte(DateTime fechaInicio, DateTime fechaFin)
        {
            var reporteInasistencias = ObtenerReporteInasistencias(fechaInicio, fechaFin);
            ReportDocument reportDocument = new ReportDocument();
            reportDocument.Load(Server.MapPath("~/Reports/ReporteInasistencias.rpt"));
            reportDocument.SetDataSource(reporteInasistencias);

            Stream stream = reportDocument.ExportToStream(ExportFormatType.PortableDocFormat);
            stream.Seek(0, SeekOrigin.Begin);

            return File(stream, "application/pdf");
        }

        public ActionResult RegistrarAsistencia()
        {
            return View();
        }

        public ActionResult GenerarQR()
        {
            // Obtener las horas de clase guardadas
            if (classStartTime.HasValue && classEndTime.HasValue)
            {
                TimeSpan currentTime = DateTime.Now.TimeOfDay;

                if (currentTime < classStartTime.Value || currentTime > classEndTime.Value)
                {
                    qrEnabled = false; // Deshabilitar QR fuera de las horas de clase
                }
                else
                {
                    qrEnabled = true;
                }
            }

            // Generar un nuevo código QR cada vez que se llama a este método si no está deshabilitado
            if (currentQRData == null || (DateTime.Now - lastQRUpdateTime).TotalMinutes > 60 || !qrEnabled)
            {
                // Aquí puedes generar un enlace de Google Form con parámetros específicos si lo deseas
                string googleFormUrl = "https://docs.google.com/forms/d/e/1FAIpQLScoP2XCRy2oDlWi0ujw8Q8nLOoiTla-_snELeiv30jhMc42BQ/viewform?usp=sf_link" + Guid.NewGuid().ToString();
                currentQRData = googleFormUrl;
                lastQRUpdateTime = DateTime.Now;
                qrEnabled = true;
            }

            if (!qrEnabled)
            {
                return new HttpStatusCodeResult(403, "QR deshabilitado fuera de las horas de clase.");
            }

            QRCodeGenerator qrGenerator = new QRCodeGenerator();
            QRCodeData qrCodeData = qrGenerator.CreateQrCode(currentQRData, QRCodeGenerator.ECCLevel.Q);
            QRCode qrCode = new QRCode(qrCodeData);

            Bitmap qrCodeImage = qrCode.GetGraphic(10);

            using (MemoryStream stream = new MemoryStream())
            {
                qrCodeImage.Save(stream, System.Drawing.Imaging.ImageFormat.Png);
                byte[] imageBytes = stream.ToArray();

                return File(imageBytes, "image/png");
            }
        }

        [HttpPost]
        public JsonResult DeshabilitarQR()
        {
            qrEnabled = false;
            return Json(new { success = true, message = "Código QR deshabilitado." });
        }

        [HttpPost]
        public JsonResult GuardarHorasClase(string classStartTime, string classEndTime)
        {
            InstructorController.classStartTime = TimeSpan.Parse(classStartTime);
            InstructorController.classEndTime = TimeSpan.Parse(classEndTime);

            return Json(new { success = true, message = "Horas de clase guardadas." });
        }

        [HttpPost]
        public JsonResult EstablecerDiasParaExcusa(int dias)
        {
            int instructorId = ObtenerIdInstructor();

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = "UPDATE usuarios SET dias_para

_excusa = @dias WHERE id_usuario = @instructorId";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@dias", dias);
                command.Parameters.AddWithValue("@instructorId", instructorId);

                connection.Open();
                command.ExecuteNonQuery();
            }

            return Json(new { success = true, message = "Días para subir excusa actualizados." });
        }

        private List<Ficha> ObtenerFichas(int instructorId)
        {
            List<Ficha> fichas = new List<Ficha>();

            using (SqlConnection connection = new SqlConnection(Conexion))
            {
                string query = @"SELECT ficha.numero_ficha, tipo_ficha, jornada, estado_ficha, ficha.fecha_registro, foto_ficha
                                FROM usuarios, ficha
                                WHERE usuarios.id_usuario = @id_usuario AND usuarios.numero_ficha = ficha.numero_ficha ";
                SqlCommand command = new SqlCommand(query, connection);
                command.Parameters.AddWithValue("@id_usuario", instructorId);

                connection.Open();
                SqlDataReader reader = command.ExecuteReader();
                while (reader.Read())
                {
                    Ficha ficha = new Ficha
                    {
                        NumeroFicha = reader["numero_ficha"].ToString(),
                        TipoFicha = reader["tipo_ficha"].ToString(),
                        Jornada = reader["jornada"].ToString(),
                        FotoFicha = reader["foto_ficha"].ToString(),
                        EstadoFicha = reader["estado_ficha"].ToString(),
                        FechaRegistro = (DateTime)reader["fecha_registro"]
                    };
                    fichas.Add(ficha);
                }
            }

            return fichas;
        }

        public int ObtenerIdInstructor()
        {
            if (User.Identity.IsAuthenticated)
            {
                string correo_usuario = User.Identity.Name;
                int id_usuario = 0;

                string query = "SELECT id_usuario FROM usuarios WHERE correo = @correo";

                using (SqlConnection connection = new SqlConnection(Conexion))
                {
                    SqlCommand command = new SqlCommand(query, connection);
                    command.Parameters.AddWithValue("@correo", correo_usuario);

                    try
                    {
                        connection.Open();
                        SqlDataReader reader = command.ExecuteReader();
                        if (reader.Read())
                        {
                            id_usuario = reader.GetInt32(0);
                        }
                        reader.Close();
                    }
                    catch (Exception)
                    {
                        // Handle exception
                    }
                }

                return id_usuario;
            }
            else
            {
                return 0;
            }
        }
    }
}
```

### Explanation:
1. **Database Column Addition**: The column `dias_para_excusa` is added to store the number of days allowed for submitting an excuse.
2. **Setting the Deadline**: The method `EstablecerDiasParaExcusa` allows the instructor to set the number of days students have to submit their excuse.
3. **Obtaining the Instructor ID**: The `ObtenerIdInstructor` method ensures actions are taken based on the currently logged-in instructor.

### Usage:
- To set the deadline, the instructor can call the `EstablecerDiasParaExcusa` action with the desired number of days.
- When retrieving and processing reports or absence excuses, you can use the `dias_para_excusa` value to enforce the deadline logic.

You will need to create corresponding views and ensure that the front-end calls these actions appropriately, but this code provides the backend foundation for setting and utilizing the 3-day excuse submission window.