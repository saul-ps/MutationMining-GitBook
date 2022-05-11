# OMIM

Se han realizado pruebas para esta BBDD, pero está desaconsejada por mostrar información poco relevante. De todos modos, se mantienen los scripts por si en un futuro pudieran contener información de interés.

### Descarga de archivos

En el siguiente [enlace](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf\_GRCh38/) debemos descargar el fichero llamado "clinvar.vcf.gz". Además, para trabajar con este archivo tendremos que dirigirnos a [Download SnpEff](https://snpeff.blob.core.windows.net/versions/snpEff\_latest\_core.zip) y así poder descargar el archivo comprimido necesario para el filtrado de datos. Posteriormente, extraemos el contenido del paquete y nos quedaremos únicamente con el fichero "SnpSift.jar".&#x20;

Para obtener los ficheros del proyecto OMIM necesarios para la ejecución del script descrito en el siguiente apartado tendremos que solicitarlos en el siguiente [enlace](https://www.omim.org/downloads) rellenando el formulario que aparece a pie de página. Una vez recibido el correo por parte de dicha organización, únicamente nos descargamos el archivo llamado "genemap2.txt".

Seleccionamos todos los ficheros y los guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/GenomeDDBB/OMIM
{% endhint %}

### Crear RData

Para limpiar el archivo "clinvar.vcf.gz" y poder usarlo de forma adecuada en el proyecto debemos ejecutar las siguientes sentencias:

#### Terminal de Ubuntu

```bash
cd /data/MutationMiningData/GenomeDDBB/OMIM
java -jar SnpSift.jar extractFields -s "," -e "." clinvar.vcf.gz CHROM POS ID RS REF ALT CLNDN CLNSIG CLNDISDB > clinvar_Pathogenic_DNprovided.tab
```

De este manera generamos un archivo en formato de tabla con las columnas requeridas para el funcionamiento del siguiente bloque de ejecución:

#### R

```r
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

PATH_OMIM <- "/data/MutationMiningData/GenomeDDBB/OMIM/"
file_clinvar <- "clinvar_Pathogenic_DNprovided.tab"
file_omim <- "genemap2.txt"

################################### CLINVAR ###################################
clinvar_clean <- as_tibble(read.table(
    file =
        paste0(PATH_OMIM, file_clinvar),
    sep = "\t", fill = TRUE, header = TRUE, quote = ""
))

omim_numbers <- str_extract_all(clinvar_clean$CLNDISDB, "(?<=OMIM:).[:alnum:]*")

clinvar_clean <- clinvar_clean %>%
    mutate(OMIM = omim_numbers) %>%
    filter(CLNDN != "." & CLNDN != "not_provided") %>%
    select(ID, CHROM, POS, REF, ALT, RS, OMIM) %>%
    mutate(RS = paste0("rs", RS)) %>%
    mutate(RS = str_replace(RS, "rs.$", "NULL"))

colnames(clinvar_clean) <- c(
    "IdClinvar", "Chr", "Start", "Ref", "Alt", "SNPS", "OmimNumbers"
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

################################### OMIM ###################################
omim <- as_tibble(read.table(
    file =
        paste0(PATH_OMIM, file_omim),
    sep = "\t", fill = TRUE, header = FALSE, quote = ""
))

colnames(omim) <- c(
    "Chromosome", "Genomic.Position.Start", "Genomic.Position.End",
    "Cyto.Location", "Computed.Cyto.Location", "MIM.Number", "Gene.Symbols",
    "Gene.Name", "Approved.Gene.Symbol", "Entrez.Gene.ID", "Ensembl.Gene.ID",
    "Comments", "Phenotypes", "Mouse.Gene.Symbol/ID"
)

omim <- omim %>%
    select(
        Chromosome, Genomic.Position.Start, MIM.Number,
        Gene.Name, Comments, Phenotypes
    ) %>%
    mutate(MIM.Number = as.character(MIM.Number)) %>%
    mutate(Chromosome = str_replace(Chromosome, "chr", ""))

################################### FINAL ###################################
equal_omim_numbers <- unlist(A6_S39_GT012_clinvar$OmimNumbers)

A6_S39_GT012_omim <- omim %>%
    filter(MIM.Number %in% equal_omim_numbers)

equal_omim_numbers <- unlist(B6_S40_GT012_clinvar$OmimNumbers)

B6_S40_GT012_omim <- omim %>%
    filter(MIM.Number %in% equal_omim_numbers)
```
