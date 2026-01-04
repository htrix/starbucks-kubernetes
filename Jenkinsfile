pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    tools {
        jdk 'jdk-17'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        // stage('Checkout') {
        //     steps {
        //         checkout([
        //           $class: 'GitSCM',
        //           branches: [[name: '*/main']],
        //           extensions: [[
        //             $class: 'CloneOption',
        //             depth: 1,
        //             noTags: true,
        //             shallow: true
        //           ]],
        //           userRemoteConfigs: [[
        //             url: 'https://github.com/htrix/starbucks-kubernetes.git',
        //             credentialsId: 'github-token'
        //           ]]
        //         ])
        //     }
        // }

        stage('Checkout from Git'){
            steps{
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/htrix/starbucks-kubernetes.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                      ${SCANNER_HOME}/bin/sonar-scanner \
                      -Dsonar.projectName=starbucks \
                      -Dsonar.projectKey=starbucks
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci --prefer-offline --no-audit'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh '''
                if [ ! -d trivy_cache/db ]; then
                    echo "🔽 Trivy DB not found, downloading..."
                    docker run --rm \
                    -v $(pwd):/project \
                    -v trivy_cache:/root/.cache/trivy \
                    aquasec/trivy:latest \
                    fs /project
                else
                    echo "✅ Trivy DB found, skipping update"
                    docker run --rm \
                    -v $(pwd):/project \
                    -v trivy_cache:/root/.cache/trivy \
                    aquasec/trivy:latest \
                    fs /project --skip-db-update
                fi
                '''
            }
        }

        stage('Build image') {
            steps {
                script {
                    // Build usando Docker do host
                    docker.build("htrix/starbucks:latest")
                }
            }
        }

        stage('TRIVY IMAGE SCAN') {
            steps {
                sh '''
                docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -v trivy_cache:/root/.cache/trivy \
                    aquasec/trivy:latest \
                    image htrix/starbucks:latest --skip-db-update > trivyimage.txt
                '''
            }
        }

       stage('Push image') {
        steps {
            script {
                docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-token') {
                    docker.image("htrix/starbucks:latest").push()
                }
            }
        }
        }

        // stage('Docker Build & Push') {
        //     steps {
        //         docker.withDockerRegistry('https://index.docker.io/v1/', credentialsId: 'dockerhub-token') {
        //             sh '''
        //               docker build -t htrix/starbucks:latest .
        //               docker push htrix/starbucks:latest
        //             '''
        //         }
        //     }
        // }

        // stage('TRIVY IMAGE SCAN') {
        //     steps {
        //         sh 'trivy image htrix/starbucks:latest --skip-db-update > trivyimage.txt'
        //     }
        // }

        stage('Deploy') {
            steps {
                sh '''
                  docker rm -f starbucks || true
                  docker run -d --name starbucks -p 3000:3000 htrix/starbucks:latest
                '''
            }
        }
    }

    post {
        cleanup {
            cleanWs()
        }
    //     always {
    //         emailext (
    //             subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    //             body: """
    //                 <p>Jenkins Starbucks CI/CD Pipeline</p>
    //                 <p>Status: ${currentBuild.currentResult}</p>
    //                 <p>Build: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
    //             """,
    //             to: 'mohdhtrix@gmail.com',
    //             mimeType: 'text/html',
    //             attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
    //         )
    //     }
    }
}
