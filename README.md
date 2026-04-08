# Fabrica-Base-de-Datos
# Diseño de Base de Datos - Sistema de Gestión Financiera

Este repositorio contiene la documentación técnica, los modelos de datos y las reglas de negocio para el módulo de gestión financiera personal.

## 1. DEFINICIÓN DE ENTIDADES

* **USUARIO:** No será solo “quien usa la app”. Es el propietario de la información financiera, debe garantizar la integridad del acceso mediante credenciales válidas y manejar el estado de seguridad (bloqueo por intentos fallidos). **(HU01 - HU02)**
* **TRANSACCIÓN:** Es cada registro individual de entrada o salida de dinero. Es un evento financiero que requiere obligatoriamente un monto > 0, una fecha, una categoría y una cuenta origen/destino. **(HU03 - HU04)**
* **CATEGORÍA:** Es la etiqueta lógica que agrupa las transacciones para su análisis, permitiendo la organización del gasto en segmentos predefinidos o personalizados. **(HU05)**
* **PRESUPUESTO:** Es una meta o límite establecido que sirve como herramienta de monitoreo del gasto proyectado contra el ejecutado por categoría. **(HU06)**
* **CUENTA:** Es la representación física o digital de donde reside el dinero. Funciona como puente entre **USUARIO** y **TRANSACCIÓN**, permitiendo segmentar el dinero para un balance detallado. **(HU03 - HU04)**

---

## 2. PRINCIPALES CONSULTAS IDENTIFICADAS

Para la validación del modelo, se utiliza la notación de **Álgebra Relacional**, donde:
* **σ (sigma):** Se usa para filtrar filas (selección).
* **π (pi):** Se usa para proyectar columnas (proyección).
* **⋈ (bowtie):** Representa el *Join* para unir tablas mediante llaves.

### Consultas de Negocio

**A. Obtener el nombre de la cuenta y su saldo actual para un usuario específico:**
$$\pi_{nombre\_cuenta, saldo\_actual}(\sigma_{id\_usuario=\#}(CUENTA))$$

**B. Listar todas las transacciones que sean de una categoría particular:**
$$\pi_{monto, fecha, descripcion}(TRANSACCIÓN \bowtie_{id\_categoria} (\sigma_{nombre\_categoria='nombre'}(CATEGORÍA)))$$

**C. Comparar el monto límite del presupuesto contra lo gastado en una categoría para calcular porcentajes:**
$$\pi_{monto\_limite, monto}(\sigma_{id\_usuario=1}(PRESUPUESTO \bowtie_{id\_categoria} TRANSACCIÓN))$$

**D. Seleccionar los nombres de las categorías y los montos de las transacciones para generar el gráfico de torta de gastos:**
$$\pi_{nombre\_cate, monto}(CATEGORÍA \bowtie_{id\_categoria} TRANSACCION)$$

---

## 3. MODELOS GRÁFICOS
*(Asegúrate de subir tus imágenes a una carpeta llamada 'diagramas' en este repositorio)*

### Modelo Lógico
![Modelo Lógico](./diagramas/logico.png)

### Modelo Físico
![Modelo Físico](./diagramas/fisico.png)
