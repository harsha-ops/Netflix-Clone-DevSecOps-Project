pipeline {
    agent any 
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "harsha6798/netflix-clone-image:${BUILD_NUMBER}"
        GITHUB_TOKEN = credentials('github_token')
    }

    stages {

        //stage('Clean Workspace') {
        //   steps {
             //   cleanWs()
        //   }
        //}

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

        //stage('quality gate') {
         //   steps {
         //       script {
         //           waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
         //       }
         //   }
        //}

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
                sh "docker build --build-arg TMDB_V3_API_KEY=5f2f4c170b0b3abeea46efedc27d608e -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Login to Docker Registry') {
            steps{
                withCredentials([usernamePassword(
                    credentialsId: 'docker_cred', // Use your actual credentials ID
                    usernameVariable: 'DOCKER_USERNAME', 
                    passwordVariable: 'DOCKER_PASSWORD'
            )]) {
                sh 'docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD'
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

        stage('Update K8smanifest files') {
            environment {
                REPO_URL = "https://${GITHUB_TOKEN}@github.com/harsha-ops/Netflix-Clone-DevSecOps-Project.git"
            }
            steps {
                sh '''
                git config --global user.email = "harsha.xyz@gmail.com"
                git config --global user.name "Harsha"
                sed -i "s|image: .*|image: ${DOCKER_IMAGE}|g" k8smanifest/deployment.yml
                git add .
                git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                git push ${REPO_URL}
                '''
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'nharsha022@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
