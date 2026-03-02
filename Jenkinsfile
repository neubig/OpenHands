// OpenHands CI Pipeline for Jenkins
// This Jenkinsfile mirrors the GitHub Actions workflows for CI/CD

pipeline {
    agent any

    options {
        // Discard old builds to save space
        buildDiscarder(logRotator(numToKeepStr: '20'))
        // Timeout for the entire pipeline
        timeout(time: 60, unit: 'MINUTES')
        // Add timestamps to console output
        timestamps()
        // Skip checkout on restart
        skipStagesAfterUnstable()
    }

    environment {
        // Python version
        PYTHON_VERSION = '3.12'
        // Node.js version
        NODE_VERSION = '22'
        // Poetry home
        POETRY_HOME = "${WORKSPACE}/.poetry"
        PATH = "${POETRY_HOME}/bin:${env.PATH}"
    }

    triggers {
        // Run on push to main
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup') {
            parallel {
                stage('Setup Python') {
                    steps {
                        sh '''
                            # Install Poetry if not present
                            if ! command -v poetry &> /dev/null; then
                                curl -sSL https://install.python-poetry.org | python3 -
                            fi
                            poetry --version
                        '''
                    }
                }
                stage('Setup Node.js') {
                    steps {
                        sh '''
                            # Verify Node.js is available
                            node --version
                            npm --version
                        '''
                    }
                }
            }
        }

        stage('Lint') {
            parallel {
                stage('Lint Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                npm ci --frozen-lockfile
                                npm run lint
                                npm run make-i18n
                                npx tsc
                                npm run check-translation-completeness
                            '''
                        }
                    }
                }
                stage('Lint Python') {
                    steps {
                        sh '''
                            pip install pre-commit==3.7.0
                            pre-commit run --all-files --show-diff-on-failure --config ./dev_config/python/.pre-commit-config.yaml
                        '''
                    }
                }
                stage('Lint Enterprise Python') {
                    steps {
                        dir('enterprise') {
                            sh '''
                                pip install pre-commit==4.2.0
                                pre-commit run --all-files --show-diff-on-failure --config ./dev_config/python/.pre-commit-config.yaml
                            '''
                        }
                    }
                }
            }
        }

        stage('Build') {
            parallel {
                stage('Build Backend') {
                    steps {
                        sh '''
                            poetry install --with dev,test,runtime
                            make build
                        '''
                    }
                }
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                npm ci
                                npm run build
                            '''
                        }
                    }
                }
                stage('Build UI Components') {
                    when {
                        anyOf {
                            branch 'main'
                            changeset 'openhands-ui/**'
                        }
                    }
                    steps {
                        dir('openhands-ui') {
                            sh '''
                                # Install bun if not present
                                if ! command -v bun &> /dev/null; then
                                    curl -fsSL https://bun.sh/install | bash
                                    export PATH="$HOME/.bun/bin:$PATH"
                                fi
                                bun install --frozen-lockfile
                                bun run build
                            '''
                        }
                    }
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Python Unit Tests') {
                    steps {
                        sh '''
                            poetry run pip install pytest-xdist pytest-rerunfailures
                            PYTHONPATH=".:$PYTHONPATH" poetry run pytest \
                                --forked -n auto -s \
                                ./tests/unit \
                                --cov=openhands \
                                --cov-branch \
                                --cov-report=xml:coverage-unit.xml \
                                --junitxml=test-results-unit.xml
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'test-results-unit.xml'
                            publishCoverage adapters: [coberturaAdapter('coverage-unit.xml')]
                        }
                    }
                }
                stage('Runtime Tests') {
                    steps {
                        sh '''
                            PYTHONPATH=".:$PYTHONPATH" TEST_RUNTIME=cli poetry run pytest \
                                -n 5 --reruns 2 --reruns-delay 3 -s \
                                tests/runtime/test_bash.py \
                                --cov=openhands \
                                --cov-branch \
                                --cov-report=xml:coverage-runtime.xml \
                                --junitxml=test-results-runtime.xml
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'test-results-runtime.xml'
                            publishCoverage adapters: [coberturaAdapter('coverage-runtime.xml')]
                        }
                    }
                }
                stage('Enterprise Unit Tests') {
                    steps {
                        dir('enterprise') {
                            sh '''
                                poetry install --with dev,test
                            '''
                        }
                        sh '''
                            PYTHONPATH=".:$PYTHONPATH" poetry run --project=enterprise pytest \
                                --forked -n auto -s \
                                -p no:ddtrace -p no:ddtrace.pytest_bdd -p no:ddtrace.pytest_benchmark \
                                ./enterprise/tests/unit \
                                --cov=enterprise \
                                --cov-branch \
                                --cov-report=xml:coverage-enterprise.xml \
                                --junitxml=test-results-enterprise.xml
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'test-results-enterprise.xml'
                            publishCoverage adapters: [coberturaAdapter('coverage-enterprise.xml')]
                        }
                    }
                }
                stage('Frontend Unit Tests') {
                    when {
                        anyOf {
                            branch 'main'
                            changeset 'frontend/**'
                        }
                    }
                    steps {
                        dir('frontend') {
                            sh '''
                                npm ci
                                npm run build
                                npm run test:coverage
                            '''
                        }
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'frontend/coverage/junit.xml'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up workspace
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            // Uncomment to enable email notifications
            // mail to: 'team@example.com',
            //      subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
            //      body: "Something went wrong with ${env.BUILD_URL}"
        }
        unstable {
            echo 'Pipeline is unstable (tests may have failed).'
        }
    }
}
