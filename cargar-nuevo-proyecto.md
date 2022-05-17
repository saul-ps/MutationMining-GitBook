# Cargar nuevo Proyecto

Para subir un nuevo proyecto a la aplicación web debemos hacer uso del fichero R que se encuentra en la siguiente ruta:

{% hint style="info" %}
/data/MutationMining/RScript/NewProject/MutationMiningLoadInitial.R
{% endhint %}

El fichero contiene en su interior varios apartados

#### 1. SENTENCIAS INICIALES

{% code title="MutationMiningLoadInitial.R" %}
```r
# Iniciar R, ruta:
# /data/MutationMining/bin/R-4.0.4/bin/R

# Rutas
ROOT_PATH <- "/data/MutationMining/RScript/NewProject/"
RLOG_PATH <- "/data/MutationMining/RScript/log/"
PROJECT_PATH <- "/data/MutationMining/projects"
LINK_PATH <- "/data/MutationMining/web/links"
GENOMES_PATH <- "/data/MutationMining/web/genomes"
REPORT_PATH <- "/data/MutationMiningData/GenomeDDBB/"
POPS_PATH <- "/data/MutationMiningData/Populations/"

# Variables a usar (example)
email <- "example@usal.es"
project_dir <- "/data/MutationMiningData/Projects/PRJEB41088_AngelmanTrio"
name_project <- "PRJEB41088_AngelmanTrio"

# Archivo R necesario con las siguientes funciones:
# Crear/eliminar proyectos, insertar/editar/borrar en MySQL
source(paste0(ROOT_PATH, "MutationMiningLoadProject.R"))

# Obtenemos el id del usuario
user_id <- get_user(email)
```
{% endcode %}

#### 2. CREAR NUEVO PROYECTO

{% code title="MutationMiningLoadInitial.R" %}
```r
if (!user_id) {
  message(paste0(
    "El usuario con email: ", email, " no existe. ",
    "Use la función add_user para añadir un nuevo usuario. ",
    'Ejemplo: \nadd_user <- function("Nombre", "correo@email.es", "Contraseña")'
  ))

  # Crear un nuevo usuario
  # add_user(name, email, passwd)
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
  add_project_files(name_project, user_id, filetable, "hg38")
  message("Datos registrados en la BBDD")
}MutationMiningLoadInitial.R
```
{% endcode %}

#### 3. CREAR FICHEROS DE ENLACE CON LAS BASES DE DATOS GENÓMICAS & POBLACIONALES

{% code title="MutationMiningLoadInitial.R" %}
```r
PATH_CLINVAR <-
  paste0(REPORT_PATH, "CLINVAR/clinvar.RData")
PATH_CLINICAL_GENOME <-
  paste0(REPORT_PATH, "CLINICALGENOME/clinical_genome.RData")
PATH_PHARMGKB <-
  paste0(REPORT_PATH, "PHARMGKB/pharmgkb.RData")
PATH_GWAS_CATALOG <-
  paste0(REPORT_PATH, "GWASCatalog/gwas_catalog_BSgenome.RData")
PATH_1000GENOMES <-
  paste0(POPS_PATH, "1000genomes/")
PATH_HAPMAP <- 
  paste0(POPS_PATH, "HapMap/")

# Archivo R necesario para crear las tablas REPORT & POPS tsv
source(paste0(ROOT_PATH, "MutationMiningLoadDB.R"))

# Crear POPS & Report TSV & links simbolicos
project_id <- get_project(user_id, name_project)
if (exists("user_id") & exists("project_id") & exists("project_dir")) {
  create_tsv(user_id, project_id, project_dir)
} else {
  stop("Error al ejecutar la creación de Report TSV, comprueba que
   las variables user_id, project_id & project_dir estén creadas")
}
```
{% endcode %}

#### OTRAS FUNCIONES

{% code title="MutationMiningLoadInitial.R" %}
```r
# Borrar un proyecto
delete_project(email, name_project)
# Borrar todos los proyectos asociados a un usuario
delete_project(email, "AllProjects")
# Borrar un usuario y todos los proyectos asociados a este
delete_user(email)

# Borrar archivos de errores en R
delete_log()
```
{% endcode %}
