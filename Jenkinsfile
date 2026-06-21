pipeline {
    agent any

    tools {
        jdk 'Jdk21'
        maven 'maven3'
    }

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '440744242159.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY_URL = '440744242159.dkr.ecr.us-east-1.amazonaws.com/clean-cicd-aws-app-repo'
        EKS_CLUSTER_NAME = 'clean-cicd-eks'

        JENKINS_URL = 'http://44.204.63.46:8080'
        JFROG_URL = 'http://44.201.211.76:8082'
        SONARQUBE_URL = 'http://34.239.102.5:9000'

        SCANNER_HOME = tool 'sonar'

        APP_NAME = 'boardgame'
        APP_OWNER = 'cloud_team'
        KUBE_NAMESPACE = 'webapps'

        TRIVY_HOME = '/var/lib/jenkins/bin/trivy'
        TRIVY_CACHE_DIR = '/var/lib/jenkins/.cache/trivy'
        TMPDIR = '/var/lib/jenkins/tmp'

        DOCKER_IMAGE = "${ECR_REPOSITORY_URL}:${BUILD_NUMBER}"
        DOCKER_IMAGE_LATEST = "${ECR_REPOSITORY_URL}:latest"

        GIT_CRED = 'lili'
        SONARQUBE_INSTALLATION = 'sonar'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "lili",
                    url: 'https://github.com/Tchapock/Geo-APPLICATION.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Install Trivy') {
            steps {
                sh '''
                    mkdir -p /var/lib/jenkins/bin
                    mkdir -p /var/lib/jenkins/tmp
                    mkdir -p /var/lib/jenkins/.cache/trivy

                    export TMPDIR=/var/lib/jenkins/tmp

                    if [ ! -f /var/lib/jenkins/bin/trivy ]; then
                        echo "Installing Trivy..."
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /var/lib/jenkins/bin
                    else
                        echo "Trivy already installed."
                    fi

                    /var/lib/jenkins/bin/trivy --version
                '''
            }
        }

        stage('File System Scan') {
            steps {
                sh '''
                    export TMPDIR=/var/lib/jenkins/tmp
                    export TRIVY_CACHE_DIR=/var/lib/jenkins/.cache/trivy

                    /var/lib/jenkins/bin/trivy fs \
                      --format table \
                      -o trivy-fs-report.html .
                '''
            }
        }

      stage('SonarQube Analysis') {
    steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            withSonarQubeEnv("${SONARQUBE_INSTALLATION}") {
                sh '''
                    echo "Running SonarQube Analysis..."
                    echo "SCANNER_HOME is: $SCANNER_HOME"
                    ls -la $SCANNER_HOME/bin

                    $SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=Geo-Application \
                      -Dsonar.projectKey=Geo-Application \
                      -Dsonar.sources=. \
                      -Dsonar.java.binaries=target/classes \
                      -Dsonar.login=$SONAR_TOKEN
                '''
            }
        }
    }
}
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Login To AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh '''
                    docker build -t $DOCKER_IMAGE .
                    docker tag $DOCKER_IMAGE $DOCKER_IMAGE_LATEST
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh '''
                    export TMPDIR=/var/lib/jenkins/tmp
                    export TRIVY_CACHE_DIR=/var/lib/jenkins/.cache/trivy

                    /var/lib/jenkins/bin/trivy image \
                      --format table \
                      -o trivy-image-report.html \
                      $DOCKER_IMAGE
                '''
            }
        }

        stage('Push Docker Image To ECR') {
            steps {
                sh '''
                    docker push $DOCKER_IMAGE
                    docker push $DOCKER_IMAGE_LATEST
                '''
            }
        }

        stage('Update Kubeconfig For EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                '''
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                dir('geo_patient') {
                    sh '''
                        echo "Current directory:"
                        pwd

                        echo "Files here:"
                        ls -la

                        kubectl create namespace $KUBE_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

                        sed -i "s|image: .*|image: ${DOCKER_IMAGE_LATEST}|g" deployment-service.yaml

                        kubectl apply -f deployment-service.yaml -n $KUBE_NAMESPACE
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods -n $KUBE_NAMESPACE
                    kubectl get svc -n $KUBE_NAMESPACE
                    kubectl get deployments -n $KUBE_NAMESPACE
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-fs-report.html,trivy-image-report.html', allowEmptyArchive: true

            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'tchapock308@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
                )
            }
        }
    }
}