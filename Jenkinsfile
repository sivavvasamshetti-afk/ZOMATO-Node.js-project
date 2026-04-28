pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        DOCKER_IMAGE = "rajeshtutta123/zomato"
        AWS_REGION = "us-east-1"
        CLUSTER_NAME = "mycluster"
        RECIPIENTS = "rajeshtutta123@gmail.com"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/rajeshtutta/zomato.git'
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

        stage('Get Application URL') {
            steps {
                script {
                    def url = sh(
                        script: '''
                        kubectl get svc zomatosvc \
                        -o jsonpath="{.status.loadBalancer.ingress[0].hostname}{.status.loadBalancer.ingress[0].ip}"
                        ''',
                        returnStdout: true
                    ).trim()

                    env.APP_URL = url
                    echo "Application URL: ${env.APP_URL}"
                }
            }
        }
    }

    post {

        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Build SUCCESS 🎉

Application URL:
http://${env.APP_URL}

Jenkins URL:
${env.BUILD_URL}
""",
                to: "${RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Build FAILED ❌

Check logs:
${env.BUILD_URL}
""",
                to: "${RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'zomato-build.zip', fingerprint: true
        }
    }
}
