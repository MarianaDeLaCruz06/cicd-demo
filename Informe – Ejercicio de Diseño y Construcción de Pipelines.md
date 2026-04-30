# Informe de Avance — Ejercicio de Diseño y Construcción de Pipelines

**Mariana De La Cruz** — A00399618  
**Valentina Gomez** — A00389790



## 1. Contexto General

Se tomó como base el repositorio sugerido:  
[https://github.com/helderklemp/cicd-demo](https://github.com/helderklemp/cicd-demo)

Se realizó un fork hacia una cuenta propia de GitHub y posteriormente se configuró Jenkins en Docker sobre entorno local Windows, usando Visual Studio Code como editor principal:  
[https://github.com/MarianaDeLaCruz06/cicd-demo](https://github.com/MarianaDeLaCruz06/cicd-demo)


## 2. Preparación del Entorno

### 2.1 Instalación y ejecución de Jenkins en Docker

Se levantó una instancia local de Jenkins mediante contenedor Docker usando el comando indicado en la guía. Posteriormente se ajustó la infraestructura creando una imagen personalizada de Jenkins con herramientas adicionales necesarias para el laboratorio.

### 2.2 Plugins Instalados

Durante la configuración inicial se instalaron los plugins sugeridos por el ejercicio:

- **Git**
- **Pipeline**
- **Docker**
- **SonarQube Scanner**

Esto permitió la integración entre el repositorio remoto, la ejecución de pipelines declarativos y el uso de contenedores.

---

## 3. Definición del Pipeline

### 3.1 Creación del Jenkinsfile

Se reemplazó el Jenkinsfile original del proyecto por uno nuevo adaptado al alcance del ejercicio, más simple y alineado con los requerimientos del documento.

### 3.2 Etapas del pipeline implementado

| Etapa | Descripción |
|---|---|
| `Checkout scm` | Obtención automática del código fuente desde GitHub |
| `Build` | Compilación de la aplicación con Maven: `mvn clean package -DskipTests` |
| `Test` | Ejecución de pruebas dentro del pipeline: `mvn test -DskipTests` |
| `Docker Build` | Construcción automática de imagen Docker: `docker build -t mi-app:latest .` |
| `Static Analysis` | Integración con SonarQube para análisis estático y evaluación de métricas de calidad |
| `Security Scan` | Evaluación de vulnerabilidades con Trivy |
| `Deploy` | Despliegue automático del contenedor en la rama principal del repositorio |

### 3.3 Integración con Jenkins

El job fue configurado como **Pipeline script from SCM**.

---

## 4. Ajustes Técnicos Realizados Durante la Construcción

### 4.1 Corrección de Maven en Jenkins

El contenedor base de Jenkins no incluía Maven. El Dockerfile del proyecto fue ajustado de la siguiente forma:

```dockerfile
FROM eclipse-temurin:17-jre-alpine

VOLUME /tmp

COPY target/cicd-demo-*.jar app.jar

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

Para solucionarlo se creó una imagen personalizada basada en Jenkins LTS que incorporaba Maven y Docker CLI.

La configuración del job en Jenkins quedó de la siguiente manera:

![!\[\]\[image1\]](images/image1.png)


### 4.2 Permisos para Docker dentro de Jenkins

Se configuró el acceso al Docker daemon montando el socket:

```
/var/run/docker.sock
```

Y se ajustaron los permisos para permitir la construcción de imágenes desde Jenkins.

### 4.3 Corrección del Dockerfile del proyecto

La imagen base original `openjdk:12-alpine` se actualizó a `eclipse-temurin:17-jre-alpine-3.20`, con lo cual se logró construir exitosamente la imagen del proyecto desde el pipeline.

---

## 5. Estado del Pipeline

El pipeline fue ejecutado correctamente con todas sus etapas. A continuación se muestra el grafo de ejecución y el detalle de los tiempos por etapa:

![!\[\]\[image2\]](images/image2.png)



---

## 6. Integración de SonarQube

Se levantó un contenedor local de SonarQube en Docker exponiendo el puerto `9000` para administración web.

### 6.1 Configuración realizada

1. Creación del proyecto `cicd-demo`
2. Generación de token de acceso
3. Registro del token en Jenkins Credentials
4. Configuración del servidor SonarQube en Jenkins
5. Integración de la etapa `Static Analysis` en el Jenkinsfile

![!\[\]\[image3\]](images/image3.png)

El análisis estático arrojó los siguientes resultados en el dashboard de SonarQube:

![!\[\]\[image4\]](images/image4.png)




### 6.2 Estado del pipeline

El pipeline ejecutó exitosamente el análisis estático desde Jenkins, validando la calidad del código fuente del proyecto.

![!\[\]\[image5\]](images/image5.png)
---

## 7. Security Scan con Trivy

Se integró la herramienta Trivy en el pipeline con el objetivo de identificar vulnerabilidades en la imagen Docker generada.

### 7.1 Escaneo de Seguridad con Trivy

Dado que el contenedor de Jenkins no incluía Trivy por defecto, se realizó su instalación manual accediendo al contenedor en modo root y configurando el repositorio oficial de la herramienta:

```bash
docker exec -it -u root jenkins-local bash

apt-get update
apt-get install -y wget apt-transport-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | tee /etc/apt/sources.list.d/trivy.list

apt-get update
apt-get install -y trivy

trivy --version
```

### 7.2 Integración en el pipeline

La ejecución de Trivy fue incorporada al Jenkinsfile mediante la etapa `Security Scan`, lo que permitió:

- Reducir el tiempo de ejecución del escaneo
- Evitar descargas innecesarias de bases de datos
- Analizar vulnerabilidades del sistema operativo de la imagen
- Mostrar resultados sin interrumpir la ejecución del pipeline

![!\[\]\[image6\]](images/image6.png)

### 7.3 Resultados obtenidos

El escaneo identificó vulnerabilidades en la imagen Docker basada en Alpine 3.20, confirmando el correcto funcionamiento de la etapa de seguridad. Trivy generó advertencias relacionadas con:

- Uso de versiones de sistema operativo sin soporte
- Posible ausencia de actualizaciones de seguridad

Esto evidencia la capacidad del pipeline para detectar riesgos en la infraestructura del contenedor antes del despliegue.

El pipeline durante la etapa de Security Scan:

![!\[\]\[image7\]](images/image7.png) 

### 7.4 Mejora identificada — Gatekeeping

Se evaluó la implementación de un mecanismo de gatekeeping basado en los resultados del escaneo. Actualmente, el pipeline permite continuar el flujo incluso si se detectan vulnerabilidades críticas. Sin embargo, se identificó la posibilidad de configurar Trivy con la opción `--exit-code 1` para detener automáticamente el despliegue en caso de hallazgos críticos, lo cual fortalecería la seguridad del proceso CI/CD.

---

## 8. Despliegue y Validación

![!\[\]\[image8\]](images/image8.png)

La etapa de despliegue fue ejecutada exitosamente como parte del pipeline, permitiendo la puesta en marcha automática de la aplicación contenida en la imagen Docker generada.

Durante esta fase, el pipeline realiza las siguientes acciones:


1. Intenta detener una instancia previa del contenedor, si existe
2. Elimina el contenedor anterior para evitar conflictos
3. Ejecuta un nuevo contenedor a partir de la imagen `mi-app:latest`

![!\[\]\[image9\]](images/image9.png)

Como se evidencia en la figura, el contenedor mi-app se encuentra en estado activo, junto con los servicios auxiliares utilizados en el pipeline como Jenkins y SonarQube. Además, se observa que la aplicación está correctamente expuesta a través del puerto 80, lo que confirma que el despliegue fue realizado de manera satisfactoria.
La coexistencia de los contenedores jenkins-local, sonarqube y mi-app demuestra un entorno completamente funcional para la ejecución del flujo CI/CD, abarcando desde la construcción hasta el despliegue de la aplicación
Como validación final del flujo CI/CD, se realizó un cambio en el código fuente del proyecto y se ejecutó un nuevo commit al repositorio. Jenkins detectó automáticamente la modificación, ejecutó nuevamente el pipeline completo y desplegó una nueva versión de la aplicación, confirmando la correcta automatización del proceso.
 

![!\[\]\[image10\]](images/image10.png)



## Conclusión

El pipeline desarrollado permitió automatizar de manera completa el ciclo de vida del software, desde la integración del código hasta su despliegue en un entorno controlado. La incorporación de herramientas como **SonarQube** y **Trivy** permitió fortalecer la calidad y seguridad del sistema, mientras que el uso de contenedores garantizó la portabilidad y consistencia del entorno.

Este ejercicio evidencia cómo la automatización en procesos CI/CD reduce la intervención manual, mejora la eficiencia del desarrollo y aumenta la confiabilidad del software entregado.
