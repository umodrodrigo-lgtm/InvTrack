pipeline {
    agent any

    environment {
        APP_NAME        = "inventory-management-system"
        GIT_BRANCH      = "main"

        // Jenkins credentials IDs
        SSH_CREDENTIALS = credential('SERVER_HOST')

        // AWS web server details
        SERVER_USER     = "ec2-user"
        SERVER_HOST     = "YOUR_WEB_SERVER_PUBLIC_IP_OR_DNS"
        DEPLOY_PATH     = "/var/www/inventory-management-system"

        // Optional if composer/npm are not globally available
        COMPOSER_BIN    = "/usr/local/bin/composer"
        PHP_BIN         = "/usr/bin/php"
        NPM_BIN         = "/usr/bin/npm"
    }

    triggers {
        githubPush()
    }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: "${GIT_BRANCH}",
                    url: "https://github.com/umodrodrigo-lgtm/InvTrack.git"
            }
        }

        stage('Prepare Build') {
            steps {
                sh '''
                    set -e

                    echo "Checking tools..."
                    php -v
                    ${COMPOSER_BIN} --version
                    node -v
                    ${NPM_BIN} -v

                    echo "Install PHP dependencies..."
                    ${COMPOSER_BIN} install --no-interaction --prefer-dist --no-progress

                    echo "Prepare env file for build..."
                    if [ ! -f .env ]; then
                      cp .env.example .env
                    fi

                    echo "Generate app key if missing..."
                    ${PHP_BIN} artisan key:generate --force || true

                    echo "Install frontend dependencies..."
                    ${NPM_BIN} install

                    echo "Build frontend assets..."
                    ${NPM_BIN} run build
                '''
            }
        }

        stage('Package Application') {
            steps {
                sh '''
                    set -e

                    echo "Cleaning unnecessary files..."
                    rm -rf deploy-package
                    mkdir -p deploy-package

                    rsync -av --delete \
                      --exclude=".git" \
                      --exclude=".github" \
                      --exclude="node_modules" \
                      --exclude="tests" \
                      --exclude="storage/logs" \
                      --exclude=".env" \
                      ./ deploy-package/

                    tar -czf ${APP_NAME}.tar.gz -C deploy-package .
                '''
            }
        }

        stage('Upload to Web Server') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -e

                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_HOST} "mkdir -p ${DEPLOY_PATH}"
                        scp -o StrictHostKeyChecking=no ${APP_NAME}.tar.gz ${SERVER_USER}@${SERVER_HOST}:/tmp/
                    '''
                }
            }
        }

        stage('Deploy on Web Server') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -e

                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_HOST} << EOF
                            set -e

                            sudo mkdir -p ${DEPLOY_PATH}
                            sudo chown -R ${SERVER_USER}:${SERVER_USER} ${DEPLOY_PATH}

                            cd ${DEPLOY_PATH}

                            tar -xzf /tmp/${APP_NAME}.tar.gz -C ${DEPLOY_PATH}
                            rm -f /tmp/${APP_NAME}.tar.gz

                            if [ ! -f .env ]; then
                                cp .env.example .env
                            fi

                            if [ ! -d storage/framework/cache ]; then mkdir -p storage/framework/cache; fi
                            if [ ! -d storage/framework/sessions ]; then mkdir -p storage/framework/sessions; fi
                            if [ ! -d storage/framework/views ]; then mkdir -p storage/framework/views; fi
                            if [ ! -d storage/logs ]; then mkdir -p storage/logs; fi

                            composer install --no-dev --optimize-autoloader --no-interaction

                            php artisan key:generate --force || true
                            php artisan storage:link || true
                            php artisan migrate --force
                            php artisan config:cache
                            php artisan route:cache
                            php artisan view:cache

                            sudo chown -R nginx:nginx ${DEPLOY_PATH} || true
                            sudo chmod -R 775 storage bootstrap/cache || true

                            sudo systemctl reload nginx || true
                            sudo systemctl restart php-fpm || true
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}