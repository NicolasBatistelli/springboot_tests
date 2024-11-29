pipeline {
    agent any
    tools {
        maven 'Maven 3.9.9'
    }
    stages {
        stage("Build Info") {
            steps {
                script {
                    BUILD_TRIGGER_BY = currentBuild.getBuildCauses()[0].userId
                    currentBuild.displayName = "#${env.BUILD_NUMBER}"                    
                }
            }
        }
        stage('Instalar Java 17') {
            steps {
                script {
                    sh 'sudo apt-get update'
                    sh 'sudo apt-get install -y openjdk-17-jdk'
                }
            }
        }
        stage('Clonar código fuente') {
            steps {
                checkout scm
            }
        }
        stage('Instalar dependencias') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Ejecutar pruebas unitarias') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Build & Run Docker') {
            steps {
                script {
                    sh 'docker build -t myapp .'
                    sh "docker stop myapp || true"
                    sh "docker rm -f myapp || true"
                    sh 'docker run -d -p 8081:8081 --name myapp myapp'
                }
            }
        }
    }
    post {
    failure {
        script {
            // Obtener el commit y los detalles del autor
            def gitCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
            def gitAuthorName = sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim()
            def gitCommitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()

            // Leer los resultados de las pruebas unitarias
            def failedTests = []
            def testReports = sh(script: 'find target/surefire-reports/ -name "*.xml"', returnStdout: true).trim().split("\n")

            testReports.each { report ->
                def failedTestsInFile = sh(script: "xmllint --xpath '//failure' ${report} 2>/dev/null", returnStdout: true).trim()
                if (failedTestsInFile) {
                    def testCaseName = sh(script: "xmllint --xpath 'string(//failure/@message)' ${report} 2>/dev/null", returnStdout: true).trim()
                    failedTests.add("Test failed: ${report} - ${testCaseName}")
                }
            }

            // Definir el asunto y cuerpo del correo con base en el resultado del build
            def subject = "Jenkins Build #${BUILD_NUMBER} - ${currentBuild.currentResult}"
            def body = """
                <p>El build ha terminado con el siguiente resultado: ${currentBuild.currentResult}</p>
                <p>Detalles del commit:</p>
                <ul>
                    <li><strong>Commit:</strong> ${gitCommit}</li>
                    <li><strong>Autor:</strong> ${gitAuthorName}</li>
                    <li><strong>Mensaje del commit:</strong> ${gitCommitMessage}</li>
                </ul>
                <p>Ver detalles en Jenkins: ${BUILD_URL}</p>
            """

            // Si alguna prueba falló, agregar los detalles al cuerpo del correo
            if (failedTests.size() > 0) {
                body += """
                    <h3>Las siguientes pruebas fallaron:</h3>
                    <ul>
                        ${failedTests.collect { "<li>${it}</li>" }.join()}
                    </ul>
                """
            }

            // Enviar el correo
            emailext(
                subject: subject,
                body: body,
                to: 'jose.maita@eldars.com.ar',
                mimeType: 'text/html'
            )
            }
        }
    }
}
