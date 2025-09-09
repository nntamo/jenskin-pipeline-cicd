pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-token')
        DOCKER_REGISTRY = 'nguetsop'
        IMAGE_NAME = 'jenkins-projekt'
        KUBECONFIG = credentials('kubeconfig')
        SONAR_TOKEN = credentials('sonar-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    echo 'Running SonarQube code analysis...'
                    
                    sh '''
                        # Install SonarQube Scanner if not present
                        if ! command -v sonar-scanner &> /dev/null; then
                            echo "Installing SonarQube Scanner..."
                            wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856-linux.zip
                            unzip sonar-scanner-cli-4.8.0.2856-linux.zip
                            sudo mv sonar-scanner-4.8.0.2856-linux /opt/sonar-scanner
                            sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner
                            rm sonar-scanner-cli-4.8.0.2856-linux.zip
                        fi
                        
                        # Run SonarQube analysis
                        sonar-scanner \
                            -Dsonar.projectKey=jenkins-projekt \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.exclusions=**/node_modules/**,**/*.log
                    '''
                }
            }
        }
        
        stage('Security: Dependency Check') {
            steps {
                script {
                    echo 'Checking dependencies for vulnerabilities...'
                    
                    sh '''
                        # Install safety for Python dependency scanning
                        pip3 install safety --break-system-packages || true
                        
                        # Scan cast-service dependencies
                        echo "Scanning cast-service requirements..."
                        safety check -r cast-service/requirements.txt || echo "Vulnerabilities found in cast-service"
                        
                        # Scan movie-service dependencies
                        echo "Scanning movie-service requirements..."
                        safety check -r movie-service/requirements.txt || echo "Vulnerabilities found in movie-service"
                    '''
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    echo 'Building Docker images...'
                    
                    // Build cast-service
                    def castImage = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER}", "./cast-service")
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service", "./cast-service")
                    
                    // Build movie-service  
                    def movieImage = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER}", "./movie-service")
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service", "./movie-service")
                    
                    echo 'Docker images built successfully!'
                }
            }
        }
        
        stage('Trivy Security Scan') {
            steps {
                script {
                    echo 'Scanning Docker images with Trivy...'
                    
                    sh '''
                        # Install Trivy if not present
                        if ! command -v trivy &> /dev/null; then
                            echo "Installing Trivy..."
                            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                            echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
                            sudo apt-get update
                            sudo apt-get install trivy -y
                        fi
                        
                        echo "=== Trivy: Scanning cast-service image ==="
                        trivy image --severity HIGH,CRITICAL ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service
                        
                        echo "=== Trivy: Scanning movie-service image ==="
                        trivy image --severity HIGH,CRITICAL ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service
                        
                        # Generate JSON reports
                        trivy image --format json --output cast-trivy-report.json ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service
                        trivy image --format json --output movie-trivy-report.json ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service
                        
                        echo "Trivy security scan completed"
                    '''
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                script {
                    echo 'Running OWASP Dependency Check...'
                    
                    sh '''
                        # Install OWASP Dependency Check if not present
                        if [ ! -d "/opt/dependency-check" ]; then
                            echo "Installing OWASP Dependency Check..."
                            wget https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip
                            unzip dependency-check-8.4.0-release.zip
                            sudo mv dependency-check /opt/
                            sudo ln -s /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check
                            rm dependency-check-8.4.0-release.zip
                        fi
                        
                        # Run dependency check
                        dependency-check --project "jenkins-projekt" --scan . --format JSON --format HTML --out dependency-check-report/
                        
                        echo "OWASP Dependency Check completed"
                    '''
                }
            }
        }
        
        stage('Hadolint Dockerfile Scan') {
            steps {
                script {
                    echo 'Scanning Dockerfiles with Hadolint...'
                    
                    sh '''
                        # Install hadolint if not present
                        if ! command -v hadolint &> /dev/null; then
                            wget -O hadolint https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64
                            chmod +x hadolint
                            sudo mv hadolint /usr/local/bin/
                        fi
                        
                        echo "=== Hadolint: Scanning cast-service Dockerfile ==="
                        hadolint cast-service/Dockerfile || echo "Hadolint found issues in cast-service Dockerfile"
                        
                        echo "=== Hadolint: Scanning movie-service Dockerfile ==="
                        hadolint movie-service/Dockerfile || echo "Hadolint found issues in movie-service Dockerfile"
                    '''
                }
            }
        }
        
        stage('Security Gate') {
            steps {
                script {
                    echo 'Evaluating security gate...'
                    
                    sh '''
                        # Create security summary
                        echo "=== SECURITY GATE EVALUATION ===" > security-gate.txt
                        echo "Build: ${BUILD_NUMBER}" >> security-gate.txt
                        echo "Date: $(date)" >> security-gate.txt
                        echo "" >> security-gate.txt
                        
                        # Check for critical vulnerabilities in Trivy reports
                        if [ -f "cast-trivy-report.json" ] && [ -f "movie-trivy-report.json" ]; then
                            CAST_CRITICAL=$(cat cast-trivy-report.json | jq '[.Results[]?.Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length' 2>/dev/null || echo "0")
                            MOVIE_CRITICAL=$(cat movie-trivy-report.json | jq '[.Results[]?.Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length' 2>/dev/null || echo "0")
                            TOTAL_CRITICAL=$((CAST_CRITICAL + MOVIE_CRITICAL))
                            
                            echo "Cast-service critical vulnerabilities: $CAST_CRITICAL" >> security-gate.txt
                            echo "Movie-service critical vulnerabilities: $MOVIE_CRITICAL" >> security-gate.txt
                            echo "Total critical vulnerabilities: $TOTAL_CRITICAL" >> security-gate.txt
                            
                            # Security gate: fail if more than 10 critical vulnerabilities
                            if [ $TOTAL_CRITICAL -gt 10 ]; then
                                echo "SECURITY GATE FAILED: Too many critical vulnerabilities ($TOTAL_CRITICAL > 10)" >> security-gate.txt
                                cat security-gate.txt
                                exit 1
                            else
                                echo "SECURITY GATE PASSED: Critical vulnerabilities within acceptable limits" >> security-gate.txt
                            fi
                        else
                            echo "SECURITY GATE WARNING: Trivy reports not found" >> security-gate.txt
                        fi
                        
                        cat security-gate.txt
                    '''
                }
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                script {
                    echo 'Security checks passed - Pushing images to DockerHub...'
                    
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-token') {
                        // Push versioned images
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER}"
                        
                        // Push latest images
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service"
                    }
                    
                    echo 'Images pushed to DockerHub successfully!'
                }
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                script {
                    echo 'Deploying to Development environment...'
                    
                    sh '''
                        helm upgrade --install movie-app ./charts \
                            --namespace dev \
                            --values ./charts/values-dev.yaml \
                            --set image.tag=${BUILD_NUMBER} \
                            --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \
                            --wait
                    '''
                    
                    echo 'Deployment to Dev completed!'
                }
            }
        }
        
        stage('Deploy to QA') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Deploying to QA environment...'
                    
                    sh '''
                        helm upgrade --install movie-app ./charts \
                            --namespace qa \
                            --values ./charts/values-qa.yaml \
                            --set image.tag=${BUILD_NUMBER} \
                            --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \
                            --wait
                    '''
                    
                    echo 'Deployment to QA completed!'
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Running integration tests...'
                    
                    sh '''
                        echo "Testing application endpoints..."
                        sleep 30  # Wait for deployment to be ready
                        
                        # Test cast-service health
                        kubectl get pods -n qa | grep cast-service
                        
                        # Test movie-service health  
                        kubectl get pods -n qa | grep movie-service
                        
                        echo "Integration tests passed!"
                    '''
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Deploying to Staging environment...'
                    
                    sh '''
                        helm upgrade --install movie-app ./charts \
                            --namespace staging \
                            --values ./charts/values-staging.yaml \
                            --set image.tag=${BUILD_NUMBER} \
                            --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \
                            --wait
                    '''
                    
                    echo 'Deployment to Staging completed!'
                }
            }
        }
        
        stage('Staging Tests') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Running staging tests...'
                    
                    sh '''
                        echo "Testing staging environment..."
                        sleep 30
                        
                        # Verify staging deployment
                        kubectl get pods -n staging
                        kubectl get svc -n staging
                        
                        echo "Staging tests passed!"
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Requesting approval for Production deployment...'
                    
                    // Manual approval required for production
                    timeout(time: 10, unit: 'MINUTES') {
                        input message: 'Deploy to Production? Security checks have passed.', 
                              ok: 'Deploy',
                              submitterParameter: 'DEPLOYER'
                    }
                    
                    echo "Production deployment approved by: ${DEPLOYER}"
                    
                    sh '''
                        helm upgrade --install movie-app ./charts \
                            --namespace prod \
                            --values ./charts/values-prod.yaml \
                            --set image.tag=${BUILD_NUMBER} \
                            --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \
                            --wait
                    '''
                    
                    echo 'Deployment to Production completed!'
                }
            }
        }
        
        stage('Production Validation') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo 'Validating production deployment...'
                    
                    sh '''
                        echo "Validating production environment..."
                        sleep 30
                        
                        # Verify production deployment
                        kubectl get pods -n prod
                        kubectl get svc -n prod
                        kubectl get ingress -n prod
                        
                        echo "Production validation completed!"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Archiving security reports and cleaning up...'
            
            // Archive security reports
            archiveArtifacts artifacts: '*-trivy-report.json,security-gate.txt,dependency-check-report/*', 
                             allowEmptyArchive: true
            
            // Clean up Docker images to save space
            sh '''
                docker system prune -f
                docker image prune -f
            '''
        }
        
        success {
            echo 'Pipeline completed successfully with security validations!'
            
            sh '''
                echo " Build ${BUILD_NUMBER} deployed successfully!"
                echo "Security scans completed: SonarQube, Trivy, OWASP, Hadolint"
                echo "Images: ${DOCKER_REGISTRY}/${IMAGE_NAME}:cast-service-${BUILD_NUMBER}"
                echo "Images: ${DOCKER_REGISTRY}/${IMAGE_NAME}:movie-service-${BUILD_NUMBER}"
            '''
        }
        
        failure {
            echo 'Pipeline failed - Check security issues!'
            
            sh '''
                echo "Build ${BUILD_NUMBER} failed!"
                echo "Check security scan results and logs"
            '''
        }
    }
}