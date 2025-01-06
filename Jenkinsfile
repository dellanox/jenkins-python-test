pipeline {
    agent any

    triggers {
        pollSCM('*/5 * * * 1-7') // Triggers the pipeline to check the SCM every 5 minutes, every day of the week.
    }

    options {
        skipDefaultCheckout(true) // Prevents the default checkout of the repository into the workspace.
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keeps only the last 10 builds to save storage.
        timestamps() // Adds timestamps to console output for better debugging.
    }

    environment {
        PATH = "/var/lib/jenkins/miniconda3/bin:$PATH" // Ensures the PATH includes the Miniconda binary location.
    }

    stages {
        stage("Initialize Build Environment") {
            steps {
                echo "Initializing global build environment"
                sh '''
                    #!/bin/bash
                    export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

                    # Remove any pre-existing environment for cleanup
                    conda env remove --yes -n build_env || true

                    # Create a global build environment
                    conda create --yes -n build_env python=3.8

                    # Activate the global environment and install base dependencies
                    . /var/lib/jenkins/miniconda3/bin/activate build_env
                    pip install -r requirements/dev.txt
                '''
            }
        }

        stage("Checkout Code") {
            steps {
                echo "Checking out source code" // Logs the current operation for visibility.
                checkout scm // Checks out the code from the source control repository configured in the job.
            }
        }
       
        stage("Set Up Environment") {
            steps {
                echo "Setting up stage-specific virtual environment"
                sh '''
                    #!/bin/bash
                    export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

                    # Remove the stage-specific environment if it exists
                    conda env remove --yes -n ${BUILD_TAG} || true

                    # Clone the global environment to stage-specific
                    conda create --yes --name ${BUILD_TAG} --clone build_env

                    # Activate the stage-specific environment
                    . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
                '''
            }
        }

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis"
                sh '''
                    #!/bin/bash
                    . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
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
                    . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
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
                    . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
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
                    #!/bin/bash
                    . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
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
            echo "Cleaning up environments"
            sh '''
                #!/bin/bash
                . /var/lib/jenkins/miniconda3/bin/activate
                conda remove --yes -n build_env --all || true
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

