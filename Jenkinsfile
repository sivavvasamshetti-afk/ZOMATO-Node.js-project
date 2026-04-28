pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        DOCKER_IMAGE = "sivav2516/zomato"
        AWS_REGION = "us-east-1"
        CLUSTER_NAME = "mycluster"
        RECIPIENTS = "siva.vvasamshetti@gmail.com"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/sivavvasamshetti-afk/ZOMATO-Node.js-project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build App') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'sonar-scanner'
            withSonarQubeEnv('sq') {
                sh """
                ${scannerHome}/bin/sonar-scanner \
                  -Dsonar.projectKey=zomato \
                  -Dsonar.sources=src \
                  -Dsonar.projectName=Zomato-App \
                  -Dsonar.projectVersion=${BUILD_NUMBER}
                """
            }
        }
    }
}

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -r zomato-build.zip build/'
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
                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file zomato-build.zip \
                    http://localhost:8081/repository/raw-hosted/zomato-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} .
                docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }

        stage('Install Helm') {
            steps {
                sh '''
                curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                mv linux-amd64/helm ./helm
                chmod +x ./helm
                '''
            }
        }

        stage('Setup Kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                kubectl get nodes
                '''
            }
        }

        stage('Deploy Monitoring (Prometheus + Grafana)') {
            steps {
                sh '''
                ./helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                ./helm repo update

                ./helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace \
                --set grafana.service.type=LoadBalancer
                '''
            }
        }

        stage('Get Grafana Password') {
            steps {
                sh '''
                echo "Grafana Admin Password:"
                kubectl get secret monitoring-grafana \
                -n monitoring \
                -o jsonpath="{.data.admin-password}" | base64 --decode
                echo ""
                '''
            }
        }

        stage('Deploy Application to EKS') {
            steps {
                sh '''
                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        stage('Wait for LoadBalancer') {
            steps {
                sh '''
                echo "Waiting for LoadBalancer to be ready..."
                sleep 60
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
