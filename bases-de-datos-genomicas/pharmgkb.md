# PharmGKB

### Descarga de archivos

En el siguiente [enlace](https://www.pharmgkb.org/downloads) debemos descargar el fichero llamado "clinicalAnnotations.zip", posteriormente descomprimimos el archivo y nos quedamos únicamente con el fichero "clinical\_annotations.tsv".

Seleccionamos este fichero y lo guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/GenomeDDBB/PHARMGKB
{% endhint %}

### Crear RData

Para filtrar este archivo y poder usarlo de forma adecuada en el proyecto debemos ejecutar las siguientes sentencias:

#### R

```r
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

PATH_PHARMGKB <- "/data/MutationMiningData/GenomeDDBB/PHARMGKB/"
pharmgkb_file <- "clinical_annotations.tsv"

pharmgkb <- as_tibble(read.table(
    file = paste0(PATH_PHARMGKB, pharmgkb_file),
    sep = "\t", fill = TRUE, header = TRUE, quote = ""
))

pharmgkb <- pharmgkb %>%
    select(
        Clinical.Annotation.ID, Variant.Haplotypes, Phenotype.s.,
        Phenotype.Category, Drug.s., Level.of.Evidence, URL
    ) %>%
    filter(str_detect(Variant.Haplotypes, "^rs.*"))

colnames(pharmgkb) <- c(
    "IDPharmGKB", "SNPS", "PhenotypePharm",
    "PhenotypeCategoryPharm", "Drugs", "Evidence", "URL"
)

# Borramos los archivos que no son necesarios para ahorra espacio en disco
used_files <- list.files(PATH_PHARMGKB)
for (file in used_files) {
    path_file <- paste0(PATH_PHARMGKB, file)
    message(paste0("Borrando ", file, "..."))
    file.remove(path_file)
}

# Guardamos la tabla en un fichero formato RData
save(pharmgkb,
    file = paste0(PATH_PHARMGKB, "pharmgkb.RData")
)
```

### Visualización en aplicación web

A través de una función JavaScript podremos filtrar por:

* Todos los registros que tenga un nivel de evidencia 3 o 4 serán descartados.
