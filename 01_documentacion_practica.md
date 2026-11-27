# Practica 1: Recuperacion de una base de datos en MariaDB

## Portada

**Materia:** Administracion de Bases de Datos  
**Practica:** Recuperacion de una base de datos en MariaDB  
**Nombre:** Axel Damian Sanchez Hernandez  
**Profesor:** IGN Diego Alberto Bollas Paredes 
 

---

## Introduccion

Recuperar una base de datos es devolverla a un estado consistente después de un fallo, un borrado accidental u otro incidente. En MariaDB intervienen InnoDB, los respaldos y el binary log (binlog).
La práctica tiene dos partes. En la Parte A se crea una base de datos, se hace un respaldo con mariadb-dump, se simula que se borra y se restaura desde el respaldo. En la Parte B se activa el binlog para recuperar también los cambios hechos después de ese respaldo.
Todo se trabaja desde la terminal de Ubuntu con MariaDB.


---

# Desarrollo

## 1. Preparar la carpeta de trabajo

Primero se creo una carpeta para guardar los archivos relacionados con la practica.

```bash
mkdir practica1
cd practica1
```

### Explicacion

- `mkdir practica1`: crea una carpeta llamada `practica1`.
- `cd practica1`: entra a la carpeta creada.


# PARTE A: Respaldo y restauracion con mariadb-dump

## 2. Crear la base de datos y los datos

Se ingreso al cliente de MariaDB:

```bash
mariadb -u root -p
```

Despues se crearon la base de datos, la tabla y los registros:

```sql
CREATE DATABASE tienda;
USE tienda;

CREATE TABLE clientes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO clientes (nombre, email) VALUES
('Ana Torres', 'ana@correo.com'),
('Luis Perez', 'luis@correo.com'),
('Maria Gomez', 'maria@correo.com'),
('Carlos Ruiz', 'carlos@correo.com'),
('Sofia Diaz', 'sofia@correo.com');

SELECT COUNT(*) FROM clientes;
```

### Resultado esperado

Ingresamos como usuarios root a mariadb y creamos la base de datos,la tabla y los registros, el resultado es 5 ya que confirma que la tabla contiene 5 clientes.

### Evidencia

**Imagen sugerida:** 

## 3. Salir de MariaDB

```sql
EXIT;
```

Al salir se regresa a la terminal de Ubuntu.

---

## 4. Crear el respaldo

Desde la terminal se ejecuto:

```bash
mariadb-dump -u root -p --databases tienda > tienda_backup.sql
```

### Explicacion

- `mariadb-dump`: herramienta para generar un respaldo logico.
- `-u root`: indica el usuario de MariaDB.
- `-p`: solicita la contrasena.
- `--databases tienda`: indica que se respaldara la base `tienda`.
- `>` envia la salida al archivo `tienda_backup.sql`.


### Resultado esperado

Debe aparecer el archivo:

```text
tienda_backup.sql
```

con un tamano mayor a cero.

### Evidencia


## 5. Simular el desastre

Se ingreso nuevamente a MariaDB:

```bash
mariadb -u root -p
```

Despues se elimino intencionalmente la base:

```sql
DROP DATABASE tienda;
SHOW DATABASES;
```

### Resultado esperado

La base de datos `tienda` ya no debe aparecer en la lista.

### Explicacion

`DROP DATABASE tienda;` elimina la base de datos y su contenido. En esta practica se utiliza para simular un error humano o desastre.

### Evidencia



## 6. Restaurar el respaldo

Se salio de MariaDB:

```sql
EXIT;
```

Luego se ejecuto desde la terminal:

```bash
mariadb -u root -p < tienda_backup.sql
```

El operador `<` envia el contenido del archivo SQL al cliente de MariaDB para ejecutar las instrucciones almacenadas en el respaldo.

---

## 7. Verificar la recuperacion

Se ejecuto:

```bash
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

### Resultado esperado

Deben aparecer los cinco registros:

```text
Ana Torres
Luis Perez
Maria Gomez
Carlos Ruiz
Sofia Diaz
```

### Evidencia


# PARTE B: Recuperacion de datos posteriores al respaldo usando binlog

La Parte B demuestra que un respaldo realizado en un momento determinado no contiene automaticamente los cambios que ocurren despues. Para recuperar esos cambios se utiliza el binary log.

## 8. Activar el binlog

En Ubuntu se edito el archivo de configuracion:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Dentro de `[mariadbd]` se agregaron:

```ini
log_bin = /var/lib/mysql/mariadb-bin
server_id = 1
```

Se guardo el archivo y se reinicio MariaDB:

```bash
sudo systemctl restart mariadb
```


### Verificar el binlog

```bash
mariadb -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
```

### Resultado esperado

Debe indicar:

```text
log_bin    ON
```

### Evidencia



## 9. Crear un respaldo con posicion del binlog

Se ejecuto:

```bash
mariadb-dump -u root -p --databases tienda --flush-logs --master-data=2 > tienda_pitr.sql
```

### Explicacion

`--flush-logs` rota el binlog y permite comenzar un nuevo archivo de log.

`--master-data=2` guarda dentro del respaldo la informacion de la posicion del binlog que corresponde al momento del respaldo. Esto permite saber desde que punto se deben aplicar los cambios posteriores.

Para localizar la informacion se utilizo:

```bash
grep "CHANGE MASTER" tienda_pitr.sql
```

### Comando adicional aprendido

`grep` permite buscar texto dentro de archivos.

Por ejemplo:

```bash
grep "CHANGE MASTER" tienda_pitr.sql
```

### Evidencia



## 10. Agregar datos posteriores al respaldo

Se ingreso nuevamente a MariaDB:

```bash
mariadb -u root -p
```

Se agregaron dos clientes:

```sql
USE tienda;

INSERT INTO clientes (nombre, email) VALUES
('Pedro Nunez', 'pedro@correo.com'),
('Laura Vega', 'laura@correo.com');

SELECT COUNT(*) FROM clientes;
```

Ahora existen cinco clientes del respaldo original y dos clientes nuevos.


### Evidencia



## 11. Simular el error

Se espero el tiempo indicado y se ejecuto: Esto nos muestra 0 ya que confirma que se elimino correctamente

```sql
DELETE FROM clientes;

SELECT COUNT(*) FROM clientes;
```



### Evidencia




## 12. Restaurar el respaldo PITR

Desde la terminal:

```bash
mariadb -u root -p < tienda_pitr.sql
```

En este momento deben existir nuevamente los cinco clientes originales, pero Pedro y Laura todavia no deben aparecer porque fueron agregados despues del respaldo.

Se puede comprobar:

```bash
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

---

## 13. Aplicar el binlog hasta antes del error

Se utilizo el archivo y la posicion obtenidos anteriormente.

Ejemplo:

```bash
sudo mariadb-binlog --start-position=342 --stop-datetime="2026-11-26 22:40:15" \
/var/lib/mariadb/mariadb-bin.000002 | mariadb -u root -p
```



### Explicacion

- `mariadb-binlog`: permite leer y procesar los eventos registrados en el binary log.
- `--start-position=330`: indica desde que posicion comenzar.
- `--stop-datetime="..."`: indica hasta que fecha y hora aplicar los cambios.
- `|`: envia la salida de un comando al siguiente.
- `mariadb -u root -p`: recibe los eventos y los ejecuta en MariaDB.

### Comandos adicionales aprendidos

Para consultar los archivos binlog:

```sql
SHOW BINARY LOGS;
```

Para revisar eventos de un archivo:

```sql
SHOW BINLOG EVENTS IN 'mariadb-bin.000002';
```

Desde la terminal tambien se puede consultar:

```bash
sudo mariadb-binlog /var/lib/mysql/mariadb-bin.000002 | less
```

`less` permite revisar la salida por paginas sin mostrar todo de golpe.

### Evidencia

## 14. Verificar la recuperacion final

Se ejecuto:

```bash
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

### Resultado esperado

Deben aparecer siete clientes:

1. Ana Torres
2. Luis Perez
3. Maria Gomez
4. Carlos Ruiz
5. Sofia Diaz
6. Pedro Nunez
7. Laura Vega

Esto demuestra que se recuperaron los datos del respaldo y tambien los cambios posteriores al respaldo, pero no el `DELETE` accidental.

### Evidencia


# Comandos adicionales aprendidos

Durante la practica se utilizaron comandos que pueden considerarse nuevos o que ayudan a comprender mejor el proceso:

| `pwd` | Muestra la ruta actual |
| `ls` | Muestra archivos y carpetas |
| `ls -lh` | Muestra archivos con tamanos legibles |
| `head archivo` | Muestra las primeras lineas de un archivo |
| `grep "texto" archivo` | Busca texto dentro de un archivo |
| `sudo systemctl status mariadb` | Consulta el estado del servicio MariaDB |
| `sudo systemctl restart mariadb` | Reinicia el servicio MariaDB |
| `mariadb -e "SQL"` | Ejecuta SQL directamente desde la terminal |
| `SHOW VARIABLES LIKE 'log_bin';` | Comprueba una variable de MariaDB |
| `SHOW BINARY LOGS;` | Muestra los archivos binlog disponibles |
| `SHOW BINLOG EVENTS IN 'archivo';` | Muestra eventos de un binlog |
| `mariadb-binlog` | Lee y procesa eventos del binlog |
| `less` | Permite leer una salida extensa por paginas |
| `SELECT NOW();` | Obtiene fecha y hora actuales |
| `\|` | Envia la salida de un comando a otro |
| `<` | Usa un archivo como entrada de un comando |

---

# Resultados obtenidos

En la Parte A se comprobo que un respaldo creado con `mariadb-dump` permite restaurar una base de datos eliminada accidentalmente. La base `tienda` fue eliminada y posteriormente reconstruida desde `tienda_backup.sql`.

En la Parte B se comprobo que el respaldo por si solo no contiene los cambios realizados despues de su creacion. Al utilizar el binlog fue posible reaplicar los cambios posteriores al respaldo hasta antes del `DELETE` accidental.

El resultado final esperado fue de siete clientes.

---

# Conclusiones

La practica permitio comprobar la importancia de contar con respaldos y mecanismos adicionales de recuperacion. Un respaldo permite recuperar la informacion existente hasta el momento en que fue realizado, pero no necesariamente los cambios posteriores.

El binlog complementa el respaldo porque registra los cambios realizados despues del respaldo. Gracias a la posicion inicial y a una fecha y hora de corte, es posible aplicar los eventos necesarios y recuperar los datos hasta antes de un error.

Tambien se conocieron herramientas de terminal como `grep`, `ls -lh`, `systemctl`, `mariadb-binlog`, `less` y el uso de tuberias y redireccionamiento. Estos comandos facilitan la administracion y recuperacion de bases de datos desde Ubuntu.

