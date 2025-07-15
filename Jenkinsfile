pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'
        maven 'M3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SLACK_CHANNEL = "#awsrohits8"
        SLACK_WEBHOOK_CREDENTIAL_ID = "slack-webhook"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/Paras116255/app-register.git'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Docker Login & Build") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds-id', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    script {
                        def imageName = "${DOCKER_USERNAME}/${APP_NAME}"
                        env.IMAGE_NAME = imageName

                        sh """
                            echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                            docker build -t ${imageName}:${IMAGE_TAG} .
                        """
                    }
                }
            }
        }

        stage("Docker Push") {
            steps {
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                        docker run \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                            --no-progress \
                            --scanners vuln \
                            --exit-code 0 \
                            --severity HIGH,CRITICAL \
                            --format table > trivy-report.txt
                    """

                    def trivyReport = readFile('trivy-report.txt').trim()
                    if (trivyReport.length() > 1000) {
                        trivyReport = trivyReport.take(1000) + "\\n... (truncated)"
                    }

                    env.TRIVY_REPORT = trivyReport
                }
            }
        }

        stage("Send Trivy Report to Slack") {
            steps {
                withCredentials([string(credentialsId: SLACK_WEBHOOK_CREDENTIAL_ID, variable: 'WEBHOOK')]) {
                    script {
                        // Escape newlines and quotes for JSON
                        def slackMessage = env.TRIVY_REPORT.replaceAll('\n', '\\\\n').replaceAll('"', '\\"')
                        sh """
                            curl -X POST -H 'Content-type: application/json' --data '{"text":"*Trivy Scan Report for ${env.IMAGE_NAME}:${env.IMAGE_TAG}*\\n${slackMessage}"}' $WEBHOOK
                        """
                    }
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            }
        }
    }
       stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
               steps {
                   git branch: 'main', credentialsId: 'github', url: 'https://github.com/Shridatta1234/app-register-CD.git'
               }
        }

        stage("Update the Deployment Tags") {
            steps {
                sh """
                   cat deployment.yaml
                   sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' deployment.yaml
                   cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                   git config --global user.name "rohit"
                   git config --global user.email "rohit@gmail.com"
                   git add deployment.yaml
                   git commit -m "Updated Deployment Manifest"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                  sh "git push https://github.com/Shridatta1234/app-register-CD.git main"
                }
            }
        }
    post {
        success {
            withCredentials([string(credentialsId: SLACK_WEBHOOK_CREDENTIAL_ID, variable: 'WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' --data '{"text":"✅ Build SUCCESS for ${env.JOB_NAME} [#${env.BUILD_NUMBER}]"}' $WEBHOOK
                """
            }
        }
        failure {
            withCredentials([string(credentialsId: SLACK_WEBHOOK_CREDENTIAL_ID, variable: 'WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' --data '{"text":"❌ Build FAILED for ${env.JOB_NAME} [#${env.BUILD_NUMBER}]"}' $WEBHOOK
                """
            }
        }
    }
}
