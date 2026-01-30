# Examen-Parcial-2

Solucion: 
```Scala
package Programacion

import cats.effect.{IO, IOApp}
import cats.implicits._
import doobie._
import doobie.implicits._
import fs2.text
import fs2.io.file.{Files, Path}
import fs2.data.csv._
import fs2.data.csv.generic.semiauto._


// 1. MODELO
case class Candidatos(
                         ID: Int,
                         Candidato: String,
                         Partido_Politico: String,
                         Evento: String,
                         Fecha_Evento: String,
                         Ubicacion: String,
                         Asistentes_Estimados: Int,
                         Campana_Activa: Boolean
                       )

given CsvRowDecoder[Candidatos, String] = deriveCsvRowDecoder[Candidatos]


// 3. BASE DE DATOS

object BaseDeDatos {
  val xa = Transactor.fromDriverManager[IO](
    driver = "oracle.jdbc.OracleDriver",
    url    = "jdbc:oracle:thin:@localhost:1521/XEPDB1",
    user   = "Candidatos",
    password = "Candidatos",
    logHandler = None
  )

  def crearTabla: ConnectionIO[Int] =
    sql"""
      CREATE TABLE Candidatos (
        ID NUMERIC,
        Candidato VARCHAR(50),
        Partido_Politico VARCHAR(100),
        Evento VARCHAR(100),
        Fecha_Evento VARCHAR(20),
        Ubicacion VARCHAR(100),
        Asistentes_Estimados NUMERIC,
        Campana_Activa NUMERIC
      )
    """.update.run

  def insertar(m: Candidatos): ConnectionIO[Int] =
    sql"""
      INSERT INTO Candidatos (ID, Candidato, Partido_Politico, Evento, Fecha_Evento,Ubicacion,Asistentes_Estimados,Campana_Activa)
      VALUES (${m.ID}, ${m.Candidato}, ${m.Partido_Politico}, ${m.Evento}, ${m.Fecha_Evento}, ${m.Ubicacion}, ${m.Asistentes_Estimados}, ${m.Campana_Activa})
    """.update.run

  // --- Listar ---
  def listar: ConnectionIO[List[Candidatos]] =
    sql"SELECT * FROM Candidatos"
      .query[Candidatos]
      .to[List]
}


// PROGRAMA PRINCIPAL

object Examen extends IOApp.Simple {

  val filePath = Path(
    "C:\\Users\\Usuario iTC\\Documents\\PROGRAMACION FUN Y REAC\\Proyecto Integrador\\src\\main\\resources\\data\\politica.csv"
  )

  val run: IO[Unit] = {
    val lecturaCSV: IO[List[Candidatos]] =
      Files[IO]
        .readAll(filePath)
        .through(text.utf8.decode)
        .through(decodeUsingHeaders[Candidatos](','))
        .compile
        .toList

    for {
      // Lectura
      _ <- IO.println("--- 1. Leyendo Archivo CSV ---")
      datos <- lecturaCSV
      _ <- IO.println(s"--> Datos en memoria: ${datos.length}")

      // --- Insercion ---
      _ <- IO.println("\n--- 2. Insertando en Oracle ---")
      _ <- BaseDeDatos.crearTabla.transact(BaseDeDatos.xa).attempt
      filas <- datos.traverse(d => BaseDeDatos.insertar(d).transact(BaseDeDatos.xa))
      _ <- IO.println(s"EXITO: Se guardaron ${filas.size} registros.")

      // --- Verificacion de Guardado ---
      _ <- IO.println("\n--- 3. Consultando Registros desde Oracle ---")
      registrosBD <- BaseDeDatos.listar.transact(BaseDeDatos.xa)

      _ <- IO(registrosBD.foreach(r => println(s"  -> ${r.ID} | ${r.Candidato} | ${r.Partido_Politico} | ${r.Evento} | ${r.Fecha_Evento} | ${r.Ubicacion} | ${r.Asistentes_Estimados} | ${r.Campana_Activa}")))
}
```
<img width="1197" height="554" alt="image" src="https://github.com/user-attachments/assets/0063e21e-ef33-44b1-93ab-1af67fcea411" />
<img width="1585" height="994" alt="image" src="https://github.com/user-attachments/assets/82e18f35-d2a6-48b0-992a-ee6cb4211f37" />



    } yield ()
  }
}
