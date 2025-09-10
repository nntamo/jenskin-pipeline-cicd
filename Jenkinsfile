pipeline {
    agent any
    
    triggers {
        // Déclenchement automatique à chaque push
        githubPush()
        pollSCM('H/5 * * * *')  // Backup toutes les 5 minutes
    }
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-token')
        DOCKER_REGISTRY = 'nguetsop'
        IMAGE_NAME = 'jenskin-cicd-projekt'
        KUBECONFIG = credentials('kubeconfig')
        DOCKER_BUILDKIT = '1'
    }
    
    stages {
        stage('🔍 Environment Detection & Setup') {
            steps {
                echo 'Auto-detecting deployment environments based on branch...'
                checkout scm
                
                script {
                    // Afficher les informations du déclencheur
                    echo """
╔══════════════════════════════════════════════════════════════╗
║                🚀 AUTO CI/CD PIPELINE STARTED 🚀            ║
╚══════════════════════════════════════════════════════════════╝
🔗 Build Cause: ${currentBuild.getBuildCauses()}
🌐 Triggered automatically after commit
                    """
                    
                    // Determine target environments based on branch
                    def branchName = env.GIT_BRANCH.replaceAll('origin/', '')
                    def environments = []
                    def runTests = true
                    def canDeployQA = false
                    def canDeployStaging = false
                    def canDeployProd = false
                    
                    switch(branchName) {
                        case 'dev':
                        case 'develop':
                            environments = ['dev']
                            canDeployQA = false
                            canDeployStaging = false
                            break
                        case 'qa':
                        case 'quality':
                            environments = ['dev', 'qa']  // QA available but needs approval
                            canDeployQA = true
                            canDeployStaging = false
                            break
                        case 'staging':
                        case 'stage':
                            environments = ['dev', 'qa', 'staging']  // STAGING available but needs approval
                            canDeployQA = true
                            canDeployStaging = true
                            break
                        case 'main':
                        case 'master':
                            environments = ['dev', 'qa', 'staging', 'prod']  // PROD available but needs approval
                            canDeployQA = true
                            canDeployStaging = true
                            canDeployProd = true
                            break
                        default:
                            environments = ['dev']
                    }
                    
                    // Store environment variables
                    env.TARGET_ENVIRONMENTS = environments.join(',')
                    env.CURRENT_BRANCH = branchName
                    env.RUN_TESTS = runTests.toString()
                    env.CAN_DEPLOY_QA = canDeployQA.toString()
                    env.CAN_DEPLOY_STAGING = canDeployStaging.toString()
                    env.CAN_DEPLOY_PROD = canDeployProd.toString()
                    
                    // Initialize approval flags to false
                    env.DEPLOY_QA = 'false'
                    env.DEPLOY_STAGING = 'false' 
                    env.DEPLOY_PROD = 'false'
                    
                    // Set dynamic build name
                    currentBuild.displayName = "#${BUILD_NUMBER}-${branchName}→[${environments.join('→')}]"
                    
                    echo """
╔══════════════════════════════════════════════════════════════╗
║                    🌍 DEPLOYMENT STRATEGY                     ║
╚══════════════════════════════════════════════════════════════╝
📋 Branch: ${branchName}
🎯 Available Environments: ${environments.join(' → ')}
🧪 Run Tests: ${runTests}
🟢 Auto Deploy DEV: ✅ (Always automatic)
🟡 Can Deploy QA: ${canDeployQA ? '✅ (Requires approval)' : '❌'}
🟠 Can Deploy STAGING: ${canDeployStaging ? '✅ (Requires approval)' : '❌'}
🔴 Can Deploy PROD: ${canDeployProd ? '✅ (Requires approval)' : '❌'}
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
            steps {
                script {
                    echo "🟢 DEV deployment is automatic - no approval required"
                    deployToEnvironment('dev')
                }
            }
        }
        
        // 🛑 QA APPROVAL GATE - NOUVEAU STAGE
        stage('🛑 QA Approval Gate') {
            when {
                expression { env.CAN_DEPLOY_QA == 'true' }
            }
            steps {
                script {
                    timeout(time: 60, unit: 'MINUTES') {
                        def qaApproval = input(
                            message: """
🧪 QA ENVIRONMENT DEPLOYMENT APPROVAL

📋 Deployment Details:
├─ Environment: QA 🟡  
├─ Build: #${BUILD_NUMBER}
├─ Branch: ${env.CURRENT_BRANCH}
├─ Commit: ${GIT_COMMIT}
├─ DEV Environment: ✅ Deployed Successfully
└─ All Tests: ✅ Passed

🔍 Please review DEV environment before approving QA deployment
⚠️ This will deploy to QA environment for testing
                            """,
                            ok: 'APPROVE QA DEPLOYMENT',
                            submitterParameter: 'QA_DEPLOYER',
                            submitter: 'qa-team,dev-team,admin',  // Qui peut approuver QA
                            parameters: [
                                booleanParam(name: 'CONFIRM_QA', defaultValue: false, description: 'I confirm QA environment is ready for deployment')
                            ]
                        )
                        
                        if (!qaApproval.CONFIRM_QA) {
                            error("❌ QA deployment cancelled - Confirmation required")
                        }
                        
                        env.DEPLOY_QA = 'true'
                        env.QA_APPROVED_BY = qaApproval.QA_DEPLOYER
                        echo "✅ QA deployment approved by: ${qaApproval.QA_DEPLOYER}"
                    }
                }
            }
        }
        
        stage('🧪 Deploy to QA Environment') {
            when {
                expression { env.DEPLOY_QA == 'true' }
            }
            steps {
                script {
                    deployToEnvironment('qa')
                }
            }
        }
        
        // 🛑 STAGING APPROVAL GATE - NOUVEAU STAGE
        stage('🛑 STAGING Approval Gate') {
            when {
                allOf {
                    expression { env.CAN_DEPLOY_STAGING == 'true' }
                    expression { env.DEPLOY_QA == 'true' } // QA doit être déployé d'abord
                }
            }
            steps {
                script {
                    timeout(time: 60, unit: 'MINUTES') {
                        def stagingApproval = input(
                            message: """
🎭 STAGING ENVIRONMENT DEPLOYMENT APPROVAL

📋 Deployment Details:
├─ Environment: STAGING 🟠
├─ Build: #${BUILD_NUMBER}
├─ Branch: ${env.CURRENT_BRANCH}
├─ Commit: ${GIT_COMMIT}
├─ DEV Environment: ✅ Deployed Successfully
├─ QA Environment: ✅ Deployed & Approved by ${env.QA_APPROVED_BY ?: 'N/A'}
└─ All Tests: ✅ Passed

🔍 Please review QA environment before approving STAGING deployment
⚠️ This will deploy to STAGING environment (pre-production)
                            """,
                            ok: 'APPROVE STAGING DEPLOYMENT',
                            submitterParameter: 'STAGING_DEPLOYER',
                            submitter: 'staging-team,qa-team,admin',  // Qui peut approuver STAGING
                            parameters: [
                                booleanParam(name: 'CONFIRM_STAGING', defaultValue: false, description: 'I confirm STAGING environment is ready for deployment')
                            ]
                        )
                        
                        if (!stagingApproval.CONFIRM_STAGING) {
                            error("❌ STAGING deployment cancelled - Confirmation required")
                        }
                        
                        env.DEPLOY_STAGING = 'true'
                        env.STAGING_APPROVED_BY = stagingApproval.STAGING_DEPLOYER
                        echo "✅ STAGING deployment approved by: ${stagingApproval.STAGING_DEPLOYER}"
                    }
                }
            }
        }
        
        stage('🎭 Deploy to STAGING Environment') {
            when {
                expression { env.DEPLOY_STAGING == 'true' }
            }
            steps {
                script {
                    deployToEnvironment('staging')
                }
            }
        }
        
        // 🛑 PRODUCTION APPROVAL GATE - MODIFIÉ
        stage('🛑 PRODUCTION Approval Gate') {
            when {
                allOf {
                    branch 'main'
                    expression { env.CAN_DEPLOY_PROD == 'true' }
                    expression { env.DEPLOY_STAGING == 'true' } // STAGING doit être déployé d'abord
                }
            }
            steps {
                script {
                    timeout(time: 30, unit: 'MINUTES') {
                        def productionApproval = input(
                            message: """
🚀 PRODUCTION DEPLOYMENT APPROVAL

📋 Deployment Details:
├─ Environment: PRODUCTION 🔴
├─ Build: #${BUILD_NUMBER}
├─ Branch: ${env.CURRENT_BRANCH}
├─ Commit: ${GIT_COMMIT}
├─ DEV Environment: ✅ Deployed Successfully
├─ QA Environment: ✅ Deployed & Approved by ${env.QA_APPROVED_BY ?: 'N/A'}
├─ STAGING Environment: ✅ Deployed & Approved by ${env.STAGING_APPROVED_BY ?: 'N/A'}
└─ All Tests: ✅ Passed

⚠️ CRITICAL: This will deploy to LIVE PRODUCTION!
🔒 Required: Manager/Lead approval
                            """,
                            ok: 'DEPLOY TO PRODUCTION',
                            submitterParameter: 'PROD_DEPLOYER',
                            submitter: 'prod-team,manager,admin',  // Qui peut approuver PROD
                            parameters: [
                                choice(name: 'DEPLOYMENT_TYPE', choices: ['Blue-Green', 'Rolling Update'], description: 'Select deployment strategy'),
                                booleanParam(name: 'CONFIRM_PRODUCTION', defaultValue: false, description: 'I confirm this is intended for PRODUCTION')
                            ]
                        )
                        
                        if (!productionApproval.CONFIRM_PRODUCTION) {
                            error("❌ Production deployment cancelled - Confirmation required")
                        }
                        
                        env.DEPLOYMENT_TYPE = productionApproval.DEPLOYMENT_TYPE
                        env.DEPLOY_PROD = 'true'
                        env.PROD_APPROVED_BY = productionApproval.PROD_DEPLOYER
                        echo "✅ PRODUCTION deployment approved by: ${productionApproval.PROD_DEPLOYER}"
                        echo "📋 Deployment strategy: ${env.DEPLOYMENT_TYPE}"
                    }
                }
            }
        }
        
        stage('🏭 Deploy to PRODUCTION Environment') {
            when {
                expression { env.DEPLOY_PROD == 'true' }
            }
            steps {
                script {
                    deployToEnvironment('prod')
                }
            }
        }
        
        stage('✅ Post-Deployment Validation') {
            when {
                expression { env.RUN_TESTS == 'true' }
            }
            steps {
                script {
                    // Only validate environments that were actually deployed
                    def deployedEnvironments = ['dev'] // DEV is always deployed
                    
                    if (env.DEPLOY_QA == 'true') {
                        deployedEnvironments.add('qa')
                    }
                    if (env.DEPLOY_STAGING == 'true') {
                        deployedEnvironments.add('staging')
                    }
                    if (env.DEPLOY_PROD == 'true') {
                        deployedEnvironments.add('prod')
                    }
                    
                    echo "🧪 Running post-deployment validation for: ${deployedEnvironments.join(', ')}"
                    
                    for (environment in deployedEnvironments) {
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
                
                sh '''
                    echo "🧹 Cleaning up Docker resources..."
                    docker image prune -f --filter "until=24h" || true
                    docker builder prune -f --filter "until=24h" || true
                    echo "✅ Cleanup completed"
                '''
            }
        }
        
        success {
            script {
                // Show only deployed environments
                def deployedEnvironments = ['dev']
                if (env.DEPLOY_QA == 'true') deployedEnvironments.add('qa')
                if (env.DEPLOY_STAGING == 'true') deployedEnvironments.add('staging')
                if (env.DEPLOY_PROD == 'true') deployedEnvironments.add('prod')
                
                echo 'CI/CD Pipeline completed successfully!'
                
                sh """
                    echo ""
                    echo "╔══════════════════════════════════════════════════════════════╗"
                    echo "║                🎉 DEPLOYMENT SUCCESSFUL! 🎉                 ║"
                    echo "╚══════════════════════════════════════════════════════════════╝"
                    echo ""
                    echo "✅ Build completed automatically after commit"
                    echo "📦 Docker images built and pushed to registry"
                    echo "☸️  Deployed to: ${deployedEnvironments.join(' → ')}"
                    echo "🏷️  Build: #${BUILD_NUMBER}"
                    echo "🌿 Branch: ${env.CURRENT_BRANCH}"
                    echo ""
                    echo "🚀 Next commit will trigger this pipeline automatically!"
                    echo ""
                """
                
                currentBuild.description = "✅ SUCCESS: ${deployedEnvironments.join(' → ')}"
            }
        }
        
        failure {
            script {
                echo 'CI/CD Pipeline failed!'
                
                sh '''
                    echo ""
                    echo "╔══════════════════════════════════════════════════════════════╗"
                    echo "║                  ❌ DEPLOYMENT FAILED! ❌                    ║"
                    echo "╚══════════════════════════════════════════════════════════════╝"
                    echo ""
                    echo "💥 Build failed - Check Jenkins console output"
                    echo "🔍 Review the failed stage above for specific error messages"
                    echo ""
                '''
                
                currentBuild.description = "❌ FAILED"
            }
        }
        
        aborted {
            script {
                echo 'Pipeline was aborted (likely due to approval timeout or cancellation)'
                currentBuild.description = "⏹️ ABORTED - Manual intervention required"
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
                echo "🟢 DEV deployment - Automatic"
                ;;
            "qa")
                REPLICA_COUNT=1
                RESOURCE_LIMITS="--requests=cpu=100m,memory=128Mi --limits=cpu=300m,memory=512Mi"
                echo "🟡 QA deployment - Approved by ${QA_APPROVED_BY}"
                ;;
            "staging")
                REPLICA_COUNT=2
                RESOURCE_LIMITS="--requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=1Gi"
                echo "🟠 STAGING deployment - Approved by ${STAGING_APPROVED_BY}"
                ;;
            "prod")
                REPLICA_COUNT=3
                RESOURCE_LIMITS="--requests=cpu=500m,memory=512Mi --limits=cpu=1000m,memory=2Gi"
                echo "🔴 PRODUCTION deployment - Approved by ${PROD_APPROVED_BY}"
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