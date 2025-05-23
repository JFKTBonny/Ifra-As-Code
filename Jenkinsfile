pipeline {
    agent any
    

    environment {
        // Define SonarQube environment variables
        SONARQUBE_SERVER = 'sonar server'  
        GITHUB_REPO = 'https://github.com/JFKTBonny/Ifra-As-Code.git'
        // GITHUB_REPO_MANIFEST = 'https://github.com/yahialm/ArgoCD-pipeline-manifest-files.git'
        SONAR_HOST_URL = 'http://192.168.1.40:9000/'
        SONAR_AUTH_TOKEN = credentials('sonar-token')
        NVD_API_KEY = credentials('NVD-API')
        GITHUB_EMAIL = credentials('github-email')
        GITHUB_TOKEN = credentials('github-jenkins-token')
        DOCKERHUB_CREDENTIALS = "dockerhub-access-credentials" 
        DOCKER_IMAGE_NAME = 'santonix/spring-app'
        SONAR_PROJECT_KEY = 'CI_CD_MAVEN_PROJECT'
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {

        stage('Checkout') {
            steps {
                // Checkout the code from GitHub
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('Build') {
            steps {
                // Give permissions to mvnw
                sh 'chmod +x mvnw'

                // Build the Spring Boot project using Maven
                sh './mvnw clean install'
            }
        }

        stage('Test') {
            steps {
                // Run tests using Maven
                sh './mvnw test'
            }
        }

        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=CI_CD_MAVEN_PROJECT \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=CI_CD_MAVEN_PROJECT '''
    
                }
            }
        }
        stage('Quality Gate') {
            steps {
                // Wait for SonarQube quality gate result and abort pipeline if it fails
                timeout(time: 1, unit: 'HOURS') { // optional timeout
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('OWASP Dependency Check Scan and Publish') {
            steps {
                script {
                    // Inject the secret securely from Jenkins credentials
                    withCredentials([string(credentialsId: 'NVD-API', variable: 'NVD_API_KEY')]) {
                        // Pass additionalArguments as a single string without Groovy interpolation for the secret
                        dependencyCheck additionalArguments: "-o './' -s './' -f 'ALL' --prettyPrint --nvdApiKey $NVD_API_KEY", odcInstallation: "DP-CHECK"

                        dependencyCheckPublisher pattern: 'dependency-check-report.html'
                    }
                }
            }

        }



        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image from the Dockerfile
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    // Save Trivy scan result as an HTML report
                    def trivyHtmlReportFile = "trivy-report-${env.BUILD_NUMBER}.html"
                    sh """
                        trivy image --format template --template @/usr/local/share/trivy/templates/html.tpl \
                        ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} > ${trivyHtmlReportFile}
                    """
                    
                    // Publish the HTML report
                    publishHTML([
                        reportName: 'Trivy Security Scan',
                        reportDir: '',
                        reportFiles: trivyHtmlReportFile,
                        keepAll: true,
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        includes: '**/*'
                    ])
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    withDockerRegistry([credentialsId: "dockerhub-access-credentials", url: "https://index.docker.io/v1/"]) {
                        // Push the Docker image to DockerHub
                        sh "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    // Use credentials to authenticate with GitHub
                    withCredentials([string(credentialsId: 'github-jenkins-token', variable: 'GITHUB_TOKEN')]) {
                        // Clone the private manifest repo using the token
                        sh """
                        if [ -d "Ifra-As-Code" ]; then
                             rm -rf Ifra-As-Code

                        fi
                        git clone https://${GITHUB_TOKEN}@github.com/JFKTBonny/Ifra-As-Code.git
                        cd Ifra-As-Code/k8s
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}|' deploy.yaml
                        """

                        // Commit and push the changes back to the repo
                        // Each sh command has its own specific context, e.g if cd Argo... command was executed individually wrapped
                        // in its own sh """""", that might create problems for the next commands because they will  
                        // be executed in the default path which is the workspace but not inside ArgoCD-pipeline-manifest-files
                        sh """
                            cd Ifra-As-Code
                            git config --global user.email "${GITHUB_EMAIL}"
                            git config --global user.name "JFKTBonny"
                            git add k8s/deploy.yaml
                            git commit -m "Updated image to ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/JFKTBonny/Ifra-As-Code.git main
                        """
                    }
                }
            }
        }

    }

    post {
        always {
             // Archive the built artifacts and test results
             archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true

            // Clean the workspace
            cleanWs()
        }
        failure {
            // Notify on failure
            echo 'Build failed!'
        }
        success {
            // Notify on success
            echo 'Build succeeded!'
        }
    }
}
