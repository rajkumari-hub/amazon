
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
            defaultValue: 'main',
            description: 'Target branch for Pull Request'
        )
    }

    environment {
        APP_NAME = 'amazon'
        NEXUS_URL = 'http://13.218.184.207:8081'
        NEXUS_REPO = 'amazon-config'
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
            }
        }

        stage('Checkout Code') {
            steps {
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
                    echo "Current Directory"
                    pwd

                    echo "Repository Files"
                    ls -la

                    echo "Git Branch"
                    git branch --show-current

                    echo "Latest Commit"
                    git log -1 --oneline
                '''
            }
        }

        stage('Determine JSON File') {
            steps {
                script {
                    env.JSON_FILE =
                        "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"

                    echo "Selected JSON File: ${env.JSON_FILE}"

                    if (!fileExists(env.JSON_FILE)) {
                        error("JSON file does not exist: ${env.JSON_FILE}")
                    }
                }
            }
        }

        stage('Validate Existing JSON') {
            steps {
                sh '''
                    python3 -m json.tool "$JSON_FILE" > /dev/null
                    echo "Existing JSON validation successful."
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
                    python3 -m json.tool "$JSON_FILE" > /dev/null
                    echo "Updated JSON is valid."
                '''
            }
        }

        stage('Show Changes') {
            steps {
                sh '''
                    git diff -- "$JSON_FILE"
                '''
            }
        }

        stage('Create Feature Branch') {
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "jenkins/${params.CONFIG_TYPE}-${params.ENVIRONMENT}-build-${env.BUILD_NUMBER}"

                    echo "Creating branch: ${env.FEATURE_BRANCH}"

                    sh '''
                        git checkout -b "$FEATURE_BRANCH"
                    '''
                }
            }
        }

        stage('Commit Changes') {
            steps {
                sh '''
                    git config user.name "Jenkins"
                    git config user.email "jenkins@localhost"

                    git add "$JSON_FILE"

                    if git diff --cached --quiet; then
                        echo "No changes to commit."
                        exit 1
                    fi

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
                        set -eu

                        cat > git-askpass.sh <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
    *Password*) printf '%s\\n' "$GIT_PASSWORD" ;;
esac
EOF

                        chmod 700 git-askpass.sh

                        export GIT_ASKPASS="$PWD/git-askpass.sh"
                        export GIT_TERMINAL_PROMPT=0

                        git push origin "$FEATURE_BRANCH"
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=amazon \
                              -Dsonar.projectName=amazon \
                              -Dsonar.projectVersion=${params.VERSION} \
                              -Dsonar.sources=environment,node \
                              -Dsonar.json.activate=true
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate(abortPipeline: true)
                }

                echo "SonarQube Quality Gate passed."
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                    ARTIFACT_NAME="amazon-${CONFIG_TYPE}-${ENVIRONMENT}-${VERSION}-build-${BUILD_NUMBER}.tar.gz"

                    tar -czf "$ARTIFACT_NAME" "$JSON_FILE"

                    echo "Artifact created: $ARTIFACT_NAME"
                    ls -lh "$ARTIFACT_NAME"
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu

                        ARTIFACT_NAME="amazon-${CONFIG_TYPE}-${ENVIRONMENT}-${VERSION}-build-${BUILD_NUMBER}.tar.gz"

                        curl --fail --silent --show-error \
                            -u "$NEXUS_USER:$NEXUS_PASSWORD" \
                            --upload-file "$ARTIFACT_NAME" \
                            "$NEXUS_URL/repository/$NEXUS_REPO/$ARTIFACT_NAME"

                        echo "Artifact uploaded successfully."
                    '''
                }
            }
        }

        stage('Create Pull Request') {
            steps {
                script {
                    withCredentials([
                        string(
                            credentialsId: 'github-pr-token',
                            variable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            python3 <<'PYTHON'
import json
import os

data = {
    "title": (
        f"Update {os.environ['JSON_FILE']} "
        f"from Jenkins build {os.environ['BUILD_NUMBER']}"
    ),
    "head": os.environ["FEATURE_BRANCH"],
    "base": os.environ["TARGET_BRANCH"],
    "body": (
        f"Automated Pull Request created by Jenkins build "
        f"{os.environ['BUILD_NUMBER']}."
    )
}

with open("pull-request.json", "w") as file:
    json.dump(data, file)
PYTHON

                            RESPONSE=$(curl --fail-with-body --silent --show-error \
                                -X POST \
                                -H "Accept: application/vnd.github+json" \
                                -H "Authorization: Bearer $GITHUB_TOKEN" \
                                -H "X-GitHub-Api-Version: 2022-11-28" \
                                -H "Content-Type: application/json" \
                                https://api.github.com/repos/rajkumari-hub/amazon/pulls \
                                --data-binary @pull-request.json)

                            printf '%s' "$RESPONSE" | python3 -c '
import json
import sys

data = json.load(sys.stdin)

if "number" not in data or "html_url" not in data:
    print(data.get("message", "Unexpected GitHub API response"),
          file=sys.stderr)
    sys.exit(1)

with open("pr-number.txt", "w") as file:
    file.write(str(data["number"]))

print("Pull Request created successfully.")
print("PR number:", data["number"])
print("PR URL:", data["html_url"])
'
                        '''
                    }

                    env.PR_NUMBER = readFile('pr-number.txt').trim()

                    echo "Created Pull Request: #${env.PR_NUMBER}"
                }
            }
        }

        /*
         * AUTOMATIC MERGE
         * Runs only after every preceding stage succeeds.
         */
        stage('Automatic PR Merge') {
            steps {
                script {
                    if (!env.PR_NUMBER?.trim()) {
                        error("Pull Request number is missing.")
                    }

                    withCredentials([
                        string(
                            credentialsId: 'github-pr-token',
                            variable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            echo "======================================"
                            echo "Automatically merging Pull Request"
                            echo "PR Number: $PR_NUMBER"
                            echo "Target Branch: $TARGET_BRANCH"
                            echo "======================================"

                            RESPONSE=$(curl --fail-with-body --silent --show-error \
                                -X PUT \
                                "https://api.github.com/repos/rajkumari-hub/amazon/pulls/${PR_NUMBER}/merge" \
                                -H "Accept: application/vnd.github+json" \
                                -H "Authorization: Bearer ${GITHUB_TOKEN}" \
                                -H "X-GitHub-Api-Version: 2022-11-28" \
                                -H "Content-Type: application/json" \
                                -d '{"merge_method":"squash"}')

                            printf '%s' "$RESPONSE" | python3 -c '
import json
import sys

data = json.load(sys.stdin)

if data.get("merged") is not True:
    print(data.get("message", "GitHub did not merge the PR"),
          file=sys.stderr)
    sys.exit(1)

print("Pull Request merged successfully.")
print("Merge SHA:", data.get("sha", "not returned"))
'
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'AMAZON CI/CD PIPELINE SUCCESSFUL'
            echo '======================================'
            echo "Build Number : ${env.BUILD_NUMBER}"
            echo "Configuration: ${params.CONFIG_TYPE}"
            echo "Environment  : ${params.ENVIRONMENT}"
            echo "Version      : ${params.VERSION}"
            echo "PR Number    : ${env.PR_NUMBER}"
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo '======================================'
            echo "Build Number: ${env.BUILD_NUMBER}"
            echo 'Check the failed stage in Console Output.'
            echo '======================================'
        }

        always {
            sh '''
                rm -f git-askpass.sh
                rm -f pull-request.json
                rm -f pr-number.txt
            '''
        }
    }
}
