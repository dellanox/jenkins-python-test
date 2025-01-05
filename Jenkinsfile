pipeline {
    agent any

    // Trigger the pipeline based on SCM polling (every 5 minutes, Monday to Friday)
    triggers {
        pollSCM('*/5 * * * 1-5')
    }

    options {
        skipDefaultCheckout(true) // Skip automatic SCM checkout
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keep the 10 most recent builds
        timestamps() // Add timestamps to console output for better debugging
    }

    environment {
        // Add Miniconda's path to the environment PATH variable
        PATH="/var/lib/jenkins/miniconda3/bin:$PATH"
    }

    stages {

        // Stage to pull the latest code from SCM
        stage ("Code pull") {
            steps {
                checkout scm // Perform code checkout from the source control
            }
        }

        // Stage to create and set up the Conda virtual environment
        stage('Build environment') {
            steps {
                echo "Building virtual environment"
                sh  ''' 
                    conda create --yes -n ${BUILD_TAG} python // Create a new Conda environment with the build tag as the name
                    source activate ${BUILD_TAG} // Activate the newly created environment
                    pip install -r requirements/dev.txt // Install dependencies from the development requirements file
                '''
            }
        }

        // Stage to run static code analysis and generate reports
        stage('Static code metrics') {
            steps {
                echo "Running raw metrics analysis"
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    radon raw --json irisvmpy > raw_report.json // Generate raw metrics report in JSON format
                    radon cc --json irisvmpy > cc_report.json // Generate Cyclomatic Complexity report
                    radon mi --json irisvmpy > mi_report.json // Generate Maintainability Index report
                    sloccount --duplicates --wide irisvmpy > sloccount.sc // Generate Source Lines of Code report
                '''

                echo "Running test coverage analysis"
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    coverage run irisvmpy/iris.py 1 1 2 3 // Run coverage tests
                    python -m coverage xml -o reports/coverage.xml // Generate coverage report in XML format
                '''

                echo "Running style check"
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    pylint irisvmpy || true // Run pylint for style checks; allow failures
                '''
            }
            post {
                always {
                    // Publish Cobertura test coverage report
                    step([$class: 'CoberturaPublisher',
                          autoUpdateHealth: false,
                          autoUpdateStability: false,
                          coberturaReportFile: 'reports/coverage.xml',
                          failNoReports: false,
                          failUnhealthy: false,
                          failUnstable: false,
                          maxNumberOfBuilds: 10,
                          onlyStable: false,
                          sourceEncoding: 'ASCII',
                          zoomCoverageChart: false])
                }
            }
        }

        // Stage to run unit tests and collect results
        stage('Unit tests') {
            steps {
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    python -m pytest --verbose --junit-xml reports/unit_tests.xml // Run unit tests and generate XML report
                '''
            }
            post {
                always {
                    // Archive unit test results in Jenkins
                    junit allowEmptyResults: true, testResults: 'reports/unit_tests.xml'
                }
            }
        }

        // Stage to run acceptance tests and generate reports
        stage('Acceptance tests') {
            steps {
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    behave -f=formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true // Run acceptance tests and generate report
                '''
            }
            post {
                always {
                    // Publish Cucumber acceptance test reports
                    cucumber (buildStatus: 'SUCCESS',
                              fileIncludePattern: '**/*.json',
                              jsonReportDirectory: './reports/',
                              parallelTesting: true,
                              sortingMethod: 'ALPHABETICAL')
                }
            }
        }

        // Stage to build the Python package
        stage('Build package') {
            when {
                // Execute this stage only if the build is successful or has no result yet
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                sh  ''' 
                    source activate ${BUILD_TAG} // Activate the Conda environment
                    python setup.py bdist_wheel // Build the Python package as a wheel file
                '''
            }
            post {
                always {
                    // Archive the built package in Jenkins
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*whl', fingerprint: true
                }
            }
        }

        // Uncomment this stage to deploy the package to PyPI
        // stage("Deploy to PyPI") {
        //     steps {
        //         sh """twine upload dist/*
        //         """
        //     }
        // }
    }

    post {
        // Always clean up the Conda environment
        always {
            sh 'conda remove --yes -n ${BUILD_TAG} --all'
        }

        // Send an email notification on failure
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                         <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']])
        }
    }
}


