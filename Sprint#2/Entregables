# Entregable Datos Spring 2

---

## 1. MODELO ENTIDAD - RELACIÓN (MER)

*![alt text](<Base De Datos-Modelo Entidad-Relacion.drawio (1).png>)*

---

## 2. MODELO FÍSICO (REDEFINIDO)


Ajustamos claves, columnas y redundancias como la que teníamos con usuario - transacción, la cual se podría llegar desde cuenta - transacción simplemente usando las claves foráneas.

![alt text](<Base De Datos-MODELO FISICO.drawio (3).png>)
---

## 3. SCRIPT DE ESTRUCTURA PARA MOTOR BD (Postgres)

```sql
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
 tipo_categoria VARCHAR(50) NOT NULL 
); 

CREATE TABLE CUENTA( 
 id_cuenta SERIAL PRIMARY KEY, 
 nombre_cuenta VARCHAR(50) NOT NULL, 
 tipo_cuenta VARCHAR(50) NOT NULL, 
 salado_inicial DECIMAL(12,2) DEFAULT 0, 
 id_usuario INT, 
 FOREIGN KEY(id_usuario) REFERENCES USUARIO(id_usuario) ON DELETE RESTRICT 
);  

CREATE TABLE PRESUPUESTO ( 
 id_presupuesto SERIAL PRIMARY KEY, 
 monto_limite DECIMAL(12,2) NOT NULL, 
 fecha DATE NOT NULL, 
 id_usuario INT NOT NULL, 
 id_categoria INT NOT NULL,  
 FOREIGN KEY (id_usuario) REFERENCES USUARIO (id_usuario) ON DELETE RESTRICT, 
 FOREIGN KEY (id_categoria) REFERENCES CATEGORIA (id_categoria) ON DELETE RESTRICT  
); 

CREATE TABLE TRANSACCION ( 
 id_transaccion SERIAL PRIMARY KEY, 
 fecha TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,  
 descripcion VARCHAR(255), 
 monto DECIMAL(12,2) NOT NULL CHECK (monto >0), 
 id_categoria INT, 
 id_cuenta INT, 
 FOREIGN KEY(id_categoria) REFERENCES CATEGORIA(id_categoria) ON DELETE RESTRICT, 
 FOREIGN KEY(id_cuenta) REFERENCES CUENTA(id_cuenta) ON DELETE RESTRICT 
);
```
---

## 4. CONSULTAS IDENTIFICADAS EN LAS HU 

```sql
-- HU-02 consulta de verificacion y traer nombre.
SELECT nombre, 
(SELECT SUM(saldo_inicial) FROM CUENTA WHERE id_usuario =1) as balance 
FROM USUARIO 
WHERE correo = 'juan@mail.com' AND contraseña = 'hash_password_123';

-- HU-03 y HU-04 listar mensual o historial de transacciones mensual 
SELECT T.fecha, C.nombre_categoria, T.monto, T.descripcion 
FROM TRANSACCION T 
JOIN CATEGORIA C ON T.id_categoria = C.id_categoria 
JOIN CUENTA CU ON T.id_cuenta = CU.id_cuenta 
WHERE CU.id_usuario = 1 
AND T.fecha >= DATE_TRUNC('month', CURRENT_DATE) 
ORDER BY T.fecha DESC;

-- HU-05 filtrar categorias denominadas "rubros" para ver donde se concentran los gastos
SELECT C.nombre_categoria AS rubro, SUM(T.monto) AS total_acumulado 
FROM TRANSACCION T 
JOIN CATEGORIA C ON T.id_categoria = C.id_categoria 
JOIN CUENTA CU ON T.id_cuenta = CU.id_cuenta 
WHERE CU.id_usuario = 1 
GROUP BY C.nombre_categoria 
ORDER BY total_acumulado DESC;

-- HU-06 cumple con el requerimiento de seguimiento y alerta del 80% que piden en criterios de aceptacion.
SELECT C.nombre_categoria, P.monto_limite, SUM(T.monto) AS gastado 
FROM PRESUPUESTO P 
JOIN CATEGORIA C ON P.id_categoria = C.id_categoria 
JOIN TRANSACCION T ON T.id_categoria = C.id_categoria 
WHERE P.id_usuario = 1 
GROUP BY C.nombre_categoria, P.monto_limite 
HAVING (SUM(T.monto)/P.monto_limite)>= 0.80;

-- HU-07 calcular valance financiero
SELECT  
SUM(CASE WHEN C.tipo_categoria = 'Ingreso' THEN T.monto ELSE 0 END) as total_ingresos,
SUM(CASE WHEN C.tipo_categoria = 'Gasto' THEN T.monto ELSE 0 END) as total_gastos,
(SUM(CASE WHEN C.tipo_categoria = 'Ingreso' THEN T.monto ELSE 0 END) -  
SUM(CASE WHEN C.tipo_categoria = 'Gasto' THEN T.monto ELSE 0 END)) as balance_neto
FROM TRANSACCION T 
JOIN CATEGORIA C ON T.id_categoria = C.id_categoria 
JOIN CUENTA CU ON T.id_cuenta = CU.id_cuenta 
WHERE CU.id_usuario = 1  
AND T.fecha >= DATE_TRUNC('month', CURRENT_DATE);

--No se realizan las Querys de "Principales consultas" realizadas en el Sprint 1, ya que cada una de esas esta dentro de las presentadas aqui.
```


## 5. DEFINICION DE VOLUMEN POR TABLA APROXIMADO:

Para los siguientes cálculos usamos:

### $L = 4 \times (\text{\# campos variables}) + \sum \text{size(fijos)} + \text{size(mapa\_bits)} + \sum \text{tamaño variables}$

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R \times L$

$B_R = \frac{T_R}{F_R}$


---

##  Tabla USUARIO = 1.28 MB

| Campo | Tipo | Tamaño |
|------|------|--------|
| id_usuario | SERIAL | 4 bytes |
| nombre | VARCHAR | 40 bytes |
| correo | VARCHAR | 30 bytes |
| contraseña | VARCHAR | 20 bytes |
| fecha_registro | TIMESTAMP | 8 bytes |
| intentos_fallidos | INT | 4 bytes |
| fecha_bloqueo | TIMESTAMP | 8 bytes |

**Cálculos:**
<center>
size(fijos) = 24 bytes ,    tamaño est. variable=90 bytes

$L = 4 \times 3 + 24 + 1 + 90 = 127 \text{ bytes por registro}$

P= 4096 bytes : tamaño de página o bloque.  

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R L$

$P = 1 + 1 + 4F_R + F_R (127)$

$4096 = 2 + 131 F_R$

$4094 = 131 F_R$

$\frac{4094}{131} = F_R$

$F_R = 31 \text{ registros por bloque (página)}$
</center>

Proyectamos una base de 10000 usuarios que seria nuestro $T_R$ siendo este el número de tuplas de la relación. Para proceder a calcular $B_R$ el cual es el número de páginas que ocupa una relación en disco.

$B_R = \frac{10000}{31} = 322 \text{ páginas}$

entonces para calcular el espacio estimado para la tabla USUARIO : 

$322 \text{ paginas} \times 4 \text{KB} = 1288 \text{ KB} = 1.28 \text{ MB}$
---

##  Tabla CATEGORIA = 8 KB

| Campo | Tipo | Tamaño |
|------|------|--------|
| id_categoria | SERIAL | 4 bytes |
| nombre_categoria | VARCHAR | 20 bytes |
| descripcion | VARCHAR | 40 bytes |
| tipo_categoria | VARCHAR | 20 bytes |

**Cálculos:**
<center>
size(fijos) = 4 bytes ,    tamaño est. variable=80 bytes

$L = 4 \times 3 + 4 + 1 + 80 = 97 \text{ bytes por registro}$

P= 4096 bytes : tamaño de página o bloque.  

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R L$

$P = 1 + 1 + 4F_R + F_R (97)$

$4096 = 2 + 101 F_R$

$4094 = 101 F_R$

$\frac{4094}{101} = F_R$

$F_R = 40 \text{ registros por bloque (página)}$

Consideramos unas 50 categorías ya que son limitadas, que sería nuestro $T_R = 50$ siendo este el número de tuplas de la relación. Para proceder a calcular $B_R$ el cual es el número de páginas que ocupa una relación en disco.

$B_R = \frac{50}{40} = 1.25 \text{ páginas}$, al ser una tabla pequeña aproximamos a 2 para prever el crecimiento.

entonces para calcular el espacio estimado para la tabla CATEGORIA :

$2 \text{ paginas} \times 4 \text{KB} = 8 \text{ KB}$
</center>
---

##  Tabla CUENTA = 2.32 MB

| Campo | Tipo | Tamaño |
|------|------|--------|
| id_cuenta | SERIAL | 4 bytes |
| nombre_cuenta | VARCHAR | 20 bytes |
| tipo_cuenta | VARCHAR | 10 bytes |
| saldo_inicial | DECIMAL | 8 bytes |
| id_usuario | INT | 4 bytes |

**Cálculos:**
<center>
size(fijos) = 16 bytes ,    tamaño est. variable=30 bytes

$L = 4 \times 2 + 16 + 1 + 30 = 55 \text{ bytes por registro}$

P= 4096 bytes : tamaño de página o bloque.  

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R L$

$P = 1 + 1 + 4F_R + F_R (55)$

$4096 = 2 + 59 F_R$

$4094 = 59 F_R$

$\frac{4094}{59} = F_R$

$F_R = 69 \text{ registros por bloque (página)}$

Consideramos que nuestros usuarios pueda tener 4 tipos de cuentas, sería nuestro $T_R = 40000$ siendo este el número de tuplas de la relación. Para proceder a calcular $B_R$ el cual es el número de páginas que ocupa una relación en disco.

$B_R = \frac{40000}{69} = 580 \text{ páginas}$

entonces para calcular el espacio estimado para la tabla CUENTA:

$580 \text{ paginas} \times 4 \text{KB} = 2320 \text{ KB} = 2.32 \text{ MB}$
</center>
---

##  Tabla TRANSACCION = 128.6 MB

| Campo | Tipo | Tamaño |
|------|------|--------|
| id_transaccion | SERIAL | 4 bytes |
| id_categoria | INT | 4 bytes |
| id_cuenta | INT | 4 bytes |
| monto | DECIMAL | 8 bytes |
| fecha | DATE | 4 bytes |
| descripcion | VARCHAR | 40 bytes |

**Cálculos:**
<center>
size(fijos) = 24 bytes ,    tamaño est. variable=40 bytes

$L = 4 \times 1 + 24 + 1 + 40 = 69 \text{ bytes por registro}$

P= 4096 bytes : tamaño de página o bloque.  

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R L$

$P = 1 + 1 + 4F_R + F_R (69)$

$4096 = 2 + 73 F_R$

$4094 = 73 F_R$

$\frac{4094}{73} = F_R$

$F_R = 56 \text{ registros por bloque (página)}$

Considerando que tenemos una proyección de 10000 usuarios, investigando la cotidianidad de la frecuencia con que una persona registra sus movimientos, tomamos un promedio de 15 transacciones al mes eso serian 180 al año entonces nuestro $T_R = 1800000$ siendo este el número de tuplas de la relación. Para proceder a calcular $B_R$ el cual es el número de páginas que ocupa una relación en disco.

$B_R = \frac{1800000}{56} = 32143 \text{ páginas}$

entonces para calcular el espacio estimado para la tabla TRANSACCIÓN:

$32143 \text{ paginas} \times 4 \text{KB} = 128572 \text{ KB} = 128.6 \text{ MB}$
</center>
---

##  Tabla PRESUPUESTO = 28.4 MB

| Campo | Tipo | Tamaño |
|------|------|--------|
| id_presupuesto | SERIAL | 4 bytes |
| monto_limite | DECIMAL | 8 bytes |
| fecha | DATE | 4 bytes |
| id_usuario | INT | 4 bytes |
| id_categoria | INT | 4 bytes |

**Cálculos:**
<center>
size(fijos) = 24 bytes ,    tamaño est. variable=0 bytes

$L = 4 \times 0 + 24 + 1 + 0 = 25 \text{ bytes por registro}$

P= 4096 bytes : tamaño de página o bloque.  

$P = 1 \text{ byte} + 1 \text{ byte} + 4F_R + F_R L$

$P = 1 + 1 + 4F_R + F_R (25)$

$4096 = 2 + 29 F_R$

$4094 = 29 F_R$

$\frac{4094}{29} = F_R$

$F_R = 141 \text{ registros por bloque (página)}$

Si tomamos un promedio en las categorías y estimamos que cada usuario tendrá 4 presupuestos, serían 100 lo cual por nuestros usuarios nuestro $T_R = 1000000$ siendo este el número de tuplas de la relación. Para proceder a calcular $B_R$ el cual es el número de páginas que ocupa una relación en disco.

$B_R = \frac{1000000}{141} = 7092 \text{ páginas}$

entonces para calcular el espacio estimado para la tabla PRESUPUESTO:

$7092 \text{ paginas} \times 4 \text{KB} = 28368 \text{ KB} = 28.4 \text{ MB}$
</center>

## 6. DEFINICIÓN DE ROLES Y ESQUEMA DE SEGURIDAD

Para este paso realizamos el esquema de **mínimo privilegio**, el cual limita el acceso de usuarios, sistema y aplicaciones estrictamente a los recursos y permisos necesarios para cumplir su función.

Con beneficios como:

- Acceso restringido  
- Seguridad proactiva  
- Implementación  
- Confianza cero  

---

###  Definición de Roles

Tenemos tres niveles de acceso principales:

**Rol Administrador (dbAdmin):**  
Tiene permisos de control total sobre la estructura y los datos. Puede crear tablas, modificar esquemas, gestionar copias de seguridad y administrar usuarios. Su uso es exclusivo para tareas de mantenimiento y migración.

**Rol Aplicación (appUser):**  
Tiene permisos de `SELECT`, `INSERT`, `UPDATE`, `DELETE` en las tablas del negocio, con restricciones sobre el borrado de tablas y modificaciones estructurales. Representa el usuario de la aplicación.

**Rol Auditor / Reportes (appAuditor):**  
Cuenta únicamente con permisos de `SELECT`. Puede leer información de transacciones y presupuestos para generar reportes o auditorías externas.

---

### Tabla de Permisos

| Tabla        | Administrador | Aplicación                          | Auditor |
|-------------|--------------|------------------------------------|--------|
| USUARIO     | ALL          | SELECT, INSERT, UPDATE             | NONE   |
| CUENTA      | ALL          | SELECT, INSERT, UPDATE, DELETE     | SELECT |
| CATEGORÍA   | ALL          | SELECT                             | SELECT |
| TRANSACCIÓN | ALL          | SELECT, INSERT, UPDATE             | SELECT |
| PRESUPUESTO | ALL          | SELECT, INSERT, UPDATE, DELETE     | SELECT |

---

## ADICIONAL
Estas son algunas de las  Querys que se usaron para probar las consultas, las adicionamos por si las llega a nesecitar.

```sql
-- 1. Insertar un Usuario
INSERT INTO USUARIO (nombre, correo, contraseña) 
VALUES ('Juan Perez', 'juan@mail.com', 'hash_password_123');

-- 2. Insertar Categorías
INSERT INTO CATEGORIA (nombre_categoria, descripcion, tipo_categoria) 
VALUES ('Comida', 'Gastos de restaurante y mercado', 'Gasto'),
       ('Sueldo', 'Ingreso mensual laboral', 'Ingreso');

-- 3. Insertar una Cuenta (Ligada al usuario 1)
INSERT INTO CUENTA (id_usuario, nombre_cuenta, tipo_cuenta, saldo_inicial) 
VALUES (1, 'Ahorros Bancolombia', 'Cuenta de Ahorros', 1000.00);

-- 4. Insertar un Presupuesto (Ligado al usuario 1 y categoría 1)
INSERT INTO PRESUPUESTO (id_usuario, id_categoria, monto_limite, fecha) 
VALUES (1, 1, 500.00, '2024-03-01');

-- 5. Insertar Transacciones
INSERT INTO TRANSACCION ( id_categoria, id_cuenta, monto, descripcion) 
VALUES ( 1, 1, 50.00, 'Almuerzo ejecutivo'),
       ( 1, 1, 30.00, 'Cena familiar'); 
```