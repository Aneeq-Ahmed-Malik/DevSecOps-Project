// DevSecOps Netflix Pipeline
//
// Prerequisites (configured automatically by Ansible JCasC):
//   - JDK tool named "jdk17"
//   - NodeJS tool named "node16"
//   - SonarQube Scanner tool named "sonar-scanner"
//   - SonarQube server named "sonar-server" (http://localhost:9000)
//   - Credentials: "sonar-token" (secret text), "docker" (username/password)
//
// Required environment variables to set in Jenkins job or here:
//   DOCKERHUB_USERNAME  - your DockerHub username

pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME       = tool 'sonar-scanner'
        IMAGE_NAME         = 'netflix'
        DOCKERHUB_USERNAME = credentials('dockerhub-username')
        TMDB_API_KEY       = credentials('tmdb-api-key')
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \\
                            -Dsonar.projectName=Netflix \\
                            -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
                archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=\${TMDB_API_KEY} -t ${IMAGE_NAME} .
                            docker tag ${IMAGE_NAME} \${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                            docker push \${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image \${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest > trivyimage.txt'
                archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
            }
        }

        stage('Deploy to Container') {
            steps {
                sh """
                    docker stop netflix-app || true
                    docker rm   netflix-app || true
                    docker run -d --name netflix-app -p 8081:80 \${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                """
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Build succeeded. ArgoCD will sync the new image to EKS automatically.'
        }
        failure {
            echo 'Build failed. Check SonarQube, OWASP, and Trivy reports above.'
        }
    }
}
