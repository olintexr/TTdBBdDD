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

---

- **Se almacena en el servidor.**  
  Esto significa que el procedimiento no vive en un archivo externo ni en la aplicación, sino dentro del propio motor de base de datos. El servidor lo conserva, lo compila, lo optimiza y lo ejecuta cuando se le solicita. Esto garantiza disponibilidad, consistencia y rendimiento, porque la lógica está físicamente cerca de los datos.

- **Puede recibir parámetros opcionales u obligatorios.**  
  Los parámetros permiten que el mismo procedimiento se ejecute con diferentes valores sin reescribir código. Algunos pueden ser obligatorios (por ejemplo, una cédula), mientras que otros pueden tener valores por defecto, lo que permite ejecutar el procedimiento incluso si no se especifican todos los argumentos.

- **Puede ejecutar operaciones de lectura y escritura.**  
  Un procedimiento puede consultar datos (SELECT), insertar (INSERT), modificar (UPDATE) o eliminar (DELETE). Esta capacidad lo convierte en una herramienta central para encapsular reglas de negocio y controlar cómo se manipula la información dentro del sistema.

- **Puede contener ciclos, condiciones, variables y bloques TRY/CATCH.**  
  Los procedimientos permiten programar lógica completa: repetir acciones con WHILE, tomar decisiones con IF/ELSE, almacenar valores temporales en variables y manejar errores con TRY/CATCH. Esto los convierte en unidades de programación completas dentro del motor.

- **No está obligado a devolver un valor.**  
  A diferencia de las funciones, un procedimiento puede ejecutar acciones sin retornar un resultado formal. Puede imprimir mensajes, modificar datos o realizar validaciones sin necesidad de producir un valor final. Esto lo hace flexible para tareas operativas.

- **Puede ser llamado desde aplicaciones, APIs o desde otros procedimientos.**  
  Los procedimientos pueden integrarse en cualquier capa del sistema. Una aplicación web, una API, un servicio backend o incluso otro procedimiento pueden invocarlo. Esto permite construir arquitecturas modulares donde la base de datos expone operaciones controladas y reutilizables.

En SQL Server se crea con:

```sql
CREATE OR ALTER PROCEDURE nombre
AS
BEGIN
    -- instrucciones
END
```

Las funciones se crean de manera similar. 

---

## 2.2 Función

Una **función** es un bloque de código que recibe parámetros y devuelve un valor.  
A diferencia de los procedimientos:

- **Debe devolver un valor (escalar o tabla).**  
  Una función siempre produce un resultado final: un número, una cadena, una fecha o incluso una tabla completa. Ese valor puede usarse inmediatamente dentro de una consulta. Esta obligación de devolver algo la convierte en una herramienta ideal para cálculos y transformaciones que deben integrarse directamente en el flujo de una sentencia SQL.

- **No puede modificar datos en SQL Server.**  
  Las funciones están diseñadas para ser deterministas y seguras dentro de una consulta. Por eso no pueden ejecutar INSERT, UPDATE o DELETE. Su propósito es calcular, transformar o evaluar, no alterar el estado de la base de datos. Esta restricción garantiza que puedan usarse sin riesgo dentro de SELECT, JOIN o WHERE.

- **Se usa dentro de consultas, SELECT, WHERE, JOIN, etc.**  
  A diferencia de los procedimientos, las funciones pueden integrarse como parte de una expresión. Pueden aparecer en un SELECT para calcular un valor, en un WHERE para filtrar, en un JOIN para relacionar datos o incluso en un ORDER BY. Esto las convierte en componentes reutilizables dentro del lenguaje declarativo.

- **Es ideal para cálculos, transformaciones y validaciones.**  
  Las funciones encapsulan lógica que se repite: formatear textos, calcular edades, validar rangos, transformar fechas, normalizar valores, etc. Son especialmente útiles cuando la misma operación debe ejecutarse en múltiples consultas, garantizando consistencia y evitando duplicación de código.

---

Ejemplo conceptual:

```sql
CREATE FUNCTION f_suma (@a INT, @b INT)
RETURNS INT
AS
BEGIN
    RETURN @a + @b;
END
GO


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


- **Para reutilizar lógica sin reescribir código.**  
  Cuando una operación se repite en distintos lugares del sistema, encapsularla en un procedimiento o función evita duplicación y reduce errores. En lugar de copiar y pegar la misma instrucción en múltiples consultas o módulos, se centraliza en un solo lugar. Esto facilita mantenimiento, actualizaciones y coherencia en el comportamiento del sistema.

- **Para centralizar reglas de negocio dentro de la base de datos.**  
  Las reglas que determinan cómo debe operar un proceso (validaciones, restricciones, cálculos, condiciones) pueden residir en la base de datos en lugar de dispersarse en distintas aplicaciones. Esto garantiza que, sin importar desde dónde se invoque la operación, la lógica se ejecuta de manera uniforme y controlada.

- **Para mejorar rendimiento, reduciendo viajes entre aplicación y servidor.**  
  Ejecutar varias operaciones dentro de un procedimiento evita enviar múltiples instrucciones desde la aplicación. En lugar de varios viajes de ida y vuelta, el servidor recibe una sola llamada y ejecuta todo internamente. Esto disminuye latencia, reduce carga en la red y mejora tiempos de respuesta.

- **Para proteger la base de datos, controlando qué operaciones están permitidas.**  
  En lugar de dar acceso directo a tablas, se otorgan permisos para ejecutar procedimientos específicos. Esto limita lo que un usuario o una aplicación puede hacer, evitando modificaciones indebidas y reduciendo riesgos de seguridad. Los procedimientos actúan como una capa de control entre el usuario y los datos.

- **Para mantener consistencia en operaciones repetitivas.**  
  Si una operación debe ejecutarse siempre de la misma manera (por ejemplo, insertar un registro con validaciones previas), un procedimiento garantiza que el proceso no varíe según quién lo ejecute o desde qué aplicación se invoque. Esto evita errores humanos y asegura uniformidad en los resultados.

- **Para automatizar tareas que deben ejecutarse de forma controlada.**  
  Procesos como auditorías, cálculos periódicos, limpiezas de datos o transformaciones pueden encapsularse en procedimientos que se ejecutan manualmente o mediante programación. Esto permite que tareas complejas se realicen de forma predecible, ordenada y sin intervención constante del usuario.

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

-- Va igual, pero con parámetros, atender a las llamadas.
create or alter procedure sp_prueba (@mensaje varchar(20) = 'Nada') as  
begin
    print('Hola, ' + @mensaje )
end
go


exec sp_prueba @mensaje= 'Villalobos'
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

