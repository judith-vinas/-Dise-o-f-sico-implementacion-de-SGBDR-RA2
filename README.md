# Diseño Físico e Implementación de SGBDR
Judith Viñas Tribaldo

---

## Escenario Técnico: Sistema "BiblioGest 2.0"

El sistema debe gestionar el inventario físico de una biblioteca. El modelo consta de:

- **SOCIOS:** (DNI, Nombre, Email, Telefono)
- **LIBROS:** (ISBN, Titulo, Autor, Paginas)
- **EJEMPLARES:** (ID_Ejemplar, ISBN, Estado) — *Relacionado con LIBROS*
- **PRESTAMOS:** (Socio, Ejemplar, Fecha) — *Relacionado con SOCIOS y EJEMPLARES*

---

## TAREA 1: Definición de Estructura y Tipos de Datos

### 1.1 Creación del esquema relacional

Código SQL para crear el esquema con Claves Primarias, Foráneas y opciones de borrado:

```sql
CREATE TABLE SOCIOS (
    DNI        VARCHAR2(9)   PRIMARY KEY,
    Nombre     VARCHAR2(100) NOT NULL,
    Email      VARCHAR2(150) UNIQUE NOT NULL,
    Telefono   VARCHAR2(15)
);

CREATE TABLE LIBROS (
    ISBN    VARCHAR2(13)  PRIMARY KEY,
    Titulo  VARCHAR2(200) NOT NULL,
    Autor   VARCHAR2(100) NOT NULL,
    Paginas NUMBER(5)     CHECK (Paginas > 0)
);

CREATE TABLE EJEMPLARES (
    ID_Ejemplar NUMBER        PRIMARY KEY,
    ISBN        VARCHAR2(13)  NOT NULL,
    Estado      VARCHAR2(20)  NOT NULL
        CHECK (Estado IN ('Nuevo', 'Bueno', 'Deteriorado')),
    CONSTRAINT fk_ejemplar_libro
        FOREIGN KEY (ISBN) REFERENCES LIBROS(ISBN)
        ON DELETE CASCADE
);

CREATE TABLE PRESTAMOS (
    ID_Prestamo      NUMBER       PRIMARY KEY,
    DNI_Socio        VARCHAR2(9)  NOT NULL,
    ID_Ejemplar      NUMBER       NOT NULL,
    Fecha_Prestamo   DATE         NOT NULL,
    Fecha_Devolucion DATE,
    CONSTRAINT fk_prestamo_socio
        FOREIGN KEY (DNI_Socio) REFERENCES SOCIOS(DNI)
        ON DELETE SET NULL,
    CONSTRAINT fk_prestamo_ejemplar
        FOREIGN KEY (ID_Ejemplar) REFERENCES EJEMPLARES(ID_Ejemplar)
        ON DELETE CASCADE
);
```

---

### 1.2 Comparativa de tipos de datos

#### Modificación de la tabla SOCIOS para añadir fechas

```sql
ALTER TABLE SOCIOS
    ADD Fecha_Registro     DATE      DEFAULT SYSDATE NOT NULL;

ALTER TABLE SOCIOS
    ADD Fecha_Modificacion TIMESTAMP DEFAULT SYSTIMESTAMP;
```

#### ¿En qué se diferencia DATE de TIMESTAMP?

| Tipo        | Precisión                              | Zona horaria      |
|-------------|----------------------------------------|-------------------|
| `DATE`      | Fecha y hora con precisión de segundos | No                |
| `TIMESTAMP` | Añade fracciones de segundo            | Opcional (con TZ) |

> `DATE` almacena fecha y hora con precisión de segundos. `TIMESTAMP` añade fracciones de segundo y opcionalmente zona horaria.

#### ¿Qué diferencia hay entre CHAR y VARCHAR2?

| Tipo          | Longitud | Relleno     | Uso recomendado                   |
|---------------|----------|-------------|-----------------------------------|
| `CHAR(n)`     | Fija     | Espacios    | Valores de longitud siempre igual |
| `VARCHAR2(n)` | Variable | Sin relleno | Valores de longitud variable      |

> Se usa `VARCHAR2` en este esquema porque los valores (nombres, emails, DNI, etc.) tienen longitudes variables, ahorrando espacio de almacenamiento.

---

### 1.3 Restricciones de integridad

Añadir restricción CHECK sobre Estado y clave candidata en Email:

```sql
ALTER TABLE EJEMPLARES
    ADD CONSTRAINT chk_estado
        CHECK (Estado IN ('Nuevo', 'Bueno', 'Deteriorado'));

ALTER TABLE SOCIOS
    ADD CONSTRAINT uq_email UNIQUE (Email);
```

---

## TAREA 2: Objetos de la Base de Datos y Optimización

### 2.1 Vista de préstamos activos

```sql
CREATE OR REPLACE VIEW VISTA_PRESTAMOS_ACTIVOS AS
    SELECT
        L.Titulo        AS Titulo_Libro,
        E.ID_Ejemplar   AS ID_Ejemplar,
        S.Nombre        AS Nombre_Socio
    FROM PRESTAMOS P
    JOIN EJEMPLARES E ON P.ID_Ejemplar = E.ID_Ejemplar
    JOIN LIBROS     L ON E.ISBN        = L.ISBN
    JOIN SOCIOS     S ON P.DNI_Socio   = S.DNI
    WHERE P.Fecha_Devolucion IS NULL;
```

> Muestra el título del libro, ID del ejemplar y nombre del socio únicamente para los préstamos que no han sido devueltos (`Fecha_Devolucion IS NULL`).

---

### 2.2 Índice sobre la tabla LIBROS

```sql
CREATE INDEX idx_libros_titulo ON LIBROS (Titulo);
```

> Este índice mejora el rendimiento en búsquedas y filtros por título de libro.

---

### 2.3 Definición de TABLESPACE en Oracle

Un **TABLESPACE** es la unidad lógica de almacenamiento en Oracle que agrupa uno o varios ficheros físicos del sistema operativo (datafiles).

**¿Por qué separar datos e índices en tablespaces distintos?**

- **Rendimiento:** Permite distribuir I/O en diferentes discos físicos, reduciendo cuellos de botella.
- **Mantenimiento independiente:** Se pueden hacer copias de seguridad, restauraciones o reorganizaciones por separado.
- **Gestión del crecimiento:** Cada tablespace puede crecer de forma autónoma según las necesidades de datos o de índices.

---

## TAREA 3: Administración y Herramientas

### 3.1 Gestión de seguridad y privilegios

#### Crear usuario `super_admin` con todos los privilegios

```sql
CREATE USER super_admin IDENTIFIED BY "contraseña";
GRANT DBA TO super_admin;
```

#### Crear usuario `tecnico_inventario`

```sql
CREATE USER tecnico_inventario IDENTIFIED BY "contraseña";
GRANT CREATE SESSION TO tecnico_inventario;
```

#### Conceder permisos de consulta y actualización sobre EJEMPLARES

```sql
GRANT SELECT, UPDATE ON EJEMPLARES TO tecnico_inventario;
```

#### Revocar permiso de actualización

```sql
REVOKE UPDATE ON EJEMPLARES FROM tecnico_inventario;
```

#### Eliminar el usuario del sistema

```sql
DROP USER tecnico_inventario CASCADE;
```

---

### 3.2 Conexión a consola

| SGBD   | Comando               |
|--------|-----------------------|
| MySQL  | `mysql -u root -p`    |
| Oracle | `sqlplus / AS SYSDBA` |

---

### 3.3 Identificación de herramientas gráficas

| Herramienta        | CLI (Consola) | Web | Escritorio (GUI) | Multi-extensión | Nativo MySQL | Nativo Oracle |
|--------------------|:---:|:---:|:---:|:---:|:---:|:---:|
| MySQL Workbench    |     |     | X   | X   | X   |     |
| phpMyAdmin         |     | X   |     | X   |     |     |
| SQL Developer      |     |     | X   | X   |     | X   |
| Visual Studio Code |     |     | X   | X   |     |     |
| mysql / sqlplus    | X   |     |     |     | X   | X   |
