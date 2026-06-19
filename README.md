Introducción y Contexto:  
Este proyecto tiene como objetivo avanzar desde una infraestructura base hacia un entorno de orquestación productivo, escalable y automatizado. Se implementó una arquitectura de microservicios reales (Venta y Despachos en spring boot, y un Frontend en React) gestionados mediante contenedores Docker. Todo el ciclo de vida del software fue automatizado utilizando prácticas de integración y Despliegue Continuo (CI/CD) para garantizar la disponibilidad y resiliencia del sistema.

Arquitectura y Configuración del Clúster:  
Para la orquestación de contenedores, se seleccionó AWS Elastic Container Service (ECS) operando bajo el modelo de cómputo serverless Fargate. Esta decisión arquitectónica se fundamenta en la necesidad de reducir la carga operativa de administrar servidores subyacentes, permitiendo un enfoque total en la entrega de valor.

Redes y Seguridad: La arquitectura se despliega sobre una Virtual Private Cloud (VPC) con múltiples Subredes Privadas (para las tareas de backend y la base de datos RDS) y Públicas (para el balanceador de carga), garantizando alta disponibilidad multi-AZ. Se configuraron Security Groups estrictos que solo permiten el tráfico desde el balanceador hacia los contenedores, y desde los contenedores hacia la base de datos MySQL.


Roles IAM: Se configuraron el ECS Task Execution Role (para permitir a ECS descargar imágenes de ECR y escribir logs en CloudWatch) y el Task Role para permisos a nivel de contenedor.

Despliegue de Servicios Frontend y Backend:  
Los microservicios fueron empaquetados en imágenes inmutables y almacenados en Amazon Elastic Container Registry (ECR).

Task Definitions y Variables de Entorno: Se crearon definiciones de tareas para “ventas-task” y “despachos-task”. Un logro crítico fue la inyección dinámica de Variables de Entorno (DB_ENDPOINT, DB_PORT, DB_NAME. DB_USERNAME, DB_PASSWORD) directamente desde la configuración de la tarea. Esto evita exponer credenciales en el código fuente de Spring Boot.


Balanceo de Carga (ALB): Se aprovisionó un Application Load Balancer (ALB) público que enruta el tráfico hacia el clúster. Se configuraron Target Groups específicos para los puertos 8081 (Ventas) y 8082 (Despachos), asegurando una comunicación fluida entre el Frontend y el Backend.

Configuración de Autoscaling:  
Para garantizar la escalabilidad y disponibilidad, se implementó una política de Target Tracking Auto Scaling en los servicios de ECS.

Métrica objetivo: Uso promedio de CPU al 70%.


Justificación: Se eligió la métrica de CPU (en lugar de memoria) debido a que las aplicaciones Java/Spring Boot tienden a consumir un bloque fijo de memoria RAM, pero presentan picos de procesamiento de CPU al recibir ráfagas de peticiones concurrentes del balanceador. Si la demanda supera el 70% sostenido, ECS aprovisionará nuevas réplicas (Tasks) de forma automática, y las destruirá cuando el tráfico disminuya, optimizando así los costos.

Pipeline CI/CD y Gestión de Secrets: 

Se construyó un pipeline de CI/CD completamente automatizado utilizando GitHub Actions. El flujo se dispara automáticamente con cada push a la rama principal e incluye las siguientes etapas:

Build: Compilación del código fuente y creación de la imagen Docker etiquetada con el hash del commit ($GITHUB_SHA).


Push: Autenticación y subida de la imagen a Amazon ECR.


Deploy: Actualización del servicio ECS mediante la estrategia Rolling Update sin interrupción del servicio, forzando un nuevo despliegue con la última imagen (--force-new-deployment).


Gestión de Credenciales (Secrets): Las credenciales de acceso a AWS (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY) nunca se expusieron en el código. Fueron almacenadas de manera cifrada en GitHub Secrets, cumpliendo con los estándares de seguridad de la industria y garantizando que las llaves permanezcan protegidas.









Validación Funcional, Logs y Métricas:

Se realizó una validación completa (Prueba de Humo) del ecosistema:

Comunicación Exitosa: Se demostró que el Frontend (React) envía peticiones POST al ALB, el cual enruta el tráfico a los contenedores de Spring Boot, y estos insertan exitosamente los registros en la base de datos RDS.


Observabilidad: El registro de eventos y errores fue centralizado en Amazon CloudWatch, lo que permitió auditar el arranque de los contenedores Tomcat (Java) y diagnosticar problemas en tiempo real.


Métricas del Pipeline: Los despliegues se ejecutan de manera consistente, manteniendo un registro de éxito continuo y asegurando la recuperación automática post-deploy.

Análisis Crítico y Lecciones Aprendidas:

Durante el desarrollo e implementación del entorno orquestado, se presentaron diferentes desafíos técnicos que nos permitieron el aprendizaje y mejora de nuestras habilidades y conocimientos:

Desafío de Inyección de propiedades en Spring Boot:


Problema: Los contenedores colapsaban al inicio porque Spring Boot leía las variables literales cómo ${DB_ENDPOINT} en lugar de los valores reales.


Solución: Se corrigieron las Task Definitions en AWS, inyectando correctamente las variables de entorno, y asegurando el versionamiento creando nuevas revisiones de las tareas.


Problemas con Health Checks y Errores 502/503:


Problema: El ALB marcaba los contenedores como Unhealthy y devolvía errores 502 Bad Gateway debido a conflictos en los puertos (8081 vs 8082) y rutas inexistentes (/).


Solución: Se unificaron los puertos en los archivos application.properties, Dockerfiles y Target Groups. Además, se ajustó el Path del Health Check del balanceador para apuntar a rutas válidas, restableciendo el tráfico.


Bloqueo de Autenticación de MySQL (Public Key Retrieval):


Problema: Las versiones modernas de MySQL rechazaban la conexión cifrada desde java.


Solución: Se actualizó la cadena de conexión en el backend agregando el parámetro “allowPublicKeyRetrieval=true”, permitiendo la conexión segura sin exponer datos.

Proyección: El entorno actual para Innovatech Chile está altamente optimizado. Como proyección, se recomienda la implementación futura de Docker Layer Caching en GitHub Actions para reducir los tiempos de Build, y la integración de pruebas unitarias automatizadas previas al despliegue para asegurar la calidad del código.
