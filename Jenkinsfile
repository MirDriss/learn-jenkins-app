pipeline {
    agent any
    
    stages {

        stage('Build'){

            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    ls -la 
                    node --version
                    npm --version
                    npm ci 
                    npm run build 
                    ls -la
                '''
            }
        }

        stage ('Test'){

            agent{
                reuseNode true
            }

            steps{
                
                sh '''
                    cd build
                    test index.html
                    npm test
                '''
            }
        }
    }
}
