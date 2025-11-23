pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd6c1d36c-fd95-47e7-ab99-f35251321738'
        NETLIFY_AUTH_TOKEN = credentials('netlify_token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        /*************************
         * 1) BUILD DOCKER IMAGE
         *************************/
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "🛠️ Building custom Playwright + Netlify image..."
                    docker build -t my-playwright .
                '''
            }
        }

        /*************************
         * 2) BUILD REACT APP
         *************************/
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "📦 Building React application"
                    npm ci
                    npm run build
                    echo "✔ Build done"
                '''
            }
        }

        /*************************
         * 3) TESTS (PARALLEL)
         *************************/
        stage('Tests') {
            parallel {

                /********** Unit tests **********/
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "🧪 Running Jest tests"
                            test -f "build/index.html"
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                /********** E2E tests local **********/
                stage('E2E Local') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "🌐 Serving build locally"
                            serve -s build &
                            sleep 10

                            echo "🧪 Running Playwright E2E tests"
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: true,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Playwright Local'
                            ])
                        }
                    }
                }
            }
        }

        /*************************
         * 4) DEPLOY STAGING
         *************************/
        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'Coming'
            }
            steps {
                sh '''
                    echo "🚀 Deploying to STAGING"
                    netlify --version
                    netlify status

                    netlify deploy --dir=build --json > deploy-output.json

                    export CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    echo "🧪 Running Playwright E2E tests on staging"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E'
                    ])
                }
            }
        }

        /*************************
         * 5) DEPLOY PROD
         *************************/
        stage('Deploy Prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "https://golden-parfait-316a2f.netlify.app"
            }
            steps {
                sh '''
                    echo "🚀 Deploying to PRODUCTION"
                    netlify --version
                    netlify status

                    netlify deploy --dir=build --prod
                    echo "Prod URL: $CI_ENVIRONMENT_URL"

                    echo "🧪 Running Playwright E2E tests on production"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E'
                    ])
                }
            }
        }
    }
}
