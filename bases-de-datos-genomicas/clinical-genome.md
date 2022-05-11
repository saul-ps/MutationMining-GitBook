# Clinical Genome

### Descarga de archivos

En el siguiente [enlace](https://erepo.clinicalgenome.org/evrepo/) debemos ir hacia el botón "Download" y pulsar la opción "Tab-delimited", esto provocará la descarga del fichero denominado "erepo.tabbed.txt".&#x20;

{% hint style="info" %}
Para que el script funcione correctamente borraremos el carácter " # " situado al inicio del fichero .txt
{% endhint %}

Seleccionamos este fichero y lo guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/GenomeDDBB/CLINICALGENOME
{% endhint %}

### Crear RData

Para filtrar este archivo y poder usarlo de forma adecuada en el proyecto debemos ejecutar las siguientes sentencias:

#### R

```r
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

PATH_CLINICAL_GENOME <- "/data/MutationMiningData/GenomeDDBB/CLINICALGENOME/"
file_clinical_genome <- "erepo.tabbed.txt"

clinical_genome <- as_tibble(read.table(
    file = paste0(PATH_CLINICAL_GENOME, file_clinical_genome),
    sep = "\t", fill = TRUE, header = TRUE, quote = ""
))

clinical_genome <- clinical_genome %>%
    select(
        ClinVar.Variation.Id, Disease, Mode.of.Inheritance, Assertion,
        Summary.of.interpretation, PubMed.Articles
    ) %>%
    mutate(ClinVar.Variation.Id = as.integer(ClinVar.Variation.Id))

colnames(clinical_genome) <- c(
    "IDClinvar", "Disease", "Mode.of.Inheritance",
    "Assertion", "Summary.of.interpretation", "PubMed.Articles"
)

# Borramos los archivos que no son necesarios para ahorra espacio en disco
used_files <- list.files(PATH_CLINICAL_GENOME)
for (file in used_files) {
    path_file <- paste0(PATH_CLINICAL_GENOME, file)
    message(paste0("Borrando ", file, "..."))
    file.remove(path_file)
}

# Guardamos la tabla en un fichero formato RData
save(clinical_genome,
    file = paste0(PATH_CLINICAL_GENOME, "clinical_genome.RData")
)
```
