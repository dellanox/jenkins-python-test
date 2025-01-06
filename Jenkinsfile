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
        stage("Checkout Code") {
            steps {
                echo "Checking out source code" // Logs the current operation for visibility.
                checkout scm // Checks out the code from the source control repository configured in the job.
            }
        }
       
        stage("Set Up Environment") {
	    steps {
		echo "Setting up virtual environment"
		sh '''
		    # Ensure Miniconda path is set
		    export PATH=/var/lib/jenkins/miniconda3/bin:$PATH

		    # Remove the environment if it exists (clean-up step)
		    conda env remove --yes -n ${BUILD_TAG} || true

		    # Create a new Conda virtual environment
		    conda create --yes -n ${BUILD_TAG} python=3.12

		    # Activate the environment for the current shell session
		    source activate ${BUILD_TAG}

		    # Install required dependencies
		    pip install -r requirements/dev.txt || exit 1
		'''
	    }
	}	

        stage("Static Code Analysis") {
            steps {
                echo "Running static code analysis" // Logs the start of static analysis.
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} // Activates the virtual environment.
                    mkdir -p reports // Ensures the reports directory exists.
                    radon raw --json irisvmpy > reports/raw_report.json // Runs raw code analysis and saves results to a JSON file.
                    radon cc --json irisvmpy > reports/cc_report.json // Runs cyclomatic complexity analysis.
                    radon mi --json irisvmpy > reports/mi_report.json // Runs maintainability index analysis.
                    sloccount --duplicates --wide irisvmpy > reports/sloccount.sc // Runs source lines of code analysis.
                '''
            }
        }

        stage("Unit Tests") {
            steps {
                echo "Running unit tests" // Logs the start of unit testing.
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} // Activates the virtual environment.
                    mkdir -p reports // Ensures the reports directory exists.
                    python -m pytest --verbose --junit-xml reports/unit_tests.xml // Runs unit tests and outputs results in JUnit format.
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/unit_tests.xml' // Publishes the test results.
                }
            }
        }

        stage("Acceptance Tests") {
            steps {
                echo "Running acceptance tests" // Logs the start of acceptance testing.
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} // Activates the virtual environment.
                    mkdir -p reports // Ensures the reports directory exists.
                    behave -f formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true // Runs acceptance tests and saves results in JSON format.
                '''
            }
            post {
                always {
                    cucumber(buildStatus: 'SUCCESS', // Publishes the cucumber reports.
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
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' // Ensures the build only happens if previous stages succeeded.
                }
            }
            steps {
                echo "Building package" // Logs the start of the package build.
                sh '''
                    . /var/lib/jenkins/miniconda3/bin/activate && conda activate ${BUILD_TAG} // Activates the virtual environment.
                    python setup.py bdist_wheel // Builds a distributable Python package.
                '''
            }
            post {
                always {
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*.whl', fingerprint: true // Archives the built package as an artifact.
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up environment" // Logs the cleanup process.
            sh '''
                . /var/lib/jenkins/miniconda3/bin/activate // Ensures Miniconda is activated.
                conda remove --yes -n ${BUILD_TAG} --all || true // Removes the virtual environment to free resources.
            '''
        }
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'", // Sends an email notification on failure.
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'</p>
                         <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']] // Sends the email to developers associated with the job.
            )
        }
    }
}

