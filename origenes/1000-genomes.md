# 1000 Genomes

### Descarga de archivos y filtrado de frecuencias por población

Para la creación de los directorios de trabajo y la descarga de la base de datos necesitaremos ejecutar el script "index.sh" que a su vez ejecutara en segundo plano los otros scripts descritos en esta sección (debemos guardar todos los ficheros en el mismo directorio de ejecución):

{% code title="index.sh" %}
```bash
urlgenomes='https://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/20130502/'
temp='/data/1000genomes/results'
srcpath='/data/1000genomes/scripts/'
popscript=$srcpath'get_populations.R'
mergescript=$srcpath'merge.py'
chromscript=$srcpath'eachChrome.sh'

rm -R $temp
mkdir $temp
cd $temp

Rscript $popscript

count=0
for i in $(
    seq 1 22
); do
    count=$(($count + 1))
    echo $count' '$urlgenomes'ALL.chr'$i'.phase3_shapeit2_mvncall_integrated_v5b.20130502.genotypes.vcf.gz' >>fileuri.txt
done

cat fileuri.txt | parallel -j 4 -k bash $chromscript {}

rm fileuri.txt
```
{% endcode %}

{% code title="get_populations.R" %}
```r
# Nombre de muestras por población
library(openxlsx)

download.file(
  "http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/working/20130606_sample_info/20130606_sample_info.xlsx",
  "20130606_sample_info.xlsx"
)

sample_info <- read.xlsx("20130606_sample_info.xlsx", sheet = 1)
sample_info <- sample_info[, c(1, 3)]

populations <- c(
  "ACB", "ASW", "ESN", "GWD", "LWK", "MSL", "YRI", "CLM", "MXL", "PEL",
  "PUR", "CDX", "CHB", "CHS", "JPT", "KHV", "CEU", "FIN", "GBR", "IBS",
  "TSI", "BEB", "GIH", "ITU", "PJL", "STU"
)
superpop <- c("AFR", "AMR", "EAS", "EUR", "SAS")

sample_info$superpop <- NA
sample_info[sample_info[, 2] %in% populations[1:7], 3] <- "AFR"
sample_info[sample_info[, 2] %in% populations[8:11], 3] <- "AMR"
sample_info[sample_info[, 2] %in% populations[12:16], 3] <- "EAS"
sample_info[sample_info[, 2] %in% populations[17:21], 3] <- "EUR"
sample_info[sample_info[, 2] %in% populations[22:26], 3] <- "SAS"

dir.create("population_sample")

write.table(matrix(c(superpop, populations), ncol = 1),
  "population_sample/populations.txt",
  quote = F, row.names = F, col.names = F
)

for (pop in superpop) {
  write.table(matrix(sample_info[sample_info[, 3] == pop, 1], ncol = 1),
    paste0("population_sample/", pop, "_samples.txt"),
    quote = F, row.names = F, col.names = F
  )
}

for (pop in populations) {
  write.table(matrix(sample_info[sample_info[, 2] == pop, 1], ncol = 1),
    paste0("population_sample/", pop, "_samples.txt"),
    quote = F, row.names = F, col.names = F
  )
}
```
{% endcode %}

{% code title="eachChrome.sh" %}
```bash
# Descarga de ficheros vcf y tbi, filtrado de variantes (vcftools)
#!/bin/bash
srcpath='/data/1000genomes/scripts/'
mergescript=$srcpath'merge.py'

count=$(echo $1 | cut -d' ' -f1)
fileuri=$(echo $1 | cut -d' ' -f2)

wget $fileuri
wget $fileuri'.tbi'
file=$(basename $fileuri)
out=$count'.ALL'

vcftools --gzvcf $file --out $out --counts
# CHROM	  POS	    N_ALLELES	  N_CHR	  {ALLELE:COUNT}
# 1	  10177	    2	          5008	  A:2878 AC:2130

vcftools --gzvcf $file --out $out --hardy
# CHR	  POS	    OBS(HOM1/HET/HOM2)	E(HOM1/HET/HOM2)	      ChiSq_HWE	    P_HWE	        P_HET_DEFICIT	    P_HET_EXCESS
# 1	  10177	    694/1490/320        826.97/1224.07/452.97	      1.181840e+02  8.743336e-28	1.000000e+00	    5.165337e-28

python3 $mergescript $count ALL > python_error_1pop.log 2>&1
# Chr Pos   Population  A   Afreq   An    B   Bfreq   Bn    AAfreq  ABfreq  BBfreq  AAn   ABn   BBn
# 1   10177 ALL	        A   0.5747  2878  AC  0.4253  2130  0.2772  0.595   0.1278  694	  1490	320

rm $out.hwe
rm $out.log
rm $out.frq.count
for pop in $(cat 'population_sample/populations.txt'); do
  out=$count'.'$pop
  vcftools --gzvcf $file --out $out --keep 'population_sample/'$pop'_samples.txt' --counts
  vcftools --gzvcf $file --out $out --keep 'population_sample/'$pop'_samples.txt' --hardy
  python3 $mergescript $count $pop > python_error_2pop.log 2>&1
  rm $out.hwe
  rm $out.log
  rm $out.frq.count
done
rm $file
rm $file.tbi
rm $file.vcfidx
```
{% endcode %}

{% code title="merge.py" %}
```python
# Generación de frecuencias por población (txt)
import sys

chromosome = sys.argv[1]
population = sys.argv[2]

countfile = open(chromosome+'.'+population+'.frq.count')
head = countfile.readline()
# CHROM	  POS	    N_ALLELES	  N_CHR	  {ALLELE:COUNT}
# 1	  10177	    2	          5008	   A:2878 AC:2130

genofile = open(chromosome+'.'+population+'.hwe')
head = genofile.readline()
# CHR	  POS	    OBS(HOM1/HET/HOM2)	E(HOM1/HET/HOM2)	      ChiSq_HWE	    P_HWE	        P_HET_DEFICIT	    P_HET_EXCESS
# 1	  10177	    694/1490/320        826.97/1224.07/452.97	      1.181840e+02  8.743336e-28	1.000000e+00	    5.165337e-28

output = open(chromosome+'.'+population+'.tmp.txt','w')

for line in countfile:
  fields = line.rstrip().split("\t")
  if len(fields) < 4:
    continue
  pos = fields[1]
  nchr = int(fields[3])
  if fields[2]=='2':
    geno = genofile.readline().rstrip().split("\t")
    if geno[0]=='' or nchr==0:
      geno = False
    else:
      geno = geno[2].split("/")
  else:
    geno = False
  alleles = []
  for i in fields[4:]:
    # Error --> A:4973	<INS:ME:ALU>:35
    aux = i.split(":")
    if nchr!=0:
      freq = str(round(int(aux[len(aux)-1])/nchr,4))
    else:
      freq = '0'
    alleles.append([aux[0],freq,aux[len(aux)-1]])
  if nchr==0:
    alleles[0][1] = '1'
  if geno:
    output.write('\t'.join(['\t'.join([chromosome,pos,population]),'\t'.join(alleles[0]),'\t'.join(alleles[1]),'\t'.join(map(lambda x: str(round(int(x)/(nchr/2),4)),geno)),'\t'.join(geno)])+'\n')
  else:
    for allele in alleles[1:]:
      output.write('\t'.join(['\t'.join([chromosome,pos,population]),'\t'.join(alleles[0]),'\t'.join(allele),'\\N','\\N','\\N','\\N','\\N','\\N'])+'\n')

countfile.close()
genofile.close()
output.close()
```
{% endcode %}

### Creación final RData por poblaciones y superpoblaciones

Antes de abrir la consola de R y ejecutar el script que se describe debajo de este párrafo necesitamos descargar el archivo[ hg19ToHg38.over.chain.gz](https://hgdownload.soe.ucsc.edu/gbdb/hg19/liftOver/) y descomprimirlo en la ruta "/data/1000genomes/liftover":

{% code title="script_1000genomes.R" %}
```r
# Load libraries
library(rtracklayer)
library(tibble)
library(dplyr)
library(stringr)

# Paths
path_frequencies <- "/data/1000genomes/results/"
path_liftover <- "/data/1000genomes/liftover/"

# Variables
populations <- c(
    "AFR", "ACB", "ASW", "ESN", "GWD", "LWK", "MSL", "YRI",
    "AMR", "CLM", "MXL", "PEL", "PUR",
    "SAS", "BEB", "GIH", "ITU", "PJL", "STU",
    "EAS", "CDX", "CHB", "CHS", "JPT", "KHV",
    "EUR", "CEU", "FIN", "GBR", "IBS", "TSI"
)
populations_first_level <- c("AFR", "AMR", "SAS", "EAS", "EUR")
populations_second_level <- setdiff(populations, populations_first_level)

# Generando RData GRCh37 & GRCh38
for (chr in 1:22) {
    for (population in populations) {
        name_table <- paste0(chr, ".", population, ".tmp.txt")
        print(paste0("Leyendo ", name_table))
        assign(
            "Bfreq_Pop",
            read.table(
                file = paste0(path_frequencies, name_table),
                header = F, col.names = c(
                    "Chr", "Pos", "Population", "A", "Afreq",
                    "An", "B", "Bfreq", "Bn", "AAfreq", "ABfreq",
                    "BBfreq", "AAn", "ABn", "BBn"
                )
            )
        )
        print(paste0(name_table, " lectura terminada. Union de datos..."))
        if (!exists("table_freqs")) {
            table_freqs <- rbind(Bfreq_Pop[, c(1, 2, 4, 7, 8)])
            rm(Bfreq_Pop)
            print(paste0("Proceso finalizado (", name_table, ")"))
        } else {
            table_freqs <- cbind(table_freqs, Bfreq = Bfreq_Pop[["Bfreq"]])
            rm(Bfreq_Pop)
            print(paste0("Proceso finalizado (", name_table, ")"))
        }
    }

    # Creando chr_freqs_GRCh37.RData
    chr_freqs_GRCh37 <- paste0("chr", chr, "_freqs_GRCh37")
    print(paste0("Preparando estructura de la tabla ", chr_freqs_GRCh37))
    colnames(table_freqs) <- c(
        "Chr", "Start", "A", "B", paste0("Bfreq_", populations)
    )
    table_freqs[is.na(table_freqs)] <- 0
    table_freqs <- as_tibble(table_freqs)
    table_freqs[["Chr"]] <- paste0("chr", table_freqs[["Chr"]])
    print(paste0("Creando ", chr_freqs_GRCh37, ".RData"))
    assign(chr_freqs_GRCh37, table_freqs)
    save(
        list = chr_freqs_GRCh37,
        file = paste0(path_frequencies, chr_freqs_GRCh37, ".RData")
    )
    rm(list = chr_freqs_GRCh37)

    # Creando chr_freqs_GRCh38.RData
    print("Convert data GRCh37 to GRCh38")
    chr_freqs_GRCh38 <- paste0("chr", chr, "_freqs_GRCh38")
    chainObject <- import.chain(paste0(path_liftover, "hg19ToHg38.over.chain"))
    grObject <- GRanges(
        seqnames = table_freqs[["Chr"]],
        ranges = IRanges(table_freqs[["Start"]], width = 1)
    )
    liftoverdata <- liftOver(grObject, chainObject)
    crossref <- as.data.frame(liftoverdata@partitioning)
    liftoverdata <- as.data.frame(liftoverdata)[, c(1, 3, 4)]
    table_freqs[, "Chr"] <- liftoverdata[crossref$start, 2]
    table_freqs[, "Start"] <- liftoverdata[crossref$start, 3]
    table_freqs <- table_freqs[crossref[, 3] != 0, ]
    table_freqs[["Chr"]] <- as.character(table_freqs[["Chr"]])
    print(paste0("Creando ", chr_freqs_GRCh38, ".RData"))
    assign(chr_freqs_GRCh38, table_freqs)
    save(
        list = chr_freqs_GRCh38,
        file = paste0(path_frequencies, chr_freqs_GRCh38, ".RData")
    )
    rm(list = chr_freqs_GRCh38)
    # Clean data
    rm(table_freqs)
    rm(chainObject)
    rm(grObject)
    rm(liftoverdata)
    rm(crossref)
    print(paste0("Lectura del chr", chr, " finalizada."))
}


# GRCh38 by population
grch38_by_population <- function(populations_level, level) {
    print(paste0("Creación RData para las ", level, ": ", populations_level))
    col_freqs <- 5
    Bfreq_cols <- paste0("Bfreq_", populations_level)
    for (chr in 1:22) {
        table_search <- paste0("chr", chr, "_freqs_GRCh38")
        print(paste0("Leyendo ", table_search, ".RData"))
        load(paste0(path_frequencies, table_search, ".RData"))
        assign("table", get(table_search))
        rm(list = table_search)
        table <- table[c(colnames(table)[1:(col_freqs - 1)], Bfreq_cols)]
        Bfreq_max <- apply(table[col_freqs:ncol(table)], 1, max)
        Bfreq_min <- apply(table[col_freqs:ncol(table)], 1, min)
        table <- cbind(table, Bfreq_max, Bfreq_min)
        table[["Chr"]] <- str_replace_all(table[["Chr"]], "chr", "")
        table <- as_tibble(table)
        table_level <- paste0("chr", chr, "_", level, "_freqs_GRCh38")
        assign(table_level, table)
        save(
            list = table_level,
            file = paste0(path_frequencies, table_level, ".RData")
        )
        print(paste0(
            "Proceso finalizado: ", table_level, ".RData ha sido generado"
        ))
        rm(list = table_level)
        rm(table)
    }
}

grch38_by_population(populations_first_level, "super_populations")
grch38_by_population(populations_second_level, "populations")
```
{% endcode %}

Este script generará un archivo .RData por cada cromosoma con el sufijo "_poblations\_freqs\_GRCh38" y "super\_poblations\_freqs\_GRCh38_", cortamos estos achivos y los llevamos a la ruta

{% hint style="info" %}
/data/MutationMiningData/Populations/1000genomes
{% endhint %}
