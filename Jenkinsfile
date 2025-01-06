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
        PYTHON_VERSION = "3.8" // Target Python version for all operations.
    }

    stages {
        stage("Initialize Global Environment") {
            steps {
                echo "Initializing global build environment with Python ${env.PYTHON_VERSION}"
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                    sh '''
                        #!/bin/bash
                        export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

                        # Clean up existing global environment
                        conda env remove --yes -n build_env || true

                        # Create and configure global build environment with Python 3.8
                        conda create --yes -n build_env python=$PYTHON_VERSION
                        . /var/lib/jenkins/miniconda3/bin/activate build_env
                        pip install --upgrade pip
                        pip install --requirement requirements/dev.txt --only-binary=:all:
                    '''
                }
            }
        }

        stage("Checkout Code") {
            steps {
                echo "Checking out source code"
                checkout scm
            }
        }

        stage("Set Up Stage-Specific Environment") {
            steps {
                echo "Setting up stage-specific environment with Python ${env.PYTHON_VERSION}"
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                    sh '''
                        #!/bin/bash
                        export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

                        # Remove existing stage-specific environment if present
                        conda env remove --yes -n ${BUILD_TAG} || true

                        # Clone the global environment to stage-specific
                        conda create --yes --name ${BUILD_TAG} --clone build_env
                        . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
                    '''
                }
            }
        }

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis"
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
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
        }

        stage("Unit Tests") {
            steps {
                echo "Running unit tests"
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                    sh '''
                        #!/bin/bash
                        . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
                        mkdir -p reports
                        python -m pytest --verbose --junit-xml reports/unit_tests.xml
                    '''
                }
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
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                    sh '''
                        #!/bin/bash
                        . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
                        mkdir -p reports
                        behave -f formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true
                    '''
                }
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
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                    sh '''
                        #!/bin/bash
                        . /var/lib/jenkins/miniconda3/bin/activate ${BUILD_TAG}
                        python setup.py bdist_wheel
                    '''
                }
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
            wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
                sh '''
                    #!/bin/bash
                    export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

                    # Cleanup environments
                    conda remove --yes -n build_env --all || true
                    conda remove --yes -n ${BUILD_TAG} --all || true
                '''
            }
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

