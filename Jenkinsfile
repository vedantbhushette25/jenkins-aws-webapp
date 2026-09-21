pipeline {
    agent any

    environment {
        DEPLOY_USER = 'ubuntu'
        DEPLOY_HOST = '172.31.28.163'
        APP_DIR = '/opt/webapp'
        APP_PORT = '3000'
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                sh 'git log -1 --oneline'
                sh 'ls -la'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'node -v && npm -v'
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Package') {
            steps {
                sh 'tar -czf webapp-${BUILD_NUMBER}.tar.gz app.js package.json'
                archiveArtifacts artifacts: 'webapp-*.tar.gz', fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['app-server-key']) {
                    sh '''
                        OPTS="-o StrictHostKeyChecking=no"
                        TARGET="${DEPLOY_USER}@${DEPLOY_HOST}"

                        scp $OPTS webapp-${BUILD_NUMBER}.tar.gz $TARGET:/tmp/webapp.tar.gz

                        ssh $OPTS $TARGET "
                            tar -xzf /tmp/webapp.tar.gz -C ${APP_DIR} &&
                            cd ${APP_DIR} &&
                            npm install --omit=dev &&
                            sudo systemctl restart webapp &&
                            sleep 2 &&
                            sudo systemctl is-active webapp
                        "
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    URL=http://${DEPLOY_HOST}:${APP_PORT}/health

                    curl --fail --silent \
                    --retry 5 \
                    --retry-delay 2 \
                    --retry-connrefused \
                    $URL
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: build #${env.BUILD_NUMBER} deployed"
        }

        failure {
            echo 'FAILED: read the error in the failed stage above'
        }
    }
}
