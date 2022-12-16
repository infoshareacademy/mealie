pipeline {
    agent {
        label 'node01'
    }
    environment { 
        DB_ENGINE = 'postgres'
        POSTGRES_SERVER = 'localhost'
    }
    tools {
      nodejs 'nodejs'
    }
    stages {
        stage('Install dependencies') {
            steps {
                script {
                    container = docker.image('postgres').run('-e POSTGRES_USER=mealie -e POSTGRES_PASSWORD=mealie -e POSTGRES_DB=mealie --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5 -p 5432:5432')
                }
                sh '''
                    sudo apt update
                    sudo apt install python3 make libsasl2-dev libldap2-dev libssl-dev tesseract-ocr-all -y
                    curl -sSL https://install.python-poetry.org | python3 -
                    export PATH=/var/lib/jenkins/.local/bin:$PATH
                    pip install psycopg2-binary
                    poetry env use python3.10
                    poetry install
                    npm i --global yarn
                    cd frontend
                    yarn
                '''
            }
        }
        stage('Tests') {
            steps {
                sh '''
                    cd frontend
                    yarn test
                    cd ..
                    export PATH=/var/lib/jenkins/.local/bin:$PATH
                    make backend-test
                '''
                snykSecurity(
                  snykInstallation: 'snyk',
                  snykTokenId: 'snyk',
                  failOnIssues: false,
                  additionalArguments: '--all-projects'
                )
            }
            post {
              always {
                junit 'test-report.xml'
              }
            }
        }
        stage('SonarQube Analysis') {
            tools {
              nodejs 'nodejs'
            }
            parameters {
              booleanParam defaultValue: false, description: 'Do you want to run Sonarqube test?', name: 'enableSonarqubeScan'
            }
            when { expression { params.enableSonarqubeScan == true } }
            steps {
              script {
                scannerHome = tool 'SonarScanner';
                withSonarQubeEnv('sonar') {
                  sh 'npm i postcss-sass --save'
                  sh "${scannerHome}/bin/sonar-scanner"
                }
              }
            }
        }
        stage('Build Docker') {
            environment {
                user = "msl0"
                registryCredentialsId = "dockerhub"
            }
            steps {
              script {
                dockerImageFrontend = docker.build(user +"/mealieFront" + ":$BUILD_NUMBER", '-f frontend/Dockerfile')
                dockerImageBackend = docker.build(user +"/mealieBackend" + ":$BUILD_NUMBER", '--target production')
                docker.withRegistry('', registryCredentialsId) {
                    dockerImageFrontend.push()
                    dockerImageBackend.push()
                }
              }
            }
        }
        stage('Run DEV') {
            steps {
              make docker-dev
            }
            post { 
                always { 
                    input 'Is the application working properly?'
                }
            }
        }
        stage('Run PROD') {
            steps {
              make docker-prod
            }
        }
    }
    post { 
        always { 
            script {
                container.stop()
            }
        }
    }
}
