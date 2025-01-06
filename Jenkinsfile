pipeline {
    agent any

    triggers {
        pollSCM('*/5 * * * 1-7') // Polls the SCM every 5 minutes, daily.
    }

    options {
        skipDefaultCheckout(true) // Prevents default repository checkout.
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keeps the last 10 builds.
        timestamps() // Adds timestamps to console logs.
    }

    environment {
        PATH = "/var/lib/jenkins/miniconda3/bin:$PATH" // Includes Miniconda binaries in PATH.
        PYTHON_VERSION = "3.11" // Target Python version for all operations.
        VENV_DIR = "venv" // Directory for the virtual environment.
    }

    stages {
        stage("Checkout Code") {
            steps {
                echo "Checking out source code"
                checkout scm
            }
        }

        stage("Set Up Virtual Environment") {
            steps {
                echo "Setting up virtual environment with Python ${env.PYTHON_VERSION}"
                sh '''
                    #!/bin/bash
                    # Clean up any existing virtual environment
                    if [ -d "${VENV_DIR}" ]; then
                        rm -rf "${VENV_DIR}"
                    fi

                    # Create and activate a new virtual environment
                    python${PYTHON_VERSION} -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate

                    # Upgrade pip and install dependencies
                    pip install --upgrade pip
                    pip install --requirement requirements/dev.txt --only-binary=:all:
                '''
            }
        }

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis"
                sh '''
                    #!/bin/bash
                    . ${VENV_DIR}/bin/activate
                    mkdir -p reports
                    radon raw --json irisvmpy > reports/raw_report.json
                    radon cc --json irisvmpy > reports/cc_report.json
                    radon mi --json irisvmpy > reports/mi_report.json
                    sloccount --duplicates --wide irisvmpy > reports/sloccount.sc
                '''
            }
        }

        stage("Unit Tests") {
            steps {
                echo "Running unit tests"
                sh '''
                    #!/bin/bash
                    . ${VENV_DIR}/bin/activate
                    mkdir -p reports
                    python -m pytest --verbose --junit-xml reports/unit_tests.xml
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/unit_tests.xml'
                }
            }
        }

        stage("Acceptance Tests") {
            steps {
                echo "Running acceptance tests"
                sh '''
                    #!/bin/bash
                    . ${VENV_DIR}/bin/activate
                    mkdir -p reports
                    behave -f formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true
                '''
            }
            post {
                always {
                    cucumber(buildStatus: 'SUCCESS',
                             fileIncludePattern: '**/*.json',
                             jsonReportDirectory: './reports/',
                             parallelTesting: true,
                             sortingMethod: 'ALPHABETICAL')
                }
            }
        }

        stage("Build Package") {
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                echo "Building package with Python ${env.PYTHON_VERSION}"
                sh '''
                    #!/bin/bash
                    . ${VENV_DIR}/bin/activate
                    python setup.py bdist_wheel
                '''
            }
            post {
                always {
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*.whl', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up virtual environment"
            sh '''
                #!/bin/bash
                if [ -d "${VENV_DIR}" ]; then
                    rm -rf "${VENV_DIR}"
                fi
            '''
        }

        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'</p>
                     <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}


