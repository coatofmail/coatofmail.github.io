## Error Mysql Too many open files

Síntomas: Mysql se para. En el log de errores puedes ver el temido Too many open files.
Dificultad solución: Fácil.


#### ¿Qué ha pasado?
Hay demasiados archivos abiertos a la vez. Vamos, que tu seridor no da de más o lo tienes mál configurado.

#### ¿Cómo se el límite de archivos que hay configurado en mysql?
Corre esto en mysql:
```
SHOW VARIABLES LIKE 'open_files_limit';
```

#### ¿Cómo puedo saber el límite que debería haber?
Cada tabla se considera un archivo. Cada base de datos suele tener varias tablas.

Para saber cuantas tabla tiene una de las bases de datos haz lo siguiente. En vez de your_database_name pon el nombre de la base de datos que quieres medir.
```
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'your_database_name';
```

Por ultimo, corre esto y sabrás el límite que tienes que poner. Si consideras que vas a necesitar más en el futuro, pon un lñimite mayor.
```
num_files=$(echo "151 * 5000 + 1000" | bc)
echo "$num_files"
```

#### ¿Donde puedo cambiarlo?
Conozco 3 sitios:
- Archivos de configuración de mysql.
- /etc/security/limits.conf
- Systemd.

En mi caso modifiqué los archivos de configuración de mysql y /etc/security/limits.conf y aun así no cambiaban los límites. Finalmente fue en systemd donde al cambiar los límites me funcionó a la perfección.

---

### Solución mediante archivos de configuración de mysql

Primero has de saber que archivos está usando mysql para su configuración.
```
mysqld --verbose --help | grep -A 1 "Default options"
```
Normalmente son estos: /etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf

Teniendo en cuenta que mysql no suele tener un directorio propio, y que /etc/my.cnf no suele existir, nos queda /etc/mysql/my.cnf  que a su vez tiene una directiva dentro que suele incluir todos los archivos de configuración que contiene /etc/mysql/

Para no ir mirando archivo tras archivo de /etc/mysql/ con este comendo podemos saber donde se ha indicado el límite de archivos:
```
sudo grep -r "open_files_limit" /etc/mysql/
```
Edita la línea open_files_limit en los archivos. Ten en cuenta que debe estar debajo de [mysqld]. Si están debajo de otra sección, no funcionará.

Un ejemplo:
```
open_files_limit 1000000
```
Por último reinicia mysql
```
sudo systemctl restart mysql
```

### Solución mediante /etc/security/limits.conf

En /etc/security/limits.conf se configuran los límites de uso de archivos por usuarios del sistema.

Un ejemplo de lo que podrías poner:
```
mysql          soft    memlock         unlimited
mysql          hard    memlock         unlimited
mysql          soft    nofile          60000000
mysql          hard    nofile          60000000
mysql          soft    nproc           60000000
mysql          hard    nproc           60000000
```
Por último reinicia mysql
```
sudo systemctl restart mysql
```

### Solución mediante systemd

Edita  /usr/lib/systemd/system/mysql.service

En [Service] quizás veas LimitNOFILE y LimitNPROC. Si no los ves, añádelos.

Un ejemplo:
```
[Service]
LimitNOFILE=60000000
LimitNPROC=60000000
```
Por último reinicia mysql
```
sudo systemctl restart mysql
```


## Comprobar que ha funcionado.
Simplemente ve a mysql y corre esto:
```
SHOW VARIABLES LIKE 'open_files_limit';
```
