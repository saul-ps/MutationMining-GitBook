# Directorio principal y R

Comenzamos creando el directorio dónde alojaremos todos lo archivos necesarios para la instalación y funcionamiento de esta aplicación.

```bash
mkdir /data/MutationMining
```

Posteriormente, creamos una nueva carpeta que usaremos para la instalación de R.

```bash
cd /data/MutationMining
mkdir bin
cd bin
```

En este proyecto estamos usando la versión R-4.0.4, instalada desde su código fuente. En el siguiente enlace [http://cran.freestatistics.org/src/base/R-4/](http://cran.freestatistics.org/src/base/R-4/) podemos descargar el archivo comprimido.

El proceso de instalación que hemos seguido es el descrito en esta dirección [https://www.nosinmiubuntu.com/instalar-r-paquete-estadistico-desde/](https://www.nosinmiubuntu.com/instalar-r-paquete-estadistico-desde/)

Una vez instalado el software de R en dicha ruta, añadimos los paquetes necesarios para la ejecución correcta de la aplicación.

{% code title="packages.R" %}
```r
listOfPackages <- c(
    "data.table", "plyr", "dplyr", "stringr", "tidyr", "tibble",
    "BiocManager", "openxlsx", "RMySQL", "bcrypt", "openssl"
)
for (i in listOfPackages) {
    if (!i %in% installed.packages()) {
        install.packages(i, dependencies = TRUE)
    }
    require(i, character.only = TRUE)
}

listOfPackagesBioconductor <- c("BSgenome", "BSgenome.Hsapiens.NCBI.GRCh38")
for (i in listOfPackagesBioconductor) {
    if (!i %in% installed.packages()) {
        BiocManager::install(i)
    }
    require(i, character.only = TRUE)
}
```
{% endcode %}
