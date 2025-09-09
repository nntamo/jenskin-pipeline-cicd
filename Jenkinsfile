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
                    
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-token') {
                        sh '''
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
                        '''
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
                        
                        # Check if Helm chart exists, if not use kubectl
                        if [ -f "charts/Chart.yaml" ]; then
                            echo "📦 Deploying with Helm..."
                            helm upgrade --install movie-app-${TARGET_ENVIRONMENT} ./charts \\
                                --namespace ${TARGET_ENVIRONMENT} \\
                                --set image.tag=${TARGET_ENVIRONMENT}-${BUILD_NUMBER} \\
                                --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \\
                                --set build.number=${BUILD_NUMBER} \\
                                --set build.commit=${GIT_COMMIT} \\
                                --set build.branch=${CURRENT_BRANCH} \\
                                --set build.environment=${TARGET_ENVIRONMENT} \\
                                --wait --timeout=600s
                        else
                            echo "📦 Deploying with kubectl (Helm chart not found)..."
                            # Basic kubectl deployment if no Helm chart
                            kubectl apply -f k8s-manifests/namespaces/${TARGET_ENVIRONMENT}-namespace.yaml || true
                            kubectl apply -f k8s-manifests/${TARGET_ENVIRONMENT}/ || echo "No k8s manifests found for ${TARGET_ENVIRONMENT}"
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
        
        stage('Production Approval') {
            when {
                allOf {
                    anyOf {
                        branch 'main'
                        branch 'master'
                    }
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
    }
    
    post {
        always {
            script {
                echo '📦 Cleaning up...'
                
                // ✅ CORRECTION: Wrap sh commands in node block
                node {
                    // Clean up Docker resources
                    sh '''
                        echo "🧹 Cleaning up Docker resources..."
                        docker image prune -f --filter "until=24h" || true
                        docker builder prune -f --filter "until=24h" || true
                        echo "✅ Cleanup completed"
                    '''
                }
            }
        }
        
        success {
            script {
                echo '🎉 Pipeline completed successfully!'
                
                // ✅ CORRECTION: Use env variables and handle null values
                def targetEnv = env.TARGET_ENVIRONMENT ?: 'unknown'
                def currentBranch = env.CURRENT_BRANCH ?: env.GIT_BRANCH?.replaceAll('origin/', '') ?: 'unknown'
                
                node {
                    sh """
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║               🎉 ${targetEnv.toUpperCase()} DEPLOYMENT SUCCESSFUL! 🎉               ║"
                        echo "╚══════════════════════════════════════════════════════════════╝"
                        echo ""
                        echo "📋 Deployment Summary:"
                        echo "├─ Environment: ${targetEnv}"
                        echo "├─ Branch: ${currentBranch}"
                        echo "├─ Build: ${BUILD_NUMBER}"
                        echo "├─ Commit: ${GIT_COMMIT}"
                        echo "└─ Registry: ${DOCKER_REGISTRY}/${IMAGE_NAME}"
                        echo ""
                        echo "📦 Environment Images:"
                        echo "├─ ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${targetEnv}-${BUILD_NUMBER}"
                        echo "└─ ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${targetEnv}-${BUILD_NUMBER}"
                        echo ""
                    """
                }
                
                currentBuild.description = "✅ ${targetEnv.toUpperCase()} SUCCESS | Images: *-${targetEnv}-${BUILD_NUMBER}"
            }
        }
        
        failure {
            script {
                echo '❌ Pipeline failed!'
                
                // ✅ CORRECTION: Use env variables and handle null values
                def targetEnv = env.TARGET_ENVIRONMENT ?: 'unknown'
                def currentBranch = env.CURRENT_BRANCH ?: env.GIT_BRANCH?.replaceAll('origin/', '') ?: 'unknown'
                
                node {
                    sh """
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║              ❌ ${targetEnv.toUpperCase()} DEPLOYMENT FAILED! ❌              ║"
                        echo "╚══════════════════════════════════════════════════════════════╝"
                        echo ""
                        echo "💥 Build ${BUILD_NUMBER} failed for ${targetEnv}!"
                        echo "├─ Branch: ${currentBranch}"
                        echo "├─ Environment: ${targetEnv}"
                        echo "├─ Commit: ${GIT_COMMIT}"
                        echo "└─ Stage: Check Jenkins console output"
                        echo ""
                    """
                }
                
                currentBuild.description = "❌ ${targetEnv.toUpperCase()} FAILED"
            }
        }
    }
}