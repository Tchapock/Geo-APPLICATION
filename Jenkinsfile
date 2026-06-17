pipeline {
    agent any

    tools {
        jdk 'Jdk21'
        maven 'maven3'
    }

    environment {
        // AWS / Terraform output values
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '440744242159.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY_URL = '440744242159.dkr.ecr.us-east-1.amazonaws.com/clean-cicd-aws-app-repo'
        EKS_CLUSTER_NAME = 'clean-cicd-eks'

        // Server URLs from Terraform output
        JENKINS_URL = 'http://34.230.61.69:8080'
        JFROG_URL = 'http://3.94.101.203:8082'
        SONARQUBE_URL = 'http://3.86.154.83:9000'

        // Jenkins tool name
        SCANNER_HOME = tool 'sonar'

        // Application variables
        APP_NAME = 'boardgame'
        APP_OWNER = 'cloud_team'
        BRANCH_NAME = 'main'
        KUBE_NAMESPACE = 'webapps'
        TRIVY_CACHE_DIR = '/var/lib/jenkins/.cache/trivy'
        TMPDIR = '/var/lib/jenkins/tmp'
        // Docker / AWS ECR image variables
        DOCKER_IMAGE = "${ECR_REPOSITORY_URL}:${BUILD_NUMBER}"
        DOCKER_IMAGE_LATEST = "${ECR_REPOSITORY_URL}:latest"

        // Jenkins credentials IDs
        GIT_CRED = 'git-cred'
        SONARQUBE_CRED = 'sonar-token'

        // Jenkins SonarQube server installation name
        SONARQUBE_INSTALLATION = 'sonar'

        // Artifact variables
        ARTIFACTPATH = 'target/*.jar'
        ARTIFACTTARGETPATH = "release_${BUILD_ID}.jar"
        HELMARTIFACTPATH = "geo-app-${BUILD_ID}.tgz"
        HELMARTIFACTTARGET = "helm/geo-app-${BUILD_ID}.tgz"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'tchapo',
                    url: 'https://github.com/Tchapock/Geo-APPLICATION.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('Install Trivy') {
    steps {
        sh '''
            mkdir -p /var/lib/jenkins/bin

            if [ ! -x /var/lib/jenkins/bin/trivy ]; then
                echo "Installing Trivy..."
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /var/lib/jenkins/bin latest
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
            mkdir -p /var/lib/jenkins/tmp
            mkdir -p /var/lib/jenkins/.cache/trivy

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
                withSonarQubeEnv("${SONARQUBE_INSTALLATION}") {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
    steps {
        timeout(time: 2, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: false
        }
    }
}

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        /*
        This stage is disabled for now because your infrastructure has JFrog,
        not Nexus. Maven deploy will fail unless your pom.xml and Maven settings
        are configured for JFrog Artifactory.

        stage('Publish To JFrog') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-settings',
                    jdk: 'jdk17',
                    maven: 'maven3',
                    mavenSettingsConfig: '',
                    traceability: true
                ) {
                    sh "mvn deploy"
                }
            }
        }
        */

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
            mkdir -p /var/lib/jenkins/tmp
            mkdir -p /var/lib/jenkins/.cache/trivy

            export TMPDIR=/var/lib/jenkins/tmp
            export TRIVY_CACHE_DIR=/var/lib/jenkins/.cache/trivy

            /var/lib/jenkins/bin/trivy image \
              --format table \
              -o trivy-image-report.html \
              $ECR_REPOSITORY_URL:$BUILD_NUMBER
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

        stage('Verify the Deployment') {
            steps {
                sh '''
                    kubectl get pods -n $KUBE_NAMESPACE
                    kubectl get svc -n $KUBE_NAMESPACE
                '''
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
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
                    to: 'jaiswaladi246@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
                )
            }
        }
    }
}