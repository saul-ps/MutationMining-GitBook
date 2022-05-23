# PharmGKB

### Descarga de archivos

#### Script

```bash
wget https://api.pharmgkb.org/v1/download/file/data/clinicalAnnotations.zip
mkdir pharmgkb
unzip clinicalAnnotations.zip -d pharmgkb/
rm clinicalAnnotations.zip 
mv pharmgkb/clinical_annotations.tsv /data/MutationMiningData/GenomeDDBB/PHARMGKB/
rm -r pharmgkb/
```

#### Manual (opcional)

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

# Borramos los archivos que no son necesarios para ahorrar espacio en disco
# Excepto los archivos históricos RData
used_files <- str_subset(list.files(PATH_PHARMGKB), ".RData$", negate=T)
for (file in used_files) {
    path_file <- paste0(PATH_PHARMGKB, file)
    message(paste0("Borrando ", file, "..."))
    file.remove(path_file)
}

# Asignamos el nombre del archivo RData con la fecha actual
name_DDBB <- paste0("pharmgkb_", Sys.Date())
assign(name_DDBB, pharmgkb)

# Guardamos la tabla en un fichero formato RData
save(list = name_DDBB, file = paste0(PATH_PHARMGKB, name_DDBB, ".RData"))
```

### Visualización en aplicación web

A través de una función JavaScript podremos filtrar por:

* Todos los registros que tenga un nivel de evidencia 3 o 4 serán descartados.
