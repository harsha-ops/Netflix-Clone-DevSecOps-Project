pipeline {
    agent any 
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "harsha6798/netflix-clone-image:${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/harsha-ops/Netflix-Clone-DevSecOps-Project.git'
            }
        }

        stage('Sonarqube Analysis') {
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }

        stage('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build --build-arg TMDB_V3_API_KEY=5f2f4c170b0b3abeea46efedc27d608e -t "${DOCKER_IMAGE}" .'
            }
        }

        stage('Login to Docker Registry') {
            steps{
                withCredentials([usernamePassword(
                    credentialsId: 'docker_cred', // Use your actual credentials ID
                    usernameVariable: 'DOCKER_USERNAME', 
                    passwordVariable: 'DOCKER_PASSWORD'
            )]) {
                sh 'docker login -u $DOCKER_USERNAME -p $DOCKERPASSWORD'
            }

            }
            
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push ${DOCKER_IMAGE}'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image $DOCKER_IMAGE > trivyimage.txt'
            }
        }

        stage('Deploy as Container') {
            steps {
                sh 'docker run -d -p 81:80 $DOCKER_IMAGE'
            }
        }




    }
}