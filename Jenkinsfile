pipeline {
    agent any

    environment {
        REGION = "eu-west-1"
        ACCOUNT_ID = "450444046629"

        ECR_NAME = "user26-petclinic"
        ECR_REGISTRY = "${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        ECR_REPO = "${ECR_REGISTRY}/${ECR_NAME}"

        CLUSTER_NAME = "user26-cluster"
        NAMESPACE = "petclinic"

        IMAGE_TAG = "v${BUILD_NUMBER}"

        K8S_MANIFEST_REPO = "https://github.com/coworklch275/petclinic-cicd.git"
        DEPLOYMENT_FILE = "k8s/deployment.yaml"
        DEPLOY_BRANCH = "deploy"

        SONAR_PROJECT_KEY = "user26-petclinic"
        TRIVY_CACHE = "/var/jenkins_home/.cache/trivy"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/coworklch275/petclinic-cicd.git'
            }
        }
        stage('Maven Build') {
            steps {
                sh '''
                    rm -rf ${WORKSPACE}/.trivycache
                    chmod +x mvnw || true

                    if [ -f "./mvnw" ]; then
                      ./mvnw clean package -DskipTests
                    else
                      mvn clean package -DskipTests
                    fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        ./mvnw sonar:sonar -B \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.sources=src/main \
                          -Dsonar.tests=src/test
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    mkdir -p ${TRIVY_CACHE} reports

                    echo "===== Filesystem Scan (deps + secrets + misconfig) ====="
                    trivy fs . \
                      --cache-dir ${TRIVY_CACHE} \
                      --scanners vuln,secret,misconfig \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      --output reports/trivy-fs.txt
                    cat reports/trivy-fs.txt

                    echo "===== Gate: CRITICAL & fixable ====="
                    trivy fs . \
                      --cache-dir ${TRIVY_CACHE} \
                      --severity CRITICAL \
                      --ignore-unfixed \
                      --exit-code 1 \
                      --quiet
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy-fs.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${ECR_NAME}:${IMAGE_TAG} .
                    docker tag ${ECR_NAME}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    mkdir -p reports

                    echo "===== [1/2] Full report (HIGH + CRITICAL) ====="
                    trivy image ${ECR_NAME}:${IMAGE_TAG} \
                      --cache-dir ${TRIVY_CACHE} \
                      --scanners vuln,secret \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      --output reports/trivy-image.txt
                    cat reports/trivy-image.txt

                    echo "===== [2/2] Gate: CRITICAL & fixable ====="
                    trivy image ${ECR_NAME}:${IMAGE_TAG} \
                      --cache-dir ${TRIVY_CACHE} \
                      --severity CRITICAL \
                      --ignore-unfixed \
                      --exit-code 1 \
                      --quiet
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                }
            }
        }

        stage('ECR Push') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${REGION} \
                      | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        // ArgoCD가 감지할 수 있도록, 클러스터에 직접 apply하지 않고
        // git manifest의 image tag만 갱신해서 deploy 브랜치로 push한다.
        // main이 아닌 deploy로 push하므로 Jenkins(=main만 감시)가 재트리거되지 않는다.
        stage('Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        sed -i "s|image: .*|image: ${ECR_REPO}:${IMAGE_TAG}|" ${DEPLOYMENT_FILE}

                        git config user.email "jenkins-ci@petclinic.local"
                        git config user.name "jenkins-ci"

                        git add ${DEPLOYMENT_FILE}
                        git diff --cached --quiet && echo "No manifest changes" || \
                          git commit -m "ci: deploy ${IMAGE_TAG}"

                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/coworklch275/petclinic-cicd.git HEAD:${DEPLOY_BRANCH} --force
                    '''
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                sh '''
                    kubectl annotate application petclinic -n argocd \
                      argocd.argoproj.io/refresh=hard --overwrite
                '''
            }
        }

        stage('Verification') {
            steps {
                sh '''
                    kubectl rollout status deployment/petclinic -n ${NAMESPACE} --timeout=300s
                    kubectl get pods -n ${NAMESPACE} -o wide
                    kubectl get svc -n ${NAMESPACE}
                    kubectl get ingress -n ${NAMESPACE}
                    kubectl get application petclinic -n argocd \
                      -o jsonpath='sync={.status.sync.status} health={.status.health.status}{"\\n"}'
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded. Manifest updated for image: ${ECR_REPO}:${IMAGE_TAG} (deployed via ArgoCD)"
        }

        failure {
            echo "Pipeline failed. Check Jenkins console logs."
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}

