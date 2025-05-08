pipeline {
    agent any

    environment {
        WORK_DIR = "docker"  // Thư mục chứa Dockerfile của Chatwoot
        DOCKER_IMAGE_NAME = "mbw_chat"
        REGISTRY_URL = "registry.digitalocean.com"
        REGISTRY_NAME = "testerpnext"
    //    PLATFORM = "linux/amd64"
    }

    stages {
            stage('Checkout Code') {
                steps {
                    script {
                        checkout scm

                        def changes = sh(script: "git diff --name-only HEAD HEAD~1", returnStdout: true).trim().tokenize('\n')
                        def hasChanges = changes.any { it.startsWith("${WORK_DIR}/") }
                        
                        def latestCommitHash = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        def latestCommitMessage = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
                        
                        if (hasChanges) {
                            echo "Detected changes in ${WORK_DIR}. Commit: ${latestCommitHash} - ${latestCommitMessage}"
                        } else {
                            echo "No relevant changes detected in ${WORK_DIR}"
                        }

                        def proceed = false
                        try {
                            timeout(time: 1, unit: 'MINUTES') {
                                proceed = input message: "Proceed with pipeline?", ok: "Continue",
                                    parameters: [booleanParam(defaultValue: hasChanges, description: 'Confirm to proceed', name: 'CONTINUE')]
                            }
                        } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
                            echo "No input received. Aborting pipeline."
                            currentBuild.result = 'ABORTED'
                            error("Pipeline aborted.")
                        }

                        if (!proceed) {
                            currentBuild.result = 'ABORTED'
                            error("Pipeline aborted by user.")
                        }
                    }
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        def tag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        env.IMAGE_TAG = tag
                        env.FULL_IMAGE = "${REGISTRY_URL}/${REGISTRY_NAME}/${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"
                        sh """
                            docker build -t ${FULL_IMAGE} -f ${WORK_DIR}/Dockerfile .
                        """
                        sh """
                            docker image prune -f
                        """
                    }
                }
            }

        // stage('Scan Image with Trivy') {
        //     steps {
        //         script {
        //             echo "Scanning ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} with Trivy..."
        //             sh """
        //                 docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -e TRIVY_TIMEOUT=30m aquasec/trivy image ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} --exit-code 0
        //             """
        //         }
        //     }
        // }

            stage('Push to DigitalOcean Registry') {
                steps {
                    withCredentials([string(credentialsId: 'do-api-token', variable: 'DO_API_TOKEN')]) {
                        script {
                            sh '''
                                export HOME=\$(pwd)
                                mkdir -p \$HOME/.docker
                                echo $DO_API_TOKEN | docker login ${REGISTRY_URL} -u $DO_API_TOKEN -p $DO_API_TOKEN
                                docker push ${FULL_IMAGE}
                                docker rmi ${FULL_IMAGE}
                            '''
                        }
                    }
                }
            }

            stage('Deploy to Server with Docker Compose') {
                steps {
                    sshagent(credentials: ['vtdc-ssh-key']) {
                        script {
                            def confirmDeploy = input(
                                id: 'deployConfirm', message: 'Deploy to production?', parameters: [
                                    booleanParam(name: 'DEPLOY_NOW', defaultValue: true, description: 'Confirm to deploy')
                                ]
                            )

                            if (confirmDeploy) {
                                sh """
                                    ssh -o StrictHostKeyChecking=no vtdc@171.244.43.165 '
                                        cd /home/vtdc/docker/chatwoot/MBW_chatwoot &&
                                        sed -i "/^CHATWOOT_IMAGE=/d" .env &&
                                        echo "CHATWOOT_IMAGE=${FULL_IMAGE}" >> .env
                                        // docker compose pull &&
                                        // docker compose down &&
                                        // docker compose up -d
                                    '
                                """
                            } else {
                                echo "Deployment skipped by user."
                            }
                        }
                    }
                }
            }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Chatwoot CI/CD completed successfully!"
        }
        failure {
            echo "Chatwoot CI/CD failed."
        }
    }
}
