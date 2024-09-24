---
Gabriel Restrepo Giraldo: A00377741

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
![image](https://github.com/user-attachments/assets/43444925-e8f2-40c9-91a7-740c1916e898)

- **Captura de jenkins funcionando**
![image](https://github.com/user-attachments/assets/0a945c9c-5a91-4d0c-a536-32f3d538d09f)


---

## 3. Configuración del Pipeline

### 3.1. Configuración del Pipeline en Jenkins

Descripción de la configuración del pipeline en la interfaz de Jenkins. Esta sección incluye capturas de pantalla relevantes de la configuración de las stages y agentes.

#### 3.1.1. Pantallazo: Configuración del Pipeline

![image](https://github.com/user-attachments/assets/8a22764f-5837-4d98-ac61-75145d97baeb)



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
![image](https://github.com/user-attachments/assets/882bc312-0aae-45e1-bfa3-b6d88d98f156)


- **Configuración de los Nodos 1 y 2**
![image](https://github.com/user-attachments/assets/53a73a17-2e5d-4d1c-9cee-1488b4457802)


- **Configuración de Etapas Paralelas**
![image](https://github.com/user-attachments/assets/874d4f2c-1864-42b0-80bb-abe918cdf675)


---

## 4. Resultados

### 4.1. Ejecución Exitosa del Pipeline

Evidencia de la ejecución exitosa del pipeline con los detalles relevantes.

#### 4.1.1. Pantallazos de la Ejecución del Pipeline

![image](https://github.com/user-attachments/assets/342b0705-e594-46d7-a84c-a816c281db5e)


Imagen de ejecución exitosa del pipeline completo.


### 4.2. Resultados Obtenidos por Etapa (Stage)

- **Preparation**:
![image](https://github.com/user-attachments/assets/bce14e07-1dd3-45b2-bd69-39953d4a665c)


- **Build**: 
    - Resultado del build Exitoso
![image](https://github.com/user-attachments/assets/2c7cc675-3d2d-461a-a6a5-cf4a00861820)

    - Resultado de las pruebas Exitosas
![image](https://github.com/user-attachments/assets/169d6ac6-b79a-4b23-8425-03a74f1f4388)

    - Resultado de creación de imagen docker
![image](https://github.com/user-attachments/assets/d5f34fdb-2108-4667-bc4a-3fb28941d7d2)


- **Javadoc**: 
A continuación se presenta el resultado de la ejecución del javadoc. El código presenta varios warnings cuyo refinamiento no hace parte de los objetivos de esta práctica. 
    ![image](https://github.com/user-attachments/assets/5670109a-72ab-44b8-a0e8-aec254ac1dac)


- **Aplication Deploy:**
Se presentan los resultados de la prueba de integración.
    ![image](https://github.com/user-attachments/assets/5e6b1a01-cf67-438c-8d4f-262464ef138c)

Como se puede apreciar en la imagen, la consulta al despliegue de la aplicación es exitosa.

---

## 5. Evidencia de Ejecución en Paralelo

### 5.1. Ejecución Paralela: Build y Javadoc

Evidencia de la ejecución en paralelo de la etapa de Build y la generación del Javadoc.

#### 5.1.1. Pantallazo de Ejecución Paralela
![image](https://github.com/user-attachments/assets/4a918dfa-7418-40f7-bc80-246fcb0506e0)


Como se puede apreciar en el grafo de ejecución del pipeline, el build y el javadoc se llevan a cabo de forma concurrente.

## 6. Conclusión

Después de llevar a cabo el taller, me di cuenta cuán retadora puede llegar a ser la automatización de una simple tarea y, luego de 35 ejecuciones del pipeline, logré conseguir el resultado esperado siguiendo todos los criterios del enunciado del taller. Pese a la dificultad de hacer la automatización mediante pipelines en jenkins, es evidente la gran utilidad que ofrece esta herramienta para un entorno de desarrollo con constantes cambios en el repositorio remoto, por lo que realmente vale la pena desarrollar competencias relacionadas con jenkins para aportar valor en un entorno más profesional. 

