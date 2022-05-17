# VCF Project

En este apartado se describe los aspectos a tener en cuenta a la hora de generar nuestro archivo VCF:&#x20;

* Evitar comillas en nombres de muestras (samples)
* Ejecutar el siguiente código R para generar el objeto RData

```
## Modificar < mutations > por el nombre del objeto RData

load(stringr)

# Cambiar caracter hexadecimal a coma
colnames_to_change <- str_subset(colnames(mutations), "^CLN")
for (col in colnames_to_change) {
    mutations[[col]] <- str_replace_all(mutations[[col]], "\\\\x2c_", ",")
    mutations[[col]] <- str_replace_all(mutations[[col]], "\\\\x2c", ",")
}

# FUNCION: Cambiar tipo de columnas AF y DP a numeric
colnames_to_change <- str_subset(colnames(mutations), "^AF_|_DP$")
for (col in colnames_to_change) {
    mutations[[col]] <- as.numeric(mutations[[col]])
}
```

* El nombre del objeto contenido en el archivo .RData debe ser igual al nombre del fichero
