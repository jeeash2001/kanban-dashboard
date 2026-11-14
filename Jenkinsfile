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
        CONTAINER_NAME = 'kanban-dashboard'
        CANDIDATE_NAME = 'kanban-candidate'
        APP_PORT = '18082'
        CANDIDATE_PORT = '18080'
        MEMORY_LIMIT = '256m'
        CPU_LIMIT = '0.50'
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

                    docker rm -f kanban-build-test 2>/dev/null || true

                    cleanup() {
                        docker rm -f kanban-build-test \
                          >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                      --name kanban-build-test \
                      -p 127.0.0.1:18081:8080 \
                      "${IMAGE}"

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                          http://127.0.0.1:18081/health \
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

        stage('Deploy and Verify') {
            steps {
                sh '''
                    set -eu

                    OLD_IMAGE=""

                    if docker inspect "${CONTAINER_NAME}" \
                      >/dev/null 2>&1; then
                        OLD_IMAGE=$(docker inspect \
                          --format='{{.Config.Image}}' \
                          "${CONTAINER_NAME}")
                    fi

                    # Start candidate on a temporary loopback port.
                    docker rm -f "${CANDIDATE_NAME}" \
                      >/dev/null 2>&1 || true

                    docker run -d \
                      --name "${CANDIDATE_NAME}" \
                      --memory="${MEMORY_LIMIT}" \
                      --cpus="${CPU_LIMIT}" \
                      -p 127.0.0.1:${CANDIDATE_PORT}:8080 \
                      "${IMAGE}"

                    CANDIDATE_OK=false

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                          http://127.0.0.1:${CANDIDATE_PORT}/health \
                          >/dev/null; then
                            CANDIDATE_OK=true
                            break
                        fi

                        sleep 2
                    done

                    if [ "${CANDIDATE_OK}" != "true" ]; then
                        echo "Candidate health check failed."
                        docker logs "${CANDIDATE_NAME}" || true
                        docker rm -f "${CANDIDATE_NAME}" || true
                        exit 1
                    fi

                    # Candidate is healthy; replace current version.
                    docker stop "${CONTAINER_NAME}" \
                      >/dev/null 2>&1 || true

                    docker rm "${CONTAINER_NAME}" \
                      >/dev/null 2>&1 || true

                    docker rm -f "${CANDIDATE_NAME}" \
                      >/dev/null 2>&1 || true

                    if docker run -d \
                      --name "${CONTAINER_NAME}" \
                      --restart unless-stopped \
                      --memory="${MEMORY_LIMIT}" \
                      --cpus="${CPU_LIMIT}" \
                      -p 127.0.0.1:${APP_PORT}:8080 \
                      "${IMAGE}"; then
                        echo "New container started."
                    else
                        echo "New container failed to start."
                    fi

                    # Verify production endpoint.
                    DEPLOY_OK=false

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                          http://127.0.0.1:${APP_PORT}/health \
                          >/dev/null; then
                            DEPLOY_OK=true
                            break
                        fi

                        sleep 2
                    done

                    if [ "${DEPLOY_OK}" != "true" ]; then
                        echo "Production health check failed."
                        docker logs "${CONTAINER_NAME}" || true
                        docker rm -f "${CONTAINER_NAME}" \
                          >/dev/null 2>&1 || true

                        # Restore previous image if available.
                        if [ -n "${OLD_IMAGE}" ]; then
                            echo "Rolling back to ${OLD_IMAGE}"

                            docker run -d \
                              --name "${CONTAINER_NAME}" \
                              --restart unless-stopped \
                              --memory="${MEMORY_LIMIT}" \
                              --cpus="${CPU_LIMIT}" \
                              -p 127.0.0.1:${APP_PORT}:8080 \
                              "${OLD_IMAGE}"

                            for attempt in $(seq 1 10); do
                                if curl -fsS \
                                  http://127.0.0.1:${APP_PORT}/health \
                                  >/dev/null; then
                                    echo "Rollback succeeded."
                                    break
                                fi

                                sleep 2
                            done
                        fi

                        exit 1
                    fi

                    echo "Production health check passed."

                    docker inspect \
                      --format='Container status: {{.State.Status}}' \
                      "${CONTAINER_NAME}"

                    docker ps --filter "name=${CONTAINER_NAME}"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD SUCCESS: Application deployed and verified.'
        }

        failure {
            echo 'CI/CD FAILED: Review the failing stage and console output.'
        }

        always {
            sh '''
                docker rm -f kanban-build-test \
                  >/dev/null 2>&1 || true

                docker rm -f kanban-candidate \
                  >/dev/null 2>&1 || true
            '''

            echo 'Pipeline execution finished.'
        }
    }
}
