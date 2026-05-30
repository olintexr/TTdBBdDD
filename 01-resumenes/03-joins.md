
# `README.md` — Demostración de INNER JOIN, LEFT JOIN y RIGHT JOIN

## 1. Objetivo

Este módulo muestra, de forma práctica y reproducible, cómo funcionan los distintos tipos de JOIN en SQL Server utilizando dos tablas:

- **Sucursal**
- **Cuenta**

El propósito es observar cómo cambia el resultado según exista o no coincidencia entre las tablas.

---

## 2. Estructura del repositorio

```
/joins/
│
├── 01-creacion-tablas.sql
├── 02-insercion-datos.sql
├── 03-demostracion-joins.sql
└── README.md
```

---

## 3. Conceptos clave

### INNER JOIN  
Devuelve únicamente las filas donde existe coincidencia entre ambas tablas.

### LEFT JOIN  
Devuelve todas las filas de la tabla izquierda, y las coincidencias de la derecha.  
Si no hay coincidencia, las columnas de la derecha serán NULL.

### RIGHT JOIN  
Devuelve todas las filas de la tabla derecha, y las coincidencias de la izquierda.  
Si no hay coincidencia, las columnas de la izquierda serán NULL.

### FULL JOIN  
Devuelve todas las filas de ambas tablas, con NULL donde no haya coincidencias.

---

## 4. Scripts incluidos

### 01-creacion-tablas.sql

Crea las tablas **Sucursal** y **Cuenta**.

### 02-insercion-datos.sql

Inserta datos iniciales y agrega la sucursal “Limón” sin cuentas asociadas.

### 03-demostracion-joins.sql

Ejecuta los ejemplos de INNER, LEFT y RIGHT JOIN con comentarios didácticos.

---

## 5. Resultado esperado

- El **INNER JOIN** no mostrará la sucursal “Limón”.
- El **LEFT JOIN** sí la mostrará, con columnas de cuenta en NULL.
- El **RIGHT JOIN** será equivalente al INNER JOIN en este modelo.

---

## 6. Próximos temas sugeridos

- Vistas
- CTE
- FULL JOIN
- Normalización

---

# SCRIPTS SQL  
Listos para copiar y ejecutar.

---

# `01-creacion-tablas.sql`

```sql
/* ============================================================================
   CREACIÓN DE TABLAS: SUCURSAL y CUENTA
   ============================================================================ */

IF OBJECT_ID('dbo.Cuenta') IS NOT NULL DROP TABLE dbo.Cuenta;
IF OBJECT_ID('dbo.Sucursal') IS NOT NULL DROP TABLE dbo.Sucursal;
GO

CREATE TABLE Sucursal (
    SucursalID INT IDENTITY(1,1) PRIMARY KEY,
    nombre_sucursal VARCHAR(50),
    ciudad_sucursal VARCHAR(50),
    activos DECIMAL(18,2)
);
GO

CREATE TABLE Cuenta (
    CuentaID INT IDENTITY(1,1) PRIMARY KEY,
    numero_cuenta VARCHAR(20),
    saldo DECIMAL(18,2),
    SucursalID INT REFERENCES Sucursal(SucursalID)
);
GO
```

---

# `02-insercion-datos.sql`

```sql
/* ============================================================================
   INSERCIÓN DE DATOS INICIALES
   ============================================================================ */

INSERT INTO Sucursal (nombre_sucursal, ciudad_sucursal, activos) VALUES
('Central', 'San José', 5000000),
('Pacífico', 'Puntarenas', 2000000),
('Norte', 'Heredia', 1500000);
GO

INSERT INTO Cuenta (numero_cuenta, saldo, SucursalID) VALUES
('1001', 250000, 1),
('1002', 180000, 1),
('2001', 90000, 2),
('3001', 120000, 3);
GO

/* Nueva sucursal sin cuentas */
INSERT INTO Sucursal (nombre_sucursal, ciudad_sucursal, activos) VALUES
('Limón', 'Limón', 1000000);
GO
```

---

# `03-demostracion-joins.sql`

```sql
/* ============================================================================
   DEMOSTRACIÓN DE INNER JOIN, LEFT JOIN Y RIGHT JOIN
   ============================================================================ */

-- 1. INNER JOIN: solo coincidencias
SELECT *
FROM Sucursal
INNER JOIN Cuenta ON Sucursal.SucursalID = Cuenta.SucursalID;
GO

-- 2. INNER JOIN con columnas seleccionadas
SELECT Sucursal.*, Cuenta.numero_cuenta, Cuenta.saldo
FROM Sucursal
INNER JOIN Cuenta ON Sucursal.SucursalID = Cuenta.SucursalID;
GO

-- 3. INNER JOIN con columnas explícitas
SELECT 
    Sucursal.SucursalID,
    Sucursal.ciudad_sucursal,
    Cuenta.numero_cuenta,
    Cuenta.saldo
FROM Sucursal
INNER JOIN Cuenta ON Sucursal.SucursalID = Cuenta.SucursalID;
GO

-- 4. Ver todas las sucursales
SELECT * FROM Sucursal;
GO

-- 5. LEFT JOIN: incluye sucursales sin cuentas
SELECT 
    Sucursal.SucursalID,
    Sucursal.ciudad_sucursal,
    Cuenta.numero_cuenta,
    Cuenta.saldo
FROM Sucursal
LEFT JOIN Cuenta ON Sucursal.SucursalID = Cuenta.SucursalID;
GO

-- 6. RIGHT JOIN: incluye todas las cuentas
SELECT 
    Sucursal.SucursalID,
    Sucursal.ciudad_sucursal,
    Cuenta.numero_cuenta,
    Cuenta.saldo
FROM Sucursal
RIGHT JOIN Cuenta ON Sucursal.SucursalID = Cuenta.SucursalID;
GO
```

---

