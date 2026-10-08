
pipeline {

    agent any

    stages {

        stage('Display Information') {

            steps {

                echo "======================================"
                echo "Amazon Multibranch Pipeline"
                echo "======================================"

                echo "Branch: ${env.BRANCH_NAME}"
                echo "Job: ${env.JOB_NAME}"
                echo "Build Number: ${env.BUILD_NUMBER}"

                echo "======================================"
            }
        }

        stage('Checkout') {

            steps {

                echo "Repository has been checked out by Multibranch Pipeline."

                sh '''
                    echo "Current directory:"
                    pwd

                    echo ""
                    echo "Repository files:"
                    ls -la

                    echo ""
                    echo "Git branch:"
                    git branch --show-current

                    echo ""
                    echo "Latest commit:"
                    git log -1 --oneline
                '''
            }
        }

        stage('Test') {

            steps {

                echo "Multibranch Pipeline is working successfully!"
            }
        }
    }
}


