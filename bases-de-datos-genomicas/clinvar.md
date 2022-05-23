# ClinVar

### Descarga de archivos

#### Script

```bash
wget https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf_GRCh38/clinvar.vcf.gz
wget https://snpeff.blob.core.windows.net/versions/snpEff_latest_core.zip
unzip snpEff_latest_core.zip 
mv clinvar.vcf.gz /data/MutationMiningData/GenomeDDBB/CLINVAR/
mv snpEff/SnpSift.jar /data/MutationMiningData/GenomeDDBB/CLINVAR/
rm -r snpEff_latest_core.zip snpEff/    
```

#### Manual (opcional)

En el siguiente [enlace](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf\_GRCh38/) debemos descargar el fichero llamado "clinvar.vcf.gz". Además, para trabajar con este archivo tendremos que dirigirnos a [Download SnpEff](https://snpeff.blob.core.windows.net/versions/snpEff\_latest\_core.zip) y así poder descargar el archivo comprimido necesario para el filtrado de datos. Posteriormente, extraemos el contenido del paquete y nos quedaremos únicamente con el fichero "SnpSift.jar".&#x20;

Seleccionamos ambos ficheros y los guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/GenomeDDBB/CLINVAR
{% endhint %}

### Crear RData

Para limpiar el archivo "clinvar.vcf.gz" y poder usarlo de forma adecuada en el proyecto debemos ejecutar las siguientes sentencias:

#### Terminal de Ubuntu

```bash
cd /data/MutationMiningData/GenomeDDBB/CLINVAR
java -jar SnpSift.jar extractFields -s "," -e "." clinvar.vcf.gz CHROM POS ID RS REF ALT CLNDN CLNSIG CLNDISDB > clinvar_Pathogenic_DNprovided.tab
```

De este manera generamos un archivo en formato de tabla con las columnas requeridas para el funcionamiento del siguiente bloque de ejecución:

#### R

```r
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

PATH_CLINVAR <- "/data/MutationMiningData/GenomeDDBB/CLINVAR/"
file_clinvar <- "clinvar_Pathogenic_DNprovided.tab"

clinvar_clean <- as_tibble(read.table(
    file =
        paste0(PATH_CLINVAR, file_clinvar),
    sep = "\t", fill = TRUE, header = TRUE, quote = ""
))

clinvar_clean <- clinvar_clean %>%
    filter(CLNDN != "." & CLNDN != "not_provided") %>%
    select(ID, CHROM, POS, REF, ALT, RS, CLNDN, CLNSIG) %>%
    mutate(RS = paste0("rs", RS)) %>%
    mutate(RS = str_replace(RS, "rs.$", "NULL"))

colnames(clinvar_clean) <- c(
    "IDClinvar", "Chr", "Start",
    "Ref", "Alt", "SNPS", "CLNDN", "CLNSIG"
)

# Insercciones y delecciones, proceso de transformación ->
# < TTTGA / T > se convierte en < TTGA / - > y +1 en [Start]
clinvar <- clinvar_clean %>%
    mutate(
        Ref =
            ifelse(nchar(Alt) > 1 & Ref == str_sub(Alt, 1, 1), "-", Ref)
    ) %>%
    mutate(
        Alt =
            ifelse(Ref == "-", str_sub(Alt, 2, nchar(Alt)), Alt)
    ) %>%
    mutate(
        Alt =
            ifelse(nchar(Ref) > 1 & Alt == str_sub(Ref, 1, 1), "-", Alt)
    ) %>%
    mutate(
        Ref =
            ifelse(Alt == "-", str_sub(Ref, 2, nchar(Ref)), Ref)
    ) %>%
    mutate(
        Start =
            ifelse(Ref == "-" | Alt == "-", Start + 1, Start)
    )

# Borramos los archivos que no son necesarios para ahorra espacio en disco
# Excepto los archivos históricos RData
used_files <- str_subset(list.files(PATH_CLINVAR), ".RData$", negate=T)
for (file in used_files) {
    path_file <- paste0(PATH_CLINVAR, file)
    message(paste0("Borrando ", file, "..."))
    file.remove(path_file)
}

# Asignamos el nombre del archivo RData con la fecha actual
name_DDBB <- paste0("clinvar_", Sys.Date())
assign(name_DDBB, clinvar)

# Guardamos la tabla en un fichero formato RData
save(clinvar, file = paste0(PATH_CLINVAR, "clinvar.RData"))
```

### Visualización en aplicación web

A través de una función JavaScript podremos filtrar por:

* Todos los registros que no incluyan la palabra "pathogenic" en la columna "CLNSIG" serán descartados.

