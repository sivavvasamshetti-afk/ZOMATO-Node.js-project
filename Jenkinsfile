pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        DOCKER_USER = "sivav2516"
        IMAGE_NAME = "sivav2516/zomato"
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        NEXUS_URL = "http://localhost:8081/repository/zomoto"
        RECIPIENTS = "siva.vvasamshetti@gmail.com"

        CLUSTER_NAME = "mycluster"
        PROJECT_NAME = "ZOMATO-Node.js-project"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sivavvasamshetti-afk/ZOMATO-Node.js-project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sq') {
                    sh '''
                    /opt/sonar-scanner/bin/sonar-scanner \
                    -Dsonar.projectKey=game \
                    -Dsonar.sources=src \
                    -Dsonar.projectName=game-App \
                    -Dsonar.projectVersion=${BUILD_NUMBER}
                    '''
                }
            }
        }
        


        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                if [ -d dist ]; then
                    tar -czf app-${BUILD_NUMBER}.tar.gz dist
                else
                    tar -czf app-${BUILD_NUMBER}.tar.gz .
                fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app-${BUILD_NUMBER}.tar.gz \
                    $NEXUS_URL/app-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .
                docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG $DOCKER_USER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    docker push $DOCKER_USER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region ap-south-1 --name mycluster

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        stage('Install Helm') {
            steps {
                sh '''
                if [ ! -f helm ]; then
                    curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                    tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                    mv linux-amd64/helm ./helm
                    chmod +x ./helm
                fi
                '''
            }
        }

        stage('Deploy Monitoring') {
            steps {
                sh '''
                ./helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                ./helm repo update

                ./helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        stage('Expose Grafana') {
            steps {
                sh '''
                echo "Waiting for Grafana..."
                sleep 30

                kubectl patch svc monitoring-grafana \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }

        stage('Expose Prometheus') {
            steps {
                sh '''
                kubectl patch svc monitoring-kube-prometheus-prometheus \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }
    }

    // ===========================
    post {

        success {
            script {

                sleep 40

                def APP_URL = ""
                def GRAFANA_URL = ""
                def PROM_URL = ""

                for (int i = 0; i < 5; i++) {

                    APP_URL = sh(
                        script: "kubectl get svc puzzle-game-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    GRAFANA_URL = sh(
                        script: "kubectl get svc monitoring-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    PROM_URL = sh(
                        script: "kubectl get svc monitoring-kube-prometheus-prometheus -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    if (APP_URL && GRAFANA_URL && PROM_URL) {
                        break
                    }

                    sleep 20
                }

                def DOCKER_IMAGE = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"

                emailext(
                    subject: "🚀 Deployment Successful - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                    <html>
                    <body style="font-family: Arial;">

                    <h2 style="color:green;">🎉 Deployment Successful</h2>

                    <h3>📌 Project Details</h3>
                    <ul>
                        <li><b>Project:</b> ${PROJECT_NAME}</li>
                        <li><b>Cluster:</b> ${CLUSTER_NAME}</li>
                    </ul>

                    <h3>🐳 Docker Image</h3>
                    <p>${DOCKER_IMAGE}</p>

                    <h3>🌐 Application</h3>
                    <a href="http://${APP_URL}">Open Application</a>

                    <h3>📊 Grafana</h3>
                    <a href="http://${GRAFANA_URL}">Open Grafana</a>

                    <h3>🔥 Prometheus</h3>
                    <a href="http://${PROM_URL}:9090">Open Prometheus</a>

                    <h3>🛠 Jenkins</h3>
                    <ul>
                        <li>Job: ${env.JOB_NAME}</li>
                        <li>Build: ${env.BUILD_NUMBER}</li>
                        <li><a href="${env.BUILD_URL}">Open Build</a></li>
                    </ul>

                    </body>
                    </html>
                    """,
                    to: "${env.RECIPIENTS}"
                )
            }
        }

        failure {
            emailext(
                subject: "❌ Deployment Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                <html>
                <body style="font-family: Arial;">

                <h2 style="color:red;">❌ Deployment Failed</h2>

                <p><b>Project:</b> Sliding Puzzle Game</p>
                <p><b>Cluster:</b> mycluster</p>

                <h3>🔍 Logs</h3>
                <a href="${env.BUILD_URL}">View Build Logs</a>

                </body>
                </html>
                """,
                to: "${env.RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'app-*.tar.gz', fingerprint: true, allowEmptyArchive: true
        }
    }
}
