pipeline {
    agent any

    parameters {
        choice(
            name: 'CONFIG_TYPE',
            choices: ['environment', 'node'],
            description: 'Select configuration type'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'stage', 'prod'],
            description: 'Select environment'
        )

        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Application/configuration version'
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
            description: 'Target branch for Pull Request'
        )
    }

    environment {
        APP_NAME = 'amazon'
    }

    stages {

        stage('Display Parameters') {
            steps {
                echo '======================================'
                echo 'Amazon Configuration Pipeline'
                echo '======================================'
                echo "CONFIG_TYPE  : ${params.CONFIG_TYPE}"
                echo "ENVIRONMENT  : ${params.ENVIRONMENT}"
                echo "VERSION      : ${params.VERSION}"
                echo "REPLICAS     : ${params.REPLICAS}"
                echo "LOG_LEVEL    : ${params.LOG_LEVEL}"
                echo "TARGET_BRANCH: ${params.TARGET_BRANCH}"
                echo '======================================'
            }
        }

        stage('Checkout Code') {
            steps {
                echo 'Checking out amazon repository...'

                git(
                    url: 'https://github.com/rajkumari-hub/amazon.git',
                    branch: params.TARGET_BRANCH,
                    credentialsId: 'github-credentials'
                )
            }
        }

        stage('Verify Repository') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Current Directory"
                    echo "======================================"
                    pwd

                    echo "======================================"
                    echo "Repository Files"
                    echo "======================================"
                    ls -la

                    echo "======================================"
                    echo "Git Branch"
                    echo "======================================"
                    git branch --show-current

                    echo "======================================"
                    echo "Git Commit"
                    echo "======================================"
                    git log -1 --oneline
                '''
            }
        }

        stage('Determine JSON File') {
            steps {
                script {
                    env.JSON_FILE = "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"

                    echo '======================================'
                    echo 'JSON FILE SELECTION'
                    echo '======================================'
                    echo "Configuration Type : ${params.CONFIG_TYPE}"
                    echo "Environment        : ${params.ENVIRONMENT}"
                    echo "Selected JSON File : ${env.JSON_FILE}"
                    echo '======================================'

                    if (!fileExists(env.JSON_FILE)) {
                        error("JSON file does not exist: ${env.JSON_FILE}")
                    }
                }
            }
        }

        stage('Validate Existing JSON') {
            steps {
                sh '''
                    echo "Validating existing JSON..."

                    python3 -m json.tool "$JSON_FILE" > /dev/null

                    echo "JSON validation successful."
                '''
            }
        }

        stage('Update JSON') {
            steps {
                sh '''
                    python3 <<'PYTHON'
import json
import os

file_name = os.environ["JSON_FILE"]
config_type = os.environ["CONFIG_TYPE"]
version = os.environ["VERSION"]
replicas = os.environ["REPLICAS"]
log_level = os.environ["LOG_LEVEL"]

with open(file_name, "r") as file:
    data = json.load(file)

if config_type == "environment":
    data["version"] = version
    data["replicas"] = int(replicas)
    data["logLevel"] = log_level

elif config_type == "node":
    data["version"] = version

with open(file_name, "w") as file:
    json.dump(data, file, indent=2)
    file.write("\\n")

print(f"Updated: {file_name}")
PYTHON
                '''
            }
        }

        stage('Validate Updated JSON') {
            steps {
                sh '''
                    echo "Validating updated JSON..."

                    python3 -m json.tool "$JSON_FILE" > /dev/null

                    echo "Updated JSON is valid."
                '''
            }
        }

        stage('Show Changes') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Git Changes"
                    echo "======================================"

                    git diff -- "$JSON_FILE"
                '''
            }
        }

        stage('Create Feature Branch') {
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "jenkins/${params.CONFIG_TYPE}-${params.ENVIRONMENT}-build-${env.BUILD_NUMBER}"

                    echo '======================================'
                    echo 'Creating Feature Branch'
                    echo '======================================'
                    echo "Source Branch  : ${params.TARGET_BRANCH}"
                    echo "Feature Branch : ${env.FEATURE_BRANCH}"
                    echo '======================================'

                    sh """
                        git checkout -b "${FEATURE_BRANCH}"
                    """
                }
            }
        }

        stage('Commit Changes') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Committing Changes"
                    echo "======================================"

                    git config user.name "Jenkins"
                    git config user.email "jenkins@localhost"

                    git add "$JSON_FILE"

                    git commit \
                        -m "Update $JSON_FILE from Jenkins build $BUILD_NUMBER"
                '''
            }
        }

        stage('Push Feature Branch') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'github-credentials',
                usernameVariable: 'GIT_USERNAME',
                passwordVariable: 'GIT_PASSWORD'
            )
        ]) {
            sh '''
                echo "======================================"
                echo "Pushing Feature Branch"
                echo "======================================"
                echo "Branch: $FEATURE_BRANCH"

                cat > git-askpass.sh <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) echo "$GIT_USERNAME" ;;
    *Password*) echo "$GIT_PASSWORD" ;;
esac
EOF

                chmod 700 git-askpass.sh

                export GIT_ASKPASS="$PWD/git-askpass.sh"
                export GIT_TERMINAL_PROMPT=0

                git push origin "$FEATURE_BRANCH"

                rm -f git-askpass.sh
            '''
        }
    }
}

	stage('Create Pull Request') {
    steps {
        withCredentials([
            string(
                credentialsId: 'github-pr-token',
                variable: 'GITHUB_TOKEN'
            )
        ]) {
            sh '''
                echo "======================================"
                echo "Creating Pull Request"
                echo "======================================"

                echo "Source Branch : $FEATURE_BRANCH"
                echo "Target Branch : $TARGET_BRANCH"

                RESPONSE=$(curl -sS -X POST \
                    -H "Accept: application/vnd.github+json" \
                    -H "Authorization: Bearer $GITHUB_TOKEN" \
                    -H "X-GitHub-Api-Version: 2022-11-28" \
                    https://api.github.com/repos/rajkumari-hub/amazon/pulls \
                    -d "{\"title\":\"Update $JSON_FILE from Jenkins build $BUILD_NUMBER\",\"head\":\"$FEATURE_BRANCH\",\"base\":\"$TARGET_BRANCH\",\"body\":\"Automated Pull Request created by Jenkins build $BUILD_NUMBER.\"}")

                echo "$RESPONSE" | python3 -c '
import json
import sys

data = json.load(sys.stdin)

if "html_url" in data:
    print("Pull Request created successfully")
    print("PR URL:", data["html_url"])
elif "message" in data:
    print("GitHub API Error:", data["message"])
    sys.exit(1)
else:
    print("Unexpected GitHub API response")
    sys.exit(1)
'
            '''
        }
    }
}

    }

    post {
        success {
            echo '======================================'
            echo 'JSON UPDATE PIPELINE SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo '======================================'
        }
    }
}
