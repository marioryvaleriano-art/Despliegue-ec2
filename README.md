# 🚀 Despliegue en K3s (EC2 en Subredes Públicas y Privadas) - Bodega Peirano

Esta carpeta contiene la arquitectura y manifiestos para desplegar la solución políglota distribuida en **AWS EC2** utilizando **K3s (Kubernetes ligero)**, con aislamiento estricto de red:
- **Subred Pública:** Master K3s, Worker Frontend y AWS Application Load Balancer (ALB).
- **Subred Privada:** Workers Backend (Spring Boot y FastAPI) y Base de Datos **MongoDB en EC2 dedicada**, con salida a internet controlada vía **NAT Gateway**.
- **Servicios K8s:** Backends con `ClusterIP` (100% privados) y Frontend con `NodePort` para recepción de tráfico del ALB.

---

## 🏛️ 1. Arquitectura de Red y Topología en AWS

```text
                                Internet (Clientes)
                                        │
                         ┌──────────────┴──────────────┐
                         ▼                             ▼
                 ┌──────────────┐               ┌──────────────┐
                 │ SSH / Admin  │               │ AWS ALB (80) │ (Público)
                 └──────┬───────┘               └──────┬───────┘
                        │                              │
════════════════════════╪══════════════════════════════╪════════════════════════════════
 SUBRED PÚBLICA (10.0.1.0/24 & 10.0.2.0/24)            │ (Forward al NodePort)
                        │                              ▼
        ┌───────────────┴───────────────┐      ┌─────────────────────────────┐
        │ EC2-Master (K3s Server)       │      │ EC2-Worker-Frontend         │
        │ - IP Pública + IP Privada     │      │ - K3s Agent (React + Nginx) │
        │ - Kubectl / Plano de Control  │      │ - NodePort: 3XXXX           │
        └───────────────────────────────┘      └──────────────┬──────────────┘
                        ▲                                     │ (Tráfico interno K8s)
                        │ (API K3s: 6443 / vxlan: 8472)        │
════════════════════════╪═════════════════════════════════════╪═════════════════════════
 SUBRED PRIVADA (10.0.10.0/24 - Sin IP Pública, Salida vía NAT Gateway)
                        │                                     ▼
        ┌───────────────┴───────────────┐      ┌─────────────────────────────┐
        │ EC2-Worker-Backend-Java       │      │ EC2-Worker-Backend-BA       │
        │ - Spring Boot WebFlux         │      │ - FastAPI + Google Gemini   │
        │ - Service: ClusterIP (8085)   │      │ - Service: ClusterIP (8000) │
        └──────────────┬────────────────┘      └──────────────┬──────────────┘
                       │                                      │
                       │             (Puerto 27017)           │
                       └──────────────────────┬───────────────┘
                                              ▼
                               ┌─────────────────────────────┐
                               │ EC2-MongoDB-Standalone      │
                               │ - Servidor MongoDB Dedicado │
                               │ - Autenticación habilitada  │
                               └─────────────────────────────┘
                                              │ (Puerto 1433)
                                              ▼
                               ┌─────────────────────────────┐
                               │ AWS RDS SQL Server          │
                               └─────────────────────────────┘
```

---

## ☁️ 2. Configuración en la Consola de AWS (VPC, Subredes y NAT Gateway)

### 2.1. Crear la VPC y el Internet Gateway y nat 
>
![alt text](image.png) ![alt text](image-1.png)
---


## 🛡️ 3. Grupos de Seguridad (Security Groups)

Crea y configura los siguientes Security Groups:

### 3.1. `sg-alb` (Load Balancer Público) (este regla omitanlo para el despliegue desde cero no es obligatorio )
- **Inbound (Entrada):**
  - `HTTP` (TCP 80): Origen `0.0.0.0/0`
  - `HTTPS` (TCP 443): Origen `0.0.0.0/0`
- **Outbound (Salida):** Todo el tráfico a `0.0.0.0/0`.

### 3.2. `k3s-public` (Master y Worker Frontend)
- **Inbound (Entrada):**
  - `SSH` (TCP 22): Origen `Tu IP Pública /32` (para tu administración).
  - `Custom TCP` (Puerto 6443 - K3s API): Origen `10.0.0.0/16` y `Tu IP /32`.
  - `Custom TCP` (NodePort 30000 - 32767): Origen `sg-alb o 0.0.0.0. (si no utilizas el balanceador )`  (tráfico del ALB hacia el Frontend).

  - `All Traffic`: Origen `10.0.0.0/16` (comunicación interna entre nodos K3s).
- **Outbound (Salida):** Todo el tráfico a `0.0.0.0/0`.

<img width="1020" height="773" alt="image" src="https://github.com/user-attachments/assets/dd5c5ff9-ea2d-44e1-8724-cbfcc36bca51" />

### 3.3. `k3s-private` ( GS Workers Backend Java y Backend BA en VPC)
- **Inbound (Entrada):**
  - `SSH` (TCP 22): Origen `sg-k3s-public` (salto SSH seguro desde el Master).
  - `All Traffic` / Overlay K3s: Origen `10.0.0.0/16` (Flannel vxlan 8472, Kubelet 10250).
- **Outbound (Salida):** Todo el tráfico a `0.0.0.0/0` (vía NAT Gateway).

<img width="1366" height="696" alt="image" src="https://github.com/user-attachments/assets/59cf363d-80a4-4ac4-b6ea-cf1038808ccd" />


### 3.4. `mongodb-ec2` (GS MongoDB Dedicada en VPC)
- **Inbound (Entrada):**
  - `Custom TCP` (TCP 27017): Origen `sg-k3s-private` (o `10.0.10.0/24`). **Bloqueado para Internet.**
  - `SSH` (TCP 22): Origen `sg-k3s-public` (para ingresar por SSH desde el Master).
- **Outbound (Salida):** Todo el tráfico a `0.0.0.0/0` (vía NAT Gateway para descargar paquetes).

  <img width="1002" height="575" alt="image" src="https://github.com/user-attachments/assets/5db8f821-99e7-440c-a467-37deada6b5fb" />


### 3.5. `rds-sqlserver` ( GS RDS SQL Server en VPC)
- **Inbound (Entrada):**
  - `MSSQL` (TCP 1433): Origen `sg-k3s-private`.
  `MSSQL` (TCP 1433): Origen MI IP o 0.0.0 (0.0.0.0. se debe utilizar solo para insertar el script incialmente luego cambiar a MI IP o eliminar la regla)

<img width="1862" height="762" alt="image" src="https://github.com/user-attachments/assets/6c3646e7-286d-4ed8-8db4-464fd35f6fb6" />

---

## 🖥️ 4. Lanzamiento de Instancias EC2

Todas con **Ubuntu 24.04 LTS** o **Ubuntu 22.04 LTS**:

| Instancia | Subred | IP Pública | Tipo EC2 | Disco | Security Group | Rol |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **EC2-Master** | `subnet-public-1a` | **Sí** | `t3.medium` | 20 GB | `sg-k3s-public` | K3s Server |
| **EC2-Worker-Frontend** | `subnet-public-1a` | **Sí** | `t3.medium` | 20 GB | `sg-k3s-public` | K3s Agent (Frontend) |
| **EC2-Worker-Backend-Java** | `subnet-private-1a` | **No** | `t3.medium` | 20 GB | `sg-k3s-private` | K3s Agent (Spring Boot) |
| **EC2-Worker-Backend-BA** | `subnet-private-1a` | **No** | `t3.medium` | 20 GB | `sg-k3s-private` | K3s Agent (FastAPI) |
| **EC2-MongoDB** | `subnet-private-1a` | **No** | `t3.medium` / `t3.small` | **30 GB** | `sg-mongodb-ec2` | MongoDB Standalone |


---

## Crear RDS SQLSERVER


| **RDS | `subnet-private-1a` | **SI** | `t3.medium o la que viene por defecto | sqlserver | plan gratuito | 20GB | `subnet-private 1a`|

1. Crear base de datos
2. Seleccionar Microsoft SQL Server
3. Configuracion completa
4. En plantillas seleccionar Capa gratuita 
5. En configuracion, Configuracion de credencial seleccionar Autoadministrado
6. En nombre de usuairo colocar : sa 
7. En contrseña maestra colocar : YourStrongPassword2026 y confirmar la contraseña
8. luego deslizar hasta Conectividad y seleccionar la vpc creada vpc-bodega-k3s
9. Acceso publico en SI
10. Grupo de segurida elegir el que se ha creado rds-sqlserver  y crear instancia 
11. Modificar la tabla de enrutamiento de las subredes privadas y reemplzar el nat gateway por el internet gateway 
12. conectarse por msqlserver, insertar el script y por ultimo en las tabla de enrutamiento privadas cambiar internet gateway por el nat gateway.




#### Scrip
```

-- =====================================================
-- SCRIPT DE INICIALIZACIÓN SQL SERVER - BODEGA PEIRANO
-- Base de datos: BodegaPeiranoDBF
-- Tablas Transaccionales (Rediseño de Promociones)
-- =====================================================

IF NOT EXISTS (SELECT * FROM sys.databases WHERE name = 'BodegaPeiranoDBF')
BEGIN
    CREATE DATABASE BodegaPeiranoDBF;
END
GO

USE BodegaPeiranoDBF;
GO

-- =====================================================
-- TABLA: INVENTARIO (TRANSACCIONAL)
-- =====================================================
CREATE TABLE inventario (
    id_inventario INT IDENTITY(1,1) NOT NULL,
    id_producto INT NOT NULL, -- Referencia lógica a MongoDB
    id_almacen INT NOT NULL,  -- Referencia lógica a MongoDB
    stock_actual DECIMAL(10,2) NOT NULL,
    stock_minimo DECIMAL(10,2) NOT NULL,
    ultima_actualizacion DATETIME NOT NULL DEFAULT GETDATE(),
    estado VARCHAR(10) NOT NULL DEFAULT 'activo',
    CONSTRAINT inventario_pk PRIMARY KEY (id_inventario),
    CONSTRAINT inventario_stock_check CHECK (stock_actual >= 0 AND stock_minimo >= 0)
);
GO

-- =====================================================
-- TABLA: VENTA (TRANSACCIÓN PRINCIPAL)
-- =====================================================
CREATE TABLE venta (
    id_venta INT IDENTITY(1,1) NOT NULL,
    fecha_venta DATETIME2 NOT NULL DEFAULT GETDATE(),
    tipo_documento VARCHAR(20) NOT NULL,
    numero_documento VARCHAR(20) NOT NULL,
    estado CHAR(1) NOT NULL DEFAULT 'C', -- C=Completada, A=Anulada
    total DECIMAL(10,2) NOT NULL,
    referencia_transaccion VARCHAR(100) NULL,
    id_cliente INT NOT NULL, -- Referencia lógica a MongoDB
    id_almacen INT NOT NULL, -- Referencia lógica a MongoDB
    id_metodo_pago INT NOT NULL, -- Referencia lógica a MongoDB
    CONSTRAINT venta_pk PRIMARY KEY (id_venta),
    CONSTRAINT venta_total_positivo CHECK (total > 0),
    CONSTRAINT venta_estado_check CHECK (estado IN ('C','A')),
    CONSTRAINT venta_tipo_doc_check CHECK (tipo_documento IN ('Boleta', 'Factura', 'Ticket')),
    CONSTRAINT venta_documento_unico UNIQUE (tipo_documento, numero_documento)
);
GO

-- =====================================================
-- TABLA: DETALLE_VENTA
-- =====================================================
CREATE TABLE detalle_venta (
    id_detalle_venta INT IDENTITY(1,1) NOT NULL,
    id_venta INT NOT NULL,
    id_producto INT NOT NULL, -- Referencia lógica a MongoDB
    cantidad DECIMAL(10,2) NOT NULL,
    precio_unitario DECIMAL(10,2) NOT NULL,
    descuento DECIMAL(10,2) NULL DEFAULT 0,
    subtotal DECIMAL(10,2) NOT NULL,
    CONSTRAINT detalle_venta_pk PRIMARY KEY (id_detalle_venta),
    CONSTRAINT detalle_cantidad_positiva CHECK (cantidad > 0),
    CONSTRAINT detalle_subtotal_check CHECK (subtotal = (cantidad * precio_unitario) - descuento)
);
GO

-- =====================================================
-- TABLA: PROMOCION (CABECERA REDISEÑADA)
-- =====================================================
CREATE TABLE promocion (
    id_promocion INT IDENTITY(1,1) NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    descripcion VARCHAR(255) NULL,
    fecha_inicio DATETIME NOT NULL,
    fecha_fin DATETIME NOT NULL,
    tipo_promocion CHAR(1) NOT NULL, -- 'P' = Porcentaje por Producto, 'K' = Pack/Combo
    descuento_porcentaje DECIMAL(5,2) NULL, -- Ej: 10.00 (para un 10%)
    monto_fijo_pack DECIMAL(10,2) NULL,    -- Si el pack tiene un precio final cerrado
    estado BIT NOT NULL DEFAULT 1,
    CONSTRAINT promocion_pk PRIMARY KEY (id_promocion),
    CONSTRAINT promocion_fechas_check CHECK (fecha_fin > fecha_inicio),
    CONSTRAINT promocion_tipo_check CHECK (tipo_promocion IN ('P', 'K')),
    -- Validación lógica: Si es 'P' debe tener porcentaje, si es 'K' puede tener monto fijo
    CONSTRAINT promocion_valores_check CHECK (
        (tipo_promocion = 'P' AND descuento_porcentaje IS NOT NULL AND monto_fijo_pack IS NULL) OR
        (tipo_promocion = 'K' AND monto_fijo_pack IS NOT NULL)
    )
);
GO

-- =====================================================
-- TABLA: PROMOCION_PRODUCTO (RELACIÓN CON CANTIDAD)
-- =====================================================
CREATE TABLE promocion_producto (
    id_promocion_producto INT IDENTITY(1,1) NOT NULL,
    id_promocion INT NOT NULL,
    id_producto INT NOT NULL, -- Referencia lógica a MongoDB
    cantidad_requerida INT NOT NULL DEFAULT 1, -- NUEVO: Para armar packs (ej. 2 Gaseosas)
    aplica_descuento_adicional BIT NOT NULL DEFAULT 0,
    CONSTRAINT promocion_producto_pk PRIMARY KEY (id_promocion_producto),
    CONSTRAINT promocion_producto_unico UNIQUE (id_promocion, id_producto),
    CONSTRAINT promocion_producto_cantidad_check CHECK (cantidad_requerida > 0)
);
GO

-- =====================================================
-- INSERCIÓN DE DATOS TRANSACCIONALES
-- =====================================================

-- 1. INVENTARIO
INSERT INTO inventario (id_producto, id_almacen, stock_actual, stock_minimo) VALUES
(1, 1, 50.00, 10.00), -- Pisco Quebranta
(2, 1, 40.00, 10.00), -- Pisco Italia
(7, 1, 100.00, 20.00), -- Vino EUMA Tinto
(10, 1, 30.00, 5.00), -- Sangría Tinta
(12, 1, 60.00, 15.00); -- Jarabe Goma
GO

-- 2. VENTAS
INSERT INTO venta (tipo_documento, numero_documento, estado, total, referencia_transaccion, id_cliente, id_almacen, id_metodo_pago) VALUES
('Boleta', 'B001-0001', 'C', 60.00, 'YAPE-123', 1, 1, 2),
('Factura', 'F001-0001', 'C', 120.00, 'TRF-456', 5, 4, 5);
GO

-- 3. DETALLE_VENTA
INSERT INTO detalle_venta (id_venta, id_producto, cantidad, precio_unitario, descuento, subtotal) VALUES
(1, 1, 2.00, 30.00, 0.00, 60.00), -- 2 Piscos Quebranta
(2, 6, 1.00, 120.00, 0.00, 120.00); -- 1 Pisco Acholado Galón
GO

-- =====================================================
-- 4. PROMOCIONES (Aplicando el Rediseño)
-- =====================================================

-- Caso 1: Promoción por Porcentaje ('P')
-- Ej: "Semana del Pisco: 15% de descuento en Piscos Peirano"
INSERT INTO promocion (nombre, descripcion, fecha_inicio, fecha_fin, tipo_promocion, descuento_porcentaje, monto_fijo_pack, estado) 
VALUES ('Semana del Pisco', '15% desc en Piscos seleccionados', '2026-07-01', '2026-07-31', 'P', 15.00, NULL, 1);

-- Caso 2: Promoción por Pack/Combo ('K')
-- Ej: "Pack Fiestas Patrias: Pisco + 2 Sangrías + 1 Jarabe por S/ 50.00"
INSERT INTO promocion (nombre, descripcion, fecha_inicio, fecha_fin, tipo_promocion, descuento_porcentaje, monto_fijo_pack, estado) 
VALUES ('Pack Fiestas Patrias', 'Pisco + 2 Sangrías + Jarabe', '2026-07-15', '2026-07-30', 'K', NULL, 50.00, 1);

-- 5. PROMOCION_PRODUCTO (Vinculando lógica)

-- Detalle de la Promo 1 (Porcentaje 'P'): Aplica a Piscos individuales
INSERT INTO promocion_producto (id_promocion, id_producto, cantidad_requerida, aplica_descuento_adicional) VALUES
(1, 1, 1, 0), -- Pisco Quebranta
(1, 2, 1, 0), -- Pisco Italia
(1, 3, 1, 0); -- Pisco Acholado

-- Detalle de la Promo 2 (Pack 'K'): Requiere múltiples productos y cantidades
INSERT INTO promocion_producto (id_promocion, id_producto, cantidad_requerida, aplica_descuento_adicional) VALUES
(2, 1, 1, 0), -- 1 Pisco Quebranta (ID 1)
(2, 10, 2, 0), -- 2 Sangrías Tintas (ID 10)
(2, 12, 1, 0); -- 1 Jarabe de Goma (ID 12)
GO

-- Verificar conteo
SELECT 'PROMOCION' AS Tabla, COUNT(*) AS Total FROM promocion
UNION ALL SELECT 'PROMOCION_PRODUCTO', COUNT(*) FROM promocion_producto;
GO
```

## 🍃 5. Instalación y Configuración de MongoDB en la Instancia EC2 Dedicada

| Instancia | Subred | IP Pública | Tipo EC2 | Disco | Security Group | Rol |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **EC2-MongoDB** | `subnet-private-1a` | **No** | `t3.medium` / `t3.small` | **30 GB** | `sg-mongodb-ec2` | MongoDB Standalone |
1. Copiar y pegar en Datos de Usario  DESDE  #!/bin/bash HASTA rm /tmp/init-bodega.js
echo "=== DESPLIEGUE FINALIZADO ==="   
2. luego crear instancia. 

#### Script 
```
#!/bin/bash
# 0. Redirigir toda la salida a un log para poder depurar (en /var/log/user-data.log)
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
export DEBIAN_FRONTEND=noninteractive

echo "=== INICIANDO DESPLIEGUE DE MONGODB PARA BODEGA PEIRANO ==="

# 1. Esperar que se libere el bloqueo de paquetes de AWS
while fuser /var/lib/dpkg/lock >/dev/null 2>&1 || fuser /var/lib/apt/lists/lock >/dev/null 2>&1; do sleep 10; done

# 2. Instalar dependencias y llave de MongoDB 7.0
apt-get update
apt-get install -y gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor --yes

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list

apt-get update
apt-get install -y mongodb-org

# 3. Iniciar el servicio y esperar a que el puerto 27017 esté listo
systemctl daemon-reload
systemctl enable mongod
systemctl start mongod
sleep 15

# 4. Crear usuario administrador
mongosh <<EOF
use admin
db.createUser({
  user: "admin",
  pwd: "Mongo12345!",
  roles: [ { role: "root", db: "admin" } ]
})
EOF

# 5. Configurar bindIp a 0.0.0.0 y activar autenticación
sed -i 's/bindIp: 127.0.0.1/bindIp: 0.0.0.0/' /etc/mongod.conf

cat >> /etc/mongod.conf <<EOF
security:
  authorization: enabled
EOF

# 6. Reiniciar MongoDB para aplicar la seguridad
systemctl restart mongod
sleep 15

# 7. Crear el script JS de inicialización de datos de la bodega
cat << 'EOF_JS' > /tmp/init-bodega.js
const db = db.getSiblingDB('BodegaPeiranoDB');

db.ubigeo.deleteMany({});
db.categoria.deleteMany({});
db.producto.deleteMany({});
db.cliente.deleteMany({});
db.almacen.deleteMany({});
db.metodo_pago.deleteMany({});

db.ubigeo.insertMany([
    { _id: "150101", distrito: "Lima", provincia: "Lima", departamento: "Lima" },
    { _id: "150102", distrito: "Asia", provincia: "Cañete", departamento: "Lima" },
    { _id: "150103", distrito: "Chilca", provincia: "Cañete", departamento: "Lima" },
    { _id: "150104", distrito: "Mala", provincia: "Cañete", departamento: "Lima" },
    { _id: "150105", distrito: "San Vicente de Cañete", provincia: "Cañete", departamento: "Lima" },
    { _id: "150106", distrito: "Imperial", provincia: "Cañete", departamento: "Lima" },
    { _id: "150108", distrito: "Lunahuana", provincia: "Cañete", departamento: "Lima" },
    { _id: "150201", distrito: "Arequipa", provincia: "Arequipa", departamento: "Arequipa" },
    { _id: "150202", distrito: "Cayma", provincia: "Arequipa", departamento: "Arequipa" },
    { _id: "150203", distrito: "Yanahuara", provincia: "Arequipa", departamento: "Arequipa" },
    { _id: "150301", distrito: "Cusco", provincia: "Cusco", departamento: "Cusco" },
    { _id: "150302", distrito: "San Jeronimo", provincia: "Cusco", departamento: "Cusco" },
    { _id: "150401", distrito: "Trujillo", provincia: "Trujillo", departamento: "La Libertad" },
    { _id: "150402", distrito: "Victor Larco", provincia: "Trujillo", departamento: "La Libertad" },
    { _id: "150501", distrito: "Piura", provincia: "Piura", departamento: "Piura" },
    { _id: "150502", distrito: "Castilla", provincia: "Piura", departamento: "Piura" }
]);

const ahora = new Date();

db.categoria.insertMany([
    { _id: 1, nombre: "Pisco",    descripcion: "Destilados de uva",                 codigo_categoria: "PIS", estado: true, fecha_creacion: ahora },
    { _id: 2, nombre: "Vino",     descripcion: "Vinos tintos, blancos y semisecos", codigo_categoria: "VIN", estado: true, fecha_creacion: ahora },
    { _id: 3, nombre: "Sangría",  descripcion: "Bebidas a base de vino y frutas",   codigo_categoria: "SAN", estado: true, fecha_creacion: ahora },
    { _id: 4, nombre: "Jarabes",  descripcion: "Jarabes para coctelería",           codigo_categoria: "JAR", estado: true, fecha_creacion: ahora }
]);

db.producto.insertMany([
    { _id: 1, nombre: "PISCO PEIRANO PURO QUEBRANTA botella 750 ml.", descripcion: "Pisco puro quebranta artesanal", codigo_barras_sku: "7750123456789", unidad_medida: "750ml", precio_unitario: 30.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 2, nombre: "PISCO PEIRANO PURO ITALIA botella 750 ml.",    descripcion: "Pisco puro italia premium",       codigo_barras_sku: "7750987654321", unidad_medida: "750ml", precio_unitario: 30.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 3, nombre: "PISCO PEIRANO ACHOLADO botella 750 ml.",       descripcion: "Pisco acholado especial",         codigo_barras_sku: "7750223344556", unidad_medida: "750ml", precio_unitario: 30.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 4, nombre: "PISCO PEIRANO PURO QUEBRANTA galon 4 Lt.",     descripcion: "Pisco quebranta galón",            codigo_barras_sku: "SKU-PIS-QUE-GAL", unidad_medida: "Galon 4L", precio_unitario: 120.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 5, nombre: "PISCO PEIRANO PURO ITALIA galon 4 Lt.",        descripcion: "Pisco italia galón",              codigo_barras_sku: "SKU-PIS-ITA-GAL", unidad_medida: "Galon 4L", precio_unitario: 120.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 6, nombre: "PISCO PEIRANO ACHOLADO galon 4 Lt.",           descripcion: "Pisco acholado galón",             codigo_barras_sku: "SKU-PIS-ACH-GAL", unidad_medida: "Galon 4L", precio_unitario: 100.00, id_categoria: 1, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 7, nombre: "Vino EUMA Tinto (Seco) botella 750 ml.",       descripcion: "Vino tinto seco",                 codigo_barras_sku: "SKU-VIN-EUM-TIN", unidad_medida: "750ml", precio_unitario: 22.00, id_categoria: 2, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 8, nombre: "Vino EUMA Tinto (Semi Seco) botella 750 ml.",  descripcion: "Vino tinto semiseco",             codigo_barras_sku: "SKU-VIN-EUM-SEM", unidad_medida: "750ml", precio_unitario: 22.00, id_categoria: 2, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 9, nombre: "Vino EUMA Blanco (Dulce) botella 750 ml.",     descripcion: "Vino blanco dulce",               codigo_barras_sku: "SKU-VIN-EUM-BLA", unidad_medida: "750ml", precio_unitario: 22.00, id_categoria: 2, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 10, nombre: "SANGRIA TINTA SANGRIA 1L.",   descripcion: "Sangría tinta lista para servir",  codigo_barras_sku: "SKU-SAN-TIN-1L", unidad_medida: "1L", precio_unitario: 12.00, id_categoria: 3, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 11, nombre: "SANGRIA BLANCA BLANGRIA 1L.", descripcion: "Sangría blanca lista para servir", codigo_barras_sku: "SKU-SAN-BLA-1L", unidad_medida: "1L", precio_unitario: 12.00, id_categoria: 3, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 12, nombre: "Jarabe de Goma JAPI botella 750 ml.",      descripcion: "Jarabe de goma para coctelería",      codigo_barras_sku: "SKU-JAR-GOM-750", unidad_medida: "750ml", precio_unitario: 7.00, id_categoria: 4, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null },
    { _id: 13, nombre: "Jarabe de Granadina JAPI botella 750 ml.", descripcion: "Jarabe de granadina para coctelería", codigo_barras_sku: "SKU-JAR-GRA-750", unidad_medida: "750ml", precio_unitario: 8.00, id_categoria: 4, estado: true, fecha_creacion: ahora, fecha_actualizacion: null, fecha_eliminacion: null, fecha_restauracion: null }
]);

db.cliente.insertMany([
    { _id: 1, tipo_documento: 'DNI', numero_documento: '72145678',  nombre: 'Sebastian',          apellido: 'Alvarez', correo: 'sebastian@mail.com',   telefono: '987654321', fecha_nacimiento: '1990-05-15', ubigeo_id: '150101', estado: true, fecha_creacion: ahora },
    { _id: 2, tipo_documento: 'DNI', numero_documento: '45678912',  nombre: 'Angel',              apellido: 'Perez',   correo: 'angel.p@mail.com',     telefono: '999888777', fecha_nacimiento: '1985-08-20', ubigeo_id: '150102', estado: true, fecha_creacion: ahora },
    { _id: 3, tipo_documento: 'CE',  numero_documento: '00123456',  nombre: 'Valeriano',          apellido: 'Gomez',   correo: 'valeriano@mail.com',   telefono: '912345678', fecha_nacimiento: '1982-12-10', ubigeo_id: '150103', estado: true, fecha_creacion: ahora },
    { _id: 4, tipo_documento: 'DNI', numero_documento: '12345678',  nombre: 'Maria',              apellido: 'Sosa',    correo: 'maria.sosa@mail.com',  telefono: '955444333', fecha_nacimiento: '1995-03-25', ubigeo_id: '150101', estado: true, fecha_creacion: ahora },
    { _id: 5, tipo_documento: 'RUC', numero_documento: '20123456789', nombre: 'Distribuidora Asia', apellido: 'SAC',   correo: 'ventas@asiacorp.pe',   telefono: '944333222', fecha_nacimiento: '1990-01-01', ubigeo_id: '150102', estado: true, fecha_creacion: ahora }
]);

db.almacen.insertMany([
    { _id: 1, nombre: "Bodega Principal - Asia", ubicacion: "Av. Principal 123, Asia",  codigo_almacen: "BOD-001", ubigeo_id: "150102", estado: "Activo", fecha_creacion: ahora },
    { _id: 2, nombre: "Bodega Costa Azul",       ubicacion: "Carretera Panamericana Sur", codigo_almacen: "BOD-002", ubigeo_id: "150103", estado: "Activo", fecha_creacion: ahora },
    { _id: 3, nombre: "Bodega Norteña",          ubicacion: "Industrial Mz A Lt 5",       codigo_almacen: "BOD-003", ubigeo_id: "150401", estado: "Activo", fecha_creacion: ahora },
    { _id: 4, nombre: "Almacén Central Lima",    ubicacion: "San Isidro 321",             codigo_almacen: "BOD-007", ubigeo_id: "150101", estado: "Activo", fecha_creacion: ahora }
]);

db.metodo_pago.insertMany([
    { _id: 1, nombre: 'Efectivo',           es_electronico: false, estado: true },
    { _id: 2, nombre: 'Yape',               es_electronico: true,  estado: true },
    { _id: 3, nombre: 'Plin',               es_electronico: true,  estado: true },
    { _id: 4, nombre: 'Tarjeta Visa',       es_electronico: true,  estado: true },
    { _id: 5, nombre: 'Transferencia BCP',  es_electronico: true,  estado: true }
]);

print("✅ MongoDB: Datos maestros insertados correctamente.");
EOF_JS

# 8. Ejecutar el script JS usando el usuario administrador
mongosh -u "admin" -p "Mongo12345!" --authenticationDatabase "admin" /tmp/init-bodega.js

# 9. Limpiar el script temporal por seguridad
rm /tmp/init-bodega.js
echo "=== DESPLIEGUE FINALIZADO ==="


```


---

## ⚡ 6. Instalación del Clúster K3s

### 6.1. En la instancia MASTER (K3s Server)
Conéctate por SSH al Master:
```powershell
ssh -i "C:\Users\user\Downloads\tu-llave.pem" ubuntu@<IP_PUBLICA_MASTER>
```

Instala K3s Server desactivando Traefik (usaremos el ALB y Nginx):
```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --node-name master
```

Obtén el token de unión del clúster:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
> 📌 Copia el **TOKEN** mostrado.

---

### 6.2. En el Worker Frontend (Público)
Conéctate al Worker Frontend por SSH directo:
```powershell
ssh -i "C:\Users\user\Downloads\tu-llave.pem" ubuntu@<IP_PUBLICA_FRONTEND>
```
Ejecuta la instalación del agente K3s:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_PRIVADA_MASTER>:6443 K3S_TOKEN=<TOKEN> sh -s - --node-name worker-frontend
```

OPCION RAPIDA DATOS DE USUARIO AWS  esto sirve para todos los backend privados tambien solo cambiar esta parte node-role=frontend-worker
```
#!/bin/bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.11.178:6443 K3S_TOKEN=K10532fc07783ccd8e3d80950a3429f6dd38b83400f2dd58de090b416fdeae2f77f::server:27e1ed16d5c95d39a4c6dbeabd7107ea sh -s - --node-name worker-frontend --node-label node-role=frontend-worker
```


---

### 6.3. En los Workers Privados (Backend Java y Backend BA)
Conéctate por SSH saltando desde el Master a cada IP privada:

```powershell
# Conectar a BACKEND JAVA:
ssh -i "tu-llave.pem" -J ubuntu@<IP_PUBLICA_MASTER> ubuntu@<IP_PRIVADA_BACKEND_JAVA>
```
Ejecuta:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_PRIVADA_MASTER>:6443 K3S_TOKEN=<TOKEN> sh -s - --node-name worker-backend-java


```
```MANERA RAPIDA PRO DATOS DE USUAARIO 
#!/bin/bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.15.5:6443 K3S_TOKEN=K107ffd5d5086ca6d0c7507b4fca91b102096381933023d7d556544c132d239b952::server:0ae2ff09dfb48073c8a11331ce36a788 sh -s - --node-name worker-backend-java
```

```powershell
# Conectar BACKEND BA:
ssh -i "tu-llave.pem" -J ubuntu@<IP_PUBLICA_MASTER> ubuntu@<IP_PRIVADA_BACKEND_BA>
```
Ejecuta:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_PRIVADA_MASTER>:6443 K3S_TOKEN=<TOKEN> sh -s - --node-name worker-backend-ba
```
```
#!/bin/bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.15.5:6443 K3S_TOKEN=K107ffd5d5086ca6d0c7507b4fca91b102096381933023d7d556544c132d239b952::server:0ae2ff09dfb48073c8a11331ce36a788 sh -s - --node-name worker-backend-ba

---

### 6.4. Verificar y Etiquetar los Nodos (En el MASTER)
Regresa a la terminal del **Master** y comprueba los nodos:

```bash
kubectl get nodes
```

Aplica las etiquetas para los `nodeSelector` de los despliegues:
```bash
kubectl label nodes worker-frontend node-role=frontend-worker --overwrite
kubectl label nodes worker-backend-java node-role=backend-worker --overwrite
kubectl label nodes worker-backend-ba node-role=backend-ba-worker --overwrite

# Verificar etiquetas asignadas:
kubectl get nodes -L node-role
```

---

## 📦 7. Despliegue de los Manifiestos K8s

### 7.1. Actualizar la IP de MongoDB en `02-configmap.yaml`
En el archivo `02-configmap.yaml`, reemplaza `IP_PRIVADA_EC2_MONGODB` por la IP privada real de tu EC2 de MongoDB (ej. `10.0.10.50`):

```yaml
  MONGO_HOST: "10.0.10.50"
  MONGO_PORT: "27017"
  MONGO_URI: "mongodb://admin:Mongo12345!@10.0.10.50:27017/BodegaPeiranoDB?authSource=admin"
```

---

### 7.2. Copiar los Manifiestos al Master con SCP
Desde tu computadora Windows:

```powershell
scp -i "C:\Users\user\Downloads\tu-llave.pem" -r "c:\Users\user\Desktop\manifest-angel\manifest-ec2" ubuntu@<IP_PUBLICA_MASTER>:~/manifest-ec2
```
scp -i "C:\Users\user\Downloads\instancia1.pem" -r "c:\Users\user\Desktop\manifest-angel\manifest-ec2" ubuntu@44.192.63.188:~/manifest-ec2 



---

### 7.3. Aplicar los Manifiestos en Orden (En el Master)
En la terminal del **Master**:

```bash
cd ~/manifest-ec2

# 1. Namespace, ConfigMap y Secret
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-secret.yaml

# 2. Backends (Spring Boot y BA - Ambos ClusterIP privados)
kubectl apply -f 04-backend-deployment.yaml
kubectl apply -f 05-backend-service.yaml
kubectl apply -f 13-backend-ba-deployment.yaml
kubectl apply -f 14-backend-ba-service.yaml

#En caso se necesite se pueden borrar los deplyments
kubectl delete -f 04-backend-deployment.yaml -f 05-backend-service.yaml -f 13-backend-ba-deployment.yaml -f 14-backend-ba-service.yaml 
kubectl delete -f 06-frontend-deployment.yaml -f 07-frontend-service.yaml 



# 3. Frontend (NodePort para ALB)
kubectl apply -f 06-frontend-deployment.yaml
kubectl apply -f 07-frontend-service.yaml

# 4. Ingress (Opcional si usas ALB Controller nativo) ) (no aplicar el manifiesto no es necesario)
kubectl apply -f 08-ingress-alb.yaml
```

---

## 🔍 8. Verificación del Despliegue

```bash
# Ver todos los pods y servicios en bodega-app
kubectl get all -n bodega-app -o wide

# Verificar que los servicios de Backend son ClusterIP y Frontend es NodePort:
kubectl get svc -n bodega-app
```
*Salida esperada:*
```text
NAME                 TYPE        CLUSTER-IP      PORT(S)
backend-service      ClusterIP   10.43.x.x       8085/TCP
backend-ba-service   ClusterIP   10.43.y.y       8000/TCP
frontend-service     NodePort    10.43.z.z       80:3XXXX/TCP
```

---

## ⚖️ 9. Configuración Completa Paso a Paso del Application Load Balancer (ALB) en AWS Console ( este paso no es necessario si no quieres aplicar el balanceador)

El Application Load Balancer (ALB) recibe el tráfico público de Internet en el puerto HTTP 80 y lo reenvía de forma balanceada y segura hacia la instancia `EC2-Worker-Frontend` al puerto `NodePort` expuesto por Kubernetes.

---

### Paso 9.1: Consultar el puerto NodePort del Frontend en el Master
En tu terminal del Master (`ubuntu@ip-10-0-15-5`), ejecuta:
```bash
kubectl get svc frontend-service -n bodega-app
```
Verás una salida similar a esta:
```text
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
frontend-service   NodePort   10.43.197.239   <none>        80:30464/TCP   30m
```
> 📌 **Anota el puerto de 5 dígitos generado** después de los dos puntos (en este ejemplo: **`30464`**).

---

### Paso 9.2: Verificar la regla del Security Group del Worker Frontend (`sg-k3s-public`)
Para que el ALB pueda entregar paquetes al NodePort de la instancia del Frontend:
1. En **AWS Console** ➔ **EC2** ➔ **Security Groups**.
2. Selecciona **`sg-k3s-public`** (asignado a `EC2-Worker-Frontend`).
3. En **Inbound rules** ➔ **Edit inbound rules**, confirma que exista la regla:
   - **Type:** `Custom TCP`
   - **Port range:** `30000 - 32767` (o el puerto exacto, ej: `30464`)
   - **Source:** Selecciona el Security Group del ALB **`sg-alb`** (o `0.0.0.0/0`).
4. Guarda las reglas.

---

### Paso 9.3: Crear el Target Group (`tg-bodega-frontend`)
1. Ve a **EC2** ➔ **Target Groups** (en el menú lateral izquierdo bajo *Load Balancing*).
2. Haz clic en el botón naranja **Create target group**.
3. **Step 1: Specify group details**:
   - **Choose a target type:** Selecciona **`Instances`**.
   - **Target group name:** `tg-bodega-frontend`.
   - **Protocol : Port:** Selecciona **`HTTP`** y en el puerto escribe el NodePort de tu Frontend (ejemplo: **`30464`**).
   - **IP address type:** `IPv4`.
   - **VPC:** Selecciona tu VPC de trabajo (**`vpc-bodega-k3s`**).
   - **Protocol version:** `HTTP1`.
   - **Health checks:**
     - **Health check protocol:** `HTTP`.
     - **Health check path:** `/health` *(devuelve status 200 rápido y limpio)* o `/`.
   - **Advanced health check settings** (Opcional recomendado):
     - *Healthy threshold:* `2`
     - *Unhealthy threshold:* `2`
     - *Timeout:* `5` segundos
     - *Interval:* `15` segundos
     - *Success codes:* `200`
4. Haz clic en **Next**.

5. **Step 2: Register targets**:
   - En la lista de instancias (*Available instances*), busca y marca la casilla de la instancia:
     **`EC2-Worker-Frontend`**.
   - En el campo **Ports for the selected instances**, verifica que figure el NodePort (ej: **`30464`**).
   - Haz clic en el botón: **Include as pending below**.
   - Verifica que en la tabla inferior (*Review pending targets*) aparezca la instancia con el puerto `30464` y estado `Pending`.
6. Haz clic en **Create target group**.

---

### Paso 9.4: Crear el Application Load Balancer (`alb-bodega-peirano`)
1. Ve a **EC2** ➔ **Load Balancers** (en el menú lateral bajo *Load Balancing*).
2. Haz clic en **Create load balancer**.
3. En la tarjeta **Application Load Balancer**, haz clic en **Create**.
4. **Basic configuration**:
   - **Load balancer name:** `alb-bodega-peirano`.
   - **Scheme:** Selecciona **`Internet-facing`** (Público, para acceso de clientes en Internet).
   - **IP address type:** `IPv4`.
5. **Network mapping**:
   - **VPC:** Selecciona tu VPC (**`vpc-bodega-k3s`**).
   - **Mappings (Subredes públicas):** Debes marcar **al menos 2 Zonas de Disponibilidad (AZs)** que contengan subredes públicas:
     - Zona 1 (ej: `us-east-1a`): Selecciona **`subnet-public-1a`**.
     - Zona 2 (ej: `us-east-1b`): Selecciona **`subnet-public-1b`**.
6. **Security groups**:
   - En la sección de Security groups, asigna **`sg-alb`** (que tiene abiertas las reglas de entrada HTTP 80 y HTTPS 443 desde `0.0.0.0/0`).
   - *Nota:* Si el grupo `default` está seleccionado, desmárcalo.
7. **Listeners and routing**:
   - **Protocol:** `HTTP`
   - **Port:** `80`
   - **Default action:** `Forward to` ➔ Selecciona tu Target Group creado: **`tg-bodega-frontend`**.
8. Revisa el resumen y haz clic al final en **Create load balancer**.
9. Espera entre **1 y 2 minutos** hasta que el **State** del Load Balancer cambie de `Provisioning` a **`Active`**.

---

### Paso 9.5: Validación de Salud y Acceso Web en el Navegador
1. Ve a **EC2** ➔ **Target Groups** ➔ Haz clic en **`tg-bodega-frontend`**.
2. Ve a la pestaña **Targets**.
3. Confirma que la instancia `EC2-Worker-Frontend` muestre su **Health status** en verde como:
   ```text
   Healthy (Health checks passing)
   ```
4. Ahora ve a **EC2** ➔ **Load Balancers** ➔ Haz clic en **`alb-bodega-peirano`**.
5. En la pestaña **Description / Details**, copia el valor de **DNS name**. Ejemplo:
   ```text
   alb-bodega-peirano-123456789.us-east-1.elb.amazonaws.com
   ```
6. Abre tu navegador web e ingresa por HTTP:
   ```text
   http://<DNS_NAME_DEL_ALB>
   ```

🎯 **¡Listo!** El ALB recibirá tus peticiones en Internet, las remitirá al Worker Frontend por el NodePort, y el Frontend Nginx servirá la aplicación React comunicándose internamente con los microservicios Spring Boot y FastAPI a través de la red privada del clúster K3s.


---

## 💡 10. Solución de Problemas Frecuentes (Troubleshooting)

### A. Si los Pods no resuelven DNS o no se comunican entre nodos (Nginx `host not found`)
1. **Regla de comunicación interna del clúster (Flannel VXLAN y CoreDNS):**
   Los Security Groups asignados a tus instancias K3s (**`sg-k3s-public`** y **`sg-k3s-private`**) deben tener en sus **Inbound Rules**:
   - **Type:** `All traffic`
   - **Protocol:** `All`
   - **Port range:** `All`
   - **Source:** `10.0.0.0/16` (o el propio Security Group ID).
   *(Esto permite que Flannel VXLAN en el puerto **UDP 8472** y CoreDNS en el puerto **UDP/TCP 53** funcionen sin bloqueos entre instancias EC2)*.

2. **Uso de FQDN en Kubernetes:**
   En `02-configmap.yaml`, utiliza siempre los nombres canónicos de Kubernetes:
   ```yaml
   VITE_API_URL: "http://backend-service.bodega-app.svc.cluster.local:8085"
   VITE_API_BA_URL: "http://backend-ba-service.bodega-app.svc.cluster.local:8000"
   ```

### B. Si el Backend Analítico da timeout contra MongoDB (`ServerSelectionTimeoutError: Timeout: 8.0s, rtt: None`)
1. **Security Group en MongoDB (`sg-mongodb-ec2`):**
   - Inbound Rule en el puerto `27017` debe permitir **`10.0.0.0/16`** o el Security Group de la subred privada (**`sg-k3s-private`**).
2. **Configuración de bindIp en la EC2 de Mongo:**
   - En `/etc/mongod.conf`, debe estar configurado como `bindIp: 0.0.0.0` y no `127.0.0.1`.
3. **Prueba rápida desde cualquier nodo o pod:**
   ```bash
   kubectl run test-mongo-ba --rm -it --image=busybox -n bodega-app --overrides='{"spec": {"nodeSelector": {"node-role": "backend-ba-worker"}}}' -- nc -zv -w 3 <IP_MONGO> 27017
   ```
4. **Reiniciar despliegues tras actualizar configuración:**
   ```bash
   kubectl rollout restart deployment frontend-deployment -n bodega-app
   kubectl rollout restart deployment backend-ba-deployment -n bodega-app
   ```




Tunnel para backned analitico desde powershell 

ssh -i "C:\Users\user\Downloads\master.pem" -L 8000:localhost:8000 ubuntu@<ip_master> "kubectl port-forward -n bodega-app svc/backend-ba-service 8000:8000"
