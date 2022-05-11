# Demonio PHP

En este apartado se detallan los comandos para iniciar y detener el modo demonio PHP, se deben ejecutar desde la terminal de Ubuntu.

```bash
# Acceder a la carpeta que realiza dicha función
cd /data/MutationMining/daemon/

# Iniciar el modo demonio
/usr/bin/php -c phpd.ini MutationMiningDaemon

# Detener el modo demonio
kill -9 $(ps aux | grep 'MutationMiningDaemon' | awk '{ print $2 }')
```
