# Entregable Datos Sprint 3

---

## 1. MODELO ENTIDAD-RELACIÓN REFINADO

### Cambios realizados respecto al Sprint 2:

| Cambio | Justificación |
|--------|---------------|
| Corrección `salado_inicial` → `saldo_inicial` en CUENTA | Typo en el campo |
| Agregar `id_usuario` (FK nullable) en CATEGORÍA | Permite categorías personalizadas por usuario (HU05). Si es NULL, es categoría global del sistema |
| Agregar `fecha_inicio` y `fecha_fin` en PRESUPUESTO (reemplaza `fecha`) | Un presupuesto necesita un rango de periodo para comparar gastos ejecutados |
| Agregar campo `saldo_actual` en CUENTA | Para mantener el saldo actualizado automáticamente con triggers |

### Diagrama actualizado:
* https://drive.google.com/file/d/1HC4ELsFqRxsqBxrwxnM5JY9yUi6MnZb2/view

### Cardinalidades refinadas:

- USUARIO (1) → (N) CUENTA
- USUARIO (1) → (N) PRESUPUESTO
- USUARIO (1) → (N) CATEGORÍA (categorías personalizadas)
- CUENTA (1) → (N) TRANSACCION
- CATEGORÍA (1) → (N) TRANSACCION
- CATEGORÍA (1) → (N) PRESUPUESTO

---

## 2. SCRIPT DE CREACIÓN DE OBJETOS (DDL + Triggers + Procedimientos)

### 2.1 Estructura de Tablas Refinada

```sql
-- ========================================
-- SCRIPT DDL REFINADO - PostgreSQL
-- Sistema de Gestión Financiera Personal
-- ========================================

CREATE TABLE USUARIO ( 
    id_usuario SERIAL PRIMARY KEY, 
    nombre VARCHAR(100) NOT NULL, 
    correo VARCHAR(100) NOT NULL UNIQUE, 
    contraseña VARCHAR(255) NOT NULL, 
    fecha_registro TIMESTAMP DEFAULT CURRENT_TIMESTAMP, 
    intentos_fallidos INT DEFAULT 0, 
    fecha_bloqueo TIMESTAMP NULL 
); 

CREATE TABLE CATEGORIA ( 
    id_categoria SERIAL PRIMARY KEY, 
    nombre_categoria VARCHAR(50) NOT NULL, 
    descripcion VARCHAR(255), 
    tipo_categoria VARCHAR(50) NOT NULL,
    id_usuario INT NULL,
    FOREIGN KEY (id_usuario) REFERENCES USUARIO(id_usuario) ON DELETE CASCADE
); 

CREATE TABLE CUENTA( 
    id_cuenta SERIAL PRIMARY KEY, 
    nombre_cuenta VARCHAR(50) NOT NULL, 
    tipo_cuenta VARCHAR(50) NOT NULL, 
    saldo_inicial DECIMAL(12,2) DEFAULT 0,
    saldo_actual DECIMAL(12,2) DEFAULT 0,
    id_usuario INT NOT NULL, 
    FOREIGN KEY(id_usuario) REFERENCES USUARIO(id_usuario) ON DELETE RESTRICT 
);  

CREATE TABLE PRESUPUESTO ( 
    id_presupuesto SERIAL PRIMARY KEY, 
    monto_limite DECIMAL(12,2) NOT NULL, 
    fecha_inicio DATE NOT NULL,
    fecha_fin DATE NOT NULL,
    id_usuario INT NOT NULL, 
    id_categoria INT NOT NULL,  
    FOREIGN KEY (id_usuario) REFERENCES USUARIO(id_usuario) ON DELETE RESTRICT, 
    FOREIGN KEY (id_categoria) REFERENCES CATEGORIA(id_categoria) ON DELETE RESTRICT,
    CHECK (fecha_fin > fecha_inicio)
); 

CREATE TABLE TRANSACCION ( 
    id_transaccion SERIAL PRIMARY KEY, 
    fecha TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,  
    descripcion VARCHAR(255), 
    monto DECIMAL(12,2) NOT NULL CHECK (monto > 0), 
    id_categoria INT NOT NULL, 
    id_cuenta INT NOT NULL, 
    FOREIGN KEY(id_categoria) REFERENCES CATEGORIA(id_categoria) ON DELETE RESTRICT, 
    FOREIGN KEY(id_cuenta) REFERENCES CUENTA(id_cuenta) ON DELETE RESTRICT 
);
```

---

### 2.2 Índices

Se crean índices sobre las columnas más consultadas en JOINs y filtros WHERE. La tabla TRANSACCIÓN es la más crítica por su volumen (1.8M registros), por lo que sin índices las consultas harían full table scans.

```sql
-- Índices en TRANSACCION (tabla más grande: 1.8M registros)
CREATE INDEX idx_transaccion_cuenta ON TRANSACCION(id_cuenta);
CREATE INDEX idx_transaccion_categoria ON TRANSACCION(id_categoria);
CREATE INDEX idx_transaccion_fecha ON TRANSACCION(fecha);
-- Índice compuesto: cubre consultas que filtran por cuenta + rango de fecha (HU03, HU04, HU07)
CREATE INDEX idx_transaccion_cuenta_fecha ON TRANSACCION(id_cuenta, fecha);

-- Índices en CUENTA
CREATE INDEX idx_cuenta_usuario ON CUENTA(id_usuario);

-- Índices en PRESUPUESTO
CREATE INDEX idx_presupuesto_usuario ON PRESUPUESTO(id_usuario);
CREATE INDEX idx_presupuesto_categoria ON PRESUPUESTO(id_categoria);

-- Índices en CATEGORÍA
CREATE INDEX idx_categoria_usuario ON CATEGORIA(id_usuario);
```

**Impacto en almacenamiento estimado:**

| Índice | Registros | Tamaño aprox. |
|--------|-----------|---------------|
| idx_transaccion_cuenta | 1,800,000 | ~14 MB |
| idx_transaccion_categoria | 1,800,000 | ~14 MB |
| idx_transaccion_fecha | 1,800,000 | ~14 MB |
| idx_transaccion_cuenta_fecha | 1,800,000 | ~22 MB |
| idx_cuenta_usuario | 40,000 | ~320 KB |
| idx_presupuesto_usuario | 40,000 | ~320 KB |
| idx_presupuesto_categoria | 40,000 | ~320 KB |
| idx_categoria_usuario | 2,020 | ~16 KB |
| **Total índices** | | **~65 MB** |

> El costo adicional de ~65 MB en disco es justificable dado que las consultas principales pasan de O(n) full scan a O(log n) con B-Tree, especialmente crítico en TRANSACCIÓN.

---

### 2.3 Triggers

#### Trigger HU01/HU02: Bloqueo por intentos fallidos de login

```sql
-- Procedimiento: incrementar intentos fallidos y bloquear si llega a 3
CREATE OR REPLACE FUNCTION fn_login_fallido()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.intentos_fallidos >= 3 AND OLD.intentos_fallidos < 3 THEN
        NEW.fecha_bloqueo := CURRENT_TIMESTAMP;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_bloqueo_usuario
BEFORE UPDATE ON USUARIO
FOR EACH ROW
WHEN (NEW.intentos_fallidos > OLD.intentos_fallidos)
EXECUTE FUNCTION fn_login_fallido();
```

#### Trigger HU03/HU04: Actualización automática de saldo al registrar transacción

```sql
-- Al insertar una transacción, actualizar saldo_actual de la cuenta
CREATE OR REPLACE FUNCTION fn_actualizar_saldo()
RETURNS TRIGGER AS $$
DECLARE
    v_tipo VARCHAR(50);
BEGIN
    SELECT tipo_categoria INTO v_tipo 
    FROM CATEGORIA WHERE id_categoria = NEW.id_categoria;

    IF v_tipo = 'Ingreso' THEN
        UPDATE CUENTA SET saldo_actual = saldo_actual + NEW.monto
        WHERE id_cuenta = NEW.id_cuenta;
    ELSIF v_tipo = 'Gasto' THEN
        UPDATE CUENTA SET saldo_actual = saldo_actual - NEW.monto
        WHERE id_cuenta = NEW.id_cuenta;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_actualizar_saldo
AFTER INSERT ON TRANSACCION
FOR EACH ROW
EXECUTE FUNCTION fn_actualizar_saldo();
```

#### Trigger: Inicializar saldo_actual con saldo_inicial al crear cuenta

```sql
CREATE OR REPLACE FUNCTION fn_inicializar_saldo()
RETURNS TRIGGER AS $$
BEGIN
    NEW.saldo_actual := NEW.saldo_inicial;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_inicializar_saldo
BEFORE INSERT ON CUENTA
FOR EACH ROW
EXECUTE FUNCTION fn_inicializar_saldo();
```

---

### 2.4 Procedimientos Almacenados

#### Procedimiento HU06: Alerta de presupuesto al 80%

```sql
CREATE OR REPLACE FUNCTION sp_alerta_presupuesto(p_id_usuario INT)
RETURNS TABLE (
    categoria VARCHAR(50),
    monto_limite DECIMAL(12,2),
    gastado DECIMAL(12,2),
    porcentaje_uso DECIMAL(5,2),
    alerta TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        C.nombre_categoria,
        P.monto_limite,
        COALESCE(SUM(T.monto), 0) AS gastado,
        ROUND(COALESCE(SUM(T.monto), 0) / P.monto_limite * 100, 2) AS porcentaje_uso,
        CASE 
            WHEN COALESCE(SUM(T.monto), 0) / P.monto_limite >= 1.0 THEN 'EXCEDIDO'
            WHEN COALESCE(SUM(T.monto), 0) / P.monto_limite >= 0.8 THEN 'ALERTA: Supera el 80%'
            ELSE 'OK'
        END AS alerta
    FROM PRESUPUESTO P
    JOIN CATEGORIA C ON P.id_categoria = C.id_categoria
    LEFT JOIN TRANSACCION T ON T.id_categoria = C.id_categoria
        AND T.fecha BETWEEN P.fecha_inicio AND P.fecha_fin
        AND T.id_cuenta IN (SELECT id_cuenta FROM CUENTA WHERE id_usuario = p_id_usuario)
    WHERE P.id_usuario = p_id_usuario
        AND CURRENT_DATE BETWEEN P.fecha_inicio AND P.fecha_fin
    GROUP BY C.nombre_categoria, P.monto_limite;
END;
$$ LANGUAGE plpgsql;
```

#### Procedimiento HU07: Balance financiero mensual

```sql
CREATE OR REPLACE FUNCTION sp_balance_mensual(p_id_usuario INT)
RETURNS TABLE (
    total_ingresos DECIMAL(12,2),
    total_gastos DECIMAL(12,2),
    balance_neto DECIMAL(12,2)
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        COALESCE(SUM(CASE WHEN C.tipo_categoria = 'Ingreso' THEN T.monto ELSE 0 END), 0),
        COALESCE(SUM(CASE WHEN C.tipo_categoria = 'Gasto' THEN T.monto ELSE 0 END), 0),
        COALESCE(SUM(CASE WHEN C.tipo_categoria = 'Ingreso' THEN T.monto ELSE 0 END), 0) -
        COALESCE(SUM(CASE WHEN C.tipo_categoria = 'Gasto' THEN T.monto ELSE 0 END), 0)
    FROM TRANSACCION T
    JOIN CATEGORIA C ON T.id_categoria = C.id_categoria
    JOIN CUENTA CU ON T.id_cuenta = CU.id_cuenta
    WHERE CU.id_usuario = p_id_usuario
        AND T.fecha >= DATE_TRUNC('month', CURRENT_DATE);
END;
$$ LANGUAGE plpgsql;
```

#### Procedimiento HU01: Registrar intento de login

```sql
CREATE OR REPLACE FUNCTION sp_intentar_login(p_correo VARCHAR, p_password VARCHAR)
RETURNS TABLE (
    resultado TEXT,
    nombre_usuario VARCHAR(100)
) AS $$
DECLARE
    v_usuario RECORD;
BEGIN
    SELECT * INTO v_usuario FROM USUARIO WHERE correo = p_correo;

    IF NOT FOUND THEN
        RETURN QUERY SELECT 'USUARIO_NO_ENCONTRADO'::TEXT, NULL::VARCHAR(100);
        RETURN;
    END IF;

    IF v_usuario.fecha_bloqueo IS NOT NULL 
       AND v_usuario.fecha_bloqueo > CURRENT_TIMESTAMP - INTERVAL '30 minutes' THEN
        RETURN QUERY SELECT 'CUENTA_BLOQUEADA'::TEXT, NULL::VARCHAR(100);
        RETURN;
    END IF;

    IF v_usuario.contraseña = p_password THEN
        UPDATE USUARIO SET intentos_fallidos = 0, fecha_bloqueo = NULL 
        WHERE id_usuario = v_usuario.id_usuario;
        RETURN QUERY SELECT 'LOGIN_EXITOSO'::TEXT, v_usuario.nombre;
    ELSE
        UPDATE USUARIO SET intentos_fallidos = intentos_fallidos + 1 
        WHERE id_usuario = v_usuario.id_usuario;
        RETURN QUERY SELECT 'PASSWORD_INCORRECTO'::TEXT, NULL::VARCHAR(100);
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## 3. ANÁLISIS DE VOLUMEN DE DATOS (REFINADO)

Para los siguientes cálculos usamos:

### $L = 4 \times (\text{número de campos variables}) + \sum \text{size(fijos)} + \text{size(mapa bits)} + \sum \text{tamaño variables}$ 

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R \times L$

$B_R = \frac{T_R}{F_R}$

Tamaño de página: **P = 4096 bytes**

---

### Tabla USUARIO = 1.28 MB

| Campo | Tipo | Tamaño |
|-------|------|--------|
| id_usuario | SERIAL | 4 bytes |
| nombre | VARCHAR | 40 bytes |
| correo | VARCHAR | 30 bytes |
| contraseña | VARCHAR | 20 bytes |
| fecha_registro | TIMESTAMP | 8 bytes |
| intentos_fallidos | INT | 4 bytes |
| fecha_bloqueo | TIMESTAMP | 8 bytes |

**Cálculos:**

size(fijos) = 24 bytes, tamaño est. variable = 90 bytes

$L = 4 \times 3 + 24 + 1 + 90 = 127 \text{ bytes por registro}$

$4096 = 2 + 131 F_R \Rightarrow F_R = 31 \text{ registros/página}$

$T_R = 10000 \text{ usuarios}$

$B_R = \frac{10000}{31} = 323 \text{ páginas}$

**Espacio estimado:** $323 \times 4\text{KB} = 1.28 \text{ MB}$

---

### Tabla CATEGORÍA = 12 KB

| Campo | Tipo | Tamaño |
|-------|------|--------|
| id_categoria | SERIAL | 4 bytes |
| nombre_categoria | VARCHAR | 20 bytes |
| descripcion | VARCHAR | 40 bytes |
| tipo_categoria | VARCHAR | 20 bytes |
| id_usuario | INT (nullable) | 4 bytes |

**Cálculos:**

size(fijos) = 8 bytes, tamaño est. variable = 80 bytes

$L = 4 \times 3 + 8 + 1 + 80 = 101 \text{ bytes por registro}$

$4096 = 2 + 105 F_R \Rightarrow F_R = 39 \text{ registros/página}$

Consideramos 20 categorías globales + posibles personalizadas. Estimamos que el 10% de usuarios crea 2 categorías propias: $T_R = 20 + (1000 \times 2) = 2020$

$B_R = \frac{2020}{39} = 52 \text{ páginas}$

**Espacio estimado:** $52 \times 4\text{KB} = 208 \text{ KB}$

> **Nota:** Este valor crece respecto al Sprint 2 porque ahora las categorías pueden ser por usuario.

---

### Tabla CUENTA = 2.56 MB

| Campo | Tipo | Tamaño |
|-------|------|--------|
| id_cuenta | SERIAL | 4 bytes |
| nombre_cuenta | VARCHAR | 20 bytes |
| tipo_cuenta | VARCHAR | 10 bytes |
| saldo_inicial | DECIMAL | 8 bytes |
| saldo_actual | DECIMAL | 8 bytes |
| id_usuario | INT | 4 bytes |

**Cálculos:**

size(fijos) = 24 bytes, tamaño est. variable = 30 bytes

$L = 4 \times 2 + 24 + 1 + 30 = 63 \text{ bytes por registro}$

$4096 = 2 + 67 F_R \Rightarrow F_R = 61 \text{ registros/página}$

$T_R = 40000$ (promedio 4 cuentas por usuario)

$B_R = \frac{40000}{61} = 656 \text{ páginas}$

**Espacio estimado:** $656 \times 4\text{KB} = 2624 \text{ KB} = 2.56 \text{ MB}$

---

### Tabla TRANSACCIÓN = 128.6 MB

| Campo | Tipo | Tamaño |
|-------|------|--------|
| id_transaccion | SERIAL | 4 bytes |
| id_categoria | INT | 4 bytes |
| id_cuenta | INT | 4 bytes |
| monto | DECIMAL | 8 bytes |
| fecha | TIMESTAMP | 8 bytes |
| descripcion | VARCHAR | 40 bytes |

**Cálculos:**

size(fijos) = 28 bytes, tamaño est. variable = 40 bytes

$L = 4 \times 1 + 28 + 1 + 40 = 73 \text{ bytes por registro}$

$4096 = 2 + 77 F_R \Rightarrow F_R = 53 \text{ registros/página}$

$T_R = 1,800,000$ (10,000 usuarios × 15 transacciones/mes × 12 meses)

$B_R = \frac{1800000}{53} = 33962 \text{ páginas}$

**Espacio estimado:** $33962 \times 4\text{KB} = 135848 \text{ KB} = 132.7 \text{ MB}$

---

### Tabla PRESUPUESTO = 1.12 MB

| Campo | Tipo | Tamaño |
|-------|------|--------|
| id_presupuesto | SERIAL | 4 bytes |
| monto_limite | DECIMAL | 8 bytes |
| fecha_inicio | DATE | 4 bytes |
| fecha_fin | DATE | 4 bytes |
| id_usuario | INT | 4 bytes |
| id_categoria | INT | 4 bytes |

**Cálculos:**

size(fijos) = 28 bytes, tamaño est. variable = 0 bytes

$L = 4 \times 0 + 28 + 1 + 0 = 29 \text{ bytes por registro}$

$4096 = 2 + 33 F_R \Rightarrow F_R = 124 \text{ registros/página}$

$T_R = 40,000$ (10,000 usuarios × 4 presupuestos promedio)

$B_R = \frac{40000}{124} = 323 \text{ páginas}$

**Espacio estimado:** $323 \times 4\text{KB} = 1292 \text{ KB} = 1.26 \text{ MB}$

---

### Resumen de Volumen Total Estimado

| Tabla | Registros ($T_R$) | Espacio |
|-------|-------------------|---------|
| USUARIO | 10,000 | 1.28 MB |
| CATEGORÍA | 2,020 | 208 KB |
| CUENTA | 40,000 | 2.56 MB |
| TRANSACCIÓN | 1,800,000 | 132.7 MB |
| PRESUPUESTO | 40,000 | 1.26 MB |
| **TOTAL** | **1,892,020** | **≈ 138 MB** |
| **+ Índices** | — | **~65 MB** |
| **TOTAL CON ÍNDICES** | — | **≈ 203 MB** |

---

## 4. DATOS DE PRUEBA

```sql
-- Insertar usuarios
INSERT INTO USUARIO (nombre, correo, contraseña) 
VALUES ('Juan Perez', 'juan@mail.com', 'hash_password_123'),
       ('Maria Lopez', 'maria@mail.com', 'hash_password_456');

-- Insertar categorías globales (id_usuario = NULL)
INSERT INTO CATEGORIA (nombre_categoria, descripcion, tipo_categoria, id_usuario) 
VALUES ('Comida', 'Gastos de restaurante y mercado', 'Gasto', NULL),
       ('Sueldo', 'Ingreso mensual laboral', 'Ingreso', NULL),
       ('Transporte', 'Gastos de movilidad', 'Gasto', NULL);

-- Insertar categoría personalizada del usuario 1
INSERT INTO CATEGORIA (nombre_categoria, descripcion, tipo_categoria, id_usuario) 
VALUES ('Freelance', 'Ingresos por proyectos', 'Ingreso', 1);

-- Insertar cuentas
INSERT INTO CUENTA (id_usuario, nombre_cuenta, tipo_cuenta, saldo_inicial) 
VALUES (1, 'Ahorros Bancolombia', 'Cuenta de Ahorros', 1000.00),
       (1, 'Nequi', 'Billetera Digital', 200.00);

-- Insertar presupuestos
INSERT INTO PRESUPUESTO (id_usuario, id_categoria, monto_limite, fecha_inicio, fecha_fin) 
VALUES (1, 1, 500.00, '2025-01-01', '2025-01-31'),
       (1, 3, 200.00, '2025-01-01', '2025-01-31');

-- Insertar transacciones (el trigger actualiza saldo_actual)
INSERT INTO TRANSACCION (id_categoria, id_cuenta, monto, descripcion) 
VALUES (1, 1, 50.00, 'Almuerzo ejecutivo'),
       (1, 1, 30.00, 'Cena familiar'),
       (2, 1, 2000.00, 'Salario enero'),
       (3, 2, 15.00, 'Uber al trabajo');

-- Probar alerta de presupuesto
SELECT * FROM sp_alerta_presupuesto(1);

-- Probar balance mensual
SELECT * FROM sp_balance_mensual(1);

-- Probar login fallido (ejecutar 3 veces para bloquear)
SELECT * FROM sp_intentar_login('juan@mail.com', 'password_incorrecta');
```
