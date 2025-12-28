pipeline {
    agent any

    parameters {
        string(name: 'PROJECT_ID', description: 'Project ID to build')
        string(name: 'BUILD_ID', description: 'Build ID to build')
        string(name: 'APPLICATION_SLUGS', description: 'Slugs of applications separated with a pipe (|).')
    }

    environment {
        // MinIO Configuration
        MINIO_CREDENTIALS_ID = 'minio-credentials'  // Jenkins credential ID for MinIO
        MINIO_URL = 'http://nginx:9000'
        MINIO_BUCKET = 'builds'

        // Docker Configuration
        DOCKER_IMAGE_NAME = 'test-image'            // Update with your image name
        DOCKER_IMAGE_TAG = "v1"                     // Or use your preferred tagging strategy
        DOCKER_REGISTRY = ''                        // Optional: Docker registry URL (e.g., 'registry.example.com')
        DOCKER_CREDENTIALS_ID = ''                  // Optional: Jenkins credential ID for Docker registry
    }

    stages {
        stage('Process Applications') {
            steps {
                script {
                    def INDIVIDUAL_APPLICATION_SLUGS = params.APPLICATION_SLUGS.tokenize('|')
                    echo "Processing Application slugs: ${INDIVIDUAL_APPLICATION_SLUGS}"

                    def BUILD_DIR = "${params.BUILD_ID}"
                    def parallelStages = [:]

                    for (slug in INDIVIDUAL_APPLICATION_SLUGS) {
                        def currentSlug = slug
                        parallelStages[currentSlug] = {
                            stage("Download ${currentSlug}") {
                                def MINIO_FILE = "/${params.PROJECT_ID}/builds/${params.BUILD_ID}/${currentSlug}.tar.gz"
                                echo "Downloading ${MINIO_FILE} from MinIO bucket ${MINIO_BUCKET}..."

                                minioDownload(
                                    bucket: "${MINIO_BUCKET}",
                                    file: "${MINIO_FILE}",
                                    host: "${MINIO_URL}",
                                    credentialsId: "${MINIO_CREDENTIALS_ID}",
                                    targetFolder: "${WORKSPACE}"
                                )
                            }

                            stage("Extract ${currentSlug}") {
                                echo "Extracting ${currentSlug}.tar.gz archive..."
                                sh """
                                    mkdir -p ${BUILD_DIR}/${currentSlug}
                                    tar -xzf ${currentSlug}.tar.gz -C ${BUILD_DIR}/${currentSlug}
                                """
                                echo "Removing ${currentSlug}.tar.gz archive"
                                sh """
                                    rm ${currentSlug}.tar.gz
                                """
                            }

                            stage("Install Dependencies ${currentSlug}") {
                                nodejs(nodeJSInstallationName: 'node24') {
                                    sh """
                                        cd ${BUILD_DIR}/${currentSlug}
                                        # Install pnpm globally if not already installed
                                        if ! command -v pnpm &> /dev/null; then
                                            npm install -g pnpm
                                        fi

                                        # Run pnpm install
                                        pnpm install
                                    """
                                }
                            }

//                             stage("Create Migrations ${currentSlug}") {
//                                 nodejs(nodeJSInstallationName: 'node24') {
//                                     sh """
//                                         cd ${BUILD_DIR}/${currentSlug}
//                                         # Install pnpm globally if not already installed
//                                         if ! command -v pnpm &> /dev/null; then
//                                             npm install -g pnpm
//                                         fi
//
//                                         # Run pnpm install
//                                         pnpm migrations-create --initial
//                                     """
//                                 }
//                             }

                            stage("Build Docker Image ${currentSlug}") {
                                echo "Building Docker image for ${currentSlug}..."

                                // sh """
                                //     cd ${BUILD_DIR}/${currentSlug}
                                //     docker build -t "${DOCKER_IMAGE_NAME}-${currentSlug}:${DOCKER_IMAGE_TAG}" .
                                // """

                                // // Using Docker Pipeline plugin
                                // def dockerImage = docker.build("${DOCKER_IMAGE_NAME}-${currentSlug}:${DOCKER_IMAGE_TAG}", "${BUILD_DIR}/${currentSlug}")

                                // // Tag as latest
                                // dockerImage.tag('latest')

                                // // Optional: Push to registry if configured
                                // if (env.DOCKER_REGISTRY && env.DOCKER_CREDENTIALS_ID) {
                                //     docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                                //         dockerImage.push("${DOCKER_IMAGE_TAG}")
                                //         dockerImage.push('latest')
                                //     }
                                // }
                            }
                        }
                    }

                    parallel parallelStages
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            echo "Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            // Cleanup
            script {
                sh """
                    rm -rf ${params.BUILD_ID}
                """
            }
        }
    }
}
