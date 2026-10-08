pipeline {

    agent any

    parameters {

        choice(
            name: 'CONFIG_TYPE',
            choices: ['environment', 'nodes'],
            description: 'Select the configuration folder'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'stage', 'prod'],
            description: 'Select environment'
        )

        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Application version'
        )

        string(
            name: 'REPLICAS',
            defaultValue: '2',
            description: 'Number of replicas'
        )

        choice(
            name: 'LOG_LEVEL',
            choices: ['DEBUG', 'INFO', 'WARN', 'ERROR'],
            description: 'Application log level'
        )

        string(
            name: 'TARGET_BRANCH',
            defaultValue: 'feature',
            description: 'Branch where Pull Request should be raised'
        )
    }

    environment {
        APP_NAME = "amazon"
    }

    stages {

        stage('Checkout') {

            steps {

                echo "Checking out repository..."

                checkout scm
            }
        }

        stage('Display Parameters') {

            steps {

                echo "================================"
                echo "Configuration Type : ${params.CONFIG_TYPE}"
                echo "Environment        : ${params.ENVIRONMENT}"
                echo "Version            : ${params.VERSION}"
                echo "Replicas           : ${params.REPLICAS}"
                echo "Log Level          : ${params.LOG_LEVEL}"
                echo "Target Branch      : ${params.TARGET_BRANCH}"
                echo "================================"
            }
        }

        stage('Determine JSON File') {

            steps {

                script {

                    env.JSON_FILE =
                        "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"

                    echo "JSON file selected: ${env.JSON_FILE}"
                }
            }
        }

        stage('Validate Existing JSON') {

            steps {

                sh """
                    python3 -m json.tool ${JSON_FILE}
                """
            }
        }

        stage('Update JSON') {

            steps {

                sh '''
python3 <<EOF

import json

file = "${JSON_FILE}"

with open(file, "r") as f:
    data = json.load(f)

data["version"] = "${VERSION}"
data["replicas"] = int("${REPLICAS}")
data["logLevel"] = "${LOG_LEVEL}"

with open(file, "w") as f:
    json.dump(data, f, indent=2)

EOF
                '''
            }
        }

        stage('Validate Updated JSON') {

            steps {

                sh """
                    python3 -m json.tool ${JSON_FILE}
                """
            }
        }

        stage('Show Changes') {

            steps {

                sh """
                    echo "Changed file:"
                    git diff -- ${JSON_FILE}
                """
            }
        }

        stage('Build') {

            steps {

                echo "Building Amazon application..."
            }
        }

        stage('Test') {

            steps {

                echo "Running tests..."
            }
        }

        stage('SonarQube') {

            steps {

                echo "SonarQube analysis will run here..."
            }
        }

        stage('Nexus') {

            steps {

                echo "Artifact will be uploaded to Nexus..."
            }
        }
    }
}

