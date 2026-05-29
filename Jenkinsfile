pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "fiazsh/django_app"
        DOCKERHUB_REPO = "fiazsh/django_app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Code Checkout') {
            steps {
                echo "Pulling code from GitHub"
                git branch: 'main',
                    changelog: false,
                    credentialsId: 'git-cred',
                    poll: false,
                    url: 'https://github.com/Fiazshafqat/notes-app-django.git'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs --format table -o trivy-fs-report.html .
                '''
            }
        }

        
        
        stage('SonarQube Code Analysis') {
            steps {
            withSonarQubeEnv('Sonar') {
            sh '''
            /opt/sonar-scanner/bin/sonar-scanner \
            -Dsonar.projectKey=notes-app \
            -Dsonar.sources=. \
            -Dsonar.exclusions=data/**,**/*.sqlite3,**/*.db,**/*.ibd,**/mysql/** \
            -Dsonar.host.url=$SONAR_HOST_URL \
            -Dsonar.login=$SONAR_AUTH_TOKEN
            '''
                }
            }
        }


        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }


        stage('OWASP Dependency Check') {
            steps {
            dependencyCheck additionalArguments: '''
            --scan ./
            --nvdApiKey 086df244-5a64-4c8f-926f-0d2662b95a94
            ''', odcInstallation: 'OWASP'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }


        stage('Build Docker Images') {
            steps {
                echo "Building Docker image"
                sh 'docker compose build'
            }
        }

        stage('Image Tag, Login & Push to Docker Hub') {
            steps {
                script {
                    echo "Tagging Docker image"

                    sh """
                    docker tag $DOCKER_IMAGE:latest $DOCKERHUB_REPO:$IMAGE_TAG
                    docker tag $DOCKER_IMAGE:latest $DOCKERHUB_REPO:latest
                    """

                    withCredentials([usernamePassword(
                        credentialsId: "docker-cred",
                        usernameVariable: "DOCKER_USER",
                        passwordVariable: "DOCKER_PASS"
                    )]) {

                        echo "Logging into Docker Hub"

                        sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        """

                        echo "Pushing image to Docker Hub"

                        sh """
                        docker push $DOCKERHUB_REPO:$IMAGE_TAG
                        docker push $DOCKERHUB_REPO:latest
                        """
                    }
                }
            }
        }

        stage('Deploy Containers') {
            steps {
                echo "Deploying containers using docker compose"
                sh 'docker compose down || true'
                sh 'docker compose up -d'
            }
        }
    }

    post {

        success {
            echo "✅ Pipeline SUCCESS: Build, Push & Deployment completed successfully"
        }

        failure {
            echo "❌ Pipeline FAILED: Check logs for errors"
        }
    }
}
