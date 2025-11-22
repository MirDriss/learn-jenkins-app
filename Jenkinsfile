pipeline {
    agent any              // On peut exécuter la pipeline sur n'importe quel agent Jenkins

    environment {
        // Variables d'environnement utilisées pour le déploiement Netlify
        NETLIFY_SITE_ID = 'd6c1d36c-fd95-47e7-ab99-f35251321738'
        
        // Récupère le token Netlify depuis les credentials Jenkins
        NETLIFY_AUTH_TOKEN = credentials('netlify_token')

    }
    
    stages {

        /*****************************
         *         STAGE 1 : BUILD
         *****************************/
        stage('Build') {

            agent {
                docker {
                    image 'node:18-alpine'    // Utilisation d’un container Docker Node 18
                    reuseNode true            // Réutilise le workspace du node Jenkins
                }
            }

            steps {
                sh '''
                    echo "Small change"
                    ls -la                     # Liste les fichiers (juste pour debug)
                    node --version            # Vérifie version Node
                    npm --version             # Vérifie version npm
                    
                    npm ci                    # Installe les dépendances propres --> package-lock.json
                    npm run build             # Compile ton projet → crée le dossier build/
                    
                    ls -la                    # Revérifie les fichiers après build
                '''
            }
        }

        /*****************************
         *         STAGE 2 : TESTS
         *****************************/
        stage ('Tests') {

            parallel {   // Exécute les tests en parallèle → gain de temps !

                /************* Tests Unitaires (Jest) *************/
                stage ('Unit test') {

                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            echo "Bien arrivé"
                            
                            test -f "build/index.html"   # Vérifie que le build existe
                            
                            npm test                     # Lance les tests unitaires Jest
                        '''
                    }

                    post {
                        always {
                            junit 'jest-results/junit.xml'  //Publie les résultats Jest dans Jenkins
                        }
                    }
                }

                /************* Tests End-to-End (Playwright) *************/
                stage ('E2E') {

                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-focal'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve                   # Installe un serveur statique
                            
                            node_modules/.bin/serve -s build &  # Lance ton site en background
                            sleep 10                             # Attend que le site démarre
                            
                            npx playwright test --reporter=html # Lance les tests E2E + génère un rapport HTML
                        '''
                    }

                    post {
                        always {
                            // Publie le rapport HTML Playwright dans Jenkins
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                icon: '',
                                keepAll: false,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Playwright Local',
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

         stage('Deploy staging') {

            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    npm install netlify-cli@20.1.1      # Installe CLI Netlify
                    npm install node-jq

                    node_modules/.bin/netlify --version # Vérifie version
                    
                    echo "Deploying to staging. SITE_ID = $NETLIFY_SITE_ID"
                    
                    node_modules/.bin/netlify status     # Vérifie connexion au compte Netlify
                    
                    # Déploie le dossier build dans Netlify en mode stagging
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                    
                '''

                script {
                    env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                }
            }

        }

        stage ('Stagging E2E') {

            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-focal'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL= "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test --reporter=html # Lance les tests E2E + génère un rapport HTML
                '''
            }

            post {
                always {
                    // Publie le rapport HTML Playwright dans Jenkins
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        icon: '',
                        keepAll: false,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Stagging E2E',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        /*****************************
         *         STAGE 3 : DEPLOY
         *****************************/
        stage('Deploy prod') {

            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    npm install netlify-cli@20.1.1      # Installe CLI Netlify
                    node_modules/.bin/netlify --version # Vérifie version
                    
                    echo "Deploying to production. SITE_ID = $NETLIFY_SITE_ID"
                    
                    node_modules/.bin/netlify status     # Vérifie connexion au compte Netlify
                    
                    # Déploie le dossier build dans Netlify en mode production
                    node_modules/.bin/netlify deploy --dir=build 
                '''
            }
        }

        stage ('Approval'){

            steps {

                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production ? ', ok: 'Yes, I am sur !'
                }

                
            }

        }

        stage ('Prod E2E') {

            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-focal'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL= "https://golden-parfait-316a2f.netlify.app"
            }

            steps {
                sh '''
                    npx playwright test --reporter=html # Lance les tests E2E + génère un rapport HTML
                '''
            }

            post {
                always {
                    // Publie le rapport HTML Playwright dans Jenkins
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        icon: '',
                        keepAll: false,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
