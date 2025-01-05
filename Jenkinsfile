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
                echo "Checking out source code"
                checkout scm
            }
        }

        stage("Set Up Environment") {
            steps {
                echo "Setting up virtual environment"
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda init bash
                    conda create --yes -n ${BUILD_TAG} python=3.12 || exit 1
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} || exit 1
                    pip install -r requirements/dev.txt || exit 1
                '''
            }
        }

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis"
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG}
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
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG}
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
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG}
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
                echo "Building package"
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG}
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
            echo "Cleaning up environment"
            sh '''
                . /var/lib/jenkins/miniconda3/bin/activate
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

