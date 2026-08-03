# 02 – Integridad referencial, CHECK y triggers con inserted/deleted

## 1. Objetivo del capítulo

En este capítulo se muestra, paso a paso, cómo:

- Definir una base de datos de ejemplo.
- Crear tablas relacionadas con integridad referencial.
- Usar restricciones `CHECK` y valores por defecto.
- Provocar errores controlados (reglas declarativas).
- Definir triggers de `INSERT`, `UPDATE` y `DELETE`.
- Explicar el uso de las tablas virtuales `inserted` y `deleted`.
- Implementar una regla de negocio con un trigger sobre `Factura`.

Todo el contenido se basa en el siguiente script, organizado y comentado.

---

## 2. Creación de la base de datos y limpieza inicial

```sql
CREATE DATABASE REGLAS_DB
GO

USE REGLAS_DB 
GO

IF OBJECT_ID('dbo.Factura', 'U') IS NOT NULL DROP TABLE dbo.Factura;
IF OBJECT_ID('dbo.Cliente', 'U') IS NOT NULL DROP TABLE dbo.Cliente;
GO
```

**Comentarios clave:**

- Se crea la base de datos `REGLAS_DB` y se selecciona con `USE`.
- Se eliminan las tablas `Factura` y `Cliente` si existen, para garantizar un entorno limpio y reproducible.

---

## 3. Tabla Cliente

```sql
CREATE TABLE dbo.Cliente (
    ClienteID INT NOT NULL PRIMARY KEY,
    nombre NVARCHAR(50) NOT NULL UNIQUE
);
GO
```

**Puntos importantes:**

- `ClienteID` es clave primaria manual (no es IDENTITY).
- `nombre` es obligatorio (`NOT NULL`) y único (`UNIQUE`), evitando clientes duplicados por nombre.

---

## 4. Tabla Factura con CHECK, DEFAULT y FOREIGN KEY

```sql
-- Tabla Factura: almacena encabezados de facturas emitidas
CREATE TABLE dbo.Factura (
    
    -- Clave primaria manual; no es IDENTITY
    FacturaID INT NOT NULL PRIMARY KEY,

    -- Cliente al que pertenece la factura; debe existir en dbo.Cliente
    ClienteID INT NOT NULL,

    -- Monto total de la factura; no puede ser negativo
    -- Esto quiere decir que cualquier insert o update que intente poner "total" en negativo, va a fallar
    total DECIMAL(12,2) NOT NULL CHECK (total >= 0),

    -- Fecha de emisión; si no se especifica, usa la fecha actual del servidor
    fecha DATE NOT NULL DEFAULT GETDATE(),

    -- Integridad referencial: cada factura debe apuntar a un cliente válido
    CONSTRAINT FK_Factura_Cliente FOREIGN KEY (ClienteID)
        REFERENCES dbo.Cliente(ClienteID)

/*
-- Si intentan borrar/actualizar el cliente no hagas nada si tiene hijos en factura
CONSTRAINT FK_Factura_Cliente FOREIGN KEY (ClienteID)
    REFERENCES dbo.Cliente(ClienteID)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION;
*/

/*
-- Si borran/actualizan el cliente, borra también la factura
ON DELETE CASCADE
ON UPDATE CASCADE
*/

/*
-- Si borran/actualizan el cliente, deja el cliente de factura en null
-- Para que esto tenga efecto, el campo cliente en factura debe aceptar valores null
ON DELETE SET NULL
ON UPDATE SET NULL
*/

/*
-- Si borran/actualizan el cliente, pon el valor por defecto en el cliente de factura
ON DELETE SET DEFAULT
ON UPDATE SET DEFAULT
*/
);
GO
```

**Conceptos:**

- `CHECK (total >= 0)` impide totales negativos en `INSERT` y `UPDATE`.
- `DEFAULT GETDATE()` asigna la fecha del sistema si no se especifica `fecha`.
- `FOREIGN KEY (ClienteID)` garantiza que toda factura apunte a un cliente existente.
- En los comentarios se muestran las variantes de comportamiento de la clave foránea:
  - `NO ACTION`
  - `CASCADE`
  - `SET NULL`
  - `SET DEFAULT`

---

## 5. Inserción y validación de datos en Cliente y Factura

### 5.1 Inserción de clientes

```sql
-- Inserción de algunos clientes
Insert into dbo.Cliente values (1, 'Olinto'), (2, 'Aaron'), (3, 'Gustavo')
GO

SELECT * FROM dbo.Cliente
GO
```

### 5.2 Intentos de inserción de facturas incorrectas

```sql
-- Intento de inserción de facturas incorrectas
-- (Total negativo)
INSERT INTO dbo.Factura VALUES (1, 80, -100, GETDATE() )
GO

-- Intento de inserción de facturas incorrectas
-- (Cliente no está)
INSERT INTO dbo.Factura VALUES (1, 80, 100, GETDATE() )
GO
```

**Qué se espera:**

- El primer `INSERT` falla por el `CHECK (total >= 0)`.
- El segundo `INSERT` falla por la `FOREIGN KEY` (cliente 80 no existe).

### 5.3 Inserción correcta

```sql
-- Intento de inserción de factura correcta
INSERT INTO dbo.Factura VALUES (1, 1, 100, GETDATE() )
GO

SELECT * FROM dbo.Factura
GO
```

### 5.4 Actualización errónea (CHECK)

```sql
-- Intento de actualización errónea (valor negativo)
UPDATE dbo.Factura SET total = -10 WHERE FacturaID = 1
GO

SELECT * FROM dbo.Factura
GO
```

**Resultado esperado:**

- El `UPDATE` falla por violar el `CHECK (total >= 0)`.

### 5.5 Intento de borrar un cliente con facturas

```sql
-- intento de borrar el cliente que está en la factura
DELETE FROM dbo.Cliente WHERE ClienteID = 1
GO
```

Con la configuración actual (sin `ON DELETE CASCADE`), esta operación debe fallar por integridad referencial.

### 5.6 Inserción de factura sin fecha (uso de DEFAULT)

```sql
-- Inserción de factura SIN FECHA (Tomará la del sistema)
INSERT INTO [dbo].[Factura]
           ([FacturaID]
           ,[ClienteID]
           ,[total])
     VALUES
           (2
           ,2
           ,2000)
GO

SELECT * FROM Factura
GO 
```

**Punto clave:**

- La columna `fecha` toma automáticamente `GETDATE()` gracias al `DEFAULT`.

---

## 6. Explicación de triggers e inserted/deleted

Los triggers (disparadores) son procedimientos que se ejecutan automáticamente en respuesta a ciertos eventos sobre una tabla, como `INSERT`, `UPDATE` o `DELETE`. Su propósito principal suele ser auditar cambios, validar reglas de negocio o evitar operaciones no deseadas.

En SQL Server, los triggers pueden ser de tipo `AFTER` (se ejecutan después del evento) o `INSTEAD OF` (se ejecutan en lugar del evento). En otros manejadores, el concepto es similar, aunque la sintaxis cambia: por ejemplo, en MySQL se suelen declarar como `BEFORE INSERT`, `AFTER UPDATE`, etc., y en PostgreSQL pueden asociarse a eventos de tabla o de fila, incluso sobre vistas.

Algunas características importantes de los triggers son:

- No se llaman manualmente; se ejecutan automáticamente cuando ocurre el evento.
- No importa si la modificación proviene de una aplicación o de la misma base de datos, el trigger se dispara igualmente.
- Pueden deshabilitarse cuando sea necesario.
- Son útiles para generar auditoría, validar reglas de negocio o prevenir operaciones que no deben permitirse.
- Si se usan en exceso, pueden afectar el rendimiento si no se administran correctamente.

Dentro de un trigger en SQL Server, el motor expone dos tablas virtuales:

- `inserted`: contiene las filas nuevas en un `INSERT` o los valores nuevos en un `UPDATE`.
- `deleted`: contiene las filas eliminadas en un `DELETE` o los valores anteriores en un `UPDATE`.

En un `INSERT` solo existe `inserted`; en un `DELETE` solo existe `deleted`; y en un `UPDATE` existen ambas tablas, porque se comparan los valores anteriores y los nuevos.

> Importante: SQL Server procesa las operaciones por conjuntos, por lo que los triggers deben diseñarse pensando en varias filas, no solo en una.

```sql
/*
Ejemplo conceptual de uso de las tablas virtuales:
- inserted: filas nuevas o valores nuevos.
- deleted: filas eliminadas o valores anteriores.
*/
```

---

## 7. Triggers sobre Cliente

### 7.1 Trigger de INSERT en Cliente

```sql
CREATE OR ALTER TRIGGER TR_Cliente_Insert
ON dbo.Cliente
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    -- Evento: INSERT
    -- Tabla virtual disponible: inserted (contiene las filas nuevas)

    SELECT 
        'Cliente insertado' AS evento,
        i.ClienteID,
        i.nombre
    FROM inserted i;
END;
GO

-- Este insert llamará al trigger de insert de la tabla de clientes
INSERT INTO Cliente(ClienteID, nombre) VALUES (4,'Pedrito')
GO
```

**Idea didáctica:**

- Cada vez que se inserta un cliente, el trigger muestra las filas de `inserted`.

---

### 7.2 Trigger de UPDATE en Cliente

```sql
CREATE OR ALTER TRIGGER TR_Cliente_Update
ON dbo.Cliente
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Evento: UPDATE
    -- Tablas virtuales disponibles:
    --   deleted  = valores anteriores
    --   inserted = valores nuevos

    SELECT 
        'Valores anteriores' AS estado,
        d.ClienteID,
        d.nombre
    FROM deleted d;

    SELECT 
        'Valores nuevos' AS estado,
        i.ClienteID,
        i.nombre
    FROM inserted i;
END;
GO

SELECT * FROM Cliente
GO

UPDATE Cliente SET nombre = 'Juancho' WHERE ClienteID = 4 
GO

SELECT * FROM Cliente
GO
```

**Punto clave:**

- Se muestran los valores antes y después del `UPDATE` usando `deleted` e `inserted`.

---

### 7.3 Trigger de DELETE en Cliente

```sql
CREATE OR ALTER TRIGGER TR_Cliente_Delete
ON dbo.Cliente
AFTER DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Evento: DELETE
    -- Tabla virtual disponible: deleted (contiene las filas eliminadas)

    SELECT
        'Cliente eliminado' AS evento,
        d.ClienteID,
        d.nombre
    FROM deleted d;
END;
GO

SELECT * FROM Cliente 
GO

DELETE FROM Cliente WHERE ClienteID = 4
GO

SELECT * FROM Cliente 
GO

SELECT * FROM Factura
GO
```

**Uso didáctico:**

- Permite ver qué filas fueron eliminadas, leyendo la tabla `deleted`.

---

## 8. Trigger de regla de negocio sobre Factura

### 8.1 Definición del trigger

```sql
-- Trigger de regla de negocio
-- No se puede cambiar el código de cliente en una factura 
CREATE OR ALTER TRIGGER TR_Factura_NoCambiarCliente
ON dbo.Factura
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN deleted d ON i.FacturaID = d.FacturaID
        WHERE i.ClienteID <> d.ClienteID
    )
    BEGIN
        RAISERROR('No se puede reasignar una factura a otro cliente.', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
END;
GO
```

**Concepto:**

- Este trigger implementa una regla de negocio: una vez creada la factura, no se puede cambiar el cliente asociado.
- Se compara `ClienteID` antes (`deleted`) y después (`inserted`) del `UPDATE`.

### 8.2 Pruebas del trigger de Factura

```sql
SELECT * FROM Factura 
GO

UPDATE Factura SET total = 1000 WHERE FacturaID = 1
GO

SELECT * FROM Factura 
GO

UPDATE Factura SET ClienteID = 2 WHERE FacturaID = 1
GO

SELECT * FROM Factura 
GO
```

**Resultados esperados:**

- El `UPDATE` que cambia solo `total` es válido.
- El `UPDATE` que intenta cambiar `ClienteID` debe fallar con el mensaje:
  
  `No se puede reasignar una factura a otro cliente.`

---

## 9. Mini‑quiz

1. ¿Qué diferencia hay entre un `CHECK` y un `FOREIGN KEY`?  
2. ¿Por qué un `INSERT` con total negativo falla en la tabla `Factura`?  
3. ¿Qué tabla virtual contiene los valores anteriores en un `UPDATE`?  
4. ¿Por qué el trigger `TR_Factura_NoCambiarCliente` usa `inserted` y `deleted` a la vez?  
5. ¿Qué ocurriría si se definiera `ON DELETE CASCADE` en la relación `Factura–Cliente`?

---

## 10. Ejercicios propuestos

1. Crear un trigger en `Factura` que impida borrar facturas con `total > 0`.  
2. Crear una tabla de auditoría y un trigger que registre cada `DELETE` en `Cliente`.  
3. Modificar el modelo para permitir `ON DELETE SET NULL` y ajustar `ClienteID` en `Factura` para aceptar `NULL`.  
4. Crear un trigger que valide que `fecha` de `Factura` no sea futura.  
5. Escribir un procedimiento almacenado que inserte facturas y pruebe los triggers definidos.