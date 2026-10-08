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

        // Change this to your actual Nexus server URL
        NEXUS_URL = 'http://<NEXUS-IP>:8081'

        // Nexus raw hosted repository
        NEXUS_REPO = 'amazon-config'
    }

    stages {

        /*
         * ============================================================
         * 1. DISPLAY PARAMETERS
         * ============================================================
         */
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


        /*
         * ============================================================
         * 2. CHECKOUT TARGET BRANCH
         * ============================================================
         */
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


        /*
         * ============================================================
         * 3. VERIFY REPOSITORY
         * ============================================================
         */
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


        /*
         * ============================================================
         * 4. DETERMINE JSON FILE
         * ============================================================
         */
        stage('Determine JSON File') {
            steps {
                script {
                    env.JSON_FILE =
                        "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"

                    echo '======================================'
                    echo 'JSON FILE SELECTION'
                    echo '======================================'
                    echo "Configuration Type : ${params.CONFIG_TYPE}"
                    echo "Environment        : ${params.ENVIRONMENT}"
                    echo "Selected JSON File : ${env.JSON_FILE}"
                    echo '======================================'

                    if (!fileExists(env.JSON_FILE)) {
                        error(
                            "JSON file does not exist: ${env.JSON_FILE}"
                        )
                    }
                }
            }
        }


        /*
         * ============================================================
         * 5. VALIDATE EXISTING JSON
         * ============================================================
         */
        stage('Validate Existing JSON') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Validating Existing JSON"
                    echo "======================================"

                    python3 -m json.tool "$JSON_FILE" > /dev/null

                    echo "JSON validation successful."
                '''
            }
        }


        /*
         * ============================================================
         * 6. UPDATE JSON
         * ============================================================
         */
        stage('Update JSON') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Updating JSON"
                    echo "======================================"

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

                    echo "JSON update completed."
                '''
            }
        }


        /*
         * ============================================================
         * 7. VALIDATE UPDATED JSON
         * ============================================================
         */
        stage('Validate Updated JSON') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Validating Updated JSON"
                    echo "======================================"

                    python3 -m json.tool "$JSON_FILE" > /dev/null

                    echo "Updated JSON is valid."
                '''
            }
        }


        /*
         * ============================================================
         * 8. SHOW CHANGES
         * ============================================================
         */
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


        /*
         * ============================================================
         * 9. CREATE JENKINS BRANCH
         * ============================================================
         */
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


        /*
         * ============================================================
         * 10. COMMIT CHANGES
         * ============================================================
         */
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


        /*
         * ============================================================
         * 11. PUSH JENKINS BRANCH
         * ============================================================
         */
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


        /*
         * ============================================================
         * 12. SONARQUBE ANALYSIS
         * ============================================================
         */
        stage('SonarQube Analysis') {
            steps {
                script {

                    echo "======================================"
                    echo "SonarQube Analysis"
                    echo "======================================"

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


        /*
         * ============================================================
         * 13. QUALITY GATE
         * ============================================================
         */
        stage('Quality Gate') {
            steps {

                echo "======================================"
                echo "Waiting for SonarQube Quality Gate"
                echo "======================================"

                timeout(
                    time: 10,
                    unit: 'MINUTES'
                ) {
                    waitForQualityGate(
                        abortPipeline: true
                    )
                }

                echo "SonarQube Quality Gate passed."
            }
        }


        /*
         * ============================================================
         * 14. PACKAGE ARTIFACT
         * ============================================================
         */
        stage('Package Artifact') {
            steps {

                sh '''
                    echo "======================================"
                    echo "Packaging Artifact"
                    echo "======================================"

                    ARTIFACT_NAME="amazon-${CONFIG_TYPE}-${ENVIRONMENT}-${VERSION}-build-${BUILD_NUMBER}.tar.gz"

                    echo "Selected JSON : $JSON_FILE"
                    echo "Artifact Name : $ARTIFACT_NAME"

                    tar -czf "$ARTIFACT_NAME" "$JSON_FILE"

                    echo "======================================"
                    echo "Artifact Created"
                    echo "======================================"

                    ls -lh "$ARTIFACT_NAME"
                '''
            }
        }


        /*
         * ============================================================
         * 15. UPLOAD ARTIFACT TO NEXUS
         * ============================================================
         */
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
                        echo "======================================"
                        echo "Uploading Artifact to Nexus"
                        echo "======================================"

                        ARTIFACT_NAME="amazon-${CONFIG_TYPE}-${ENVIRONMENT}-${VERSION}-build-${BUILD_NUMBER}.tar.gz"

                        echo "Nexus Repository : $NEXUS_REPO"
                        echo "Artifact         : $ARTIFACT_NAME"

                        curl --fail --silent --show-error \
                            -u "$NEXUS_USER:$NEXUS_PASSWORD" \
                            --upload-file "$ARTIFACT_NAME" \
                            "$NEXUS_URL/repository/$NEXUS_REPO/$ARTIFACT_NAME"

                        echo "======================================"
                        echo "Artifact Uploaded Successfully"
                        echo "======================================"

                        echo "Repository URL:"
                        echo "$NEXUS_URL/repository/$NEXUS_REPO/$ARTIFACT_NAME"
                    '''
                }
            }
        }


        /*
         * ============================================================
         * 16. CREATE GITHUB PULL REQUEST
         * ============================================================
         */
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

                        python3 <<PYTHON
import json
import os

data = {
    "title": f"Update {os.environ['JSON_FILE']} from Jenkins build {os.environ['BUILD_NUMBER']}",
    "head": os.environ["FEATURE_BRANCH"],
    "base": os.environ["TARGET_BRANCH"],
    "body": f"Automated Pull Request created by Jenkins build {os.environ['BUILD_NUMBER']}."
}

with open("pull-request.json", "w") as file:
    json.dump(data, file)

print("Pull request JSON created successfully.")
PYTHON

                        echo "======================================"
                        echo "Request Body"
                        echo "======================================"

                        cat pull-request.json

                        RESPONSE=$(curl -sS -X POST \
                            -H "Accept: application/vnd.github+json" \
                            -H "Authorization: Bearer $GITHUB_TOKEN" \
                            -H "X-GitHub-Api-Version: 2022-11-28" \
                            https://api.github.com/repos/rajkumari-hub/amazon/pulls \
                            --data-binary @pull-request.json)

                        echo "======================================"
                        echo "GitHub API Response"
                        echo "======================================"

                        echo "$RESPONSE"

                        echo "$RESPONSE" | python3 -c '
import json
import sys

data = json.load(sys.stdin)

if "html_url" in data:
    print("======================================")
    print("Pull Request created successfully")
    print("PR URL:", data["html_url"])
    print("======================================")

elif "message" in data:
    print("======================================")
    print("GitHub API Error:", data["message"])
    print("======================================")
    sys.exit(1)

else:
    print("Unexpected GitHub API response")
    sys.exit(1)
'

                        rm -f pull-request.json
                    '''
                }
            }
        }
    }


    /*
     * ================================================================
     * POST ACTIONS
     * ================================================================
     */
    post {

        success {
            echo '======================================'
            echo 'AMAZON CI/CD PIPELINE SUCCESSFUL'
            echo '======================================'
            echo "Build Number : ${env.BUILD_NUMBER}"
            echo "Configuration: ${params.CONFIG_TYPE}"
            echo "Environment  : ${params.ENVIRONMENT}"
            echo "Version      : ${params.VERSION}"
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo '======================================'
            echo "Build Number : ${env.BUILD_NUMBER}"
            echo 'Check the failed stage in Console Output.'
            echo '======================================'
        }

        always {
            sh '''
                rm -f git-askpass.sh
                rm -f pull-request.json
            '''
        }
    }
}
