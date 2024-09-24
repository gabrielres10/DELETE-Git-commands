---
title: Informe de Jenkins

---

### Gabriel Restrepo Giraldo
**Código de Estudiante**: A00377741

---
# Taller de Despliegue y Ejecución del Pipeline en Jenkins

## 1. Introducción

En este documento se describe el proceso de configuración y ejecución de un pipeline en Jenkins, que incluye la preparación del entorno en Docker, la configuración del pipeline y la ejecución exitosa del mismo. Se presenta evidencia de la ejecución paralela de la generación de Javadoc y el build de la aplicación, junto con los resultados obtenidos.

---

## 2. Preparación

### 2.1. Configuración del Entorno Jenkins en Docker

Descripción del proceso seguido para instalar y configurar Jenkins dentro de un contenedor Docker.

#### Script Utilizado para Desplegar

```bash
docker run --name jenkins-container -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

#### Pantallazo de la Ejecución del Contenedor Jenkins

- **Contenedor docker ejecutando**
![image](https://hackmd.io/_uploads/r1YPUilR0.png)
- **Captura de jenkins funcionando**
![image](https://hackmd.io/_uploads/ry7TUigAA.png)

---

## 3. Configuración del Pipeline

### 3.1. Configuración del Pipeline en Jenkins

Descripción de la configuración del pipeline en la interfaz de Jenkins. Esta sección incluye capturas de pantalla relevantes de la configuración de las stages y agentes.

#### 3.1.1. Pantallazo: Configuración del Pipeline

![image](https://hackmd.io/_uploads/BkkPPilAR.png)


#### 3.1.2. Script de Pipeline Utilizado

```groovy
pipeline {
    agent none 
    stages {
        // Download stage
        stage('Preparation') {
            agent { label 'node1' }  
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/spring-projects/spring-petclinic']]])
                stash includes: '**', name: 'source'  
            }
        }
        
        // Parallel stages: Build and Javadoc generation
        stage('Build and Javadoc') {
            parallel {
                stage('Build') {
                    agent { label 'node1' } 
                    steps {
                        unstash 'source'
                        script {
                            sh './gradlew clean build'  // Run compilation and running tests
                            sh './mvnw spring-boot:build-image' // Prepare docker images
                        }
                    }
                    post {
                        always {
                            junit 'build/test-results/test/*.xml' 
                            stash includes: 'build/libs/*.jar', name: 'app'  // Save resulting .jar
                        }
                    }
                }

                stage('Generate Javadoc') {
                    agent { label 'node1' }
                    steps {
                        unstash 'source'
                        script {
                            sh './gradlew javadoc'  // Run Javadoc generation
                        }
                        stash includes: 'build/docs/javadoc/**', name: 'javadoc'  // Save Javadoc artifacts
                    }
                }
            }
        }

        // Deployment stage
        stage('Deploy Application') {
            agent { label 'node2' }
            steps {
                unstash 'app'
                script {
                    // Delete petclinic-app if it exists
                    sh '''
                    if [ "$(docker ps -q -f ancestor=spring-petclinic:3.3.0-SNAPSHOT)" ]; then
                        docker stop $(docker ps -q -f ancestor=spring-petclinic:3.3.0-SNAPSHOT)
                    fi
                    '''
                    sh 'docker run -d -p 8081:8080 spring-petclinic:3.3.0-SNAPSHOT' // Running the docker application
                
                    sleep(30) //Time to wait for deploy
                }
                // Integration test
                script {
                    sh 'curl --fail http://localhost:8081' // Assuming node1 can reach node2 via this URL
                    echo 'Pruebas de integración ejecutadas exitosamente.'
                }
            }
        }
    }

    // Notification
    post {
        success {
            echo 'Pipeline ejecutado con éxito.'
        }
        failure {
            echo 'Pipeline fallido.'
        }
    }
}

```

#### 3.1.3. Pantallazos Relevantes de la Configuración del Pipeline

- **Agentes usados**
![image](https://hackmd.io/_uploads/ryugOse0C.png)

- **Configuración de los Nodos 1 y 2**
![image](https://hackmd.io/_uploads/B1LmOjeRC.png)

- **Configuración de Etapas Paralelas**
![image](https://hackmd.io/_uploads/Byb2dilCC.png)

---

## 4. Resultados

### 4.1. Ejecución Exitosa del Pipeline

Evidencia de la ejecución exitosa del pipeline con los detalles relevantes.

#### 4.1.1. Pantallazos de la Ejecución del Pipeline

![image](https://hackmd.io/_uploads/rJCq6hgRC.png)

Imagen de ejecución exitosa del pipeline completo.


### 4.2. Resultados Obtenidos por Etapa (Stage)

- **Preparation**:
![image](https://hackmd.io/_uploads/BkFm5nxAA.png)

- **Build**: 
    - Resultado del build Exitoso
![image](https://hackmd.io/_uploads/ryFPchxC0.png)
    - Resultado de las pruebas Exitosas
![image](https://hackmd.io/_uploads/SJnJi2eCA.png)
    - Resultado de creación de imagen docker
![image](https://hackmd.io/_uploads/S1Iqhhx00.png)

- **Javadoc**: 
A continuación se presenta el resultado de la ejecución del javadoc. El código presenta varios warnings cuyo refinamiento no hace parte de los objetivos de esta práctica. 
    ![image](https://hackmd.io/_uploads/Skcx6hxCA.png)

- **Aplication Deploy:**
Se presentan los resultados de la prueba de integración.
    ![image](https://hackmd.io/_uploads/Bk8NRnxC0.png)
Como se puede apreciar en la imagen, la consulta al despliegue de la aplicación es exitosa.

---

## 5. Evidencia de Ejecución en Paralelo

### 5.1. Ejecución Paralela: Build y Javadoc

Evidencia de la ejecución en paralelo de la etapa de Build y la generación del Javadoc.

#### 5.1.1. Pantallazo de Ejecución Paralela
![image](https://hackmd.io/_uploads/r19du2lA0.png)

Como se puede apreciar en el grafo de ejecución del pipeline, el build y el javadoc se llevan a cabo de forma concurrente.

## 6. Conclusión

Después de llevar a cabo el taller, me di cuenta cuán retadora puede llegar a ser la automatización de una simple tarea y, luego de 35 ejecuciones del pipeline, logré conseguir el resultado esperado siguiendo todos los criterios del enunciado del taller. Pese a la dificultad de hacer la automatización mediante pipelines en jenkins, es evidente la gran utilidad que ofrece esta herramienta para un entorno de desarrollo con constantes cambios en el repositorio remoto, por lo que realmente vale la pena desarrollar competencias relacionadas con jenkins para aportar valor en un entorno más profesional. 

