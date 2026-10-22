
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'RUN_EXTRA_CHECK',
            defaultValue: true,
            description: 'Run additional dashboard checks'
        )
    }

    environment {
        IMAGE_NAME = 'iniyan59/smart-campus-energy-monitor'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            parallel {
                stage('Validate Dashboard') {
                    steps {
                        script {
                            if (!fileExists('index.html')) {
                                error('index.html is missing')
                            }

                            def html = readFile('index.html')

                            if (!html.contains('Academic Block') ||
                                !html.contains('Library') ||
                                !html.contains('Hostel Block')) {
                                error('Required buildings are missing')
                            }

                            if (!html.contains('100')) {
                                error('Energy limit check is missing')
                            }

                            echo 'Dashboard validation passed'
                        }
                    }
                }

                stage('Check Required Files') {
                    steps {
                        script {
                            ['Dockerfile', 'deployment.yaml'].each {
                                fileName ->
                                if (!fileExists(fileName)) {
                                    error("${fileName} is missing")
                                }
                            }

                            echo 'Required files are present'
                        }
                    }
                }
            }
        }

        stage('Extra Dashboard Check') {
            when {
                expression {
                    return params.RUN_EXTRA_CHECK
                }
            }
            steps {
                echo 'Running extra dashboard checks'
                sh "grep -q 'Total Campus Consumption' index.html"
                sh "grep -q 'Highest Energy Usage' index.html"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" |
                            docker login -u "$DOCKER_USER" --password-stdin

                        docker push "$IMAGE_NAME:$IMAGE_TAG"

                        docker tag "$IMAGE_NAME:$IMAGE_TAG" "$IMAGE_NAME:latest"

                        docker push "$IMAGE_NAME:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'kuberconfig'
                ) {
                    sh 'kubectl apply -f deployment.yaml'

                    sh 'kubectl set image deployment/smart-campus-energy-monitor energy-dashboard=${IMAGE_NAME}:${IMAGE_TAG}'

                    sh 'kubectl rollout status deployment/smart-campus-energy-monitor --timeout=120s'
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts(
                artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                fingerprint: true
            )
            echo 'Pipeline completed successfully'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
