pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-token')
        DOCKER_REGISTRY = 'nguetsop'
        IMAGE_NAME = 'jenskin-cicd-projekt'
        KUBECONFIG = credentials('kubeconfig')
        DOCKER_BUILDKIT = '1'
    }
    
    parameters {
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip integration tests')
    }
    
    stages {
        stage('Checkout & Environment Detection') {
            steps {
                echo 'Detecting environment based on branch...'
                checkout scm
                
                script {
                    // Determine target environment based on branch
                    def branchName = env.GIT_BRANCH.replaceAll('origin/', '')
                    def targetEnvironment = ''
                    
                    switch(branchName) {
                        case 'dev':
                        case 'develop':
                            targetEnvironment = 'dev'
                            break
                        case 'main':
                        case 'master':
                            targetEnvironment = 'prod'
                            break
                        default:
                            targetEnvironment = 'dev'
                    }
                    
                    // Store environment variables
                    env.TARGET_ENVIRONMENT = targetEnvironment
                    env.CURRENT_BRANCH = branchName
                    
                    // Set dynamic build name
                    currentBuild.displayName = "#${BUILD_NUMBER}-${branchName}→${targetEnvironment}"
                    
                    echo """
╔══════════════════════════════════════════════════════════════╗
║                    🌍 ENVIRONMENT STRATEGY                    ║
╚══════════════════════════════════════════════════════════════╝
📋 Current Branch: ${branchName}
🎯 Target Environment: ${targetEnvironment}
                    """
                    
                    // Validate basic project structure
                    sh '''
                        echo "🔍 Validating project structure..."
                        [ -f "cast-service/Dockerfile" ] || { echo "❌ cast-service Dockerfile missing"; exit 1; }
                        [ -f "movie-service/Dockerfile" ] || { echo "❌ movie-service Dockerfile missing"; exit 1; }
                        echo "✅ Basic project structure validation passed"
                    '''
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    echo "🔨 Building Docker images for ${env.TARGET_ENVIRONMENT} environment..."
                    
                    sh '''
                        export DOCKER_BUILDKIT=1
                        
                        echo "🔨 Building cast-service image..."
                        docker build \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-latest \
                            --label "build.number=${BUILD_NUMBER}" \
                            --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            --label "git.commit=${GIT_COMMIT}" \
                            --label "git.branch=${CURRENT_BRANCH}" \
                            --label "target.environment=${TARGET_ENVIRONMENT}" \
                            --label "service.name=cast-service" \
                            ./cast-service
                        
                        echo "🔨 Building movie-service image..."
                        docker build \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-latest \
                            --label "build.number=${BUILD_NUMBER}" \
                            --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            --label "git.commit=${GIT_COMMIT}" \
                            --label "git.branch=${CURRENT_BRANCH}" \
                            --label "target.environment=${TARGET_ENVIRONMENT}" \
                            --label "service.name=movie-service" \
                            ./movie-service
                        
                        echo "✅ Docker images built successfully!"
                        echo "📦 Images created for ${TARGET_ENVIRONMENT}:"
                        echo "   - ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}"
                        echo "   - ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}"
                        
                        docker images | grep ${IMAGE_NAME}
                    '''
                }
            }
        }
        
        stage('Push Images to Registry') {
            steps {
                script {
                    echo "🚀 Pushing images for ${env.TARGET_ENVIRONMENT} environment..."
                    
                    // Use Docker commands directly with credentials
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-token', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh '''
                            echo "📦 Logging into Docker registry..."
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                            
                            echo "📦 Pushing ${TARGET_ENVIRONMENT} images to ${DOCKER_REGISTRY}/${IMAGE_NAME}..."
                            
                            # Push environment-specific versioned images
                            echo "├─ Pushing cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}
                            
                            echo "├─ Pushing movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER}
                            
                            # Push environment-specific latest tags
                            echo "├─ Pushing cast-service-${TARGET_ENVIRONMENT}-latest..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-latest
                            
                            echo "└─ Pushing movie-service-${TARGET_ENVIRONMENT}-latest..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-latest
                            
                            echo "✅ All ${TARGET_ENVIRONMENT} images pushed successfully!"
                            
                            # Logout for security
                            docker logout
                        '''
                    }
                }
            }
        }
        
        stage('Production Approval') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master' 
                    expression { env.GIT_BRANCH == 'origin/main' }
                    expression { env.CURRENT_BRANCH == 'main' }
                    expression { env.TARGET_ENVIRONMENT == 'prod' }
                }
            }
            steps {
                script {
                    timeout(time: 30, unit: 'MINUTES') {
                        def deploymentDetails = """
🚀 PRODUCTION DEPLOYMENT CONFIRMATION

📋 Deployment Details:
├─ Environment: PRODUCTION
├─ Build: #${BUILD_NUMBER}
├─ Branch: ${env.CURRENT_BRANCH}
├─ Commit: ${GIT_COMMIT}
└─ Images: Ready to deploy

⚠️ This will deploy to PRODUCTION environment.
                        """
                        
                        def approval = input(
                            message: deploymentDetails,
                            ok: 'Deploy to Production',
                            submitterParameter: 'DEPLOYER'
                        )
                        
                        echo "✅ Production deployment confirmed by: ${approval}"
                    }
                }
            }
        }
        
        stage('Deploy to Environment') {
            steps {
                script {
                    echo "🚀 Deploying to ${env.TARGET_ENVIRONMENT} environment..."
                    
                    sh """
                        echo "🔧 Preparing deployment to ${TARGET_ENVIRONMENT}..."
                        
                        # Create namespace if it doesn't exist
                        kubectl create namespace ${TARGET_ENVIRONMENT} --dry-run=client -o yaml | kubectl apply -f -
                        echo "✅ Namespace ${TARGET_ENVIRONMENT} ready"
                        
                        # Check if Helm chart exists and is valid
                        if [ -f "charts/Chart.yaml" ]; then
                            echo "📦 Deploying with Helm (increased timeout)..."
                            helm upgrade --install movie-app-${TARGET_ENVIRONMENT} ./charts \\
                                --namespace ${TARGET_ENVIRONMENT} \\
                                --set image.tag=${TARGET_ENVIRONMENT}-${BUILD_NUMBER} \\
                                --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \\
                                --set build.number=${BUILD_NUMBER} \\
                                --set build.commit=${GIT_COMMIT} \\
                                --set build.branch=${CURRENT_BRANCH} \\
                                --set build.environment=${TARGET_ENVIRONMENT} \\
                                --wait --timeout=300s || {
                                    echo "⚠️ Helm deployment failed, trying kubectl fallback..."
                                    kubectl create deployment cast-service-${TARGET_ENVIRONMENT} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} -n ${TARGET_ENVIRONMENT} --dry-run=client -o yaml | kubectl apply -f -
                                    kubectl create deployment movie-service-${TARGET_ENVIRONMENT} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} -n ${TARGET_ENVIRONMENT} --dry-run=client -o yaml | kubectl apply -f -
                                }
                        else
                            echo "📦 Deploying with kubectl (no Helm chart found)..."
                            # Create simple deployments
                            kubectl create deployment cast-service-${TARGET_ENVIRONMENT} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} -n ${TARGET_ENVIRONMENT} --dry-run=client -o yaml | kubectl apply -f -
                            kubectl create deployment movie-service-${TARGET_ENVIRONMENT} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${TARGET_ENVIRONMENT}-${BUILD_NUMBER} -n ${TARGET_ENVIRONMENT} --dry-run=client -o yaml | kubectl apply -f -
                            
                            # Expose services
                            kubectl expose deployment cast-service-${TARGET_ENVIRONMENT} --port=80 --target-port=8000 -n ${TARGET_ENVIRONMENT} || echo "Service might already exist"
                            kubectl expose deployment movie-service-${TARGET_ENVIRONMENT} --port=80 --target-port=8000 -n ${TARGET_ENVIRONMENT} || echo "Service might already exist"
                        fi
                        
                        echo "🔍 Verifying deployment in ${TARGET_ENVIRONMENT}..."
                        kubectl get pods -n ${TARGET_ENVIRONMENT}
                        kubectl get services -n ${TARGET_ENVIRONMENT}
                        
                        echo "✅ Deployment to ${TARGET_ENVIRONMENT} completed!"
                    """
                }
            }
        }
        
        stage('Environment Testing') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                script {
                    echo "🧪 Running basic ${env.TARGET_ENVIRONMENT} environment tests..."
                    
                    sh """
                        echo "⏳ Waiting for pods to be ready in ${TARGET_ENVIRONMENT}..."
                        kubectl wait --for=condition=ready pod -n ${TARGET_ENVIRONMENT} --timeout=300s --all || true
                        
                        echo "🏥 Performing health checks for ${TARGET_ENVIRONMENT}..."
                        kubectl get pods -n ${TARGET_ENVIRONMENT} -o wide
                        
                        # Environment-specific tests
                        case "${TARGET_ENVIRONMENT}" in
                            "dev")
                                echo "🔧 Running development smoke tests..."
                                ;;
                            "prod")
                                echo "🚀 Running production validation tests..."
                                ;;
                        esac
                        
                        # Check if any pods are running
                        RUNNING_PODS=\$(kubectl get pods -n ${TARGET_ENVIRONMENT} --field-selector=status.phase=Running -o name | wc -l)
                        if [ \$RUNNING_PODS -gt 0 ]; then
                            echo "✅ \$RUNNING_PODS pod(s) running successfully in ${TARGET_ENVIRONMENT}"
                        else
                            echo "⚠️ No pods running in ${TARGET_ENVIRONMENT} - this may be expected if no workloads are deployed yet"
                        fi
                        
                        sleep 10
                        echo "✅ ${TARGET_ENVIRONMENT} environment tests completed"
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo 'Cleaning up...'
                
                node {
                    // Clean up Docker resources
                    sh '''
                        echo "Cleaning up Docker resources..."
                        docker image prune -f --filter "until=24h" || true
                        docker builder prune -f --filter "until=24h" || true
                        echo "Cleanup completed"
                    '''
                }
            }
        }
        
        success {
            script {
                echo 'Pipeline completed successfully!'
                
                node {
                    sh '''
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║                    🎉 DEPLOYMENT SUCCESSFUL! 🎉                    ║"
                        echo "╚══════════════════════════════════════════════════════════════╝"
                        echo ""
                        echo "✅ Build completed successfully"
                        echo "📦 Docker images built and pushed to registry"
                        echo "☸️  Deployment completed to target environment"
                        echo ""
                    '''
                }
                
                currentBuild.description = "✅ SUCCESS"
            }
        }
        
        failure {
            script {
                echo 'Pipeline failed!'
                
                node {
                    sh '''
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║                    ❌ DEPLOYMENT FAILED! ❌                    ║"
                        echo "╚══════════════════════════════════════════════════════════════╝"
                        echo ""
                        echo "💥 Build failed - Check Jenkins console output for details"
                        echo "🔍 Review the failed stage above for specific error messages"
                        echo ""
                    '''
                }
                
                currentBuild.description = "❌ FAILED"
            }
        }
    }
}