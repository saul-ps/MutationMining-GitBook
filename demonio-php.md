# Demonio PHP

En este apartado se detallan los comandos para iniciar y detener el modo demonio PHP, se deben ejecutar desde la terminal de Ubuntu.

```bash
# Iniciar el modo demonio
nohup /usr/bin/php /data/MutationMining/daemon/MutationMiningDaemon

# Detener el modo demonio
kill -9 $(ps aux | grep 'MutationMiningDaemon' | awk '{ print $2 }')
```
