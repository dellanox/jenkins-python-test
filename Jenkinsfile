pipeline {
    agent any

    triggers {
        pollSCM('*/5 * * * 1-5')
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
        stage("Code Pull") {
            steps {
                echo "Pulling code from SCM"
                checkout scm
            }
        }

        stage('Verify Conda Presence') {
            steps {
                echo "Checking if Conda is available"
                sh '''
                if ! command -v conda &> /dev/null; then
                    echo "Conda is not installed or not in PATH. Please ensure Conda is installed."
                    exit 1
                fi
                conda --version
                '''
            }
        }

        stage('Build Environment') {
            steps {
                echo "Setting up Conda environment and updating dependencies"
                sh '''
                # Ensure Conda is up-to-date
                echo "Updating Conda..."
                conda update -n base -c defaults conda -y

                # Remove existing environment if present
                if conda info --envs | grep -q 'jenkins-env'; then
                    echo "Removing existing environment..."
                    conda remove --yes -n jenkins-env --all
                fi

                # Create a clean environment and activate it
                echo "Creating a new environment..."
                conda create --yes -n jenkins-env python=3.8
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env

                # Upgrade pip
                echo "Upgrading pip..."
                pip install --upgrade pip

                # Install dependencies
                echo "Installing dependencies from requirements..."
                pip install --upgrade -r requirements/dev.txt
                '''
            }
        }

        stage('Static Code Metrics') {
            steps {
                echo "Collecting raw metrics"
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                radon raw --json irisvmpy > raw_report.json
                radon cc --json irisvmpy > cc_report.json
                radon mi --json irisvmpy > mi_report.json
                sloccount --duplicates --wide irisvmpy > sloccount.sc
                '''
                echo "Generating test coverage report"
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                coverage run irisvmpy/iris.py 1 1 2 3
                python -m coverage xml -o reports/coverage.xml
                '''
                echo "Performing style check"
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                pylint irisvmpy || true
                '''
            }
            post {
                always {
                    step([$class: 'CoberturaPublisher',
                        coberturaReportFile: 'reports/coverage.xml',
                        failNoReports: false
                    ])
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                python -m pytest --verbose --junit-xml reports/unit_tests.xml
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/unit_tests.xml'
                }
            }
        }

        stage('Acceptance Tests') {
            steps {
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                behave -f=formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true
                '''
            }
            post {
                always {
                    cucumber(
                        buildStatus: 'SUCCESS',
                        fileIncludePattern: '**/*.json',
                        jsonReportDirectory: './reports/',
                        sortingMethod: 'ALPHABETICAL'
                    )
                }
            }
        }

        stage('Build Package') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                sh '''
                source /var/lib/jenkins/miniconda3/etc/profile.d/conda.sh
                conda activate jenkins-env
                python setup.py bdist_wheel
                '''
            }
            post {
                always {
                    archiveArtifacts(
                        artifacts: 'dist/*.whl',
                        fingerprint: true
                    )
                }
            }
        }

        // Uncomment for deployment to PyPI
        // stage("Deploy to PyPI") {
        //     steps {
        //         sh '''
        //         twine upload dist/*
        //         '''
        //     }
        // }
    }

    post {
        always {
            sh '''
            conda remove --yes -n jenkins-env --all || true
            '''
        }
        failure {
            emailext(
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'</p>
                <p>Check console output at: <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>
                """,
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}

