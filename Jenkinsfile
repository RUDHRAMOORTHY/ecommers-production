pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }
    tools {
    sonarRunner 'sonar-scanner'
    }

    environment {
        IMAGE_NAME = "rudhra2710/ecommers-flask-app"
        GIT_USER   = "RUDHRAMOORTHY"
        GIT_EMAIL  = "rudhra2710@gmail.com"
        SONARQUBE_SERVER = "sonar-server"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Environment Check') {
            steps {
                sh '''
                    echo "===== Python ====="
                    python3 --version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== kubectl ====="
                    kubectl version --client || true

                    echo "===== Git ====="
                    git --version
                '''
            }
        }
        
        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' sonar-scanner -Dsonar.projectName=ECOMMERS \
                    -Dsonar.projectKey=ECOMMERS '''
                }
            }
        }
        stage("Code Quality Gate"){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            } 
        }


        stage('OWASP Dependency Check') {
            steps {

                dependencyCheck(
                    additionalArguments: '''
                        --scan .
                        --format XML
                        --format HTML
                        --out dependency-check-report
                    ''',
                    odcInstallation: 'dp-check'
                )

                dependencyCheckPublisher(
                    pattern: 'dependency-check-report/dependency-check-report.xml'
                )
            }
        }

        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    sh """
                        docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        .
                    """
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    --ignore-unfixed \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Docker Login') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                        -u "$DOCKER_USER" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
        
        stage('Upload Python Package to Nexus') {
            when {
                branch 'main'
            }

            steps {
                script {

                    sh '''
                        python3 -m pip install --upgrade build twine

                        rm -rf dist build *.egg-info

                        python3 -m build
                    '''

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'nexus-cred',
                            usernameVariable: 'NEXUS_USER',
                            passwordVariable: 'NEXUS_PASS'
                        )
                    ]) {

                        sh '''
                            python3 -m twine upload \
                                --repository-url http://43.204.214.128:8081/repository/python-hosted/ \
                                -u "$NEXUS_USER" \
                                -p "$NEXUS_PASS" \
                                dist/*
                        '''
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
            when { branch 'main' }
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-cred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                        set -e
                        git config user.name "$GIT_USER"
                        git config user.email "$GIT_EMAIL"

                        git fetch origin
                        git checkout main
                        git reset --hard origin/main

                        sed -i "s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" k8s/deployment.yml

                        git add k8s/deployment.yml
                        git diff --cached --quiet || git commit -m "Updated image to ${IMAGE_TAG}"
                        git push https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/RUDHRAMOORTHY/ecommers-production.git main
                        """
                    }
                }
            }
        }
    }
}
