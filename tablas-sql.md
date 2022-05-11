# Tablas SQL

En este apartado se explicará todo lo necesario para añadir las Bases de Datos requeridas para la ejecución correcta de la aplicación. Primero debemos entrar en MySQL y ejecutar las siguientes sentencias:

```sql
CREATE USER 'MutationMining'@'localhost' IDENTIFIED BY 'MutationMining2021!';
DROP DATABASE IF EXISTS `MutationMining`;
CREATE DATABASE `MutationMining`;
GRANT ALL PRIVILEGES ON MutationMining.* TO 'MutationMining'@'localhost' WITH GRANT OPTION;
USE `MutationMining`;
```

Creación de tablas SQL:

```sql
-- TABLE sessions
DROP TABLE IF EXISTS `sessions`;
CREATE TABLE `sessions` (
  `id` int NOT NULL AUTO_INCREMENT,
  `project` int NOT NULL,
  `user_id` int NOT NULL,
  `time` int NOT NULL,
  `open` tinyint NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM DEFAULT CHARSET = utf8;
-- TABLE projects
DROP TABLE IF EXISTS `projects`;
CREATE TABLE `projects` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `time` int NOT NULL,
  `name` tinytext NOT NULL,
  `assembly` varchar(31) NOT NULL,
  `vcf_path` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM DEFAULT CHARSET = utf8;
-- TABLE samples
DROP TABLE IF EXISTS `samples`;
CREATE TABLE `samples` (
  `id` int NOT NULL AUTO_INCREMENT,
  `project` int NOT NULL,
  `name` tinytext NOT NULL,
  `bam_path` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM DEFAULT CHARSET = utf8;
-- TABLE filters
DROP TABLE IF EXISTS `filters`;
CREATE TABLE `filters` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `project_id` int NOT NULL,
  `time` int NOT NULL,
  `name` tinytext NOT NULL,
  `filter_name` tinytext NOT NULL,
  `num_samples` int NOT NULL,
  `name_samples` text NOT NULL,
  `num_variants` int NOT NULL,
  `num_filters` int NOT NULL,
  `filters` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM DEFAULT CHARSET = utf8;
-- TABLE filters_template
DROP TABLE IF EXISTS `filters_template`;
CREATE TABLE `filters_template` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `project_id` int NOT NULL,
  `name` tinytext NOT NULL,
  `filters` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM DEFAULT CHARSET = utf8;
-- TABLE users
DROP TABLE IF EXISTS `users`;
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `email` varchar(50) NOT NULL,
  `confirm` varchar(12) NOT NULL,
  `type` int(11) NOT NULL,
  `password` char(60) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `email_idx` (`email`)
) ENGINE = MyISAM AUTO_INCREMENT = 3 DEFAULT CHARSET = utf8;
```
