# Cuestionario - Practica 1: Recuperacion de una base de datos en MariaDB

## 1. ¿Que datos se habrian perdido si solo hubieras usado el respaldo de la Parte A?

Se habrian perdido los datos agregados despues de realizar el respaldo. En esta practica, Pedro Nunez y Laura Vega fueron agregados despues de crear el respaldo, por lo que no estaban incluidos en `tienda_backup.sql`.

El respaldo de la Parte A contenia solamente los cinco clientes que existian en el momento en que fue realizado.

## 2. ¿Para que sirve la opcion `--master-data=2`?

La opcion `--master-data=2` guarda dentro del respaldo la informacion de la posicion del binary log correspondiente al momento del respaldo.

Esta informacion permite conocer desde que archivo y posicion del binlog se pueden aplicar los cambios posteriores al respaldo para realizar una recuperacion a un punto en el tiempo.

## 3. ¿Por que es necesario activar el binlog antes de que ocurra el error?

Porque el binlog registra los cambios que se realizan en MariaDB. Si no estaba activado antes del incidente, los cambios que ocurrieron despues del respaldo no estarian registrados en el binlog y no se podrian reaplicar mediante este metodo.

Por eso el binlog debe estar funcionando antes de que ocurra el error que se desea recuperar.

## 4. ¿Por que se esperaron 5 segundos antes y despues de anotar la hora?

La espera permite crear una separacion temporal entre las operaciones realizadas durante la practica. Esto ayuda a identificar con mayor facilidad el momento en que se agregaron los datos y el momento posterior en que ocurrio el `DELETE`.

De esta forma se puede utilizar una fecha y hora de corte para aplicar el binlog hasta antes del error y evitar reaplicar el `DELETE` accidental.

