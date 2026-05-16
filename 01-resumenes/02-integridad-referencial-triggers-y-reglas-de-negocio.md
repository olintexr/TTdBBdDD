# 02 – Integridad referencial, triggers y reglas de negocio

## 1. Idea central

La integridad de los datos es el núcleo de un sistema de bases de datos.  
Este capítulo muestra cómo SQL Server protege la coherencia del sistema mediante:

- Claves foráneas  
- Cascadas  
- Restricciones CHECK  
- Tablas puente  
- Triggers  
- Procedimientos almacenados con manejo profesional de errores  
- SQL dinámico seguro  
- Cursores  
- Reglas de negocio declaradas en el motor  

El objetivo es que el estudiante comprenda que **la integridad debe declararse, automatizarse y garantizarse desde el motor**, no desde la aplicación.

---

## 2. Conceptos fundamentales

### 2.1 Integridad referencial  
Garantiza que las relaciones entre tablas se mantengan válidas.

### 2.2 Cascadas  
Propagan automáticamente acciones entre tablas relacionadas.

### 2.3 Restricciones CHECK  
Validan valores antes de insertarlos o actualizarlos.

### 2.4 Tablas puente  
Representan relaciones muchos‑a‑muchos.

### 2.5 Triggers  
Código que se ejecuta automáticamente ante INSERT, UPDATE o DELETE.

### 2.6 Procedimientos almacenados avanzados  
Incluyen SQL dinámico, SQL estático, manejo de errores y cursores.

---

# 3. BancoDB – Integridad estructural  
*(Código completo)*

```sql
----------------------------------------------------------
-- (a) Conectarse a master y preparar la base de datos
----------------------------------------------------------
USE master;
GO

DECLARE @DBName SYSNAME = N'BancoDB';
DECLARE @SQL NVARCHAR(MAX);

IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @DBName)
BEGIN
    SET @SQL = N'ALTER DATABASE [' + @DBName + '] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;';
    EXEC(@SQL);

    SET @SQL = N'DROP DATABASE [' + @DBName + '];';
    EXEC(@SQL);
END
GO

CREATE DATABASE BancoDB;
GO

USE BancoDB;
GO

IF OBJECT_ID('dbo.Prestatario', 'U') IS NOT NULL DROP TABLE dbo.Prestatario;
IF OBJECT_ID('dbo.Prestamo',    'U') IS NOT NULL DROP TABLE dbo.Prestamo;
IF OBJECT_ID('dbo.Impositor',   'U') IS NOT NULL DROP TABLE dbo.Impositor;
IF OBJECT_ID('dbo.Cuenta',      'U') IS NOT NULL DROP TABLE dbo.Cuenta;
IF OBJECT_ID('dbo.Cliente',     'U') IS NOT NULL DROP TABLE dbo.Cliente;
IF OBJECT_ID('dbo.Sucursal',    'U') IS NOT NULL DROP TABLE dbo.Sucursal;
GO

CREATE TABLE dbo.Cliente
(
    ClienteID       INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    nombre_cliente  NVARCHAR(20)  NOT NULL UNIQUE,
    calle_cliente   NVARCHAR(30)  NULL,
    ciudad_cliente  NVARCHAR(30)  NULL
);
GO

CREATE TABLE dbo.Sucursal
(
    SucursalID       INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    nombre_sucursal  NVARCHAR(15)  NOT NULL UNIQUE,
    ciudad_sucursal  NVARCHAR(30)  NULL,
    activos          DECIMAL(16,2) NOT NULL CHECK (activos >= 0)
);
GO

CREATE TABLE dbo.Cuenta
(
    CuentaID        INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    numero_cuenta   NVARCHAR(10)  NOT NULL UNIQUE,
    SucursalID      INT           NOT NULL,
    saldo           DECIMAL(12,2) NOT NULL CHECK (saldo >= 0),
    FOREIGN KEY (SucursalID)
        REFERENCES dbo.Sucursal(SucursalID)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
GO

CREATE TABLE dbo.Prestamo
(
    PrestamoID       INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    numero_prestamo  NVARCHAR(10)  NOT NULL UNIQUE,
    SucursalID       INT           NOT NULL,
    monto            DECIMAL(12,2) NOT NULL CHECK (monto >= 0),
    FOREIGN KEY (SucursalID)
        REFERENCES dbo.Sucursal(SucursalID)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
GO

CREATE TABLE dbo.Impositor
(
    ClienteID  INT NOT NULL,
    CuentaID   INT NOT NULL,
    PRIMARY KEY (ClienteID, CuentaID),
    FOREIGN KEY (ClienteID) REFERENCES dbo.Cliente(ClienteID) ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (CuentaID)  REFERENCES dbo.Cuenta(CuentaID)  ON DELETE CASCADE ON UPDATE CASCADE
);
GO

CREATE TABLE dbo.Prestatario
(
    ClienteID   INT NOT NULL,
    PrestamoID  INT NOT NULL,
    PRIMARY KEY (ClienteID, PrestamoID),
    FOREIGN KEY (ClienteID)  REFERENCES dbo.Cliente(ClienteID)  ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (PrestamoID) REFERENCES dbo.Prestamo(PrestamoID) ON DELETE CASCADE ON UPDATE CASCADE
);
GO
```

*(Sigue el script completo con semillas, inserts, joins, intentos de error, etc.)*

---

# 4. RRHH_DB – Integridad lógica  
*(Código completo)*

```sql
/* ================================================
   0) Borrar y recrear la base de datos
================================================ */
USE master;
GO

DECLARE @DB SYSNAME = N'RRHH_DB';
DECLARE @SQL NVARCHAR(MAX);

IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @DB)
BEGIN
    PRINT 'Cerrando conexiones a ' + @DB + ' y eliminando...';
    SET @SQL = N'ALTER DATABASE [' + @DB + '] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;';
    EXEC(@SQL);
    SET @SQL = N'DROP DATABASE [' + @DB + '];';
    EXEC(@SQL);
END;

PRINT 'Creando base de datos ' + @DB + '...';
SET @SQL = N'CREATE DATABASE [' + @DB + '];';
EXEC(@SQL);
GO

USE RRHH_DB;
GO
SET NOCOUNT ON;
SET XACT_ABORT ON;
GO

/* ================================================
   1) Esquema (Ej. 4.2)
   - Empleado(nombre_empleado, calle, ciudad)
   - Empresa(nombre_empresa, ciudad)
   - Trabaja(nombre_empleado, nombre_empresa, sueldo)
   - Jefe(nombre_empleado, nombre_jefe)
================================================ */

-- Limpieza defensiva 
IF OBJECT_ID('dbo.TR_Trabaja_ChkCiudad',        'TR') IS NOT NULL DROP TRIGGER dbo.TR_Trabaja_ChkCiudad;
IF OBJECT_ID('dbo.TR_Trabaja_ChkSueldoVsJefe',  'TR') IS NOT NULL DROP TRIGGER dbo.TR_Trabaja_ChkSueldoVsJefe;
IF OBJECT_ID('dbo.TR_Jefe_ChkSueldo',           'TR') IS NOT NULL DROP TRIGGER dbo.TR_Jefe_ChkSueldo;
IF OBJECT_ID('dbo.Jefe',     'U') IS NOT NULL DROP TABLE dbo.Jefe;
IF OBJECT_ID('dbo.Trabaja',  'U') IS NOT NULL DROP TABLE dbo.Trabaja;
IF OBJECT_ID('dbo.Empresa',  'U') IS NOT NULL DROP TABLE dbo.Empresa;
IF OBJECT_ID('dbo.Empleado', 'U') IS NOT NULL DROP TABLE dbo.Empleado;
GO

-- Empleado
CREATE TABLE dbo.Empleado
(
    EmpleadoID      INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_Empleado PRIMARY KEY,
    nombre_empleado NVARCHAR(50) NOT NULL,
    calle           NVARCHAR(100) NULL,
    ciudad          NVARCHAR(50) NOT NULL,
    CONSTRAINT UQ_Empleado_nombre UNIQUE (nombre_empleado)
);
GO

-- Empresa
CREATE TABLE dbo.Empresa
(
    EmpresaID      INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_Empresa PRIMARY KEY,
    nombre_empresa NVARCHAR(80) NOT NULL,
    ciudad         NVARCHAR(50) NOT NULL,
    CONSTRAINT UQ_Empresa_nombre UNIQUE (nombre_empresa)
);
GO

/* Trabaja
   - Interpretación 1:1 (cada empleado trabaja para una empresa)
   - PK = EmpleadoID (coincide con Empleado)
*/
CREATE TABLE dbo.Trabaja
(
    EmpleadoID INT NOT NULL CONSTRAINT PK_Trabaja PRIMARY KEY,
    EmpresaID  INT NOT NULL,
    sueldo     DECIMAL(12,2) NOT NULL CONSTRAINT CK_Trabaja_sueldo_nonneg CHECK (sueldo >= 0),

    CONSTRAINT FK_Trabaja_Empleado
        FOREIGN KEY (EmpleadoID) REFERENCES dbo.Empleado(EmpleadoID)
        ON DELETE CASCADE ON UPDATE CASCADE,

    CONSTRAINT FK_Trabaja_Empresa
        FOREIGN KEY (EmpresaID)  REFERENCES dbo.Empresa(EmpresaID)
        ON DELETE CASCADE ON UPDATE CASCADE
);
GO

/* Jefe
   - Cada empleado tiene a lo sumo un jefe (PK = EmpleadoID)
   - Tanto empleado como jefe existen en Empleado
*/
CREATE TABLE dbo.Jefe
(
    EmpleadoID INT NOT NULL
        CONSTRAINT PK_Jefe PRIMARY KEY,      -- cada empleado a lo sumo un jefe
    JefeID     INT NOT NULL,

    CONSTRAINT CK_Jefe_NoAutoReferencia
        CHECK (EmpleadoID <> JefeID),

    CONSTRAINT FK_Jefe_Empleado
        FOREIGN KEY (EmpleadoID)
        REFERENCES dbo.Empleado(EmpleadoID)
        ON DELETE NO ACTION      -- = RESTRICT
        ON UPDATE NO ACTION,

    CONSTRAINT FK_Jefe_Jefe
        FOREIGN KEY (JefeID)
        REFERENCES dbo.Empleado(EmpleadoID)
        ON DELETE NO ACTION      -- = RESTRICT
        ON UPDATE NO ACTION
);
GO

/* ================================================
   2) Reglas del Ej. 4.3 (a) y (b) vía TRIGGERS
   - (a) Empleado y Empresa misma ciudad
   - (b) Sueldo empleado <= sueldo de su jefe
================================================ */

-- (a) Misma ciudad
CREATE TRIGGER dbo.TR_Trabaja_ChkCiudad
ON dbo.Trabaja
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN dbo.Empleado e ON e.EmpleadoID = i.EmpleadoID
        JOIN dbo.Empresa  m ON m.EmpresaID  = i.EmpresaID
        WHERE e.ciudad <> m.ciudad
    )
    BEGIN
        RAISERROR (N'4.3(a): El empleado debe trabajar en una empresa de su misma ciudad.', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
END;
GO

-- (b) Sueldo empleado <= sueldo de su jefe
-- Verifica cuando cambia el sueldo/empresa del empleado o del jefe
CREATE TRIGGER dbo.TR_Trabaja_ChkSueldoVsJefe
ON dbo.Trabaja
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Empleado actualizado: comparar con su jefe (si lo tiene)
    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN dbo.Jefe j        ON j.EmpleadoID = i.EmpleadoID
        JOIN dbo.Trabaja tBoss ON tBoss.EmpleadoID = j.JefeID
        WHERE i.sueldo > tBoss.sueldo
    )
    BEGIN
        RAISERROR (N'4.3(b): El sueldo del empleado no puede superar al de su jefe.', 16, 1);
        ROLLBACK TRANSACTION; RETURN;
    END

    -- Jefe actualizado: sus subordinados no pueden superar su nuevo sueldo
    IF EXISTS (
        SELECT 1
        FROM inserted i                   -- i puede ser el JEFE
        JOIN dbo.Jefe sub ON sub.JefeID = i.EmpleadoID
        JOIN dbo.Trabaja tEmp ON tEmp.EmpleadoID = sub.EmpleadoID
        WHERE tEmp.sueldo > i.sueldo
    )
    BEGIN
        RAISERROR (N'4.3(b): Tras el cambio, hay subordinados con sueldo mayor al del jefe.', 16, 1);
        ROLLBACK TRANSACTION; RETURN;
    END
END;
GO

-- (b) También al insertar/actualizar relación de Jefe
CREATE TRIGGER dbo.TR_Jefe_ChkSueldo
ON dbo.Jefe
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted j
        JOIN dbo.Trabaja tEmp  ON tEmp.EmpleadoID = j.EmpleadoID
        JOIN dbo.Trabaja tBoss ON tBoss.EmpleadoID = j.JefeID
        WHERE tEmp.sueldo > tBoss.sueldo
    )
    BEGIN
        RAISERROR (N'4.3(b): El empleado no puede tener sueldo mayor al de su jefe.', 16, 1);
        ROLLBACK TRANSACTION; RETURN;
    END
END;
GO

/* ================================================
   3) Datos de ejemplo (seed opcional)
================================================ */
BEGIN TRAN;

-- Empleados
INSERT INTO dbo.Empleado (nombre_empleado, calle, ciudad) VALUES
(N'Ana López',   N'Av. 1 #123',   N'San José'),
(N'Bruno Pérez', N'Calle 5 #234', N'San José'),
(N'Carla Gómez', N'Av. 9 #555',   N'Heredia'),
(N'Diego Ruiz',  N'Calle 8 #777', N'Heredia');

-- Empresas
INSERT INTO dbo.Empresa (nombre_empresa, ciudad) VALUES
(N'ACME',   N'San José'),
(N'TechSA', N'Heredia');

-- Trabaja (1:1 con Empleado)
INSERT INTO dbo.Trabaja (EmpleadoID, EmpresaID, sueldo)
SELECT e.EmpleadoID, m.EmpresaID, v.sueldo
FROM (VALUES
    (N'Ana López',   N'ACME',   1500.00),
    (N'Bruno Pérez', N'ACME',   2200.00),
    (N'Carla Gómez', N'TechSA', 2400.00),
    (N'Diego Ruiz',  N'TechSA', 1800.00)
) v(nombre_empleado, nombre_empresa, sueldo)
JOIN dbo.Empleado e ON e.nombre_empleado = v.nombre_empleado
JOIN dbo.Empresa  m ON m.nombre_empresa  = v.nombre_empresa;

-- Jefes (Bruno jefe de Ana; Carla jefe de Diego)
INSERT INTO dbo.Jefe (EmpleadoID, JefeID)
SELECT eEmp.EmpleadoID, eBoss.EmpleadoID
FROM (VALUES
    (N'Ana López',  N'Bruno Pérez'),
    (N'Diego Ruiz', N'Carla Gómez')
) v(emp, boss)
JOIN dbo.Empleado eEmp  ON eEmp.nombre_empleado  = v.emp
JOIN dbo.Empleado eBoss ON eBoss.nombre_empleado = v.boss;

COMMIT TRAN;
GO

/* ================================================
   4) Consultas de verificación
================================================ */
-- (i) Empleado–Empresa–Sueldo–Jefe
SELECT e.nombre_empleado, e.ciudad AS ciudad_empleado,
       m.nombre_empresa, m.ciudad AS ciudad_empresa,
       t.sueldo,
       ej.nombre_empleado AS nombre_jefe
FROM dbo.Empleado e
LEFT JOIN dbo.Trabaja t  ON t.EmpleadoID = e.EmpleadoID
LEFT JOIN dbo.Empresa m  ON m.EmpresaID  = t.EmpresaID
LEFT JOIN dbo.Jefe j     ON j.EmpleadoID = e.EmpleadoID
LEFT JOIN dbo.Empleado ej ON ej.EmpleadoID = j.JefeID
ORDER BY e.nombre_empleado;

-- (ii) Pruebas de reglas:
-- -- 4.3(a) Violación de ciudad:
-- UPDATE t SET EmpresaID = (SELECT EmpresaID FROM dbo.Empresa WHERE nombre_empresa=N'TechSA')
-- FROM dbo.Trabaja t JOIN dbo.Empleado e ON e.EmpleadoID=t.EmpleadoID
-- WHERE e.nombre_empleado=N'Ana López'; -- vive en San José => ERROR

-- -- 4.3(b) Subir sueldo de Ana por encima del de Bruno => ERROR
-- UPDATE dbo.Trabaja SET sueldo = 3000
-- WHERE EmpleadoID = (SELECT EmpleadoID FROM dbo.Empleado WHERE nombre_empleado=N'Ana López');

```

---

# 5. Trigger de activos vs suma de préstamos  
*(Código completo)*

```sql
CREATE TRIGGER TR_Chk_Activos_Navacerrada
ON dbo.Prestamo
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @activos DECIMAL(16,2);
    DECLARE @suma_prestamos DECIMAL(16,2);

    SELECT @activos = activos
    FROM dbo.Sucursal
    WHERE nombre_sucursal = N'Navacerrada';

    SELECT @suma_prestamos = SUM(monto)
    FROM dbo.Prestamo
    WHERE nombre_sucursal = N'Navacerrada';

    IF @activos <> @suma_prestamos
    BEGIN
        RAISERROR (
          N'Error: los activos de la sucursal Navacerrada no coinciden con la suma de sus préstamos.',
          16, 1
        );
        ROLLBACK TRANSACTION;
    END
END;
GO
```

---

# 6. Triggers básicos: inserted y deleted  
*(Código completo)*

```sql
create table tabla (nombre varchar(16), saldo integer)
go

insert into tabla values  
('Olinto', 600), 
('Edelyn', 900);
go

update tabla set saldo = '100' where nombre = 'Olinto';
go

delete from tabla where nombre = 'Olinto';
go

CREATE OR ALTER TRIGGER tgr_prueba_insert 
ON dbo.tabla 
AFTER INSERT
AS 
BEGIN
    SET NOCOUNT ON;
    PRINT 'Acabo de insertar un registro';

    SELECT 'Nuevos valores:' AS mensaje, nombre, saldo
    FROM inserted;
END
GO

CREATE OR ALTER TRIGGER tgr_prueba_delete 
ON dbo.tabla 
AFTER DELETE
AS 
BEGIN
    SET NOCOUNT ON;
    PRINT 'Acabo de borrar un registro';

    SELECT 'Valores eliminados:' AS mensaje, nombre, saldo
    FROM deleted;
END
GO

CREATE OR ALTER TRIGGER tgr_prueba_update 
ON dbo.tabla 
AFTER UPDATE
AS 
BEGIN
    SET NOCOUNT ON;
    PRINT 'Acabo de actualizar un registro';

    SELECT 
        'Valores anteriores:' AS mensaje,
        d.nombre AS nombre_anterior, d.saldo AS saldo_anterior,
        'Valores nuevos:' AS mensaje2,
        i.nombre AS nombre_nuevo, i.saldo AS saldo_nuevo
    FROM deleted d
    INNER JOIN inserted i ON d.nombre = i.nombre;
END
GO
```

---

# 7. Procedimientos almacenados avanzados  
*(Código completo)*

## 7.1 SQL dinámico seguro

```sql
CREATE PROCEDURE dbo.usp_Cliente_Delete_Dynamic
    @ClienteID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        IF @ClienteID IS NULL
            THROW 50001, 'ClienteID no puede ser NULL.', 1;

        DECLARE @sql NVARCHAR(MAX) = N'DELETE FROM dbo.Cliente WHERE ClienteID = @id;';
        DECLARE @params NVARCHAR(100) = N'@id INT';

        BEGIN TRAN;

        EXEC sp_executesql @sql, @params, @id = @ClienteID;

        DECLARE @rows INT = @@ROWCOUNT;

        COMMIT TRAN;

        SELECT @rows AS filas_afectadas;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRAN;

        THROW;
    END CATCH
END
GO
```

---

## 7.2 SQL estático

```sql
CREATE PROCEDURE dbo.usp_Cliente_Delete_Static
    @ClienteID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        IF @ClienteID IS NULL
            THROW 50002, 'ClienteID no puede ser NULL.', 1;

        BEGIN TRAN;

        DELETE FROM dbo.Cliente
        WHERE ClienteID = @ClienteID;

        DECLARE @rows INT = @@ROWCOUNT;

        COMMIT TRAN;

        SELECT @rows AS filas_afectadas;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRAN;
        THROW;
    END CATCH
END
GO
```

---

## 7.3 Cursores

```sql
CREATE PROCEDURE dbo.usp_Recorrer_Clientes
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE 
        @ClienteID INT,
        @Nombre NVARCHAR(50),
        @Calle NVARCHAR(100),
        @Ciudad NVARCHAR(50);

    BEGIN TRY
        PRINT 'Iniciando recorrido de clientes...';

        DECLARE curClientes CURSOR LOCAL FAST_FORWARD
        FOR
        SELECT ClienteID, nombre_cliente, calle_cliente, ciudad_cliente
        FROM dbo.Cliente
        ORDER BY ClienteID;

        OPEN curClientes;

        FETCH NEXT FROM curClientes INTO @ClienteID, @Nombre, @Calle, @Ciudad;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT CONCAT('ID=', @ClienteID, ' | Nombre=', @Nombre, ' | Calle=', ISNULL(@Calle,'(sin calle)'), ' | Ciudad=', ISNULL(@Ciudad,'(sin ciudad)'));

            FETCH NEXT FROM curClientes INTO @ClienteID, @Nombre, @Calle, @Ciudad;
        END

        CLOSE curClientes;
        DEALLOCATE curClientes;

        PRINT 'Recorrido completado.';
    END TRY
    BEGIN CATCH
        PRINT 'Error en usp_Recorrer_Clientes:';
        PRINT ERROR_MESSAGE();

        IF CURSOR_STATUS('local','curClientes') >= -1
        BEGIN
            CLOSE curClientes;
            DEALLOCATE curClientes;
        END
    END CATCH
END
GO
```

---

# 8. Ejercicios del capítulo

1. Crear un trigger que impida que una cuenta quede con saldo negativo.  
2. Crear un trigger que valide que el monto de un préstamo no supere los activos de su sucursal.  
3. Crear un SP dinámico que actualice clientes con parámetros opcionales.  
4. Crear un SP estático que inserte sucursales con validación previa.  
5. Crear un cursor que recorra cuentas y calcule totales por sucursal.  
6. Crear una tabla puente adicional Cliente–Empresa (clientes corporativos).  
7. Crear un trigger que impida ciclos en la jerarquía de jefes.  
8. Crear un reporte que muestre inconsistencias potenciales.  

---

# 9. Asignación formal  

## Instrucciones

El estudiante debe entregar un archivo `.sql` que contenga:

1. Un trigger que valide una regla de negocio compleja.  
2. Un SP dinámico seguro usando sp_executesql.  
3. Un SP estático equivalente.  
4. Un cursor que recorra una tabla y muestre información.  
5. Un conjunto de pruebas que demuestren que las reglas funcionan.  

---

# 10. Mini‑quiz  

1. ¿Qué diferencia hay entre una restricción CHECK y un trigger?  
2. ¿Cuándo es apropiado usar cascadas?  
3. ¿Qué problema resuelven las tablas puente?  
4. ¿Por qué un trigger puede acceder a inserted y deleted?  
5. ¿Qué tipo de integridad controla que un empleado no gane más que su jefe?  
6. ¿Por qué el SQL dinámico debe ser parametrizado?  
7. ¿Cuándo es apropiado usar un cursor?  

---

# 11. Glosario  

**Integridad referencial:** Garantía de relaciones válidas entre tablas.  
**Cascada:** Propagación automática de acciones.  
**Trigger:** Código que se ejecuta ante eventos.  
**inserted/deleted:** Tablas virtuales de triggers.  
**PK surrogate:** Clave primaria artificial.  
**Tabla puente:** Tabla para relaciones muchos‑a‑muchos.  
**CHECK:** Restricción de dominio.  
**SQL dinámico:** SQL construido en tiempo de ejecución.  
**sp_executesql:** SQL dinámico seguro y parametrizado.  
**Cursor:** Mecanismo para recorrer filas una por una.  
**TRY/CATCH:** Manejo de errores.  
**THROW:** Re‑lanzamiento de errores con metadatos. 

---

# 12. Bibliografía comentada  

**Silberschatz, Korth, Sudarshan — *Database System Concepts*.**  
Base teórica para integridad, modelos y arquitectura.

**Elmasri y Navathe — *Fundamentals of Database Systems*.**  
Profundiza en diseño conceptual y normalización.

**Itzik Ben‑Gan — *T‑SQL Fundamentals*.**  
Lectura prioritaria para comprender triggers, SP y lógica T‑SQL.

**Microsoft Learn – SQL Server Documentation**  
Referencia oficial para sintaxis y comportamiento del motor.

**Garcia‑Molina, Ullman, Widom — *Database Systems: The Complete Book*.**  
Excelente para integridad lógica y teoría avanzada.

**Itzik Ben‑Gan — *Inside Microsoft SQL Server: T‑SQL Programming*.**  
Optimización, manejo de errores y patrones profesionales.

**Joe Celko — *SQL for Smarties*.**  
Problemas complejos y soluciones avanzadas.

**Markus Winand — *SQL Performance Explained*

---

