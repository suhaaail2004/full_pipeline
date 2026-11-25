pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = '8089278330/foodista'
        SONAR_SERVER_NAME     = 'sonarqube_env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/suhaaail2004/full_pipeline.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=foodista-key \
                                -Dsonar.projectName=foodista-project \
                                -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login \
                                -u "\$DOCKER_USER" --password-stdin

                            docker build \
                                -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                                -t ${DOCKER_IMAGE}:latest \
                                -f Dockerfile .

                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {
                    git branch: "${MANIFEST_BRANCH}",
                        credentialsId: "${GIT_CREDENTIALS_ID}",
                        url: "${MANIFEST_REPO}"

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            usernameVariable: 'GH_USER',
                            passwordVariable: 'GH_TOKEN'
                        )
                    ]) {
                        sh """
                            git remote set-url origin \
                                https://\${GH_USER}:\${GH_TOKEN}@github.com/suhaaail2004/full_pipeline.git

                            git pull --rebase origin ${MANIFEST_BRANCH}

                            sed -i '/name: foodista/{n;s|image:.*|image: 8089278330/foodista:${env.BUILD_NUMBER}|}' \
                                k8s/deployment.yaml

                            git config user.name "suhaaail2004"
                            git config user.email "suhailshihab444@gmail.com"

                            git add k8s/deployment.yaml
                            git commit -m "chore(manifest): update Foodista image to ${env.BUILD_NUMBER} [skip ci]"
                            git push origin ${MANIFEST_BRANCH}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Foodista image pushed and manifest updated.'
        }

        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}