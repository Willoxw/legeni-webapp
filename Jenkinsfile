pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKERHUB_ID',
            defaultValue: 'legeni07',
            description: 'Compte Docker Hub qui possède le dépôt legeni-webapp'
        )
        string(
            name: 'NOTIFICATION_CREDENTIAL_ID',
            defaultValue: '',
            description: 'ID Jenkins optionnel du webhook Discord (credential Secret text)'
        )
    }

    environment {
        ANSIBLE_CONFIG = 'ansible/ansible.cfg'
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Contrôles qualité') {
            parallel {
                stage('Tests et couverture') {
                    steps {
                        sh '''
                            set -eu
                            python3 -m venv .venv-tests
                            .venv-tests/bin/pip install -q -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            .venv-tests/bin/pytest -q \
                                --junitxml=reports/junit.xml \
                                --cov=app \
                                --cov-report=xml:reports/coverage.xml \
                                --cov-report=term \
                                --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                            recordCoverage(
                                tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']]
                            )
                        }
                    }
                }

                stage('Lint Python') {
                    steps {
                        sh '''
                            set -eu
                            python3 -m venv .venv-flake8
                            .venv-flake8/bin/pip install -q flake8==7.1.1
                            mkdir -p reports
                            .venv-flake8/bin/flake8 --output-file=reports/flake8.txt .
                        '''
                    }
                    post {
                        always {
                            recordIssues(
                                tools: [flake8(pattern: 'reports/flake8.txt')]
                            )
                        }
                    }
                }

                stage('Lint Dockerfile') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile'
                    }
                }

                stage('Syntaxe Ansible') {
                    steps {
                        sh '''
                            set -eu
                            VAULT_BACKUP=$(mktemp)
                            cp ansible/vars/vault.yml "$VAULT_BACKUP"
                            restore_vault() {
                                cp "$VAULT_BACKUP" ansible/vars/vault.yml
                                rm -f "$VAULT_BACKUP"
                            }
                            trap restore_vault EXIT
                            printf '%s\\n' \
                                'vault_redis_password: ci-syntax-only' \
                                'vault_jenkins_admin_password: ci-syntax-only' \
                                > ansible/vars/vault.yml
                            ansible-playbook -i ansible/inventory.ini \
                                ansible/provision.yml --syntax-check
                            ansible-playbook -i ansible/inventory.ini \
                                ansible/deploy.yml --syntax-check
                        '''
                    }
                }
            }
        }

        stage('Construire l’image') {
            steps {
                script {
                    env.SEMVER = readFile('VERSION').trim()
                    env.IMAGE_TAG = "${env.SEMVER}-${env.BUILD_NUMBER}"
                    env.IMAGE_REF = "${params.DOCKERHUB_ID}/legeni-webapp:${env.IMAGE_TAG}"
                }
                sh '''
                    docker build \
                        --build-arg APP_VERSION="$IMAGE_TAG" \
                        --tag "$IMAGE_REF" .
                '''
            }
        }

        stage('Scan Trivy') {
            steps {
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy_cache:/root/.cache/ \
                        aquasec/trivy:0.57.1 image \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --exit-code 1 \
                        "$IMAGE_REF"
                '''
            }
        }

        stage('Publier sur Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'registry-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        DOCKER_CONFIG=$(mktemp -d)
                        chmod 700 "$DOCKER_CONFIG"
                        export DOCKER_CONFIG
                        cleanup_docker_config() {
                            docker logout >/dev/null 2>&1 || true
                            rm -f "$DOCKER_CONFIG/config.json"
                            rmdir "$DOCKER_CONFIG"
                        }
                        trap cleanup_docker_config EXIT
                        if [ "$REGISTRY_USER" != "$DOCKERHUB_ID" ]; then
                            echo "DOCKERHUB_ID doit correspondre au credential registry-credentials." >&2
                            exit 1
                        fi
                        printf '%s' "$REGISTRY_TOKEN" |
                            docker login --username "$REGISTRY_USER" --password-stdin
                        docker push "$IMAGE_REF"
                    '''
                }
            }
        }

        stage('Déployer en staging') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    string(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_CONTENT'
                    ),
                    string(
                        credentialsId: 'ansible-vault-pass',
                        variable: 'ANSIBLE_VAULT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        VAULT_FILE=$(mktemp)
                        KUBECONFIG_FILE=$(mktemp)
                        trap 'rm -f "$VAULT_FILE" "$KUBECONFIG_FILE"' EXIT
                        chmod 600 "$VAULT_FILE" "$KUBECONFIG_FILE"
                        printf '%s' "$ANSIBLE_VAULT_PASSWORD" > "$VAULT_FILE"
                        printf '%s' "$KUBECONFIG_CONTENT" > "$KUBECONFIG_FILE"
                        export KUBECONFIG
                        KUBECONFIG="$KUBECONFIG_FILE"
                        ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
                            --extra-vars "target_env=staging app_image=$DOCKERHUB_ID/legeni-webapp app_version=$IMAGE_TAG" \
                            --vault-password-file "$VAULT_FILE"
                    '''
                }
            }
        }

        stage('Validation humaine avant production') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Staging est validé. Déployer ${env.IMAGE_REF} en production ?",
                        ok: 'Déployer en production'
                    )
                }
            }
        }

        stage('Déployer en production') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    string(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_CONTENT'
                    ),
                    string(
                        credentialsId: 'ansible-vault-pass',
                        variable: 'ANSIBLE_VAULT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        VAULT_FILE=$(mktemp)
                        KUBECONFIG_FILE=$(mktemp)
                        trap 'rm -f "$VAULT_FILE" "$KUBECONFIG_FILE"' EXIT
                        chmod 600 "$VAULT_FILE" "$KUBECONFIG_FILE"
                        printf '%s' "$ANSIBLE_VAULT_PASSWORD" > "$VAULT_FILE"
                        printf '%s' "$KUBECONFIG_CONTENT" > "$KUBECONFIG_FILE"
                        export KUBECONFIG
                        KUBECONFIG="$KUBECONFIG_FILE"
                        ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
                            --extra-vars "target_env=prod app_image=$DOCKERHUB_ID/legeni-webapp app_version=$IMAGE_TAG" \
                            --vault-password-file "$VAULT_FILE"
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline terminé pour ${env.JOB_NAME} #${env.BUILD_NUMBER}."
        }
        failure {
            echo "Échec de ${env.JOB_NAME} #${env.BUILD_NUMBER}; consulter la console Jenkins."
            script {
                if (env.BRANCH_NAME == 'main' && params.NOTIFICATION_CREDENTIAL_ID?.trim()) {
                    withCredentials([
                        string(
                            credentialsId: params.NOTIFICATION_CREDENTIAL_ID,
                            variable: 'NOTIFICATION_WEBHOOK'
                        )
                    ]) {
                        sh '''
                            set +x
                            MESSAGE="Pipeline échoué: $JOB_NAME #$BUILD_NUMBER - $BUILD_URL"
                            python3 -c 'import json,sys; print(json.dumps({"content":sys.argv[1]}))' \
                                "$MESSAGE" |
                                curl --fail --silent --show-error \
                                    -H 'Content-Type: application/json' \
                                    --data-binary @- "$NOTIFICATION_WEBHOOK"
                        '''
                    }
                } else {
                    echo 'Notification externe non configurée: définir un credential webhook Secret text.'
                }
            }
        }
    }
}
