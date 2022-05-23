# GWAS Catalog

### Descarga de archivos

En el siguiente [enlace](https://www.ebi.ac.uk/gwas/docs/file-downloads) debemos descargar el fichero llamado "All associations v1.0". Una vez descargado cambiamos su nombre por "gwas\_catalog.tsv".

{% hint style="info" %}
Para realizar el filtrado de este archivo se ha usado una tabla en la que cada registro se corresponde a un artículo de GWAS Catalog, dividida en cuatros columnas:

* Pubmed
* Revista de publicación
* Número de citas
* Fecha de publicación.
{% endhint %}

{% file src="../.gitbook/assets/articles.tab" %}
Tabla impacto de artículos GWAS Catalog
{% endfile %}

Seleccionamos estos ficheros y los guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/GenomeDDBB/GWASCatalog
{% endhint %}

### Crear RData

Para filtrar estos archivos y poder usarlos de forma adecuada en el proyecto debemos ejecutar las siguientes sentencias:

#### R

```r
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

PATH_GWAS_CATALOG <- "/data/MutationMiningData/GenomeDDBB/GWASCatalog/"
file_gwas_catalog <- "gwas_catalog.tsv"

PATH_GWAS_CATALOG_ARTICLES <-
    "/data/MutationMiningData/GenomeDDBB/GWASCatalog/articles.tab"

# Articles GWAS Catalog
articles_gwas <-
    as_tibble(read.table(
        file = PATH_GWAS_CATALOG_ARTICLES,
        sep = ",", header = F, fill = TRUE, quote = '"'
    ))

colnames(articles_gwas) <-
    c("PUBMEDID", "JOURNAL", "N_CIT", "DATE_JOURNAL")

articles_gwas <- articles_gwas %>%
    mutate(DATE_JOURNAL = as.numeric(str_sub(DATE_JOURNAL, 1, 4)))

current_year <- max(articles_gwas$DATE_JOURNAL) + 1

articles_gwas <- articles_gwas %>%
    mutate(
        N_CIT_Y =
            round(N_CIT / (current_year - DATE_JOURNAL), 2)
    ) %>%
    select(PUBMEDID, N_CIT, N_CIT_Y)

# BBDD GWAS Catalog
gwas_catalog <- as_tibble(read.table(
    file = paste0(PATH_GWAS_CATALOG, file_gwas_catalog),
    sep = "\t", fill = TRUE, header = TRUE, quote = ""
))

# CITATIONS
gwas_catalog <- inner_join(gwas_catalog, articles_gwas, by = "PUBMEDID")

gwas_catalog_citations <- gwas_catalog %>%
    distinct(
        PUBMEDID, DISEASE.TRAIT, STUDY, CHR_ID, CHR_POS, SNPS,
        STRONGEST.SNP.RISK.ALLELE, RISK.ALLELE.FREQUENCY, REPORTED.GENE.S.,
        P.VALUE, OR.or.BETA, LINK, N_CIT, N_CIT_Y
    ) %>%
    group_by(
        PUBMEDID, DISEASE.TRAIT, STUDY, CHR_ID, CHR_POS, SNPS,
        STRONGEST.SNP.RISK.ALLELE, REPORTED.GENE.S.,
        LINK, N_CIT, N_CIT_Y
    ) %>%
    summarise(
        # OR.or.BETA related to min(P.VALUE)
        OR.or.BETA = OR.or.BETA[match(min(P.VALUE), P.VALUE)],
        P.VALUE = min(P.VALUE),
        RISK.ALLELE.FREQUENCY =
            round(mean(as.numeric(RISK.ALLELE.FREQUENCY), na.rm = T), 2),
        .groups = "drop"
    ) %>%
    select(
        PUBMEDID, DISEASE.TRAIT, STUDY, CHR_ID, CHR_POS, SNPS,
        STRONGEST.SNP.RISK.ALLELE, RISK.ALLELE.FREQUENCY, REPORTED.GENE.S.,
        P.VALUE, OR.or.BETA, LINK, N_CIT, N_CIT_Y
    )

colnames(gwas_catalog_citations) <- c(
    "PUBMEDID", "DISEASE.TRAIT", "STUDY", "CHR_ID", "CHR_POS", "SNPS",
    "RISK.ALL", "RISK.ALL.FREQ", "REPORTED.GENE.S.", "P.VALUE",
    "OR.or.BETA", "URL", "N_CIT", "N_CIT_Y"
)

gwas_catalog_citations <- gwas_catalog_citations %>%
    # Separate common snp 
    mutate(SNPS = str_split(SNPS, ";")) %>%
    unnest(SNPS) %>%
    mutate(SNPS = str_replace(SNPS, " ", "")) %>%
    filter(str_detect(SNPS, "^rs.*")) %>%
    # Delete SNP interaction
    filter(!str_detect(SNPS, "x"))

citations_snp_disease <- gwas_catalog_citations %>%
    group_by(DISEASE.TRAIT, SNPS) %>%
    summarise(
        N_ART = n(),
        N_CIT = sum(N_CIT, na.rm = T),
        N_CIT_Y = sum(N_CIT_Y, na.rm = T),
        .groups = "drop"
    )

gwas_catalog_citations <- gwas_catalog_citations %>%
    select(
        PUBMEDID, DISEASE.TRAIT, STUDY, CHR_ID, CHR_POS, SNPS,
        RISK.ALL, RISK.ALL.FREQ, REPORTED.GENE.S., P.VALUE,
        OR.or.BETA, URL
    ) %>%
    inner_join(citations_snp_disease,
        by = c("DISEASE.TRAIT", "SNPS")
    )

# Add REF
gwas_catalog_BSgenome <- gwas_catalog_citations %>%
    mutate(
        RISK.ALL =
            str_extract_all(RISK.ALL, paste0("(?<=", SNPS, "-).[:alnum:]*"))
    ) %>%
    mutate(RISK.ALL = as.character(RISK.ALL)) %>%
    filter(str_detect(RISK.ALL, "[:alpha:]")) %>%
    mutate(CHR_POS = as.double(CHR_POS)) %>%
    filter(nchar(CHR_ID) > 0) %>%
    filter(CHR_POS > 0) %>%
    mutate(RISK.ALL = str_replace(RISK.ALL, " ", ""))

# BSgenome ; REF CHR
chr_number <- gwas_catalog_BSgenome$CHR_ID
chr_position <- gwas_catalog_BSgenome$CHR_POS
genome <- getBSgenome("BSgenome.Hsapiens.NCBI.GRCh38")
myseqs <- data.frame(
    chr = chr_number,
    start = chr_position,
    end = chr_position
)
chr_ref <- as.character(getSeq(genome, myseqs$chr,
    start = myseqs$start, end = myseqs$end
))

# Final Results
gwas_catalog_BSgenome <- gwas_catalog_BSgenome %>%
    mutate(REF = chr_ref) %>%
    select(
        PUBMEDID, URL, DISEASE.TRAIT, STUDY, CHR_ID, CHR_POS, SNPS,
        RISK.ALL, REF, RISK.ALL.FREQ, REPORTED.GENE.S., P.VALUE,
        OR.or.BETA, N_ART, N_CIT, N_CIT_Y
    ) %>%
    # Los SNP con REF = ALT no interesan porque no se pueden cruzar,
    # dado que las letras se sacan por genotipo.
    # Si en un futuro se quisiera utilizar, juntar las columnas REF & ALT
    # y realizar la misma operacion por cada muestra en el proyecto
    # Ejemplo: REF = A , ALT = T || Secuencia GWAS: AT |
    # Genotipo_0 = AA , Genotipo_1 = AT , Genotipo_2 = TT
    filter(RISK.ALL != REF)

# Borramos los archivos que no son necesarios para ahorrar espacio en disco
# Excepto los archivos históricos RData
used_files <- str_subset(list.files(PATH_GWAS_CATALOG), ".RData$", negate=T)
for (file in used_files) {
    path_file <- paste0(PATH_GWAS_CATALOG, file)
    message(paste0("Borrando ", file, "..."))
    file.remove(path_file)
}

# Asignamos el nombre del archivo RData con la fecha actual
name_DDBB <- paste0("gwas_catalog_BSgenome_", Sys.Date())
assign(name_DDBB, gwas_catalog_BSgenome)

# Guardamos la tabla en un fichero formato RData
save(list = name_DDBB, file = paste0(PATH_GWAS_CATALOG, name_DDBB, ".RData"))
```

### Visualización en aplicación web

A través de una función JavaScript podremos filtrar por diferentes características, cualquier registro que incluya una de estas condiciones será descartado:

* El registro que incluya alguna de estas palabras:\
  levels, related, count, mean, percentage, pressure, bmi, dose, obesity, height, body mass, menopause, plateletcrit, hematocrit, cholesterol, body fat, rate, distribution, fraction, concentration, circumference, interval, distribution
* Los registros que no sean " exonic "
* Cualquier registro que no tenga más de un artículo asociado y no cuente con más de diez citas anuales

### Aspectos a tener en cuenta

Posibles errores:

* position - 1 -> retirar primer alelo (insercciones, delecciones);&#x20;
* rs141501074 was merged into rs59586144;&#x20;
* risk.allele = ? - character(0);&#x20;
* rs9996856 x rs952995 (interacciones de SNP's)
