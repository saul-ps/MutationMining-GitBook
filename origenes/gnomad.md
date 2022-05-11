# gnomAD

Se han realizado pruebas para esta BBDD, pero está desaconsejada por mostrar información poco relevante. De todos modos, se mantienen los scripts por si en un futuro pudieran contener información de interés.

### Descarga de archivos

En el siguiente [enlace](https://gnomad.broadinstitute.org/downloads#v2-liftover-variants) debemos descargar el fichero llamado "All chromosomes VCF" en el apartado: Variants (GRCh38 liftover) --> Exomes --> All chromosomes VCF --> Download from Google (example). Además, para trabajar con este archivo tendremos que dirigirnos a [Download SnpEff](https://snpeff.blob.core.windows.net/versions/snpEff\_latest\_core.zip) y así poder descargar el archivo comprimido necesario para el filtrado de datos. Posteriormente, extraemos el contenido del paquete y nos quedaremos únicamente con el fichero "SnpSift.jar".&#x20;

Seleccionamos ambos ficheros y los guardamos en la ruta:

{% hint style="info" %}
/data/MutationMiningData/Ancestry/gnomAD
{% endhint %}

### Crear RData

#### Terminal

```
cd /data/MutationMiningData/Ancestry/gnomAD
gunzip -c gnomad.exomes.r2.1.1.sites.liftover_grch38.vcf.bgz > gnomAD_v2_liftover.vcf
java -jar SnpSift.jar extractFields gnomAD_v2_liftover.vcf CHROM POS ID REF ALT AF_afr AF_amr AF_asj AF_eas AF_eas_jpn AF_eas_kor AF_eas_oea AF_fin AF_nfe AF_nfe_bgr AF_nfe_est AF_nfe_nwe AF_nfe_onf AF_nfe_seu AF_nfe_swe AF_oth AF_popmax AF_sas > gnomAD_v2_liftover.tab
```

#### R

```
# Ruta para iniciar R
# /data/MutationMining/bin/R-4.0.4/bin/R

# Paquetes necesarios para la ejecución del script
source("/data/MutationMining/RScript/packages.R")

path_file <- "/data/MutationMiningData/Ancestry/gnomAD/gnomAD_v2_liftover.tab"

gnomAD_v2_liftover <-
    as_tibble(read.table(file = path_file, sep = "\t", fill = TRUE, quote = ""))

# Modificar NA por -1
regions <- str_subset(colnames(gnomAD_v2_liftover), "AF_")
for (region in regions) {
    gnomAD_v2_liftover[[region]] <-
        ifelse(is.na(gnomAD_v2_liftover[[region]]),
            -1,
            gnomAD_v2_liftover[[region]]
        )
}

# Guardamos Base de Datos
gnomAD_v2_liftover_no_NA <- gnomAD_v2_liftover
rm(gnomAD_v2_liftover)
path_RData <- 
    "/data/MutationMiningData/Ancestry/gnomAD/gnomAD_v2_liftover_no_NA.RData"
save(gnomAD_v2_liftover_no_NA, file = path_RData)

# Búsqueda de SNPS exclusivos
regions_no_search <- c("AF_popmax", "AF_oth")
idx <- which(regions %in% regions_no_search)
regions <- regions[-idx]
for (region in regions) {
    sentence <- paste0(
        "gnomAD_v2_liftover_no_NA %>% filter(", region, " > 0.4)"
    )
    for (region_min in regions) {
        if (region != region_min) {
            sentence <- paste0(
                sentence, " %>% filter(", region_min, " < 0.1)"
            )
        } else {
            next
        }
    }
    message(sentence)
    sentence_dplyr <- parse(text = sentence)
    assign(
        region,
        eval(sentence_dplyr)
    )
}

# Para cruzar estos datos con una muestra, multiplicamos GT_012 * AF_freq
```

### Description of columns AF\_

* AF\_afr - Alternate allele frequency in samples of African-American ancestry
* AF\_amr - Alternate allele frequency in samples of Latino ancestry
* AF\_asj - Alternate allele frequency in samples of Ashkenazi Jewish ancestry
* AF\_eas - Alternate allele frequency in samples of East Asian ancestry
* AF\_eas\_jpn - Alternate allele frequency in samples of Japanese ancestry
* AF\_eas\_kor - Alternate allele frequency in samples of Korean ancestry
* AF\_eas\_oea - Alternate allele frequency in samples of non-Korean, non-Japanese East Asian ancestry&#x20;
* AF\_fin - Alternate allele frequency in samples of Finnish ancestry
* AF\_nfe - Alternate allele frequency in samples of non-Finnish European ancestry
* AF\_nfe\_bgr - Alternate allele frequency in samples of Bulgarian ancestry
* AF\_nfe\_est - Alternate allele frequency in samples of Estonian ancestry
* AF\_nfe\_nwe - Alternate allele frequency in samples of North-Western European ancestry
* AF\_nfe\_onf - Alternate allele frequency in samples of non-Finnish but otherwise indeterminate European ancestry
* AF\_nfe\_seu - Alternate allele frequency in samples of Southern European ancestry
* AF\_nfe\_swe - Alternate allele frequency in samples of Swedish ancestry
* AF\_oth - Alternate allele frequency in samples of uncertain ancestry
* AF\_popmax - Maximum allele frequency across populations (excluding samples of Ashkenazi, Finnish, and indeterminate ancestry
* AF\_sas - Alternate allele frequency in samples of South Asian ancestry
