# Cargar nuevo Proyecto

Para añadir un nuevo proyecto a la página web debemos copiar la carpeta del proyecto, cuyo contenido se divide en diferentes archivos `bam`, `bai`, `vcf` y el fichero `RData`, dentro de la siguiente ruta

{% hint style="info" %}
/data/MutationMiningData/Projects
{% endhint %}

A continuación, haremos uso del fichero R que se encuentra en la siguiente ruta:

{% hint style="info" %}
/data/MutationMining/RScript/NewProject/MutationMiningLoadInitial.R
{% endhint %}

El fichero contiene en su interior varios apartados

### Sentencias Iniciales (EJECUTAR AL INICIO SIN MODIFICAR)

{% code title="MutationMiningLoadInitial.R" %}
```r
# Iniciar R, ruta:
# /data/MutationMining/bin/R-4.0.4/bin/R

# Rutas MutationMining
ROOT_PATH <- "/data/MutationMining/RScript/NewProject/"
RLOG_PATH <- "/data/MutationMining/RScript/log/"
PROJECT_PATH <- "/data/MutationMining/projects"
LINK_PATH <- "/data/MutationMining/web/links"
GENOMES_PATH <- "/data/MutationMining/web/genomes"
REPORT_PATH <- "/data/MutationMiningData/GenomeDDBB/"
PATH_CLINVAR <- paste0(REPORT_PATH, "CLINVAR/")
PATH_CLINICAL_GENOME <- paste0(REPORT_PATH, "CLINICALGENOME/")
PATH_PHARMGKB <- paste0(REPORT_PATH, "PHARMGKB/")
PATH_GWAS_CATALOG <- paste0(REPORT_PATH, "GWASCatalog/")
POPS_PATH <- "/data/MutationMiningData/Populations/"
PATH_1000GENOMES <- paste0(POPS_PATH, "1000genomes/")
PATH_HAPMAP <- paste0(POPS_PATH, "HapMap/")

# Archivo R necesario con las siguientes funciones:
# Crear/eliminar proyectos, insertar/editar/borrar en MySQL
source(paste0(ROOT_PATH, "MutationMiningLoadProject.R"))

# Crear un nuevo usuario
# add_user(name, email, passwd)
```
{% endcode %}

### Crear un proyecto nuevo (EJECUCIÓN NO OBLIGATORIA)

#### Variables necesarias (Modificar según proyecto)

{% code title="MutationMiningLoadInitial.R" %}
```r
email <- "example@usal.es"
project_dir <- "/data/MutationMiningData/Projects/PRJEB41088_AngelmanTrio"
name_project <- "PRJEB41088_AngelmanTrio"

# Declarar para proyectos humanos
clinvar_RData <- "clinvar_2022-05-23.RData"
clinvar_version <- "2022-05-18"
clinical_genome_RData <- "clinical_genome_2022-05-23.RData"
clinical_genome_version <- "2021-02-15"
pharmgkb_RData <- "pharmgkb_2022-05-23.RData"
pharmgkb_version <- "2022-05-05"
gwas_catalog_RData <- "gwas_catalog_BSgenome_2022-05-25.RData"
gwas_catalog_version <- "2022-04-07"
genomes1000_version <- "2013-05-02"
hapmap_version <- "2009-01_phaseIII"
versions <- rbind(
  clinvar_version, clinical_genome_version, pharmgkb_version,
  gwas_catalog_version, genomes1000_version, hapmap_version
)
description <- "Descripción sobre cómo fue generado el proyecto"
```
{% endcode %}

#### Carga inicial y registro del proyecto en la BBDD (Sólo ejecutar)

{% code title="MutationMiningLoadInitial.R" %}
```r
# Obtenemos el id del usuario
user_id <- get_user(email)

if (!user_id) {
  message(paste0(
    "El usuario con email: ", email, " no existe. ",
    "Use la función add_user para añadir un nuevo usuario. ",
    'Ejemplo: \nadd_user("nombre", "correo@email.es", "contraseña")'
  ))
} else {
  # Comprobamos si el nombre del proyecto ya existe para este usuario
  exist_project_name(user_id, name_project)
  # Rutas de archivos necesarios para crear un proyecto incluidos en project_dir
  rdata <- dir(project_dir, pattern = "*.RData$")
  vcf <- dir(project_dir, pattern = "*.vcf.gz$")
  bams <- dir(project_dir, pattern = "*.bam$")
  filetable <- data.frame(
    type = c("RData", "vcf", rep("bam", length(bams))),
    path = paste0(project_dir, "/", c(rdata, vcf, bams))
  )
  add_project_files(name_project, user_id, filetable, description, "hg38")
  message("Datos registrados en la BBDD")
}
```
{% endcode %}

#### Añadir reportes por muestra y predicciones de población (Sólo ejecutar)

{% code title="MutationMiningLoadInitial.R" %}
```r
# Archivo R necesario para crear las tablas REPORT & POPS tsv
source(paste0(ROOT_PATH, "MutationMiningLoadDB.R"))

# Crear POPS & Report TSV & links simbolicos
if (exists("user_id") & exists("name_project") &
  exists("project_dir") & exists("versions")) {
  project_id <- get_project(user_id, name_project)
  if (exists("project_id")) {
    create_tsv(user_id, project_id, project_dir, versions)
  }
} else {
  stop("¿Has inicializado las variables de la sección \"Variables necesarias\"?")
}
```
{% endcode %}

### Otras funciones

#### Modificar los valores en función de lo parámetros requeridos

{% code title="MutationMiningLoadInitial.R" %}
```r
# Asociar proyectos a un usuario
associate_project("email_from", "name_project", "email_to")

# Borrar un proyecto
delete_project("email", "name_project")

# Borrar todos los  proyectos asociados a un usuario ("AllProjects" not change)
delete_project("email", "AllProjects")

# Borrar un usuario y todos los proyectos asociados a este
delete_user("email")

# Borrar archivos de errores en R
delete_log()
```
{% endcode %}
