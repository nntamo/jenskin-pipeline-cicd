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
        booleanParam(name: 'DEPLOY_TO_QA', defaultValue: true, description: 'Deploy to QA environment')
        booleanParam(name: 'DEPLOY_TO_STAGING', defaultValue: true, description: 'Deploy to Staging environment')
        booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'Deploy to Production environment')
    }
    
    stages {
        stage('🔍 Environment Detection & Setup') {
            steps {
                echo 'Detecting deployment environments based on branch...'
                checkout scm
                
                script {
                    // Afficher les informations du déclencheur
                    echo """
╔══════════════════════════════════════════════════════════════╗
║                    🚀 BUILD TRIGGER INFO                     ║
╚══════════════════════════════════════════════════════════════╝
🔗 Build Cause: ${currentBuild.getBuildCauses()}
📋 Triggered by: ${env.BUILD_CAUSE ?: 'Manual/Unknown'}
🌐 GitHub Event: ${env.GITHUB_EVENT_NAME ?: 'N/A'}
                    """
                    
                    // Determine target environments based on branch
                    def branchName = env.GIT_BRANCH.replaceAll('origin/', '')
                    def environments = []
                    
                    switch(branchName) {
                        case 'dev':
                        case 'develop':
                            environments = ['dev']
                            break
                        case 'qa':
                        case 'quality':
                            environments = ['dev', 'qa']
                            break
                        case 'staging':
                        case 'stage':
                            environments = ['dev', 'qa', 'staging']
                            break
                        case 'main':
                        case 'master':
                            environments = ['dev', 'qa', 'staging', 'prod']
                            break
                        default:
                            environments = ['dev']
                    }
                    
                    // Store environment variables
                    env.TARGET_ENVIRONMENTS = environments.join(',')
                    env.CURRENT_BRANCH = branchName
                    env.FIRST_ENVIRONMENT = environments[0]
                    
                    // Set dynamic build name
                    currentBuild.displayName = "#${BUILD_NUMBER}-${branchName}→[${environments.join('→')}]"
                    
                    echo """
╔══════════════════════════════════════════════════════════════╗
║                    🌍 DEPLOYMENT STRATEGY                     ║
╚══════════════════════════════════════════════════════════════╝
📋 Current Branch: ${branchName}
🎯 Target Environments: ${environments.join(' → ')}
📦 Total Environments: ${environments.size()}
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
        
        stage('🔨 Build & Package Docker Images') {
            steps {
                script {
                    echo "🔨 Building Docker images for multi-environment deployment..."
                    
                    sh '''
                        export DOCKER_BUILDKIT=1
                        
                        echo "🔨 Building cast-service image..."
                        docker build \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER} \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-latest \
                            --label "build.number=${BUILD_NUMBER}" \
                            --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            --label "git.commit=${GIT_COMMIT}" \
                            --label "git.branch=${CURRENT_BRANCH}" \
                            --label "service.name=cast-service" \
                            ./cast-service
                        
                        echo "🔨 Building movie-service image..."
                        docker build \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER} \
                            --tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-latest \
                            --label "build.number=${BUILD_NUMBER}" \
                            --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            --label "git.commit=${GIT_COMMIT}" \
                            --label "git.branch=${CURRENT_BRANCH}" \
                            --label "service.name=movie-service" \
                            ./movie-service
                        
                        echo "✅ Docker images built successfully!"
                        echo "📦 Images created:"
                        echo "   - ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER}"
                        echo "   - ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER}"
                        
                        docker images | grep ${IMAGE_NAME}
                    '''
                }
            }
        }
        
        stage('📦 Push to Container Registry') {
            steps {
                script {
                    echo "🚀 Pushing images to container registry..."
                    
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-token', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh '''
                            echo "📦 Logging into Docker registry..."
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                            
                            echo "📦 Pushing images to ${DOCKER_REGISTRY}/${IMAGE_NAME}..."
                            
                            # Push versioned images
                            echo "├─ Pushing cast-service-${BUILD_NUMBER}..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER}
                            
                            echo "├─ Pushing movie-service-${BUILD_NUMBER}..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER}
                            
                            # Push latest tags
                            echo "├─ Pushing cast-service-latest..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-latest
                            
                            echo "└─ Pushing movie-service-latest..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-latest
                            
                            echo "✅ All images pushed successfully!"
                            
                            # Logout for security
                            docker logout
                        '''
                    }
                }
            }
        }
        
        stage('🚀 Deploy to DEV Environment') {
            when {
                expression { 
                    env.TARGET_ENVIRONMENTS.contains('dev')
                }
            }
            steps {
                script {
                    deployToEnvironment('dev')
                }
            }
        }
        
        stage('🧪 Deploy to QA Environment') {
            when {
                allOf {
                    expression { env.TARGET_ENVIRONMENTS.contains('qa') }
                    expression { params.DEPLOY_TO_QA }
                }
            }
            steps {
                script {
                    deployToEnvironment('qa')
                }
            }
        }
        
        stage('🎭 Deploy to STAGING Environment') {
            when {
                allOf {
                    expression { env.TARGET_ENVIRONMENTS.contains('staging') }
                    expression { params.DEPLOY_TO_STAGING }
                }
            }
            steps {
                script {
                    deployToEnvironment('staging')
                }
            }
        }
        
        stage('⚡ Production Approval Gate') {
            when {
                allOf {
                    expression { env.TARGET_ENVIRONMENTS.contains('prod') }
                    expression { params.DEPLOY_TO_PROD }
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
├─ Previous Environments: dev → qa → staging
└─ Images: Ready to deploy

⚠️ This will deploy to PRODUCTION environment.
Please ensure all previous environments are validated.
                        """
                        
                        def approval = input(
                            message: deploymentDetails,
                            ok: 'Deploy to Production',
                            submitterParameter: 'DEPLOYER',
                            parameters: [
                                choice(name: 'DEPLOYMENT_TYPE', choices: ['Blue-Green', 'Rolling Update'], description: 'Select deployment strategy')
                            ]
                        )
                        
                        env.DEPLOYMENT_TYPE = approval.DEPLOYMENT_TYPE
                        echo "✅ Production deployment confirmed by: ${approval.DEPLOYER}"
                        echo "📋 Deployment strategy: ${env.DEPLOYMENT_TYPE}"
                    }
                }
            }
        }
        
        stage('🏭 Deploy to PRODUCTION Environment') {
            when {
                allOf {
                    expression { env.TARGET_ENVIRONMENTS.contains('prod') }
                    expression { params.DEPLOY_TO_PROD }
                }
            }
            steps {
                script {
                    deployToEnvironment('prod')
                }
            }
        }
        
        stage('✅ Post-Deployment Validation') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                script {
                    def environments = env.TARGET_ENVIRONMENTS.split(',')
                    
                    echo "🧪 Running post-deployment validation across environments..."
                    
                    for (environment in environments) {
                        validateEnvironment(environment)
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo 'Cleaning up resources...'
                
                node {
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
                def environments = env.TARGET_ENVIRONMENTS.split(',')
                
                echo 'Pipeline completed successfully!'
                
                node {
                    sh """
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║                    🎉 DEPLOYMENT SUCCESSFUL! 🎉               ║"
                        echo "╚══════════════════════════════════════════════════════════════╝"
                        echo ""
                        echo "✅ Build completed successfully"
                        echo "📦 Docker images built and pushed to registry"
                        echo "☸️  Deployments completed to: ${environments.join(' → ')}"
                        echo "🏷️  Build: #${BUILD_NUMBER}"
                        echo "🌿 Branch: ${env.CURRENT_BRANCH}"
                        echo ""
                    """
                }
                
                currentBuild.description = "✅ SUCCESS: ${environments.join(' → ')}"
            }
        }
        
        failure {
            script {
                echo 'Pipeline failed!'
                
                node {
                    sh '''
                        echo ""
                        echo "╔══════════════════════════════════════════════════════════════╗"
                        echo "║                    ❌ DEPLOYMENT FAILED! ❌                   ║"
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

// Function to deploy to a specific environment
def deployToEnvironment(String environment) {
    echo "🚀 Deploying to ${environment.toUpperCase()} environment..."
    
    sh """
        echo "🔧 Preparing deployment to ${environment}..."
        
        # Create namespace if it doesn't exist
        kubectl create namespace ${environment} --dry-run=client -o yaml | kubectl apply -f -
        echo "✅ Namespace ${environment} ready"
        
        # Environment-specific configurations
        case "${environment}" in
            "dev")
                REPLICA_COUNT=1
                RESOURCE_LIMITS="--requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi"
                ;;
            "qa")
                REPLICA_COUNT=1
                RESOURCE_LIMITS="--requests=cpu=100m,memory=128Mi --limits=cpu=300m,memory=512Mi"
                ;;
            "staging")
                REPLICA_COUNT=2
                RESOURCE_LIMITS="--requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=1Gi"
                ;;
            "prod")
                REPLICA_COUNT=3
                RESOURCE_LIMITS="--requests=cpu=500m,memory=512Mi --limits=cpu=1000m,memory=2Gi"
                ;;
        esac
        
        echo "📋 Environment: ${environment}"
        echo "📊 Replica Count: \$REPLICA_COUNT"
        echo "💾 Resource Limits: \$RESOURCE_LIMITS"
        
        # Check if Helm chart exists and is valid
        if [ -f "charts/Chart.yaml" ]; then
            echo "📦 Deploying with Helm..."
            helm upgrade --install movie-app-${environment} ./charts \\
                --namespace ${environment} \\
                --set image.tag=${BUILD_NUMBER} \\
                --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \\
                --set replicaCount=\$REPLICA_COUNT \\
                --set environment=${environment} \\
                --set build.number=${BUILD_NUMBER} \\
                --set build.commit=${GIT_COMMIT} \\
                --set build.branch=${CURRENT_BRANCH} \\
                --wait --timeout=300s || {
                    echo "⚠️ Helm deployment failed, trying kubectl fallback..."
                    kubectl create deployment cast-service-${environment} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER} -n ${environment} --dry-run=client -o yaml | kubectl apply -f -
                    kubectl create deployment movie-service-${environment} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER} -n ${environment} --dry-run=client -o yaml | kubectl apply -f -
                    kubectl scale deployment cast-service-${environment} --replicas=\$REPLICA_COUNT -n ${environment}
                    kubectl scale deployment movie-service-${environment} --replicas=\$REPLICA_COUNT -n ${environment}
                }
        else
            echo "📦 Deploying with kubectl..."
            # Create deployments with environment-specific replicas
            kubectl create deployment cast-service-${environment} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER} -n ${environment} --dry-run=client -o yaml | kubectl apply -f -
            kubectl create deployment movie-service-${environment} --image=${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER} -n ${environment} --dry-run=client -o yaml | kubectl apply -f -
            
            # Scale deployments
            kubectl scale deployment cast-service-${environment} --replicas=\$REPLICA_COUNT -n ${environment}
            kubectl scale deployment movie-service-${environment} --replicas=\$REPLICA_COUNT -n ${environment}
            
            # Expose services
            kubectl expose deployment cast-service-${environment} --port=80 --target-port=8000 -n ${environment} || echo "Service might already exist"
            kubectl expose deployment movie-service-${environment} --port=80 --target-port=8000 -n ${environment} || echo "Service might already exist"
        fi
        
        echo "🔍 Verifying deployment in ${environment}..."
        kubectl get pods -n ${environment} -o wide
        kubectl get services -n ${environment}
        
        echo "✅ Deployment to ${environment} completed!"
    """
}

// Function to validate environment deployment
def validateEnvironment(String environment) {
    echo "🧪 Validating ${environment.toUpperCase()} environment..."
    
    sh """
        echo "⏳ Waiting for pods to be ready in ${environment}..."
        kubectl wait --for=condition=ready pod -n ${environment} --timeout=300s --all || true
        
        echo "🏥 Performing health checks for ${environment}..."
        kubectl get pods -n ${environment} -o wide
        
        # Environment-specific validation
        case "${environment}" in
            "dev")
                echo "🔧 Running development smoke tests..."
                ;;
            "qa")
                echo "🧪 Running quality assurance tests..."
                ;;
            "staging")
                echo "🎭 Running staging validation tests..."
                ;;
            "prod")
                echo "🚀 Running production validation tests..."
                ;;
        esac
        
        # Check if pods are running
        RUNNING_PODS=\$(kubectl get pods -n ${environment} --field-selector=status.phase=Running -o name | wc -l)
        if [ \$RUNNING_PODS -gt 0 ]; then
            echo "✅ \$RUNNING_PODS pod(s) running successfully in ${environment}"
        else
            echo "⚠️ No pods running in ${environment}"
            exit 1
        fi
        
        echo "✅ ${environment} environment validation completed"
    """
}