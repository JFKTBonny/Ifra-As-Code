pipeline {
    agent any

    environment {
        // Define SonarQube environment variables
        SONARQUBE_SERVER = 'sonar-server'  
        GITHUB_REPO = 'https://github.com/yahialm/CICD-pipeline-with-Jenkins-ArgoCD-Sonar-and-K8s.git'
        GITHUB_REPO_MANIFEST = 'https://github.com/yahialm/ArgoCD-pipeline-manifest-files.git'
        SONAR_PROJECT_KEY = credentials('sonar-project') 
        SONARQUBE_TOKEN = credentials('sonar-token')
        NVD_API_KEY = credentials('NVD-API')
        GITHUB_EMAIL = credentials('github-email')
        // GITHUB_TOKEN = credentials('github-token')
        DOCKERHUB_CREDENTIALS = "docker-hub-credentials-id" 
        DOCKER_IMAGE_NAME = 'yahialm/spring'
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Run SonarQube analysis
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh "./mvnw sonar:sonar " +
                           "-Dsonar.projectKey=${SONAR_PROJECT_KEY} " +
                           "-Dsonar.login=${SONARQUBE_TOKEN}"
                    }
                }
            }
        }

        stage('OWASP Dependency Check Scan and Publish') {
            steps {
                script {
                    dependencyCheck additionalArguments: """
                    -o './' 
                    -s './'
                    -f 'ALL' 
                    --prettyPrint
                    --nvdApiKey ${NVD_API_KEY} """, odcInstallation: "dependency-check"

                    dependencyCheckPublisher pattern: 'dependency-check-report.html'
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
                    withDockerRegistry([credentialsId: "docker-hub-credentials-id", url: "https://index.docker.io/v1/"]) {
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
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        // Clone the private manifest repo using the token
                        sh """
                        if [ -d "ArgoCD-pipeline-manifest-files" ]; then
                             rm -rf ArgoCD-pipeline-manifest-files
                        fi
                        git clone https://${GITHUB_TOKEN}@github.com/yahialm/ArgoCD-pipeline-manifest-files.git
                        cd ArgoCD-pipeline-manifest-files/k3s
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}|' deployment.yaml
                        """

                        // Commit and push the changes back to the repo
                        // Each sh command has its own specific context, e.g if cd Argo... command was executed individually wrapped
                        // in its own sh """""", that might create problems for the next commands because they will  
                        // be executed in the default path which is the workspace but not inside ArgoCD-pipeline-manifest-files
                        sh """
                            cd ArgoCD-pipeline-manifest-files
                            git config --global user.email "${GITHUB_EMAIL}"
                            git config --global user.name "yahialm"
                            git add k3s/deployment.yaml
                            git commit -m "Updated image to ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/yahialm/ArgoCD-pipeline-manifest-files.git main
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
