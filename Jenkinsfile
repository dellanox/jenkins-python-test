pipeline {
    agent any

    triggers {
        pollSCM('*/5 * * * 1-7')
    }

    options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        PATH = "/var/lib/jenkins/miniconda3/bin:$PATH"
    }

    stages {
        stage("Checkout Code") {
            steps {
                checkout scm
            }
        }

        stage("Set Up Environment") {
            steps {
                echo "Creating virtual environment"
                sh '''
                    # Initialize Conda environment for the current shell
                    source /var/lib/jenkins/miniconda3/bin/activate && conda init bash
                    # Create a unique environment for this build
                    conda create --yes -n ${BUILD_TAG} python || exit 1
                    # Activate the newly created environment
                    source /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
                    # Install dependencies
                    pip install -r requirements/dev.txt || exit 1
                '''
            }
        }

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis"
                sh '''
                    source /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
                    radon raw --json irisvmpy > reports/raw_report.json || exit 1
                    radon cc --json irisvmpy > reports/cc_report.json || exit 1
                    radon mi --json irisvmpy > reports/mi_report.json || exit 1
                    sloccount --duplicates --wide irisvmpy > reports/sloccount.sc || exit 1
                '''
            }
            post {
                always {
                    echo "Publishing coverage reports"
                    step([$class: 'CoberturaPublisher',
                          coberturaReportFile: 'reports/coverage.xml',
                          failNoReports: false])
                }
            }
        }

        stage("Unit Tests") {
            steps {
                echo "Running unit tests"
                sh '''
                    source /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
                    python -m pytest --verbose --junit-xml reports/unit_tests.xml || exit 1
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
                    source /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
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
                echo "Building package"
                sh '''
                    source /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
                    python setup.py bdist_wheel || exit 1
                '''
            }
            post {
                always {
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*whl', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up environment"
            sh '''
                source /var/lib/jenkins/miniconda3/bin/activate
                conda remove --yes -n ${BUILD_TAG} --all || true
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

