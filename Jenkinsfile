
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    triggers {
        githubPush()
    }

    environment {
        REGISTRY_CREDENTIALS = 'dockerhub-creds'

        MEMORY_LIMIT = '256m'
        CPU_LIMIT = '0.50'

        BLUE_PORT = '18082'
        GREEN_PORT = '18083'
        TEST_PORT = '18081'
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm

                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=8 HEAD',
                        returnStdout: true
                    ).trim()
                }

                echo 'Source code checked out successfully.'
            }
        }

        stage('Prepare Image Version') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${REGISTRY_CREDENTIALS}",
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    script {
                        env.IMAGE_REPO =
                            "${env.DOCKERHUB_USER}/kanban-dashboard"

                        env.IMAGE_TAG =
                            "${env.BUILD_NUMBER}-${env.GIT_SHA}"

                        env.IMAGE =
                            "${env.IMAGE_REPO}:${env.IMAGE_TAG}"
                    }

                    echo "Image version: ${env.IMAGE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      --pull \
                      --label "org.opencontainers.image.revision=${GIT_SHA}" \
                      --label "org.opencontainers.image.version=${BUILD_NUMBER}" \
                      -t "${IMAGE}" \
                      .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker rm -f kanban-build-test \
                      >/dev/null 2>&1 || true

                    cleanup() {
                        docker rm -f kanban-build-test \
                          >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                      --name kanban-build-test \
                      --memory="${MEMORY_LIMIT}" \
                      --cpus="${CPU_LIMIT}" \
                      -p 127.0.0.1:${TEST_PORT}:8080 \
                      "${IMAGE}"

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                          http://127.0.0.1:${TEST_PORT}/health \
                          >/dev/null; then

                            echo "Image test passed."
                            exit 0
                        fi

                        sleep 2
                    done

                    echo "Image test failed."
                    docker logs kanban-build-test
                    exit 1
                '''
            }
        }

        stage('Login and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${REGISTRY_CREDENTIALS}",
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        echo "$DOCKERHUB_TOKEN" |
                          docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker push "${IMAGE}"

                        docker logout
                    '''
                }
            }
        }

        stage('Pull Image from Registry') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${REGISTRY_CREDENTIALS}",
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        echo "$DOCKERHUB_TOKEN" |
                          docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker pull "${IMAGE}"

                        docker logout
                    '''
                }
            }
        }

        stage('Blue-Green Deployment') {
            steps {
                sh '''
                    set -eu

                    NGINX_UPSTREAM="/etc/nginx/conf.d/kanban-upstream.conf"

                    ACTIVE_PORT=$(sed -n \
                      's/.*127.0.0.1:\\([0-9][0-9]*\\);.*/\\1/p' \
                      "$NGINX_UPSTREAM")

                    if [ "$ACTIVE_PORT" = "$BLUE_PORT" ]; then
                        OLD_PORT="$BLUE_PORT"
                        NEW_PORT="$GREEN_PORT"

                        OLD_NAME="kanban-blue"
                        NEW_NAME="kanban-green"

                        # One-time migration of existing production name.
                        if ! docker inspect kanban-blue \
                          >/dev/null 2>&1 && \
                           docker inspect kanban-dashboard \
                          >/dev/null 2>&1; then

                            docker rename kanban-dashboard kanban-blue
                        fi

                    elif [ "$ACTIVE_PORT" = "$GREEN_PORT" ]; then
                        OLD_PORT="$GREEN_PORT"
                        NEW_PORT="$BLUE_PORT"

                        OLD_NAME="kanban-green"
                        NEW_NAME="kanban-blue"

                    else
                        echo "Unexpected active Nginx port: $ACTIVE_PORT"
                        exit 1
                    fi

                    echo "Current production: $OLD_NAME on $OLD_PORT"
                    echo "New deployment: $NEW_NAME on $NEW_PORT"

                    # Ensure the old production container is running.
                    if [ "$(docker inspect \
                      --format='{{.State.Running}}' "$OLD_NAME")" \
                      != "true" ]; then

                        echo "Existing production container is not running."
                        exit 1
                    fi

                    # Remove stale inactive container, if present.
                    docker rm -f "$NEW_NAME" \
                      >/dev/null 2>&1 || true

                    # Start new version alongside old production.
                    docker run -d \
                      --name "$NEW_NAME" \
                      --restart unless-stopped \
                      --memory="${MEMORY_LIMIT}" \
                      --cpus="${CPU_LIMIT}" \
                      -p 127.0.0.1:${NEW_PORT}:8080 \
                      "${IMAGE}"

                    # Check new container health before switching traffic.
                    CANDIDATE_OK=false

                    for attempt in $(seq 1 30); do
                        if curl -fsS \
                          "http://127.0.0.1:${NEW_PORT}/health" \
                          >/dev/null; then

                            CANDIDATE_OK=true
                            break
                        fi

                        sleep 2
                    done

                    if [ "$CANDIDATE_OK" != "true" ]; then
                        echo "New container health check failed."

                        docker logs "$NEW_NAME" || true
                        docker rm -f "$NEW_NAME" || true

                        exit 1
                    fi

                    echo "New container is healthy."

                    # Switch Nginx to new production.
                    if ! sudo -n /usr/local/sbin/switch-kanban-upstream \
                      "$NEW_PORT"; then

                        echo "Nginx traffic switch failed."
                        docker rm -f "$NEW_NAME" || true
                        exit 1
                    fi

                    # Verify application through Nginx.
                    DEPLOY_OK=false

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                          http://127.0.0.1/health \
                          >/dev/null; then

                            DEPLOY_OK=true
                            break
                        fi

                        sleep 2
                    done

                    # Roll back if public application validation fails.
                    if [ "$DEPLOY_OK" != "true" ]; then
                        echo "New production validation failed."
                        echo "Switching traffic back to old production."

                        sudo -n /usr/local/sbin/switch-kanban-upstream \
                          "$OLD_PORT"

                        docker logs "$NEW_NAME" || true

                        docker rm -f "$NEW_NAME" || true

                        if curl -fsS \
                          http://127.0.0.1/health >/dev/null; then
                            echo "Rollback successful. Old version is serving."
                        else
                            echo "WARNING: Old version also failed validation."
                        fi

                        exit 1
                    fi

                    echo "New production deployment successful."

                    # Allow a short period for existing requests to drain.
                    sleep 5

                    # Remove old version only after successful validation.
                    docker rm -f "$OLD_NAME"

                    echo "Old container removed."
                    echo "Blue-Green deployment completed."

                    docker ps
                    docker inspect \
                      --format='Health: {{.State.Health.Status}}' \
                      "$NEW_NAME"

                    docker inspect \
                      --format='Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}}' \
                      "$NEW_NAME"

                    curl -fsS http://127.0.0.1/health
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD SUCCESS: Blue-Green deployment completed.'
        }

        failure {
            echo 'CI/CD FAILED: Check console output and rollback logs.'
        }

        always {
            sh '''
                docker rm -f kanban-build-test \
                  >/dev/null 2>&1 || true
            '''

            echo 'Pipeline execution finished.'
        }
    }
}
