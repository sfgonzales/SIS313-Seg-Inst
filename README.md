# 🌐 Diseño e Implementación de una Infraestructura de TI Web Escalable

Este documento describe paso a paso la creación y configuración de una infraestructura compuesta por 4 instancias virtuales usando VirtualBox, con hardening básico y servicios instalados.

---

## 🖥️ **Infraestructura**

| VM                   | Rol                     | IP                   | Servicios |
|-----------------------|------------------------|----------------------|-----------|
| Balanceador           | Nginx                   | 192.168.100.248      | Nginx (puerto 80) |
| App Server 1          | Node.js app             | 192.168.100.249     | Node.js (puerto 3000) |
| App Server 2          | Node.js app             | 192.168.100.250      | Node.js (puerto 3000) |
| Base de Datos         | MariaDB + RAID 5        | 192.168.100.251      | MariaDB (puerto 3306) |

---

## ⚙️ **Pasos de Configuración**

### 1️⃣ Crear máquinas virtuales en VirtualBox

  - Balanceador y 2 servidores de aplicaciones: 10 GB
  - Base de datos: 10 GB + 3 discos adicionales de 1 GB para RAID 5
---
### 2️⃣ Instalar Nginx en el balanceador
- Instalar NGINX

```
sudo apt update
sudo apt install nginx -y
```
- Configurar Nginx como balanceador
Editar archivo:

```
sudo nano /etc/nginx/sites-available/default
```
Reemplazar todo el contenido por:

```
upstream app_servers {
    server 192.168.100.249:3000;
    server 192.168.100.250:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
- Verificar y reiniciar Nginx

```
sudo nginx -t
sudo systemctl restart nginx
```

### 3️⃣ Configuración de Servidores Node.js (192.168.100.249 y 192.168.100.250)
- Instalar Node.js y dependencias

```
sudo apt update
sudo apt install nodejs npm -y
```
- Crear la app 

```
mkdir myapp
cd myapp
npm init -y
npm install express mysql2 body-parser
```
- Crear archivo index.js con CRUD

```
const express = require('express');
const bodyParser = require('body-parser');
const mysql = require('mysql2');

const app = express();
app.use(bodyParser.json());

const db = mysql.createConnection({
  host: '192.168.100.251',
  user: 'root',
  password: 'sas123',
  database: 'crud_db'
});

db.connect(err => {
  if (err) {
    console.error('Error al conectar a la base de datos:', err);
    process.exit(1);
  }
  console.log('Conectado a la base de datos');
});

app.get('/items', (req, res) => {
  db.query('SELECT * FROM items', (err, results) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(results);
  });
});

app.post('/items', (req, res) => {
  const { name } = req.body;
  db.query('INSERT INTO items (name) VALUES (?)', [name], (err, result) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ id: result.insertId, name });
  });
});

app.put('/items/:id', (req, res) => {
  const { id } = req.params;
  const { name } = req.body;
  db.query('UPDATE items SET name = ? WHERE id = ?', [name, id], (err, result) => {
    if (err) return res.status(500).json({ error: err.message });
    if (result.affectedRows === 0) return res.status(404).json({ error: 'Item no encontrado' });
    res.json({ id, name });
  });
});

app.delete('/items/:id', (req, res) => {
  const { id } = req.params;
  db.query('DELETE FROM items WHERE id = ?', [id], (err, result) => {
    if (err) return res.status(500).json({ error: err.message });
    if (result.affectedRows === 0) return res.status(404).json({ error: 'Item no encontrado' });
    res.json({ message: `Item con id ${id} eliminado` });
  });
});

app.listen(3000,'0.0.0.0', () => {
  console.log('Aplicación escuchando en puerto 3000');
});
```
- Ejecutar la app

``
node index.js
``
### 4️⃣ Configuración del Servidor de Base de Datos + RAID5 (192.168.100.251)
#### 4.1 Creación de la Base de Datos

- Instalar MariaDB y mdadm

```
sudo apt update
sudo apt install mariadb-server mdadm -y
```

- Crear base de datos y usuario

```
sudo mysql -u root
```
Dentro de MariaDB:

```
CREATE DATABASE crud_db;
USE crud_db;
CREATE TABLE items (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255));
CREATE USER 'root'@'%' IDENTIFIED BY 'sas123';
GRANT ALL PRIVILEGES ON crud_db.* TO 'root'@'%';
FLUSH PRIVILEGES;
EXIT;
```
- Editar 50-server.cnf:

```
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Cambiar:

```
bind-address = 0.0.0.0
```
- Reiniciar:

```
sudo systemctl restart mariadb
```

#### 4.2 Implementación de RAID

- Verificar los discos

```
lsblk
```
- Crear el RAID 5

```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
```
- Verificar:

```
cat /proc/mdstat
```
- Crear el sistema de archivos:

```
sudo mkfs.ext4 /dev/md0
sudo mkdir /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

- Hacerlo persistente:

```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
echo '/dev/md0 /mnt/raid ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

>No olvidar crear un dico adicional para la restauración 

- Simular una falla

```
sudo mdadm --fail /dev/md0 /dev/sdb
sudo mdadm --remove /dev/md0 /dev/sdb
```

- Verificar

```
cat /proc/mdstat
```

- Reemplazar disco (por ejemplo añadir un disco nuevo /dev/sde):
 
```
sudo mdadm --add /dev/md0 /dev/sde
```

- Verificar la reconstrucción RAID

```
sudo mdadm --add /dev/md0 /dev/sde
```

### 5️⃣ Hardening Básico
#### 5.1 Implementacion de SSH
- Instalación del Servidor SSH (si no está instalado):

```
sudo apt update
sudo apt install openssh-server
```
- Verificación del Estado del Servicio SSH:
Una vez instalado (o si ya estaba), verifica que el servicio SSH esté en ejecución:

```
sudo systemctl status ssh
```
La salida debería indicar que el servicio está active (running). Si no lo está, iniciarlo con:
```
sudo systemctl start ssh
```
Y para que se inicie automáticamente en cada arranque:
```
sudo systemctl enable ssh
```
- Configuración del Servidor SSH (Archivo sshd_config):
El archivo de configuración principal del servidor SSH es /etc/ssh/sshd_config. Ábrelo con un editor de texto con privilegios de superusuario:

```
sudo nano /etc/ssh/sshd_config
```
>Primeramente se recomienda cambiar el puerto por defecto 22 (`Port 22`) a otro valor, ayudando a reducir ataques automatizados, ya que muchos bots buscan específicamente ese puerto. Además, desactivar el acceso directo como root (`PermitRootLogin no`) evita que un atacante pueda obtener control total del sistema desde el primer intento, obligando al uso de cuentas con menos privilegios y luego escalar mediante sudo, lo que añade una capa de seguridad y trazabilidad.

#### 5.2 Implementacion de UFW
- Configuración del Firewall (UFW - Uncomplicated Firewall):
Es importante configurar el firewall para permitir las conexiones entrantes al puerto SSH. Si UFW está habilitado en el sistema, se puede permitir el tráfico SSH con el siguiente comando:

```
sudo ufw allow ssh
```
Si se cambio el puerto SSH en el archivo `/etc/ssh/sshd_config`, se debe especificar el nuevo puerto en la regla del firewall. Por ejemplo, si el nuevo puerto es 2222, se usaría el comando:

```
sudo ufw allow 2222/tcp
```
Entonces para cada servidor configuramos el UFW de la siguiente forma:
- Balanceador:

```
sudo ufw allow 80/tcp
sudo ufw allow 2222/tcp
sudo ufw enable
```
>Permitimos el tráfico web (puerto 80) para recibir las peticiones HTTP y el puerto 2222 para el acceso SSH

- Servidores de Aplicaciones:

```
sudo ufw allow 2222/tcp
sudo ufw allow from 192.168.100.248 to any port 3000
sudo ufw enable
```
>Permitimos el puerto 2222 para SSH y el acceso al puerto 3000 solo desde el balanceador (192.168.100.248), para mayor seguridad.

- Servidor de la Base de Datos:

```
sudo ufw allow 2222/tcp
sudo ufw allow from 192.168.100.249 to any port 3306
sudo ufw allow from 192.168.100.250 to any port 3306
sudo ufw enable
```
>Permitimos el puerto 2222 para SSH y el acceso al puerto 3306 solo desde los servidores de aplicación (192.168.100.249 y 192.168.100.250), restringiendo el acceso externo.

Para verificar el estado del firewall y asegurarse de que la regla para SSH (o el puerto personalizado) esté activa, usar el comando:

```
sudo ufw status
```
