pipeline {
    agent any

    parameters {
        string(name: 'PROJECT_ID', description: 'Project ID to build')
        string(name: 'BUILD_ID', description: 'Build ID to build')
        string(name: 'APPLICATION_SLUGS', description: 'Slugs of applications separated with a pipe (|).')
    }

    environment {
        INDIVIDUAL_APPLICATION_SLUGS = params.APPLICATION_SLUGS.tokenize('\\|')

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
        stage('Download from MinIO') {
            steps {
                script {
                    echo "Iterating over Application slugs ${INDIVIDUAL_APPLICATION_SLUGS}"

                    for (slug in applicationSlugs) {
                        def MINIO_FILE = "/${params.PROJECT_ID}/builds/${params.BUILD_ID}/${slug}.tar.gz"

                        echo "Downloading ${MINIO_FILE} from MinIO bucket ${MINIO_BUCKET}..."

                        // Using MinIO plugin
                        minioDownload(
                            bucket: "${MINIO_BUCKET}",
                            file: "${MINIO_FILE}",
                            host: "${MINIO_URL}",
                            credentialsId: "${MINIO_CREDENTIALS_ID}",
                            targetFolder: "${WORKSPACE}"
                        )
                    }
                }
            }
        }

        stage('Extract Archive') {
            steps {
                script {
                    echo 'Extracting tar.gz archive...'
                    sh """
                        tar -xzf applicationName.tar.gz
                    """
                    echo 'Removing tar.gz archive'
                    sh """
                        rm applicationName.tar.gz
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                nodejs(nodeJSInstallationName: 'node24') {
                    sh '''
                        # Install pnpm globally if not already installed
                        if ! command -v pnpm &> /dev/null; then
                            npm install -g pnpm
                        fi

                        # Run pnpm install
                        pnpm install
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image using Docker Pipeline plugin...'

                    // sh """
                    //     docker build -t "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}" .
                    // """

                    // // Using Docker Pipeline plugin
                    // def dockerImage = docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")

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
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            echo "Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline failed!'
        }
//         always {
//             // Cleanup
//             script {
//                 sh """
//                     rm -rf *
//                 """
//             }
//         }
    }
}
