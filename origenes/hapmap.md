# HapMap

### Descarga del programa PLINK y los archivos HapMap

{% code title="script_hapmap.sh" %}
```bash
wget https://zzz.bwh.harvard.edu/plink/dist/plink-1.07-x86_64.zip
path_hapmap='/data/hapmap'
mkdir -p $path_hapmap
mv plink-1.07-x86_64.zip $path_hapmap
cd $path_hapmap
unzip plink-1.07-x86_64.zip
rm plink-1.07-x86_64.zip
plinkpath=$path_hapmap/plink-1.07-x86_64

# Directory HapMap
ftppath='ftp://ftp.ncbi.nlm.nih.gov/hapmap/genotypes/2009-01_phaseIII/plink_format'

# File paths
hapmap_readme='00README.txt'
hapmap_doc='phase_3_samples.doc'
hapmap_consensuns_map='hapmap3_r2_b36_fwd.consensus.qc.poly.map.bz2'
hapmap_consensuns_ped='hapmap3_r2_b36_fwd.consensus.qc.poly.ped.bz2'
hapmap_relationships='relationships_w_pops_121708.txt'
hapmap_poly='hapmap3_r2_b36_fwd.qc.poly.tar.bz2'

# File downloads
wget $ftppath/$hapmap_readme -P $path_hapmap
wget $ftppath/$hapmap_doc -P $path_hapmap
wget $ftppath/$hapmap_consensuns_map -P $path_hapmap
wget $ftppath/$hapmap_consensuns_ped -P $path_hapmap
wget $ftppath/$hapmap_relationships -P $path_hapmap
wget $ftppath/$hapmap_poly -P $path_hapmap

# Extraction of information by population
# This command create a new folder /hapmap3_pop
bunzip2 -c $hapmap_poly | tar xvf - 

# Extract SNP consensuns
bunzip2 -c $hapmap_consensuns_map > hapmap_consensuns.map
bunzip2 -c $hapmap_consensuns_ped > hapmap_consensuns.ped

# SNPs - A1 - A2
path_freqs=$path_hapmap/freqs
mkdir -p $path_freqs
cd $plinkpath

# plink
path_map_ped=$path_hapmap/hapmap3_pop/hapmap3_r2_b36_fwd
for table in ASW CEU CHB CHD GIH JPT LWK MEX MKK TSI YRI
do
    ./plink --file $path_map_ped.$table.qc.poly --freq --out $path_freqs/freq_$table --noweb
done
cd $path_freqs
find . -name '*.log' -delete
find . -name '*.hh' -delete
```
{% endcode %}

### SNP de referencia para todas las poblaciones

Una vez realizado este proceso iniciamos la consola de R y ejecutamos el siguiente script para obtener el SNP y alelo de referencia para cada variante:

{% code title="script_hapmap.R" %}
```r
# Load libraries
listOfPackages <- c("tibble", "dplyr", "stringr")

for (i in listOfPackages) {
    if (!i %in% installed.packages()) {
        install.packages(i, dependencies = T)
    }
    require(i, character.only = TRUE)
}

# Create SNP reference TXT
path_freqs <- "/data/hapmap/freqs/"
files_freq <- str_subset(dir(path_freqs), ".frq")

for (table_freq in files_freq) {
    path <- paste0(path_freqs, table_freq)
    print(path)
    name_table <- gsub(".frq", "", table_freq)
    assign(
        name_table,
        read.table(
            file = path, header = T
        )[, 1:5] %>% setNames(c("Chr", "SNP", "A", "B", name_table))
    )
    print(paste0(name_table, " created"))
}

name_tables_freq <- gsub(".frq", "", files_freq)
first_lap <- TRUE
for (name_table in name_tables_freq) {
    print(name_table)
    if (first_lap) {
        table_freqs <- get(name_table)[, 1:4]
        first_lap <- FALSE
    } else {
        current_table <- get(name_table)
        rs_codes <- current_table[[2]]
        rs_codes_freqs <- table_freqs[[2]]
        rs_codes_diff <- setdiff(rs_codes, rs_codes_freqs)
        rs_rows <- current_table %>%
            select(Chr, SNP, A, B) %>%
            filter(SNP %in% rs_codes_diff)
        table_freqs <- rbind(table_freqs, rs_rows)
    }
    print(paste0(name_table, " checked"))
}

table_rs <- table_freqs[c(2, 3)]
write.table(table_rs,
    file = paste0(path_freqs, "rs_ref.txt"),
    row.names = F, col.names = F, quote = F
)
```
{% endcode %}

### Variantes ordenados por SNP/Ref

{% code title="script_hapmap.sh" %}
```bash
cd $path_freqs
find . -name '*.frq' -delete
cd $plinkpath
for table in ASW CEU CHB CHD GIH JPT LWK MEX MKK TSI YRI
do
    ./plink --file $path_map_ped.$table.qc.poly --reference-allele $path_freqs/rs_ref.txt --keep-allele-order --freq --out $path_freqs/freq_$table --noweb
done
cd $path_freqs
find . -name '*.log' -delete
find . -name '*.hh' -delete
```
{% endcode %}

### Generación de RData

{% code title="script_hapmap.R" %}
```r
# Load libraries
listOfPackages <- c("tibble", "dplyr", "stringr")

for (i in listOfPackages) {
    if (!i %in% installed.packages()) {
        install.packages(i, dependencies = T)
    }
    require(i, character.only = TRUE)
}

path_freqs <- "/data/hapmap/freqs/"
files_freq <- str_subset(dir(path_freqs), ".frq")

first_lap <- TRUE
for (table_freq in files_freq) {
  path <- paste0(path_freqs, table_freq)
  print(path)
  name_table <- gsub(".frq", "", table_freq)
  assign(
    name_table,
    read.table(
      file = path, header = T
    )[, 1:5] %>% setNames(c("Chr", "SNP", "A", "B", name_table))
  )
  print(paste0(name_table, " fue cargada con éxito"))
  if (first_lap) {
    table_freqs <- get(name_table)
    first_lap <- FALSE
  } else {
    table_freqs <- full_join(table_freqs, get(name_table),
      by = c("Chr", "SNP", "A", "B")
    )
  }
  print("Proceso de unión finalizado")
}

table_freqs[is.na(table_freqs)] <- 0
table_freqs$Chr <- paste0("chr", table_freqs$Chr)
freq_max <- apply(table_freqs[5:15], 1, max)
freq_min <- apply(table_freqs[5:15], 1, min)
table_freqs <- cbind(table_freqs, freq_max, freq_min)
hapmap_freqs <- as_tibble(table_freqs)

path_hapmap <- "/data/MutationMiningData/Populations/HapMap/"
dir.create(path_hapmap, recursive = TRUE)
save(hapmap_freqs, file = paste0(path_hapmap, "hapmap_freqs.RData"))
```
{% endcode %}
