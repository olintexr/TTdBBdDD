---

# Procedimientos y Funciones en SQL  
## Capítulo 1 — Tópicos de Bases de Datos

Este capítulo introduce los **procedimientos almacenados** y las **funciones**, dos mecanismos esenciales para encapsular lógica dentro de la base de datos.  
El enfoque principal será **SQL Server**, pero los estudiantes pueden practicar también en **MySQL** o **PostgreSQL**, entendiendo que la sintaxis puede variar ligeramente.

---

# 1. Idea central

Los **procedimientos almacenados (Stored Procedures)** y las **funciones** permiten que la base de datos deje de ser un simple contenedor de tablas y se convierta en un **entorno programable**.  
Esto significa que parte de la lógica del sistema puede vivir dentro del motor, cerca de los datos, con beneficios en rendimiento, seguridad y consistencia.

---

# 2. Conceptos fundamentales

A continuación se presentan los conceptos esenciales que un estudiante debe dominar antes de escribir su primer procedimiento.

---

## 2.1 Procedimiento almacenado (Stored Procedure)

Un **procedimiento almacenado** es un programa guardado dentro de la base de datos.  
Puede ejecutar múltiples instrucciones SQL, recibir parámetros, manejar errores, realizar validaciones y modificar datos.

Características principales:

- Se almacena en el servidor.  
- Puede recibir parámetros opcionales u obligatorios.  
- Puede ejecutar operaciones de lectura y escritura.  
- Puede contener ciclos, condiciones, variables y bloques TRY/CATCH.  
- No está obligado a devolver un valor.  
- Puede ser llamado desde aplicaciones, APIs o desde otros procedimientos.

En SQL Server se crea con:

```sql
CREATE OR ALTER PROCEDURE nombre
AS
BEGIN
    -- instrucciones
END
```

---

## 2.2 Función

Una **función** es un bloque de código que recibe parámetros y devuelve un valor.  
A diferencia de los procedimientos:

- Debe devolver un valor (escalar o tabla).  
- No puede modificar datos en SQL Server.  
- Se usa dentro de consultas, SELECT, WHERE, JOIN, etc.  
- Es ideal para cálculos, transformaciones y validaciones.

Ejemplo conceptual:

```sql
CREATE FUNCTION f_suma (@a INT, @b INT)
RETURNS INT
AS
BEGIN
    RETURN @a + @b;
END
```

---

## 2.3 Datos de conexión (concepto esencial)

Antes de que una aplicación pueda ejecutar procedimientos o funciones, necesita **conectarse** al motor de base de datos.  
Esto requiere cuatro elementos básicos:

1. **Servidor**  
   Puede ser una dirección IP, un nombre de máquina o un servicio en la nube.  
   Ejemplo:  
   - `localhost`  
   - `192.168.1.50`  
   - `sqlserver.miempresa.com`

2. **Base de datos**  
   Es el contenedor lógico donde viven las tablas, procedimientos y funciones.  
   Ejemplo:  
   - `pruebas_sp`  
   - `ventas2024`

3. **Usuario**  
   Identidad que se conecta al servidor.  
   Puede ser un usuario SQL o un usuario de Windows.

4. **Contraseña**  
   Clave asociada al usuario.

En la práctica profesional, la aplicación **no se conecta directamente al servidor**, sino a una **API** que actúa como intermediaria.  
La API valida permisos, aplica reglas de negocio y evita exponer el servidor directamente a internet.

---

## 2.4 ¿Por qué existen los procedimientos y funciones?

- Para **reutilizar lógica** sin reescribir código.  
- Para **centralizar reglas de negocio** dentro de la base de datos.  
- Para **mejorar rendimiento**, reduciendo viajes entre aplicación y servidor.  
- Para **proteger la base de datos**, controlando qué operaciones están permitidas.  
- Para **mantener consistencia** en operaciones repetitivas.  
- Para **automatizar tareas** que deben ejecutarse de forma controlada.

---

## 2.5 ¿Cuándo usarlos?

- Cuando una operación se repite muchas veces.  
- Cuando se requiere validar datos antes de insertarlos.  
- Cuando se necesita manejar errores.  
- Cuando se desea encapsular lógica compleja.  
- Cuando se quiere exponer una API interna desde la base de datos.  
- Cuando se necesita garantizar que una operación siempre se ejecute igual.

---

# 3. Ejemplos prácticos en SQL Server

A continuación se presentan ejemplos ejecutables en SQL Server Management Studio (SSMS).  
Los estudiantes pueden adaptarlos a MySQL o PostgreSQL con cambios mínimos.

---

## 3.1 Crear base de datos y procedimiento simple

```sql
create database pruebas_sp
go

use pruebas_sp 
go

create or alter procedure sp_prueba as 
begin
    print('Hola')
end
go

exec sp_prueba
go
```

---

## 3.2 Crear tabla de ejemplo

```sql
create table persona (
    cedula integer not null primary key,
    nombre varchar(32) not null,
    fecha date not null
)
go
```

---

## 3.3 Procedimiento para insertar personas con manejo de errores

```sql
CREATE or alter PROCEDURE sp_insertar_persona
    @cedula_p INT,
    @nombre VARCHAR(32),
    @fecha DATE
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        select 'a' as mensaje, * from persona;

        INSERT INTO persona (cedula, nombre, fecha)
        VALUES (@cedula_p, @nombre, @fecha);

        select 'b' as mensaje, * from persona;
    END TRY

    BEGIN CATCH
        PRINT 'Error al insertar el registro.';
        PRINT 'Mensaje de error: ' + ERROR_MESSAGE();
        PRINT 'Número de error: ' + CAST(ERROR_NUMBER() AS VARCHAR(10));
        PRINT 'Severidad: ' + CAST(ERROR_SEVERITY() AS VARCHAR(10));
        PRINT 'Estado: ' + CAST(ERROR_STATE() AS VARCHAR(10));
        PRINT 'Línea: ' + CAST(ERROR_LINE() AS VARCHAR(10));
    END CATCH
END
GO
```

### Ejecuciones

```sql
EXEC sp_insertar_persona 12345678, 'Juan Pérez', '2020-10-25';
go

EXEC sp_insertar_persona 12345678, 'Juan Pérez', '2020-10-25'; -- error por duplicación
go

EXEC sp_insertar_persona 12345679, 'Andrea Carter', '2020-10-25';
go

EXEC sp_insertar_persona 33345679, 'Tony Stark', '2000-10-25';
go

EXEC sp_insertar_persona 99345679, 'Natasha Romanoff', '1990-11-20';
go
```

---

## 3.4 Ciclos en procedimientos

### Ciclo simple

```sql
CREATE OR ALTER PROCEDURE sp_ciclo_simple
    @maximo INT = 7
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @i INT = 1;

    WHILE @i <= @maximo
    BEGIN
        PRINT 'Valor actual: ' + CAST(@i AS VARCHAR(10));
        SET @i = @i + 1;
    END
END
GO

exec sp_ciclo_simple
go
```

### Números pares

```sql
CREATE OR ALTER PROCEDURE sp_numeros_pares
AS
BEGIN
    DECLARE @i INT = 1;

    WHILE @i <= 10
    BEGIN
        IF @i % 2 = 0
            PRINT 'Número par: ' + CAST(@i AS VARCHAR(10));
        SET @i += 1;
    END
END
GO

exec sp_numeros_pares
go
```

---

## 3.5 Condiciones

```sql
CREATE OR ALTER PROCEDURE sp_evaluar_numero
    @numero INT
AS
BEGIN
    SET NOCOUNT ON;

    IF @numero > 0
        PRINT cast (@numero as varchar) + ' es positivo.';
    ELSE IF @numero < 0
        PRINT cast (@numero as varchar) + ' es negativo.';
    ELSE
        PRINT cast (@numero as varchar) + ' es cero.';
END
GO

exec sp_evaluar_numero 100
go 
exec sp_evaluar_numero -99
go 
exec sp_evaluar_numero 0
go
```

---

## 3.6 Consultar persona

```sql
CREATE OR ALTER PROCEDURE sp_consultar_persona
    @cedula_p INT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        IF EXISTS (SELECT 1 FROM persona WHERE cedula = @cedula_p)
        BEGIN
            SELECT * FROM persona WHERE cedula = @cedula_p;
        END
        ELSE
        BEGIN
            PRINT 'No se encontró una persona con la cédula especificada.';
        END
    END TRY

    BEGIN CATCH
        PRINT 'Error al consultar el registro.';
        PRINT 'Mensaje de error: ' + ERROR_MESSAGE();
        PRINT 'Número de error: ' + CAST(ERROR_NUMBER() AS VARCHAR(10));
        PRINT 'Severidad: ' + CAST(ERROR_SEVERITY() AS VARCHAR(10));
        PRINT 'Estado: ' + CAST(ERROR_STATE() AS VARCHAR(10));
        PRINT 'Línea: ' + CAST(ERROR_LINE() AS VARCHAR(10));
    END CATCH
END
GO

exec sp_consultar_persona 33345679
go 
exec sp_consultar_persona 99345679
go
exec sp_consultar_persona 55345679
go
```

---

# 4. Bibliografía recomendada

**Silberschatz, Abraham; Korth, Henry; Sudarshan, S. — *Database System Concepts*.**  
Referencia fundamental para comprender los conceptos centrales de bases de datos: modelos, SQL, transacciones, concurrencia y arquitectura. Es la base teórica principal para entender el contexto donde operan los procedimientos y funciones.

**Elmasri, Ramez; Navathe, Shamkant — *Fundamentals of Database Systems*.**  
Complementa a Silberschatz con mayor profundidad en diseño conceptual, normalización y modelos. Es ideal para reforzar fundamentos antes de avanzar hacia programación en SQL.

**Itzik Ben‑Gan — *T‑SQL Fundamentals*.**  
El mejor punto de partida práctico para aprender T‑SQL. Explica con claridad procedimientos, funciones, variables, control de flujo y manejo de errores. Es la lectura prioritaria para este capítulo.

**Microsoft Learn – SQL Server Documentation — https://learn.microsoft.com/sql**  
Referencia oficial, precisa y actualizada. Es indispensable para consultar sintaxis exacta, ejemplos y comportamiento del motor mientras se desarrollan procedimientos y funciones.

**Garcia‑Molina, Hector; Ullman, Jeffrey; Widom, Jennifer — *Database Systems: The Complete Book*.**  
Más formal y profundo. Excelente para quienes quieran entender teoría avanzada, transacciones, recuperación y concurrencia. Recomendado después de dominar los fundamentos.

**Itzik Ben‑Gan — *Inside Microsoft SQL Server: T‑SQL Programming*.**  
Lectura avanzada enfocada en optimización, comportamiento interno del motor y técnicas profesionales de T‑SQL. Ideal para estudiantes que ya dominan lo básico.

**Joe Celko — *SQL for Smarties*.**  
Libro desafiante orientado a problemas complejos y soluciones elegantes en SQL. Recomendado para estudiantes avanzados.

**Markus Winand — *SQL Performance Explained*.**  
Guía clara para entender índices, planes de ejecución y optimización. Muy útil para quienes quieran escribir código eficiente.

**PostgreSQL Documentation — https://www.postgresql.org/docs/**  
Referencia oficial para comparar sintaxis y conceptos en un motor libre.

**MySQL Reference Manual — https://dev.mysql.com/doc/**  
Permite contrastar diferencias en procedimientos, funciones y manejo de errores en MySQL.

---

# 5. Asignación

1. Crear un procedimiento almacenado que borre una persona si existe.  
2. Crear un procedimiento almacenado que actualice nombre y fecha si la persona existe.  
3. Mejorar el procedimiento de inserción para evitar duplicados.  
4. Crear un procedimiento combinado: insertar si no existe, actualizar si existe.  
5. Crear un procedimiento que devuelva el número total de personas usando `RETURN` o un parámetro `OUTPUT`.

---

# Reglas de entrega de la asignación

**Subject del correo:**  
TAREA 02 - PROCEDIMIENTOS EN SQL - APELLIDOS Y NOMBRES COMPLETOS DEL ALUMNO EN MAYÚSCULA

Ejemplo:  
PEREZ BONALDE, JUAN ANTONIO

**Nombre del archivo:**  
TAREA 02 - PROCEDIMIENTOS EN SQL - APELLIDOS Y NOMBRES COMPLETOS DEL ALUMNO EN MAYÚSCULA.sql

Enviar el script al correo electrónico indicado en la sesión de clases.

---

## Reglas para la entrega del archivo

Estas reglas son obligatorias y forman parte de la evaluación.  
El incumplimiento de cualquiera de ellas afecta la nota final.

1. **El archivo debe ser un script SQL (.sql).**  
   No se aceptan Word, PDF, imágenes, capturas de pantalla, fotos de cuadernos ni ningún otro formato.

2. **El archivo debe contener únicamente las instrucciones SQL solicitadas en la asignación.**  
   No debe incluir explicaciones, comentarios extensos, portadas, dedicatorias, texto decorativo ni contenido ajeno al código.

3. **El archivo no debe incluir el nombre del estudiante dentro del contenido.**  
   El nombre completo va únicamente en el asunto del correo y en el nombre del archivo, según el formato indicado.

4. **El archivo debe comenzar directamente con el código SQL.**  
   No debe incluir encabezados, títulos, introducciones ni texto adicional.

5. **El script debe ser escrito y editado directamente en un editor de texto o en el entorno SQL correspondiente.**  
   No se aceptan digitalizaciones, fotos ni capturas de pantalla del código.

6. **El archivo debe enviarse con el nombre correcto y en el formato correcto.**  
   Esto forma parte de la evaluación.

7. **El estudiante debe verificar que el script ejecuta sin errores en su entorno antes de enviarlo.**  
   Enviar código que no compila o no ejecuta también afecta la calificación.

