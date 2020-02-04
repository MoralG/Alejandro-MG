---
title: "Interconexión entre Bases de Datos"
categories:
- Base de Datos
excerpt: |
  Las interconexiones de servidores de bases de datos trata de acceder a datos que no están almacenados en nuestra base de datos, pudiendo combinarlos con los que ya tenemos.
aside: true
---

![iSCSI](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/Inter.jpg?raw=true)

**Las interconexiones de servidores de bases de datos trata de acceder a datos que no están almacenados en nuestra base de datos, pudiendo combinarlos con los que ya tenemos.**

#### Las interconexiones de servidores de bases de datos son operaciones que pueden ser muy útiles en diferentes contextos. Básicamente, se trata de acceder a datos que no están almacenados en nuestra base de datos, pudiendo combinarlos con los que ya tenemos.

#### En esta práctica veremos varias formas de crear un enlace entre distintos servidores de bases de datos.

#### Los servidores enlazados siempre tendrán que estar instalados en máquinas diferentes.

## 1. Enlace ORACLE - ORACLE

#### Realizar un enlace entre dos servidores de bases de datos ORACLE, explicando la configuración necesaria en ambos extremos y demostrando su funcionamiento.

###### Para enlazar un Cliente Oracle con dirección `172.22.44.76/16` a un Servidor Oracle con dirección `172.22.7.59/16`, tenemos que realizar unas configuraciones en cada una de las partes para que realice el enlace. Estas configuraciones se explicarán a continuación:

##### Configuración Servidor Oracle

###### En el Servidor 2 tendremos que crear un usuario con los privilegios que deseemos, ya que accedemos a través de dicho usuario, desde el cliente, a la base de datos. En este caso utilizaremos el usuario `paloma` que tiene privilegios para ver sus tablas creadas.

###### Configuramos el fichero `/opt/oracle/product/12.2.0.1/dbhome_1/network/admin/listener.ora` para escuchar en una determinada dirección y un determinado puerto.

~~~
# listener.ora Network Configuration File: /opt/oracle/product/12.2.0.1/dbhome_1/network/admin/listener.$
# Generated by Oracle configuration tools.

SID_LIST_LISTENER =
 (SID_LIST =
  (SID_DESC =
   (GLOBAL_DBNAME = orcl)
   (ORACLE_HOME = /opt/oracle/product/12.2.0.1/dbhome_1)
   (SID_NAME = orcl)
  )
 )

LISTENER=
 (DESCRIPTION_LIST =
  (DESCRIPTION =
   (ADDRESS_LIST =
    (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
   )
   (ADDRESS_LIST =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 172.22.44.76)(PORT = 1521))
   )
  )
 )
~~~

> `LISTENER`: Donde especificamos las direcciones y puertos desde donde se pueden conectar remotamente a nuestro servidor.
>
> `SID_LIST_LISTENER`: Donde se indican los nombres de los servicios.

##### Configuración Cliente Oracle

###### Vamos a modificar el fichero `/opt/oracle/product/12.2.0.1/dbhome_1/network/admin/tnsnames.ora`.

> * Si no esta en la dirección indicada, tendremos que crearlo nosostros.

~~~
sudo nano /opt/oracle/product/12.2.0.1/dbhome_1/network/admin/tnsnames.ora
~~~

###### Añadimos las siguientes lineas en dicho fichero con las caracteristicas que hacen referencia al Servidor Oracle al que nos vamos a enlazar y añadimos la información refente a nuestro servidor si no teniamos el fichero creado.

~~~
LISTENER_ORCL =
 (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))

ORCL =
 (DESCRIPTION = Mi Servidor Oracle
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = orcl)
    )
 )

OraclePaloma =
 (DESCRIPTION = Servidor Oracle de Paloma
    (ADDRESS = (PROTOCOL = TCP)(HOST = 172.22.7.59)(PORT = 1521))
    (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = orcl)
    )
 )
~~~

> `ADDRESS`: Especificamos el protocolo, la dirección y el puerto de la máquina que queremos conectarnos.
>
> `SERVER = DEDICATED`: Para crear un proceso del servidor para atender solo a la conexión indicada
>
> `SERVICE_NAME`: Especificamos el nombre del sevicio de la máquina que queremos conectarnos. Este se especifica en el parámetro `CONNECT_DATA`.

###### Ahora tenemos que reiniciar el servicio para que se apliquen los cambios:

~~~
lsnrctl stop
lsnrctl start
~~~

###### Nos devuelve el siguiente mensaje al reiniciar el listener:

~~~
El servicio "orcl" tiene 1 instancia(s).
  La instancia "orcl", con estado UNKNOWN, tiene 1 manejador(es) para este servicio...
~~~

###### Asignamos privilegios al usuario `moralg` de nuestra base de datos, para que pueda crear el enlace.

```sql
GRANT CREATE DATABASE LINK to moralg;
```

###### Ya solo queda crear un enlace en nuestro Servidor Oracle para que pueda enlazarse al Servidor Oracle 2. Para crear el link se realiza de la siguiente forma:

``` sql
CREATE DATABASE LINK ConexionPaloma
CONNECT TO paloma
IDENTIFIED BY paloma
USING 'orcl';
```

> * Indicamos el nombre del link `ConexionPaloma` para que se conecte al usuario `paloma`, con la contraseña `paloma` usando el nombre del servicio `orcl`

###### Comprobamos el enlace, realizando un `SELECT` a una de la tablas de paloma y combinandola con una de moralg:

![PruebaOra_Ora](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/PruebaOra_Ora.png?raw=true)

> * La consulta nos muestra el campo `DESCRPCION` de la tabla `ASPECTOS` al hacer la conexión por el enlace `ConexionPaloma` combinando con el campo `NOMBREEQUIPOS` de la tabla `PALMARES`.

## 2. Enlace POSTGRES - POSTGRES

#### Realizar un enlace entre dos servidores de bases de datos Postgres, explicando la configuración necesaria en ambos extremos y demostrando su funcionamiento.

###### Vamos a realizar un enlace entre un Servidor de Postgres con la dirección `192.168.43.141/24` y un Cliente de Postgres con la dirección `192.168.43.66/24`

##### Configuración del Servidor Postgres

###### Modificamos el fichero `/etc/postgresql/11/main/postgresql.conf`, descomentando una línea y añadiendo la dirección del cliente que quiere crear el enlace para que pueda escucharlo.

~~~
listen_addresses = '192.168.43.66, localhost'
~~~

###### Reiniciamos el servicio del servidor:

~~~
sudo systemctl restart postgresql.service
~~~

##### Configuración del Cliente Postgres

###### Vamos a intalar el paquete `postgresql-contrib` para poder utilizar los módulos con `CREATE EXTENSION`

~~~
sudo apt install postgresql-contrib
~~~

###### Ahora vamos a añadir un nuevo registro de autentificación en el fichero `/etc/postgresql/11/main/pg_hba.conf` indicandole la dirección del servidor y el tipo de autentificación `md5` para todas las base de datos y usuarios.

###### La linea que hay que añadir es:

~~~
# TYPE  DATABASE        USER            ADDRESS                 METHOD

host    all             all             192.168.43.66/24        md5
~~~

###### Reiniciamos el servicio del cliente:

~~~
sudo systemctl restart postgresql.service
~~~

###### Para crear el enlace vamos a utilizar el módulo `dblink`

``` sql
CREATE EXTENSION dblink;
    CREATE EXTENSION
```

> * Solo los usuarios con superusuarios pueden crear extensiones. Este privilegio se asigna de con `ALTER ROLE <name_role> WITH superuser;`

###### Ahora podemos realizar una consulta a una base de datos del Servidor de Postgres con lo siguiente:

![PruebaPost_Post](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/PruebaPost_Post.png?raw=true)

> `dblink`: Tenemos que indicarle, para realizar el enlace, varios parámetros:
>
> `dbname`: Nombre de la base de datos del servidor.
>
> `host`: Direción del servidor.
>
> `user`: Usuario con el cual nos queremos conectar al servidor.
>
> `select .....`: Indicar una consulta.

> * Además tenemos que indicar el tipo da datos de las columnas, ya que es obligatorio.

###### Podemos utilizar `dblink_connect` y le indicamos los parámetros del servidor para realizar una conexión persistente con el servidor y no tener que indicar más, durante la sesión, dichos parámetros.

``` sql
SELECT dblink_connect('ConexionPaloma', 'dbname=restaurante host=172.22.3.28 user=paloma password=paloma');
```

> `ConexionPaloma`: Nombre que se le asigna a la conexión persistente.

###### Ahora realizamos una consulta con lo indicado anteriormente:

![PruebaPost_Post](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/PruebaPost_Post1.png?raw=true)

## 3. Enlace ORACLE - POSTGRES

#### Realizar un enlace entre un cliente ORACLE y un servidor Postgres, empleando Heterogeneus Services, explicando la configuración necesaria en ambos extremos y demostrando su funcionamiento.

### 3.1 Enlace de Cliente Oracle a Servidor Postgres
-----------------------------------------------------------------

#### Tenemos que configurar un enlace utilizando el driver `ODBC` (Open DataBase Connector) que es un estandar para el acceso a los datos de cualquier gestor de base de datos.

##### Configuración el Servidor Postgres

###### Modificamos el fichero `/etc/postgresql/11/main/postgresql.conf`, descomentando una línea y añadiendo la dirección del cliente que quiere crear el enlace para que pueda escucharlo.

~~~
listen_addresses = '192.168.43.66, localhost'
~~~

###### Ahora vamos a añadir un nuevo registro de autentificación en el fichero `/etc/postgresql/11/main/pg_hba.conf` indicandole la dirección del cliente y el tipo de autentificación `md5` para todas las base de datos y usuarios.

###### La linea que hay que añadir es:

~~~
# TYPE  DATABASE        USER            ADDRESS                 METHOD

host    all             all             192.168.43.66/24        md5
~~~

###### Reiniciamos el servicio del servidor:

~~~
sudo systemctl restart postgresql.service
~~~

##### Configuración del Cliente Oracle

###### Instalamos el driver `ODBC`:

~~~
sudo apt install odbc-postgresql unixodbc
~~~

###### Se nos creará por defecto, al instalar los paquetes anteriores, el fichero `/etc/odbcinst.ini` donde se indicará los drivers de ODBC que vamos a utilizar.

~~~
[PostgreSQL ANSI]
Description=PostgreSQL ODBC driver (ANSI version)
Driver=psqlodbca.so
Setup=libodbcpsqlS.so
Debug=0
CommLog=1
UsageCount=1

[PostgreSQL Unicode]
Description=PostgreSQL ODBC driver (Unicode version)
Driver=psqlodbcw.so
Setup=libodbcpsqlS.so
Debug=0
CommLog=1
UsageCount=1
~~~

###### Ahora tenemos que añadir al fichero `/etc/odbc.ini` los parametros de conexión del Servidor de Postgres.

~~~
[PSQLA]
Debug = 0
CommLog = 0
ReadOnly = 1
Driver = PostgreSQL ANSI
Servername = 192.168.43.66
Username = paloma
Password = paloma
Port = 5432
Database = restaurante
Trace = 0
TraceFile = /tmp/sql.log

[PSQLU]
Debug = 0
CommLog = 0
ReadOnly = 0
Driver = PostgreSQL Unicode
Servername = 192.168.43.66
Username = paloma
Password = paloma
Port = 5432
Database = restaurante
Trace = 0
TraceFile = /tmp/sql.log

[Default]
Driver = /usr/lib/x86_64-linux-gnu/odbc/liboplodbcS.so
~~~

> `Driver`: Indicamos el driver, `PostgreSQL ANSI` y `PostgreSQL Unicode`, que hemos añadido al fichero `/etc/odbcinst.ini`.
>
> `Servername`: Dirección del Servidor de Postgres.
>
> `Username`: Nombre del usuario que vamos a utilixar para realizar la conexión al servidor.
>
> `Password`: Contraseña del usuario `paloma`.
>
> `Port`: Puerto del Servidor de Postgres.
>
> `Database`: Base de datos a la que vamos a conectarnos.

###### Podemos comprobar la configuración de los drivers y la configuración de la conexión de dichos drivers:

~~~
odbcinst -q -d
    [PostgreSQL ANSI]
    [PostgreSQL Unicode]


odbcinst -q -s
    [PSQLA]
    [PSQLU]
    [Default]
~~~

###### Comprobamos una conexión al Servidor de Postgres:

~~~
root@servidororacle:/home/oracle# isql -v PSQLU
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
SQL> select * from aspectos;
+-------+---------------------------------------------------+------------+
| codigo| descripcion                                       | importancia|
+-------+---------------------------------------------------+------------+
| COL   | Color                                             | Baja       |
| TEX   | Textura                                           | Alta       |
| VOL   | Volumen                                           | Media      |
| CAN   | Cantidad                                          | Alta       |
| PRE   | Presentacion                                      | Alta       |
| TEC   | Tecnica                                           | Media      |
| ORI   | Originalidad                                      | media      |
+-------+---------------------------------------------------+------------+
SQLRowCount returns 7
7 rows fetched
~~~

###### Ahora vamos a configurar el servicio de *Heterogeneus Services*, para esto vamos a crear el fichero `/opt/oracle/product/12.2.0.1/dbhome_1/hs/admin/initPSQLU.ora` y añadimos lo siguiente:

~~~
HS_FDS_CONNECT_INFO = PSQLU
HS_FDS_TRACE_LEVEL = Debug
HS_FDS_SHAREABLE_NAME = /usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so
HS_LANGUAGE = AMERICAN_AMERICA.WE8ISO8859P1
set ODBCINI=/etc/odbc.ini
~~~

> `HS_FDS_CONNECT_INFO`: Para indicar un nombre al servicio.
>
> `HS_FDS_TRACE_LEVEL`: Para activar el servicio.
>
> `HS_FDS_SHAREABLE_NAME`: Donde especificamos la dirección del driver configurado anteriormente
>
> `HS_LANGUAGE`: Indicamos el idioma por defecto
>
> `set ODBCINI`: Donde especificamos el fichero de configuración del driver para PSQL.

###### Configuramos el fichero `/opt/oracle/product/12.2.0.1/dbhome_1/network/admin/listener.ora` para que pueda escuchar en el driver de `ODBC`, añadiendo otra entrada al apartado de `SID_DESC`:

~~~
SID_LIST_LISTENER =
 (SID_LIST =
  (SID_DESC =
   (GLOBAL_DBNAME = orcl)
   (ORACLE_HOME = /opt/oracle/product/12.2.0.1/dbhome_1)
   (SID_NAME = orcl)
  )
  (SID_DESC =
    (SID_NAME = PSQLU)
    (PROGRAM = dg4odbc)
    (ORACLE_HOME = /opt/oracle/product/12.2.0.1/dbhome_1)
  )
 )
~~~

> `SID_NAME`: Le especificamos el nombre del servicio de 'Heterogeneus Services'.
>
> `PROGRAM`: Le indicamos el programa por defecto.

###### Reiniciamos el servicio:

~~~
lsnrctl stop
lsnrctl start
~~~

###### Configuramos el fichero `/opt/oracle/product/12.2.0.1/dbhome_1/network/admin/tnsnames.ora` creando un entrada para realizar la conexión al driver `ODBC`. Añadimos al fichero lo siguiente:

~~~
PSQLU =
 (DESCRIPTION=
 (ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521))
   (CONNECT_DATA=(SID=PSQLU))
   (HS=OK)
 )
~~~

> Le indicamos que `SID=PSQLU` ya que es el nombre que le hemos asignado anteriormente y `HS=OK` para que utilice el servicio *Heterogeneus Services*.

###### Nos devuelve el siguiente mensaje al reiniciar el listener:

~~~
El servicio "PSQLU" tiene 1 instancia(s).
  La instancia "PSQLU", con estado UNKNOWN, tiene 1 manejador(es) para este servicio...
~~~

###### Asignamos privilegios al usuario `moralg` de nuestra base de datos, para que pueda crear el enlace.

```sql
GRANT CREATE PUBLIC DATABASE LINK to moralg;
```

###### Ahora solo nos queda crear el enlace con `CREATE LINK` y realizar la consulta.

``` sql
CREATE PUBLIC DATABASE LINK ConexionPalomaPSQLU
CONNECT TO "paloma"
IDENTIFIED BY "paloma"
USING 'PSQLU';
```

> * Indicamos el nombre del link `ConexionPalomaPSQLU` para que se conecte al usuario `paloma`, con la contraseña `paloma` usando el nombre del servicio `PSQLU`

###### Comprobamos el enlace, realizando un `SELECT` a una de la tablas de paloma:

``` sql
col nombreequipo format a30;
col descripcion format a30;
```

> * Para que salga las columnas mas pequeñas y estéticas, utilizamos `col <nombre_campo> format a<tamaño>`.

![PruebaOra_Post](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/PruebaOra_Post.png?raw=true)

### 3.2 Enlace de Cliente Postgres a Servidor Oracle
-----------------------------------------------------------------

#### Para realizar el enlace de Postgres a Oracle necesitamos el cliente de Oracle y descargamos la extensión para postgres `oracle_fdw`.

##### Configuración el Cliente Postgres

###### Descargamos el cliente de oracle con `wget` desde la página oficial de oracle.

~~~
wget https://download.oracle.com/otn_software/linux/instantclient/195000/oracle-instantclient19.5-basic-19.5.0.0.0-1.x86_64.rpm
~~~

###### Descargamos dependencias y paquetes necesarios.

~~~
wget https://download.oracle.com/otn_software/linux/instantclient/195000/oracle-instantclient19.5-sqlplus-19.5.0.0.0-1.x86_64.rpm

wget https://download.oracle.com/otn_software/linux/instantclient/195000/oracle-instantclient19.5-tools-19.5.0.0.0-1.x86_64.rpm

wget https://download.oracle.com/otn_software/linux/instantclient/195000/oracle-instantclient19.5-devel-19.5.0.0.0-1.x86_64.rpm
~~~

> * Solo con el cliente de Oracle valdría para realizar el enlace, pero durante el proceso de esta tarea, nos surgieron errores de falta de ficheros que se crean automaticamente con las dependencias indicadas anteriormentes.

###### Ya tenemos descargados los ficheros ahora tenemos que instalarlos, pero tenemos el problema que son `.rpm`.
###### Tenemos que utilizar el paquete de `alien` para pasarlos a `.deb`. 

###### Instalamos el paquete:
~~~
sudo apt install alien
~~~

###### Convertimos los `.rpm` a `.deb`:
~~~
sudo alien -d oracle-instantclient19.5-basic-19.5.0.0.0-1.x86_64.rpm
sudo alien -d oracle-instantclient19.5-sqlplus-19.5.0.0.0-1.x86_64.rpm
sudo alien -d oracle-instantclient19.5-tools-19.5.0.0.0-1.x86_64.rpm
sudo alien -d oracle-instantclient19.5-devel-19.5.0.0.0-1.x86_64.rpm
~~~

###### Instalamos los paquetes con `dpkg -i`
~~~
sudo dpkg -i oracle-instantclient19.5-basic_19.5.0.0.0-2_amd64.deb
sudo dpkg -i oracle-instantclient19.5-sqlplus-19.5.0.0.0-1.x86_64.deb
sudo dpkg -i oracle-instantclient19.5-tools-19.5.0.0.0-1.x86_64.deb
sudo dpkg -i oracle-instantclient19.5-devel-19.5.0.0.0-1.x86_64.deb
~~~

###### Ahora vamos a descargarnos e instalarnos la extensión para postgres `oracle_fdw`.
###### Para descargadnos `oracle_fdw` vamos a clonar un repositorio de Github.
~~~
git clone https://github.com/laurenz/oracle_fdw.git
~~~

###### Ahora tenemos que realizar la compilación de dicho paquete, para esto vamos a utlizar `make`

###### Puede que se produzcan varios fallos durante la compilación, os mostraré los fallos que me saltaron durante la realización de esta tarea.

-------------------------------
###### FALLO 1
~~~
vagrant@Postgres:~/oracle_fdw$ make
Makefile:20: /usr/lib/postgresql/11/lib/pgxs/src/makefiles/pgxs.mk: No such file or directory
make: *** No rule to make target '/usr/lib/postgresql/11/lib/pgxs/src/makefiles/pgxs.mk'.  Stop.
~~~

###### Este fallo se produce al no tener instalado el paquete `postgresql-server-dev-all`

~~~
vagrant@Postgres:~/oracle_fdw$ sudo apt install postgresql-server-dev-all
~~~
------------------
###### FALLO 2
~~~
oracle_utils.c:22:10: fatal error: oci.h: No such file or directory
 #include <oci.h>
          ^~~~~~~
compilation terminated.
make: *** [<builtin>: oracle_utils.o] Error 1
~~~
###### Este fallo se solucciona igualando las siguiente variables a las rutas correctas.
~~~
export ORACLE_HOME="/usr/lib/oracle/19.5/client64"
export LD_LIBRARY_PATH="/usr/lib/oracle/19.5/client64/lib"
export PATH=$ORACLE_HOME:$PATH
export USE_PGXS=1
~~~
-----------------------------
###### FALLO 3

###### Tenemos que modificar el `Makefile`, ya que la versión del cliente que nos hemos descargado es la `19.5` y esta no esta incluida en el `Makefile`:

###### Tenemos que añadir `-I/usr/include/oracle/19.5/client64` en la variable `PG_CPPFLAGS` y tenemos que añadir `-L/usr/lib/oracle/19.5/client64/lib` en la variable `SHLIB_LINK`

###### MakeFile corregido para la versión 19.5
~~~
MODULE_big = oracle_fdw
OBJS = oracle_fdw.o oracle_utils.o oracle_gis.o
EXTENSION = oracle_fdw
DATA = oracle_fdw--1.1.sql oracle_fdw--1.0--1.1.sql
DOCS = README.oracle_fdw
REGRESS = oracle_fdw oracle_gis oracle_import oracle_join

# add include and library paths for both Instant Client and regular Client
PG_CPPFLAGS = -I$(ORACLE_HOME)/sdk/include -I$(ORACLE_HOME)/oci/include -I$(ORACLE_HOME)/rdbms/public -I/usr/include/oracle/19.5/client64

SHLIB_LINK = -L$(ORACLE_HOME) -L$(ORACLE_HOME)/bin -L$(ORACLE_HOME)/lib -L$(ORACLE_HOME)/lib/amd64 -l$(ORACLE_SHLIB) -L/usr/lib/oracle/19.5/client64/lib

ifdef NO_PGXS
subdir = contrib/oracle_fdw
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
else
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
endif

# Oracle's shared library is oci.dll on Windows and libclntsh elsewhere
ifeq ($(PORTNAME),win32)
ORACLE_SHLIB=oci
else
ifeq ($(PORTNAME),cygwin)
ORACLE_SHLIB=oci
else
ORACLE_SHLIB=clntsh
endif
endif
~~~
---------------------------

###### Ahora vamos a realizar la instalación, si todo ha salido bien en la compilación.
~~~
sudo make install
~~~

###### Una vez instalado la extensión, vamos a crearla en la base de datos.

###### Accedemos a la base de datos con el usuario `postgres` y creamos la extensión con `CREATE EXTENSION oracle_fdw`.
~~~
postgres@Postgres:/home/vagrant/oracle_fdw$ psql
psql (11.5 (Debian 11.5-1+deb10u1))
Type "help" for help.

postgres=# CREATE EXTENSION oracle_fdw;
ERROR:  could not load library "/usr/lib/postgresql/11/lib/oracle_fdw.so": libaio.so.1: cannot open shared object file: No such file or directory
~~~

> * Nos salta un error al crear la extension. Esto se solucciona con la instalacción de dos librerías `libaio1` y `libaio-dev`.

~~~
sudo apt-get install libaio1 libaio-dev
~~~

###### Creamos la extensión:
``` sql
CREATE EXTENSION oracle_fdw;
```

###### Creamos un nuevo servidor con la opciones del servidor de Oracle.

``` sql
CREATE SERVER ConexionOraclePaloma FOREIGN DATA WRAPPER oracle_fdw OPTIONS(dbserver '//192.168.43.58:1521/ORCL');
```

> `CREATE SERVER`: define un nuevo servidor externo. El usuario que define el servidor se convierte en su propietario.

###### Le asignamos el usuario `paloma` con la contraseña `paloma`.

``` sql
CREATE USER MAPPING for postgres SERVER ConexionOraclePaloma OPTIONS(user 'paloma', 
                                                                      password 'paloma');
```

###### Ahora tenemos que crear una tabla externa que debería de ser igual a la tabla del servidor Oracle que queremos consultar.

> * Podemos realizar, en el servidor Oracle, la consulta `DESCRIBE <nombre_tabla>;` para que nos muestre la información necesario para crear la `FOREIGN TABLE`.

###### Creamos la tabla externa:

``` sql
CREATE FOREIGN TABLE ASPECTOS(codigo varchar(10), 
                              descripcion varchar(30), 
                              importancia varchar(10)) 
SERVER ConexionOraclePaloma OPTIONS(schema 'PALOMA', 
                                     table 'ASPECTOS');
```

> * Hay que tener en cuenta que las opciones `schema` y `table` tienen que estar en mayúsculas, ya que oracle los guarda de este modo.

###### Ahora realizamos la consulta:

![PruebaPost_Ora](https://github.com/MoralG/Interconexion_de_Servidores_de_BBDD/blob/master/image/PruebaPost_Ora.png?raw=true)

> * Si queremos dar permisos a un usuario concreto de Postgres para que pueda realizar consultas a un servidor externo concreto utilizaremos `GRANT USAGE ON FOREIGN SERVER <nombre_servidor> TO <nombre_usuario>;`.