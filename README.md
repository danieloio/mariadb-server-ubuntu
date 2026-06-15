# 03 — Servidor de Base de Datos con MariaDB y phpMyAdmin

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-orange?logo=ubuntu)
![MariaDB](https://img.shields.io/badge/MariaDB-10.11-blue?logo=mariadb)
![Apache](https://img.shields.io/badge/Apache-2.4-red?logo=apache)
![phpMyAdmin](https://img.shields.io/badge/phpMyAdmin-5.2-6C78AF?logo=phpmyadmin)
![Status](https://img.shields.io/badge/Estado-Completado-brightgreen)

Despliegue de un servidor de base de datos empresarial sobre Ubuntu Server 22.04 en VirtualBox. El proyecto incluye la instalación y configuración de MariaDB, Apache y phpMyAdmin, el diseño de una base de datos relacional para la empresa multisede de los proyectos anteriores, consultas SQL con JOINs y un sistema de backup automatizado con cron.

---

## Índice

- [Infraestructura](#infraestructura)
- [Servicios instalados](#servicios-instalados)
- [Base de datos](#base-de-datos)
- [Configuración](#configuración)
- [Consultas SQL](#consultas-sql)
- [Backup automatizado](#backup-automatizado)
- [Problemas encontrados](#problemas-encontrados)

---

## Infraestructura

| Parámetro | Valor |
|---|---|
| Hipervisor | Oracle VirtualBox |
| Sistema operativo | Ubuntu Server 22.04 LTS |
| RAM | 2048 MB |
| Disco | 20 GB VDI |
| Adaptador 1 | NAT (acceso a internet) |
| Adaptador 2 | Host-Only — 192.168.56.10/24 (acceso SSH y phpMyAdmin) |
| Acceso remoto | SSH desde Windows Terminal |

![IP interfaces](screenshots/01-ip-a.png)

---

## Servicios instalados

| Servicio | Versión | Puerto |
|---|---|---|
| MariaDB | 10.11.14 | 3306 |
| Apache | 2.4.58 | 80 |
| PHP | 8.3.6 | — |
| phpMyAdmin | 5.2.1 | 80 |

### MariaDB activo

![MariaDB status](screenshots/02-status-mariadb.png)

### Apache activo

![Apache status](screenshots/03-status-apache2.png)

### phpMyAdmin — Login

![phpMyAdmin login](screenshots/04-phpmyadmin-login.png)

### phpMyAdmin — Panel principal

![phpMyAdmin panel](screenshots/05-phpmyadmin-panel.png)

---

## Base de datos

La base de datos `empresa_multisede` modela la infraestructura de la empresa con 3 sedes (Madrid, Barcelona y Valencia) trabajada en los proyectos anteriores de redes.

### Diagrama de tablas

```
sedes                    departamentos
─────────────            ─────────────────
id_sede (PK)             id_departamento (PK)
nombre                   nombre
ciudad                   descripcion
direccion
telefono
    │                          │
    └──────────┬───────────────┘
               │
          empleados
          ──────────────────
          id_empleado (PK)
          nombre
          apellidos
          email
          telefono
          cargo
          salario
          fecha_alta
          id_sede (FK)
          id_departamento (FK)
               │
          equipos
          ──────────────────
          id_equipo (PK)
          tipo
          marca
          modelo
          numero_serie
          estado (activo/baja/reparacion)
          id_sede (FK)
          id_empleado (FK)
```

### Estructura en phpMyAdmin

![DB structure](screenshots/06-db-structure.png)

### Tabla empleados

![DB empleados](screenshots/07-db-empleados.png)

---

## Configuración

### 1. Red — interfaz Host-Only estática

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

```bash
sudo netplan apply
```

### 2. Instalación de servicios

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y mariadb-server
sudo apt install -y apache2
sudo apt install -y phpmyadmin
sudo apt install -y libapache2-mod-php8.3
sudo a2enmod php8.3
sudo systemctl restart apache2
```

### 3. Seguridad de MariaDB

```bash
sudo mysql_secure_installation
```

```
Switch to unix_socket authentication: n
Change the root password:             y
Remove anonymous users:               y
Disallow root login remotely:         y
Remove test database:                 y
Reload privilege tables:              y
```

### 4. Usuario administrador

![Create dbadmin](screenshots/08-create-dbadmin.png)

```sql
CREATE USER 'dbadmin'@'localhost' IDENTIFIED BY 'TuPassword';
GRANT ALL PRIVILEGES ON *.* TO 'dbadmin'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### 5. Creación de la base de datos

```sql
CREATE DATABASE empresa_multisede CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE empresa_multisede;

CREATE TABLE sedes (
    id_sede INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL,
    ciudad VARCHAR(50) NOT NULL,
    direccion VARCHAR(100),
    telefono VARCHAR(20)
);

CREATE TABLE departamentos (
    id_departamento INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL,
    descripcion VARCHAR(100)
);

CREATE TABLE empleados (
    id_empleado INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL,
    apellidos VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    telefono VARCHAR(20),
    cargo VARCHAR(50),
    salario DECIMAL(8,2),
    fecha_alta DATE,
    id_sede INT,
    id_departamento INT,
    FOREIGN KEY (id_sede) REFERENCES sedes(id_sede),
    FOREIGN KEY (id_departamento) REFERENCES departamentos(id_departamento)
);

CREATE TABLE equipos (
    id_equipo INT AUTO_INCREMENT PRIMARY KEY,
    tipo VARCHAR(50) NOT NULL,
    marca VARCHAR(50),
    modelo VARCHAR(50),
    numero_serie VARCHAR(100) UNIQUE,
    estado ENUM('activo','baja','reparacion') DEFAULT 'activo',
    id_sede INT,
    id_empleado INT,
    FOREIGN KEY (id_sede) REFERENCES sedes(id_sede),
    FOREIGN KEY (id_empleado) REFERENCES empleados(id_empleado)
);
```

---

## Consultas SQL

### Consulta 1 — Empleados con sede y departamento (JOIN múltiple)

```sql
SELECT e.nombre, e.apellidos, e.cargo, s.ciudad, d.nombre AS departamento
FROM empleados e
JOIN sedes s ON e.id_sede = s.id_sede
JOIN departamentos d ON e.id_departamento = d.id_departamento;
```

![Consulta 1](screenshots/09-query-empleados-sede-dept.png)

---

### Consulta 2 — Salario medio por sede (GROUP BY + AVG)

```sql
SELECT s.ciudad, COUNT(e.id_empleado) AS total_empleados,
ROUND(AVG(e.salario), 2) AS salario_medio
FROM sedes s
LEFT JOIN empleados e ON s.id_sede = e.id_sede
GROUP BY s.ciudad;
```

![Consulta 2](screenshots/10-query-salario-medio.png)

---

### Consulta 3 — Equipos asignados por empleado (LEFT JOIN)

```sql
SELECT e.nombre, e.apellidos, eq.tipo, eq.marca, eq.modelo, eq.estado
FROM empleados e
LEFT JOIN equipos eq ON e.id_empleado = eq.id_empleado
ORDER BY e.nombre;
```

![Consulta 3](screenshots/11-query-equipos-empleados.png)

---

## Backup automatizado

### Script de backup comprimido

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
mysqldump -u dbadmin -pTuPassword empresa_multisede | gzip > ~/backups/backup_$DATE.sql.gz
echo "Backup completado: backup_$DATE.sql.gz"
```

### Tarea cron — ejecución diaria a las 2:00 AM

```bash
0 2 * * * /home/daniel/backup_db.sh
```

### Restaurar un backup

```bash
gunzip < ~/backups/backup_FECHA.sql.gz | mysql -u dbadmin -p empresa_multisede
```

---

## Problemas encontrados

**Problema:** La interfaz Host-Only (enp0s8) no levantaba al arrancar.
**Causa:** No estaba configurada en netplan.
**Solución:** Añadir enp0s8 con IP estática 192.168.56.10/24 en `/etc/netplan/50-cloud-init.yaml` y ejecutar `sudo netplan apply`.

---

**Problema:** phpMyAdmin mostraba el código PHP en lugar de la interfaz web.
**Causa:** El módulo PHP 8.3 no estaba habilitado en Apache.
**Solución:** Instalar `libapache2-mod-php8.3`, habilitar con `a2enmod php8.3` y añadir `AddType application/x-httpd-php .php` en la configuración de phpMyAdmin.

---

*Laboratorio realizado con Ubuntu Server 22.04 LTS — Daniel Moisés Loyo Vásquez*
