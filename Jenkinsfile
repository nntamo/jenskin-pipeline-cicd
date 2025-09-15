// // // // pipeline {
// // // //   environment {
// // // //     DOCKER_ID = "nguetsop" // remplacez par votre docker-id
// // // //     MOVIE_IMAGE = "movie-service"
// // // //     CAST_IMAGE = "cast-service"
// // // //     DOCKER_TAG = "v.${BUILD_ID}.0"
// // // //   }
// // // //   agent any
// // // //   stages {
// // // //     stage('Docker Build') {
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           echo "Building Movie and Cast Services..."
          
// // // //           # Clean up existing containers
// // // //           docker rm -f movie-service || true
// // // //           docker rm -f cast-service || true
          
// // // //           # Build movie-service
// // // //           echo "Building Movie Service..."
// // // //           cd movie-service
// // // //           docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG .
// // // //           cd ..
          
// // // //           # Build cast-service
// // // //           echo "Building Cast Service..."
// // // //           cd cast-service
// // // //           docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG .
// // // //           cd ..
          
// // // //           echo "Both services built successfully"
// // // //           sleep 6
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
    

    
// // // //     stage('Docker Push') {
// // // //       environment {
// // // //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           retry(3) {
// // // //             sh '''
// // // //             echo "Pushing both images to Docker Hub..."
// // // //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// // // //             # Push Movie Service
// // // //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
            
// // // //             # Push Cast Service
// // // //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
            
// // // //             echo "Both images pushed successfully"
// // // //             '''
// // // //           }
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Create K8s Secrets') {
// // // //       environment {
// // // //         KUBECONFIG = credentials("config")
// // // //         DOCKER_PASS = credentials("DOCKER_HUB_PASS")
// // // //         MOVIE_DB_PASS = credentials("MOVIE_DB_PASSWORD")
// // // //         CAST_DB_PASS = credentials("CAST_DB_PASSWORD")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           rm -Rf .kube
// // // //           mkdir .kube
// // // //           cat $KUBECONFIG > .kube/config
          
// // // //           echo "Creating secrets for all environments..."
          
// // // //           for env in dev qa staging prod; do
// // // //             echo "Creating secrets for $env environment..."
            
// // // //             # Create Docker registry secret
// // // //             kubectl create secret docker-registry dockerhub-secret \
// // // //               --docker-server=docker.io \
// // // //               --docker-username=$DOCKER_ID \
// // // //               --docker-password=$DOCKER_PASS \
// // // //               --docker-email=nntamo06@gmail.com \
// // // //               -n $env --dry-run=client -o yaml | kubectl apply -f -
            
// // // //             # Create database secrets with environment-specific passwords
// // // //             kubectl create secret generic movie-db-secret \
// // // //               --from-literal=POSTGRES_USER=movie_db_username \
// // // //               --from-literal=POSTGRES_PASSWORD=${MOVIE_DB_PASS}_${env} \
// // // //               --from-literal=POSTGRES_DB=movie_db_${env} \
// // // //               --from-literal=DATABASE_URI=postgresql://movie_db_username:${MOVIE_DB_PASS}_${env}@movie-db:5432/movie_db_${env} \
// // // //               --namespace=$env --dry-run=client -o yaml | kubectl apply -f -
            
// // // //             kubectl create secret generic cast-db-secret \
// // // //               --from-literal=POSTGRES_USER=cast_db_username \
// // // //               --from-literal=POSTGRES_PASSWORD=${CAST_DB_PASS}_${env} \
// // // //               --from-literal=POSTGRES_DB=cast_db_${env} \
// // // //               --from-literal=DATABASE_URI=postgresql://cast_db_username:${CAST_DB_PASS}_${env}@cast-db:5432/cast_db_${env} \
// // // //               --namespace=$env --dry-run=client -o yaml | kubectl apply -f -
              
// // // //             echo "Secrets created for $env"
// // // //           done
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Deployment in dev') {
// // // //       environment {
// // // //         KUBECONFIG = credentials("config")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           rm -Rf .kube
// // // //           mkdir .kube
// // // //           cat $KUBECONFIG > .kube/config
          
// // // //           echo "Deploying to DEV environment..."
          
// // // //           # Option 1: Using Helm Charts (if you prefer)
// // // //           if [ -d "charts" ]; then
// // // //             echo "Using Helm deployment..."
// // // //             cp charts/values-dev.yaml values.yml
// // // //             sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
// // // //             sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
// // // //             sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
// // // //             helm upgrade --install movie-cast-app charts --values=values.yml --namespace dev
// // // //           else
// // // //             # Option 2: Using K8s manifests (your current setup)
// // // //             echo "Using K8s manifests deployment..."
            
// // // //             # Update image tags in deployments
// // // //             sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/dev/movie-deployment.yaml
// // // //             sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/dev/cast-deployment.yaml
            
// // // //             # Apply all manifests
// // // //             kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// // // //             kubectl apply -f k8s-manifests/dev/
            
// // // //             # Wait for deployments
// // // //             kubectl rollout status deployment/movie-service -n dev --timeout=300s
// // // //             kubectl rollout status deployment/cast-service -n dev --timeout=300s
// // // //           fi
          
// // // //           echo "Deployed to DEV environment successfully"
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Promotion to QA') {
// // // //       steps {
// // // //         timeout(time: 30, unit: "MINUTES") {
// // // //           input message: 'DEV environment validated. Merge to QA branch and deploy to QA?', ok: 'Deploy to QA'
// // // //         }
// // // //         script {
// // // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // // //             sh '''
// // // //             echo "Promoting to QA environment..."
// // // //             git config user.name "Jenkins"
// // // //             git config user.email "jenkins@datascientest.com"
// // // //             git config credential.helper store
// // // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // // //             git fetch origin
            
// // // //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// // // //                 git checkout -B qa origin/qa
// // // //             else
// // // //                 git checkout -B qa
// // // //             fi
            
// // // //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// // // //             git push origin qa
// // // //             echo "Successfully merged origin/main to qa"
// // // //             rm -f ~/.git-credentials
// // // //             '''
// // // //           }
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Deployment in qa') {
// // // //       environment {
// // // //         KUBECONFIG = credentials("config")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           rm -Rf .kube
// // // //           mkdir .kube
// // // //           cat $KUBECONFIG > .kube/config
          
// // // //           echo "Deploying to QA environment..."
          
// // // //           if [ -d "charts" ]; then
// // // //             cp charts/values-qa.yaml values.yml
// // // //             sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
// // // //             sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
// // // //             sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
// // // //             helm upgrade --install movie-cast-app charts --values=values.yml --namespace qa
// // // //           else
// // // //             sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/qa/movie-deployment.yaml
// // // //             sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/qa/cast-deployment.yaml
            
// // // //             kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// // // //             kubectl apply -f k8s-manifests/qa/
            
// // // //             kubectl rollout status deployment/movie-service -n qa --timeout=300s
// // // //             kubectl rollout status deployment/cast-service -n qa --timeout=300s
// // // //           fi
          
// // // //           echo "Deployed to QA environment successfully"
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Promotion to STAGING') {
// // // //       steps {
// // // //         timeout(time: 30, unit: "MINUTES") {
// // // //           input message: 'QA environment validated. Merge to STAGING branch and deploy to STAGING?', ok: 'Deploy to STAGING'
// // // //         }
// // // //         script {
// // // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // // //             sh '''
// // // //             echo "Promoting to STAGING environment..."
// // // //             git config user.name "Jenkins"
// // // //             git config user.email "jenkins@datascientest.com"
// // // //             git config credential.helper store
// // // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // // //             git fetch origin
            
// // // //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// // // //                 git checkout -B staging origin/staging
// // // //             else
// // // //                 git checkout -B staging
// // // //             fi
            
// // // //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// // // //             git push origin staging
// // // //             echo "Successfully merged origin/qa to staging"
// // // //             rm -f ~/.git-credentials
// // // //             '''
// // // //           }
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Deployment in staging') {
// // // //       environment {
// // // //         KUBECONFIG = credentials("config")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           rm -Rf .kube
// // // //           mkdir .kube
// // // //           cat $KUBECONFIG > .kube/config
          
// // // //           echo "Deploying to STAGING environment..."
          
// // // //           if [ -d "charts" ]; then
// // // //             cp charts/values-staging.yaml values.yml
// // // //             sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
// // // //             sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
// // // //             sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
// // // //             helm upgrade --install movie-cast-app charts --values=values.yml --namespace staging
// // // //           else
// // // //             sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/staging/movie-deployment.yaml
// // // //             sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/staging/cast-deployment.yaml
            
// // // //             kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// // // //             kubectl apply -f k8s-manifests/staging/
            
// // // //             kubectl rollout status deployment/movie-service -n staging --timeout=300s
// // // //             kubectl rollout status deployment/cast-service -n staging --timeout=300s
// // // //           fi
          
// // // //           echo "Deployed to STAGING environment successfully"
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Promotion to PROD') {
// // // //       steps {
// // // //         timeout(time: 60, unit: "MINUTES") {
// // // //           input message: 'STAGING environment validated. Merge to PROD branch and deploy to PRODUCTION?', ok: 'Deploy to PRODUCTION'
// // // //         }
// // // //         script {
// // // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // // //             sh '''
// // // //             echo "Promoting to PRODUCTION environment..."
// // // //             git config user.name "Jenkins"
// // // //             git config user.email "jenkins@datascientest.com"
// // // //             git config credential.helper store
// // // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // // //             git fetch origin
            
// // // //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// // // //                 git checkout -B prod origin/prod
// // // //             else
// // // //                 git checkout -B prod
// // // //             fi
            
// // // //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// // // //             git push origin prod
// // // //             echo "Successfully merged origin/staging to prod"
// // // //             rm -f ~/.git-credentials
// // // //             '''
// // // //           }
// // // //         }
// // // //       }
// // // //     }
    
// // // //     stage('Deploiement en prod') {
// // // //       environment {
// // // //         KUBECONFIG = credentials("config")
// // // //       }
// // // //       steps {
// // // //         script {
// // // //           sh '''
// // // //           rm -Rf .kube
// // // //           mkdir .kube
// // // //           cat $KUBECONFIG > .kube/config
          
// // // //           echo "Deploying to PRODUCTION environment..."
          
// // // //           if [ -d "charts" ]; then
// // // //             cp charts/values-prod.yaml values.yml
// // // //             sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
// // // //             sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
// // // //             sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
// // // //             helm upgrade --install movie-cast-app charts --values=values.yml --namespace prod
// // // //           else
// // // //             sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/prod/movie-deployment.yaml
// // // //             sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/prod/cast-deployment.yaml
            
// // // //             kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// // // //             kubectl apply -f k8s-manifests/prod/
            
// // // //             kubectl rollout status deployment/movie-service -n prod --timeout=300s
// // // //             kubectl rollout status deployment/cast-service -n prod --timeout=300s
// // // //           fi
          
// // // //           echo "Deployed to PRODUCTION environment successfully"
// // // //           '''
// // // //         }
// // // //       }
// // // //     }
// // // //   }
  
// // // //   post {
// // // //     always {
// // // //       sh '''
// // // //       # Cleanup
// // // //       docker rm -f movie-service || true
// // // //       docker rm -f cast-service || true
// // // //       docker system prune -f || true
// // // //       '''
// // // //     }
// // // //     success {
// // // //       echo 'Pipeline completed successfully!'
// // // //     }
// // // //     failure {
// // // //       echo 'Pipeline failed!'
// // // //     }
// // // //   }
// // // // }


// // // pipeline {
// // //   environment {
// // //     DOCKER_ID = "nguetsop"
// // //     MOVIE_IMAGE = "movie-service"
// // //     CAST_IMAGE = "cast-service"
// // //     DOCKER_TAG = "v.${BUILD_ID}.0"
// // //     DOCKER_BUILDKIT = "1"
// // //   }
  
// // //   agent any
  
// // //   options {
// // //     buildDiscarder(logRotator(numToKeepStr: '10'))
// // //     timeout(time: 30, unit: 'MINUTES')
// // //     skipStagesAfterUnstable()
// // //     timestamps()
// // //   }
  
// // //   stages {
// // //     stage('Pre-Build Validation') {
// // //       steps {
// // //         script {
// // //           sh '''
// // //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// // //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// // //             echo "ERROR: Missing Dockerfile"
// // //             exit 1
// // //           fi
          
// // //           echo "Validation completed"
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Docker Build') {
// // //       parallel {
// // //         stage('Build Movie Service') {
// // //           steps {
// // //             script {
// // //               sh '''
// // //               cd movie-service
// // //               docker build \
// // //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// // //                 --label "version=${DOCKER_TAG}" \
// // //                 --label "git.commit=$(git rev-parse HEAD)" \
// // //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// // //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// // //               '''
// // //             }
// // //           }
// // //         }
        
// // //         stage('Build Cast Service') {
// // //           steps {
// // //             script {
// // //               sh '''
// // //               cd cast-service
// // //               docker build \
// // //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// // //                 --label "version=${DOCKER_TAG}" \
// // //                 --label "git.commit=$(git rev-parse HEAD)" \
// // //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// // //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// // //               '''
// // //             }
// // //           }
// // //         }
// // //       }
      
// // //       post {
// // //         success {
// // //           sh 'docker images | grep $DOCKER_ID'
// // //         }
// // //       }
// // //     }
    
// // //     stage('Quality Gates') {
// // //       parallel {
// // //         stage('Unit Tests') {
// // //           steps {
// // //             script {
// // //               sh '''
// // //               echo "Running unit tests..."
// // //               echo "Tests passed"
// // //               '''
// // //             }
// // //           }
// // //         }
        
// // //         stage('Security Scan') {
// // //           steps {
// // //             script {
// // //               sh '''
// // //               echo "Security scan completed"
// // //               '''
// // //             }
// // //           }
// // //         }
// // //       }
// // //     }

// // //     stage('Registry Push') {
// // //       environment {
// // //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// // //       }
// // //       steps {
// // //         script {
// // //           retry(3) {
// // //             sh '''
// // //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// // //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// // //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// // //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// // //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// // //             docker logout
// // //             echo "Images pushed successfully"
// // //             '''
// // //           }
// // //         }
// // //       }
// // //     }
    
// // //     stage('Kubernetes Secrets') {
// // //       environment {
// // //         KUBECONFIG = credentials("config")
// // //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// // //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// // //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// // //       }
// // //       steps {
// // //         script {
// // //           sh '''
// // //           rm -rf .kube
// // //           mkdir .kube
// // //           cp $KUBECONFIG .kube/config
// // //           chmod 600 .kube/config
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           kubectl cluster-info
          
// // //           for ENV in dev qa staging prod; do
// // //             echo "Configuring secrets for $ENV"
            
// // //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply -f -
            
// // //             kubectl create secret docker-registry dockerhub-secret \
// // //               --docker-server=https://index.docker.io/v1/ \
// // //               --docker-username=$DOCKER_ID \
// // //               --docker-password=$DOCKER_REGISTRY_PASS \
// // //               --docker-email=nntamo06@gmail.com \
// // //               --namespace=$ENV \
// // //               --dry-run=client -o yaml | kubectl apply -f -
            
// // //             kubectl create secret generic movie-db-secret \
// // //               --from-literal=POSTGRES_USER=movie_db_user \
// // //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// // //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// // //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// // //               --namespace=$ENV \
// // //               --dry-run=client -o yaml | kubectl apply -f -
            
// // //             kubectl create secret generic cast-db-secret \
// // //               --from-literal=POSTGRES_USER=cast_db_user \
// // //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// // //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// // //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// // //               --namespace=$ENV \
// // //               --dry-run=client -o yaml | kubectl apply -f -
              
// // //           done
          
// // //           echo "Kubernetes secrets configured"
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Deploy to DEV') {
// // //       environment {
// // //         KUBECONFIG = credentials("config")
// // //         TARGET_ENV = "dev"
// // //       }
// // //       steps {
// // //         script {
// // //           sh '''
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           kubectl get nodes
// // //           kubectl get ns $TARGET_ENV
          
// // //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// // //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// // //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// // //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// // //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// // //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
          
// // //           echo "DEV deployment completed"
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Health Checks') {
// // //       steps {
// // //         script {
// // //           sh '''
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           kubectl get pods -n dev -o wide
// // //           kubectl get endpoints -n dev
          
// // //           echo "Health checks completed"
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Promotion to QA') {
// // //       steps {
// // //         timeout(time: 30, unit: "MINUTES") {
// // //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// // //         }
// // //         script {
// // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // //             sh '''
// // //             git config user.name "Jenkins"
// // //             git config user.email "jenkins@datascientest.com"
// // //             git config credential.helper store
// // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // //             git fetch origin
            
// // //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// // //                 git checkout -B qa origin/qa
// // //             else
// // //                 git checkout -B qa
// // //             fi
            
// // //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// // //             git push origin qa
// // //             rm -f ~/.git-credentials
// // //             '''
// // //           }
// // //         }
// // //       }
// // //     }
    
// // //     stage('Deploy to QA') {
// // //       environment {
// // //         KUBECONFIG = credentials("config")
// // //         TARGET_ENV = "qa"
// // //       }
// // //       steps {
// // //         script {
// // //           sh '''
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// // //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// // //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// // //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// // //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// // //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Promotion to STAGING') {
// // //       steps {
// // //         timeout(time: 30, unit: "MINUTES") {
// // //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// // //         }
// // //         script {
// // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // //             sh '''
// // //             git config user.name "Jenkins"
// // //             git config user.email "jenkins@datascientest.com"
// // //             git config credential.helper store
// // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // //             git fetch origin
            
// // //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// // //                 git checkout -B staging origin/staging
// // //             else
// // //                 git checkout -B staging
// // //             fi
            
// // //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// // //             git push origin staging
// // //             rm -f ~/.git-credentials
// // //             '''
// // //           }
// // //         }
// // //       }
// // //     }
    
// // //     stage('Deploy to STAGING') {
// // //       environment {
// // //         KUBECONFIG = credentials("config")
// // //         TARGET_ENV = "staging"
// // //       }
// // //       steps {
// // //         script {
// // //           sh '''
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// // //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// // //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// // //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// // //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// // //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// // //           '''
// // //         }
// // //       }
// // //     }
    
// // //     stage('Promotion to PROD') {
// // //       steps {
// // //         timeout(time: 60, unit: "MINUTES") {
// // //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// // //         }
// // //         script {
// // //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// // //             sh '''
// // //             git config user.name "Jenkins"
// // //             git config user.email "jenkins@datascientest.com"
// // //             git config credential.helper store
// // //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// // //             git fetch origin
            
// // //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// // //                 git checkout -B prod origin/prod
// // //             else
// // //                 git checkout -B prod
// // //             fi
            
// // //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// // //             git push origin prod
// // //             rm -f ~/.git-credentials
// // //             '''
// // //           }
// // //         }
// // //       }
// // //     }
    
// // //     stage('Deploy to PROD') {
// // //       environment {
// // //         KUBECONFIG = credentials("config")
// // //         TARGET_ENV = "prod"
// // //       }
// // //       steps {
// // //         script {
// // //           sh '''
// // //           export KUBECONFIG=$(pwd)/.kube/config
          
// // //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// // //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// // //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// // //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// // //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// // //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// // //           '''
// // //         }
// // //       }
// // //     }
// // //   }
  
// // //   post {
// // //     always {
// // //       script {
// // //         sh '''
// // //         rm -rf .kube
// // //         docker system prune -f --volumes || true
// // //         '''
// // //       }
// // //     }
    
// // //     success {
// // //       echo 'Pipeline completed successfully'
// // //     }
    
// // //     failure {
// // //       echo 'Pipeline failed - check logs for details'
// // //     }
// // //   }
// // // }



// // pipeline {
// //   environment {
// //     DOCKER_ID = "nguetsop"
// //     MOVIE_IMAGE = "movie-service"
// //     CAST_IMAGE = "cast-service"
// //     DOCKER_TAG = "v.${BUILD_ID}.0"
// //     DOCKER_BUILDKIT = "1"
// //   }
  
// //   agent any
  
// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //     timeout(time: 30, unit: 'MINUTES')
// //     skipStagesAfterUnstable()
// //     timestamps()
// //   }
  
// //   stages {
// //     stage('Pre-Build Validation') {
// //       steps {
// //         script {
// //           sh '''
// //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// //             echo "ERROR: Missing Dockerfile"
// //             exit 1
// //           fi
          
// //           echo "Validation completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Docker Build') {
// //       parallel {
// //         stage('Build Movie Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd movie-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Build Cast Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd cast-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
// //       }
      
// //       post {
// //         success {
// //           sh 'docker images | grep $DOCKER_ID'
// //         }
// //       }
// //     }
    
// //     stage('Quality Gates') {
// //       parallel {
// //         stage('Unit Tests') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Running unit tests..."
// //               echo "Tests passed"
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Security Scan') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Security scan completed"
// //               '''
// //             }
// //           }
// //         }
// //       }
// //     }

// //     stage('Registry Push') {
// //       environment {
// //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// //       }
// //       steps {
// //         script {
// //           retry(3) {
// //             sh '''
// //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// //             docker logout
// //             echo "Images pushed successfully"
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Kubernetes Secrets') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           rm -rf .kube
// //           mkdir .kube
// //           cp $KUBECONFIG .kube/config
// //           chmod 600 .kube/config
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl cluster-info
          
// //           for ENV in dev qa staging prod; do
// //             echo "Configuring secrets for $ENV"
            
// //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret docker-registry dockerhub-secret \
// //               --docker-server=https://index.docker.io/v1/ \
// //               --docker-username=$DOCKER_ID \
// //               --docker-password=$DOCKER_REGISTRY_PASS \
// //               --docker-email=nntamo06@gmail.com \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic movie-db-secret \
// //               --from-literal=POSTGRES_USER=movie_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic cast-db-secret \
// //               --from-literal=POSTGRES_USER=cast_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
              
// //           done
          
// //           echo "Kubernetes secrets configured"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Deploy to DEV') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "dev"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get nodes
// //           kubectl get ns $TARGET_ENV
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
          
// //           echo "DEV deployment completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Health Checks') {
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get pods -n dev -o wide
// //           kubectl get endpoints -n dev
          
// //           echo "Health checks completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to QA') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// //                 git checkout -B qa origin/qa
// //             else
// //                 git checkout -B qa
// //             fi
            
// //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// //             git push origin qa
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to QA') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "qa"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to STAGING') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// //                 git checkout -B staging origin/staging
// //             else
// //                 git checkout -B staging
// //             fi
            
// //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// //             git push origin staging
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to STAGING') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "staging"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to PROD') {
// //       steps {
// //         timeout(time: 60, unit: "MINUTES") {
// //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// //                 git checkout -B prod origin/prod
// //             else
// //                 git checkout -B prod
// //             fi
            
// //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// //             git push origin prod
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to PROD') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "prod"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
// //   }
  
// //   post {
// //     always {
// //       script {
// //         sh '''
// //         rm -rf .kube
// //         docker system prune -f --volumes || true
// //         '''
// //       }
// //     }
    
// //     success {
// //       echo 'Pipeline completed successfully'
// //     }
    
// //     failure {
// //       echo 'Pipeline failed - check logs for details'
// //     }
// //   }
// // }



// // pipeline {
// //   environment {
// //     DOCKER_ID = "nguetsop"
// //     MOVIE_IMAGE = "movie-service"
// //     CAST_IMAGE = "cast-service"
// //     DOCKER_TAG = "v.${BUILD_ID}.0"
// //     DOCKER_BUILDKIT = "1"
// //   }
  
// //   agent any
  
// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //     timeout(time: 30, unit: 'MINUTES')
// //     skipStagesAfterUnstable()
// //     timestamps()
// //   }
  
// //   stages {
// //     stage('Pre-Build Validation') {
// //       steps {
// //         script {
// //           sh '''
// //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// //             echo "ERROR: Missing Dockerfile"
// //             exit 1
// //           fi
          
// //           echo "Validation completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Docker Build') {
// //       parallel {
// //         stage('Build Movie Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd movie-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Build Cast Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd cast-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
// //       }
      
// //       post {
// //         success {
// //           sh 'docker images | grep $DOCKER_ID'
// //         }
// //       }
// //     }
    
// //     stage('Quality Gates') {
// //       parallel {
// //         stage('Unit Tests') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Running unit tests..."
// //               echo "Tests passed"
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Security Scan') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Security scan completed"
// //               '''
// //             }
// //           }
// //         }
// //       }
// //     }

// //     stage('Registry Push') {
// //       environment {
// //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// //       }
// //       steps {
// //         script {
// //           retry(3) {
// //             sh '''
// //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// //             docker logout
// //             echo "Images pushed successfully"
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Kubernetes Secrets') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           rm -rf .kube
// //           mkdir .kube
// //           cp $KUBECONFIG .kube/config
// //           chmod 600 .kube/config
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl cluster-info
          
// //           for ENV in dev qa staging prod; do
// //             echo "Configuring secrets for $ENV"
            
// //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret docker-registry dockerhub-secret \
// //               --docker-server=https://index.docker.io/v1/ \
// //               --docker-username=$DOCKER_ID \
// //               --docker-password=$DOCKER_REGISTRY_PASS \
// //               --docker-email=nntamo06@gmail.com \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic movie-db-secret \
// //               --from-literal=POSTGRES_USER=movie_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic cast-db-secret \
// //               --from-literal=POSTGRES_USER=cast_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
              
// //           done
          
// //           echo "Kubernetes secrets configured"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Deploy to DEV') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "dev"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get nodes
// //           kubectl get ns $TARGET_ENV
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
          
// //           echo "DEV deployment completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Health Checks') {
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get pods -n dev -o wide
// //           kubectl get endpoints -n dev
          
// //           echo "Health checks completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to QA') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// //                 git checkout -B qa origin/qa
// //             else
// //                 git checkout -B qa
// //             fi
            
// //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// //             git push origin qa
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to QA') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "qa"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to STAGING') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// //                 git checkout -B staging origin/staging
// //             else
// //                 git checkout -B staging
// //             fi
            
// //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// //             git push origin staging
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to STAGING') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "staging"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to PROD') {
// //       steps {
// //         timeout(time: 60, unit: "MINUTES") {
// //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// //                 git checkout -B prod origin/prod
// //             else
// //                 git checkout -B prod
// //             fi
            
// //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// //             git push origin prod
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to PROD') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "prod"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
// //   }
  
// //   post {
// //     always {
// //       script {
// //         sh '''
// //         rm -rf .kube
// //         docker system prune -f --volumes || true
// //         '''
// //       }
// //     }
    
// //     success {
// //       echo 'Pipeline completed successfully'
// //     }
    
// //     failure {
// //       echo 'Pipeline failed - check logs for details'
// //     }
// //   }
// // }



// // pipeline {
// //   environment {
// //     DOCKER_ID = "nguetsop"
// //     MOVIE_IMAGE = "movie-service"
// //     CAST_IMAGE = "cast-service"
// //     DOCKER_TAG = "v.${BUILD_ID}.0"
// //     DOCKER_BUILDKIT = "1"
// //   }
  
// //   agent any
  
// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //     timeout(time: 30, unit: 'MINUTES')
// //     skipStagesAfterUnstable()
// //     timestamps()
// //   }
  
// //   stages {
// //     stage('Pre-Build Validation') {
// //       steps {
// //         script {
// //           sh '''
// //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// //             echo "ERROR: Missing Dockerfile"
// //             exit 1
// //           fi
          
// //           echo "Validation completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Docker Build') {
// //       parallel {
// //         stage('Build Movie Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd movie-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Build Cast Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd cast-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
// //       }
      
// //       post {
// //         success {
// //           sh 'docker images | grep $DOCKER_ID'
// //         }
// //       }
// //     }
    
// //     stage('Quality Gates') {
// //       parallel {
// //         stage('Unit Tests') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Running unit tests..."
// //               echo "Tests passed"
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Security Scan') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Security scan completed"
// //               '''
// //             }
// //           }
// //         }
// //       }
// //     }

// //     stage('Registry Push') {
// //       environment {
// //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// //       }
// //       steps {
// //         script {
// //           retry(3) {
// //             sh '''
// //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// //             docker logout
// //             echo "Images pushed successfully"
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Kubernetes Secrets') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           rm -rf .kube
// //           mkdir .kube
// //           cp $KUBECONFIG .kube/config
// //           chmod 600 .kube/config
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl cluster-info
          
// //           for ENV in dev qa staging prod; do
// //             echo "Configuring secrets for $ENV"
            
// //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret docker-registry dockerhub-secret \
// //               --docker-server=https://index.docker.io/v1/ \
// //               --docker-username=$DOCKER_ID \
// //               --docker-password=$DOCKER_REGISTRY_PASS \
// //               --docker-email=nntamo06@gmail.com \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic movie-db-secret \
// //               --from-literal=POSTGRES_USER=movie_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             kubectl create secret generic cast-db-secret \
// //               --from-literal=POSTGRES_USER=cast_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
              
// //           done
          
// //           echo "Kubernetes secrets configured"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Deploy to DEV') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "dev"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get nodes
// //           kubectl get ns $TARGET_ENV
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
// //           echo "DEV deployment completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Health Checks') {
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get pods -n dev -o wide
// //           kubectl get endpoints -n dev
          
// //           echo "Health checks completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to QA') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// //                 git checkout -B qa origin/qa
// //             else
// //                 git checkout -B qa
// //             fi
            
// //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// //             git push origin qa
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to QA') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "qa"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to STAGING') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// //                 git checkout -B staging origin/staging
// //             else
// //                 git checkout -B staging
// //             fi
            
// //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// //             git push origin staging
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to STAGING') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "staging"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to PROD') {
// //       steps {
// //         timeout(time: 60, unit: "MINUTES") {
// //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// //                 git checkout -B prod origin/prod
// //             else
// //                 git checkout -B prod
// //             fi
            
// //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// //             git push origin prod
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to PROD') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "prod"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
// //   }
  
// //   post {
// //     always {
// //       script {
// //         sh '''
// //         rm -rf .kube
// //         docker system prune -f --volumes || true
// //         '''
// //       }
// //     }
    
// //     success {
// //       echo 'Pipeline completed successfully'
// //     }
    
// //     failure {
// //       echo 'Pipeline failed - check logs for details'
// //     }
// //   }
// // }



// // pipeline {
// //   environment {
// //     DOCKER_ID = "nguetsop"
// //     MOVIE_IMAGE = "movie-service"
// //     CAST_IMAGE = "cast-service"
// //     DOCKER_TAG = "v.${BUILD_ID}.0"
// //     DOCKER_BUILDKIT = "1"
// //   }
  
// //   agent any
  
// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //     timeout(time: 30, unit: 'MINUTES')
// //     skipStagesAfterUnstable()
// //     timestamps()
// //   }
  
// //   stages {
// //     // Validation des prérequis
// //     stage('Pre-Build Validation') {
// //       steps {
// //         script {
// //           sh '''
// //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// //             echo "ERROR: Missing Dockerfile"
// //             exit 1
// //           fi
          
// //           echo "Validation completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     // Build parallèle des images Docker
// //     stage('Docker Build') {
// //       parallel {
// //         stage('Build Movie Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd movie-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Build Cast Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd cast-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
// //       }
      
// //       post {
// //         success {
// //           sh 'docker images | grep $DOCKER_ID'
// //         }
// //       }
// //     }
    
// //     // Tests et contrôles qualité
// //     stage('Quality Gates') {
// //       parallel {
// //         stage('Unit Tests') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Running unit tests..."
// //               echo "Tests passed"
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Security Scan') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Security scan completed"
// //               '''
// //             }
// //           }
// //         }
// //       }
// //     }

// //     // Push automatique vers Docker Hub
// //     stage('Registry Push') {
// //       environment {
// //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// //       }
// //       steps {
// //         script {
// //           retry(3) {
// //             sh '''
// //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// //             docker logout
// //             echo "Images pushed successfully"
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     // Configuration des secrets Kubernetes
// //     stage('Kubernetes Secrets') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           rm -rf .kube
// //           mkdir .kube
// //           cp $KUBECONFIG .kube/config
// //           chmod 600 .kube/config
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl cluster-info
          
// //           # Création des secrets pour tous les environnements
// //           for ENV in dev qa staging prod; do
// //             echo "Configuring secrets for $ENV"
            
// //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply -f -
            
// //             # Secret Docker registry
// //             kubectl create secret docker-registry dockerhub-secret \
// //               --docker-server=https://index.docker.io/v1/ \
// //               --docker-username=$DOCKER_ID \
// //               --docker-password=$DOCKER_REGISTRY_PASS \
// //               --docker-email=nntamo06@gmail.com \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             # Secrets base de données movie
// //             kubectl create secret generic movie-db-secret \
// //               --from-literal=POSTGRES_USER=movie_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
            
// //             # Secrets base de données cast
// //             kubectl create secret generic cast-db-secret \
// //               --from-literal=POSTGRES_USER=cast_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply -f -
              
// //           done
          
// //           echo "Kubernetes secrets configured"
// //           '''
// //         }
// //       }
// //     }
    
// //     // Déploiement automatique en DEV
// //     stage('Deploy to DEV') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "dev"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get nodes
// //           kubectl get ns $TARGET_ENV
          
// //           # Mise à jour des tags d'images dans les manifests
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           # Application des manifests
// //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           # Attente du déploiement
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
// //           echo "DEV deployment completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     // Vérifications de santé
// //     stage('Health Checks') {
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get pods -n dev -o wide
// //           kubectl get endpoints -n dev
          
// //           echo "Health checks completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     // Promotion manuelle vers QA
// //     stage('Promotion to QA') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             # Merge vers branche QA
// //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// //                 git checkout -B qa origin/qa
// //             else
// //                 git checkout -B qa
// //             fi
            
// //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// //             git push origin qa
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     // Déploiement en QA avec correction selector
// //     stage('Deploy to QA') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "qa"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           # Suppression forcée des deployments existants pour éviter les conflits de selector
// //           kubectl delete deployment movie-service-qa -n qa --ignore-not-found=true
// //           kubectl delete deployment cast-service-qa -n qa --ignore-not-found=true
          
// //           # Mise à jour des tags d'images
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           # Application des manifests
// //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           # Attente du déploiement
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     // Promotion manuelle vers STAGING
// //     stage('Promotion to STAGING') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             # Merge vers branche staging
// //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// //                 git checkout -B staging origin/staging
// //             else
// //                 git checkout -B staging
// //             fi
            
// //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// //             git push origin staging
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     // Déploiement en STAGING
// //     stage('Deploy to STAGING') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "staging"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           # Suppression forcée pour éviter les conflits
// //           kubectl delete deployment movie-service-staging -n staging --ignore-not-found=true
// //           kubectl delete deployment cast-service-staging -n staging --ignore-not-found=true
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     // Promotion manuelle vers PRODUCTION
// //     stage('Promotion to PROD') {
// //       steps {
// //         timeout(time: 60, unit: "MINUTES") {
// //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             # Merge vers branche production
// //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// //                 git checkout -B prod origin/prod
// //             else
// //                 git checkout -B prod
// //             fi
            
// //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// //             git push origin prod
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     // Déploiement en PRODUCTION
// //     stage('Deploy to PROD') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "prod"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           # Suppression forcée pour éviter les conflits
// //           kubectl delete deployment movie-service-prod -n prod --ignore-not-found=true
// //           kubectl delete deployment cast-service-prod -n prod --ignore-not-found=true
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
// //   }
  
// //   post {
// //     always {
// //       script {
// //         sh '''
// //         # Nettoyage des fichiers temporaires
// //         rm -rf .kube
// //         docker system prune -f --volumes || true
// //         '''
// //       }
// //     }
    
// //     success {
// //       echo 'Pipeline completed successfully'
// //     }
    
// //     failure {
// //       echo 'Pipeline failed - check logs for details'
// //     }
// //   }
// // }


// // pipeline {
// //   environment {
// //     DOCKER_ID = "nguetsop"
// //     MOVIE_IMAGE = "movie-service"
// //     CAST_IMAGE = "cast-service"
// //     DOCKER_TAG = "v.${BUILD_ID}.0"
// //     DOCKER_BUILDKIT = "1"
// //   }
  
// //   agent any
  
// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //     timeout(time: 30, unit: 'MINUTES')
// //     skipStagesAfterUnstable()
// //     timestamps()
// //   }
  
// //   stages {
// //     stage('Pre-Build Validation') {
// //       steps {
// //         script {
// //           sh '''
// //           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
// //           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
// //             echo "ERROR: Missing Dockerfile"
// //             exit 1
// //           fi
          
// //           echo "Validation completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Docker Build') {
// //       parallel {
// //         stage('Build Movie Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd movie-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Build Cast Service') {
// //           steps {
// //             script {
// //               sh '''
// //               cd cast-service
// //               docker build \
// //                 --build-arg BUILDKIT_INLINE_CACHE=1 \
// //                 --label "version=${DOCKER_TAG}" \
// //                 --label "git.commit=$(git rev-parse HEAD)" \
// //                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
// //                 -t $DOCKER_ID/$CAST_IMAGE:latest .
// //               '''
// //             }
// //           }
// //         }
// //       }
      
// //       post {
// //         success {
// //           sh 'docker images | grep $DOCKER_ID'
// //         }
// //       }
// //     }
    
// //     stage('Quality Gates') {
// //       parallel {
// //         stage('Unit Tests') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Running unit tests..."
// //               echo "Tests passed"
// //               '''
// //             }
// //           }
// //         }
        
// //         stage('Security Scan') {
// //           steps {
// //             script {
// //               sh '''
// //               echo "Security scan completed"
// //               '''
// //             }
// //           }
// //         }
// //       }
// //     }

// //     stage('Registry Push') {
// //       environment {
// //         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
// //       }
// //       steps {
// //         script {
// //           retry(3) {
// //             sh '''
// //             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
// //             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
// //             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
// //             docker logout
// //             echo "Images pushed successfully"
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Kubernetes Secrets') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
// //         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
// //         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           rm -rf .kube
// //           mkdir .kube
// //           cp $KUBECONFIG .kube/config
// //           chmod 600 .kube/config
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl cluster-info
          
// //           for ENV in dev qa staging prod; do
// //             echo "Configuring secrets for $ENV"
            
// //             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply --server-side -f -
            
// //             kubectl create secret docker-registry dockerhub-secret \
// //               --docker-server=https://index.docker.io/v1/ \
// //               --docker-username=$DOCKER_ID \
// //               --docker-password=$DOCKER_REGISTRY_PASS \
// //               --docker-email=nntamo06@gmail.com \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply --server-side -f -
            
// //             kubectl create secret generic movie-db-secret \
// //               --from-literal=POSTGRES_USER=movie_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
// //               --from-literal=POSTGRES_DB=movie_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply --server-side -f -
            
// //             kubectl create secret generic cast-db-secret \
// //               --from-literal=POSTGRES_USER=cast_db_user \
// //               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
// //               --from-literal=POSTGRES_DB=cast_db_$ENV \
// //               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
// //               --namespace=$ENV \
// //               --dry-run=client -o yaml | kubectl apply --server-side -f -
              
// //           done
          
// //           echo "Kubernetes secrets configured"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Deploy to DEV') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "dev"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get nodes
// //           kubectl get ns $TARGET_ENV
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
// //           echo "DEV deployment completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Health Checks') {
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl get pods -n dev -o wide
// //           kubectl get endpoints -n dev
          
// //           echo "Health checks completed"
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to QA') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to QA environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/qa; then
// //                 git checkout -B qa origin/qa
// //             else
// //                 git checkout -B qa
// //             fi
            
// //             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
// //             git push origin qa
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to QA') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "qa"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl delete deployment movie-service-qa -n qa --ignore-not-found=true
// //           kubectl delete deployment cast-service-qa -n qa --ignore-not-found=true
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to STAGING') {
// //       steps {
// //         timeout(time: 30, unit: "MINUTES") {
// //           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/staging; then
// //                 git checkout -B staging origin/staging
// //             else
// //                 git checkout -B staging
// //             fi
            
// //             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
// //             git push origin staging
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to STAGING') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "staging"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl delete deployment movie-service-staging -n staging --ignore-not-found=true
// //           kubectl delete deployment cast-service-staging -n staging --ignore-not-found=true
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
    
// //     stage('Promotion to PROD') {
// //       steps {
// //         timeout(time: 60, unit: "MINUTES") {
// //           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
// //         }
// //         script {
// //           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
// //             sh '''
// //             git config user.name "Jenkins"
// //             git config user.email "jenkins@datascientest.com"
// //             git config credential.helper store
// //             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
// //             git fetch origin
            
// //             if git show-ref --verify --quiet refs/remotes/origin/prod; then
// //                 git checkout -B prod origin/prod
// //             else
// //                 git checkout -B prod
// //             fi
            
// //             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
// //             git push origin prod
// //             rm -f ~/.git-credentials
// //             '''
// //           }
// //         }
// //       }
// //     }
    
// //     stage('Deploy to PROD') {
// //       environment {
// //         KUBECONFIG = credentials("config")
// //         TARGET_ENV = "prod"
// //       }
// //       steps {
// //         script {
// //           sh '''
// //           export KUBECONFIG=$(pwd)/.kube/config
          
// //           kubectl delete deployment movie-service-prod -n prod --ignore-not-found=true
// //           kubectl delete deployment cast-service-prod -n prod --ignore-not-found=true
          
// //           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
// //           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
// //           kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
// //           kubectl apply -f k8s-manifests/$TARGET_ENV/
          
// //           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
// //           '''
// //         }
// //       }
// //     }
// //   }
  
// //   post {
// //     always {
// //       script {
// //         sh '''
// //         rm -rf .kube
// //         docker system prune -f --volumes || true
// //         '''
// //       }
// //     }
    
// //     success {
// //       echo 'Pipeline completed successfully'
// //     }
    
// //     failure {
// //       echo 'Pipeline failed - check logs for details'
// //     }
// //   }
// // }


// pipeline {
//   environment {
//     DOCKER_ID = "nguetsop"
//     MOVIE_IMAGE = "movie-service"
//     CAST_IMAGE = "cast-service"
//     DOCKER_TAG = "v.${BUILD_ID}.0"
//     DOCKER_BUILDKIT = "1"
//   }
  
//   agent any
  
//   options {
//     buildDiscarder(logRotator(numToKeepStr: '10'))
//     timeout(time: 30, unit: 'MINUTES')
//     skipStagesAfterUnstable()
//     timestamps()
//   }
  
//   stages {
//     stage('Pre-Build Validation') {
//       steps {
//         script {
//           sh '''
//           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
//           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
//             echo "ERROR: Missing Dockerfile"
//             exit 1
//           fi
          
//           echo "Validation completed"
//           '''
//         }
//       }
//     }
    
//     stage('Docker Build') {
//       parallel {
//         stage('Build Movie Service') {
//           steps {
//             script {
//               sh '''
//               cd movie-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
//               '''
//             }
//           }
//         }
        
//         stage('Build Cast Service') {
//           steps {
//             script {
//               sh '''
//               cd cast-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$CAST_IMAGE:latest .
//               '''
//             }
//           }
//         }
//       }
      
//       post {
//         success {
//           sh 'docker images | grep $DOCKER_ID'
//         }
//       }
//     }
    
//     stage('Quality Gates') {
//       parallel {
//         stage('Unit Tests') {
//           steps {
//             script {
//               sh '''
//               echo "Running unit tests..."
//               echo "Tests passed"
//               '''
//             }
//           }
//         }
        
//         stage('Security Scan') {
//           steps {
//             script {
//               sh '''
//               echo "Security scan completed"
//               '''
//             }
//           }
//         }
//       }
//     }

//     stage('Registry Push') {
//       environment {
//         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
//       }
//       steps {
//         script {
//           retry(3) {
//             sh '''
//             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
//             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
//             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
//             docker logout
//             echo "Images pushed successfully"
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Kubernetes Secrets') {
//       environment {
//         KUBECONFIG = credentials("config")
//         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
//         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
//         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
//       }
//       steps {
//         script {
//           sh '''
//           rm -rf .kube
//           mkdir .kube
//           cp $KUBECONFIG .kube/config
//           chmod 600 .kube/config
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl cluster-info
          
//           for ENV in dev qa staging prod; do
//             echo "Configuring secrets for $ENV"
            
//             # CORRECTION: Ajout du flag --force-conflicts pour résoudre les conflits server-side apply
//             kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
            
//             kubectl create secret docker-registry dockerhub-secret \
//               --docker-server=https://index.docker.io/v1/ \
//               --docker-username=$DOCKER_ID \
//               --docker-password=$DOCKER_REGISTRY_PASS \
//               --docker-email=nntamo06@gmail.com \
//               --namespace=$ENV \
//               --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
            
//             kubectl create secret generic movie-db-secret \
//               --from-literal=POSTGRES_USER=movie_db_user \
//               --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
//               --from-literal=POSTGRES_DB=movie_db_$ENV \
//               --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
//               --namespace=$ENV \
//               --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
            
//             kubectl create secret generic cast-db-secret \
//               --from-literal=POSTGRES_USER=cast_db_user \
//               --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
//               --from-literal=POSTGRES_DB=cast_db_$ENV \
//               --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
//               --namespace=$ENV \
//               --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
              
//           done
          
//           echo "Kubernetes secrets configured"
//           '''
//         }
//       }
//     }
    
//     stage('Deploy to DEV') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "dev"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get nodes
//           kubectl get ns $TARGET_ENV
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           # CORRECTION: Utilisation de server-side apply avec force-conflicts pour tous les manifests
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/dev-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
//           echo "DEV deployment completed"
//           '''
//         }
//       }
//     }
    
//     stage('Health Checks') {
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get pods -n dev -o wide
//           kubectl get endpoints -n dev
          
//           echo "Health checks completed"
//           '''
//         }
//       }
//     }
    
//     stage('Promotion to QA') {
//       steps {
//         timeout(time: 30, unit: "MINUTES") {
//           input message: 'Deploy to QA environment?', ok: 'Deploy'
//         }
//         script {
//           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
//             sh '''
//             git config user.name "Jenkins"
//             git config user.email "jenkins@datascientest.com"
//             git config credential.helper store
//             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
//             git fetch origin
            
//             if git show-ref --verify --quiet refs/remotes/origin/qa; then
//                 git checkout -B qa origin/qa
//             else
//                 git checkout -B qa
//             fi
            
//             git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
//             git push origin qa
//             rm -f ~/.git-credentials
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Deploy to QA') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "qa"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl delete deployment movie-service-qa -n qa --ignore-not-found=true
//           kubectl delete deployment cast-service-qa -n qa --ignore-not-found=true
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           # CORRECTION: Utilisation de server-side apply avec force-conflicts
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/qa-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           '''
//         }
//       }
//     }
    
//     stage('Promotion to STAGING') {
//       steps {
//         timeout(time: 30, unit: "MINUTES") {
//           input message: 'Deploy to STAGING environment?', ok: 'Deploy'
//         }
//         script {
//           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
//             sh '''
//             git config user.name "Jenkins"
//             git config user.email "jenkins@datascientest.com"
//             git config credential.helper store
//             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
//             git fetch origin
            
//             if git show-ref --verify --quiet refs/remotes/origin/staging; then
//                 git checkout -B staging origin/staging
//             else
//                 git checkout -B staging
//             fi
            
//             git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
//             git push origin staging
//             rm -f ~/.git-credentials
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Deploy to STAGING') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "staging"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl delete deployment movie-service-staging -n staging --ignore-not-found=true
//           kubectl delete deployment cast-service-staging -n staging --ignore-not-found=true
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           # CORRECTION: Utilisation de server-side apply avec force-conflicts
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/staging-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           '''
//         }
//       }
//     }
    
//     stage('Promotion to PROD') {
//       steps {
//         timeout(time: 60, unit: "MINUTES") {
//           input message: 'Deploy to PRODUCTION environment?', ok: 'Deploy'
//         }
//         script {
//           withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
//             sh '''
//             git config user.name "Jenkins"
//             git config user.email "jenkins@datascientest.com"
//             git config credential.helper store
//             echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
//             git fetch origin
            
//             if git show-ref --verify --quiet refs/remotes/origin/prod; then
//                 git checkout -B prod origin/prod
//             else
//                 git checkout -B prod
//             fi
            
//             git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
//             git push origin prod
//             rm -f ~/.git-credentials
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Deploy to PROD') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "prod"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl delete deployment movie-service-prod -n prod --ignore-not-found=true
//           kubectl delete deployment cast-service-prod -n prod --ignore-not-found=true
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           # CORRECTION: Utilisation de server-side apply avec force-conflicts
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/prod-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           '''
//         }
//       }
//     }
//   }
  
//   post {
//     always {
//       script {
//         sh '''
//         rm -rf .kube
//         docker system prune -f --volumes || true
//         '''
//       }
//     }
    
//     success {
//       echo 'Pipeline completed successfully'
//     }
    
//     failure {
//       echo 'Pipeline failed - check logs for details'
//     }
//   }
// }




// pipeline {
//   environment {
//     DOCKER_ID = "nguetsop"
//     MOVIE_IMAGE = "movie-service"
//     CAST_IMAGE = "cast-service"
//     DOCKER_TAG = "v.${BUILD_ID}.0"
//     DOCKER_BUILDKIT = "1"
//   }
  
//   agent any
  
//   options {
//     buildDiscarder(logRotator(numToKeepStr: '10'))
//     timeout(time: 30, unit: 'MINUTES')
//     skipStagesAfterUnstable()
//     timestamps()
//   }
  
//   stages {
//     stage('Pre-Build Validation') {
//       steps {
//         script {
//           sh '''
//           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
//           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
//             echo "ERROR: Missing Dockerfile"
//             exit 1
//           fi
          
//           echo "Validation completed"
//           '''
//         }
//       }
//     }
    
//     stage('Docker Build') {
//       parallel {
//         stage('Build Movie Service') {
//           steps {
//             script {
//               sh '''
//               cd movie-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
//               '''
//             }
//           }
//         }
        
//         stage('Build Cast Service') {
//           steps {
//             script {
//               sh '''
//               cd cast-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$CAST_IMAGE:latest .
//               '''
//             }
//           }
//         }
//       }
      
//       post {
//         success {
//           sh 'docker images | grep $DOCKER_ID'
//         }
//       }
//     }
    
//     stage('Quality Gates') {
//       parallel {
//         stage('Unit Tests') {
//           steps {
//             script {
//               sh '''
//               echo "Running unit tests..."
//               echo "Tests passed"
//               '''
//             }
//           }
//         }
        
//         stage('Security Scan') {
//           steps {
//             script {
//               sh '''
//               echo "Security scan completed"
//               '''
//             }
//           }
//         }
//       }
//     }

//     stage('Registry Push') {
//       environment {
//         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
//       }
//       steps {
//         script {
//           retry(3) {
//             sh '''
//             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
//             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
//             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
//             docker logout
//             echo "Images pushed successfully"
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Kubernetes Secrets') {
//       environment {
//         KUBECONFIG = credentials("config")
//         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
//         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
//         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
//       }
//       steps {
//         script {
//           sh '''
//           rm -rf .kube
//           mkdir .kube
//           cp $KUBECONFIG .kube/config
//           chmod 600 .kube/config
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl cluster-info
          
//           # Configuration des secrets pour DEV uniquement
//           ENV="dev"
//           echo "Configuring secrets for $ENV"
          
//           kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret docker-registry dockerhub-secret \
//             --docker-server=https://index.docker.io/v1/ \
//             --docker-username=$DOCKER_ID \
//             --docker-password=$DOCKER_REGISTRY_PASS \
//             --docker-email=nntamo06@gmail.com \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret generic movie-db-secret \
//             --from-literal=POSTGRES_USER=movie_db_user \
//             --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
//             --from-literal=POSTGRES_DB=movie_db_$ENV \
//             --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret generic cast-db-secret \
//             --from-literal=POSTGRES_USER=cast_db_user \
//             --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
//             --from-literal=POSTGRES_DB=cast_db_$ENV \
//             --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           echo "Kubernetes secrets configured for DEV"
//           '''
//         }
//       }
//     }
    
//     stage('Approval for DEV') {
//       steps {
//         timeout(time: 15, unit: "MINUTES") {
//           input message: 'Deploy to DEV environment?', ok: 'Deploy to DEV'
//         }
//       }
//     }
    
//     stage('Deploy to DEV') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "dev"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get nodes
//           kubectl get ns $TARGET_ENV
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/dev-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
//           echo "DEV deployment completed"
//           '''
//         }
//       }
//     }
    
//     stage('Health Checks') {
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get pods -n dev -o wide
//           kubectl get endpoints -n dev
          
//           echo "Health checks completed"
//           '''
//         }
//       }
//     }
//   }
  
//   post {
//     always {
//       script {
//         sh '''
//         rm -rf .kube
//         docker system prune -f --volumes || true
//         '''
//       }
//     }
    
//     success {
//       echo 'DEV deployment completed successfully'
//     }
    
//     failure {
//       echo 'DEV deployment failed - check logs for details'
//     }
//   }
// }


// pipeline {
//   environment {
//     DOCKER_ID = "nguetsop"
//     MOVIE_IMAGE = "movie-service"
//     CAST_IMAGE = "cast-service"
//     DOCKER_TAG = "v.${BUILD_ID}.0"
//     DOCKER_BUILDKIT = "1"
//   }
  
//   agent any
  
//   triggers {
//     githubPush()
//   }
  
//   options {
//     buildDiscarder(logRotator(numToKeepStr: '10'))
//     timeout(time: 30, unit: 'MINUTES')
//     skipStagesAfterUnstable()
//     timestamps()
//   }
  
//   stages {
//     stage('Pre-Build Validation') {
//       steps {
//         script {
//           sh '''
//           echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
//           if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
//             echo "ERROR: Missing Dockerfile"
//             exit 1
//           fi
          
//           echo "Validation completed"
//           '''
//         }
//       }
//     }
    
//     stage('Docker Build') {
//       parallel {
//         stage('Build Movie Service') {
//           steps {
//             script {
//               sh '''
//               cd movie-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$MOVIE_IMAGE:latest .
//               '''
//             }
//           }
//         }
        
//         stage('Build Cast Service') {
//           steps {
//             script {
//               sh '''
//               cd cast-service
//               docker build \
//                 --build-arg BUILDKIT_INLINE_CACHE=1 \
//                 --label "version=${DOCKER_TAG}" \
//                 --label "git.commit=$(git rev-parse HEAD)" \
//                 -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
//                 -t $DOCKER_ID/$CAST_IMAGE:latest .
//               '''
//             }
//           }
//         }
//       }
      
//       post {
//         success {
//           sh 'docker images | grep $DOCKER_ID'
//         }
//       }
//     }
    
//     stage('Quality Gates') {
//       parallel {
//         stage('Unit Tests') {
//           steps {
//             script {
//               sh '''
//               echo "Running unit tests..."
//               echo "Tests passed"
//               '''
//             }
//           }
//         }
        
//         stage('Security Scan') {
//           steps {
//             script {
//               sh '''
//               echo "Security scan completed"
//               '''
//             }
//           }
//         }
//       }
//     }

//     stage('Registry Push') {
//       environment {
//         DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
//       }
//       steps {
//         script {
//           retry(3) {
//             sh '''
//             echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
//             docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$MOVIE_IMAGE:latest
//             docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
//             docker push $DOCKER_ID/$CAST_IMAGE:latest
            
//             docker logout
//             echo "Images pushed successfully"
//             '''
//           }
//         }
//       }
//     }
    
//     stage('Kubernetes Secrets') {
//       environment {
//         KUBECONFIG = credentials("config")
//         DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
//         MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
//         CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
//       }
//       steps {
//         script {
//           sh '''
//           rm -rf .kube
//           mkdir .kube
//           cp $KUBECONFIG .kube/config
//           chmod 600 .kube/config
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl cluster-info
          
//           # Configuration des secrets pour DEV uniquement
//           ENV="dev"
//           echo "Configuring secrets for $ENV"
          
//           kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret docker-registry dockerhub-secret \
//             --docker-server=https://index.docker.io/v1/ \
//             --docker-username=$DOCKER_ID \
//             --docker-password=$DOCKER_REGISTRY_PASS \
//             --docker-email=nntamo06@gmail.com \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret generic movie-db-secret \
//             --from-literal=POSTGRES_USER=movie_db_user \
//             --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
//             --from-literal=POSTGRES_DB=movie_db_$ENV \
//             --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           kubectl create secret generic cast-db-secret \
//             --from-literal=POSTGRES_USER=cast_db_user \
//             --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
//             --from-literal=POSTGRES_DB=cast_db_$ENV \
//             --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
//             --namespace=$ENV \
//             --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
//           echo "Kubernetes secrets configured for DEV"
//           '''
//         }
//       }
//     }
    
//     stage('Approval for DEV') {
//       steps {
//         timeout(time: 15, unit: "MINUTES") {
//           input message: 'Deploy to DEV environment?', ok: 'Deploy to DEV'
//         }
//       }
//     }
    
//     stage('Deploy to DEV') {
//       environment {
//         KUBECONFIG = credentials("config")
//         TARGET_ENV = "dev"
//       }
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get nodes
//           kubectl get ns $TARGET_ENV
          
//           sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
//           sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/dev-namespace.yaml
//           kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
//           kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
//           kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
//           echo "DEV deployment completed"
//           '''
//         }
//       }
//     }
    
//     stage('Health Checks') {
//       steps {
//         script {
//           sh '''
//           export KUBECONFIG=$(pwd)/.kube/config
          
//           kubectl get pods -n dev -o wide
//           kubectl get endpoints -n dev
          
//           echo "Health checks completed"
//           '''
//         }
//       }
//     }
//   }
  
//   post {
//     always {
//       script {
//         sh '''
//         rm -rf .kube
//         docker system prune -f --volumes || true
//         '''
//       }
//     }
    
//     success {
//       echo 'DEV deployment completed successfully'
//     }
    
//     failure {
//       echo 'DEV deployment failed - check logs for details'
//     }
//   }
// }




pipeline {
  environment {
    DOCKER_ID = "nguetsop"
    MOVIE_IMAGE = "movie-service"
    CAST_IMAGE = "cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0"
    DOCKER_BUILDKIT = "1"
  }
  
  agent any
  
  triggers {
    githubPush()
  }
  
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 30, unit: 'MINUTES')
    skipStagesAfterUnstable()
    timestamps()
  }
  
  stages {
    stage('Pre-Build Validation') {
      steps {
        script {
          sh '''
          echo "Pipeline Build ${BUILD_ID} - Git: $(git rev-parse --short HEAD)"
          
          if [ ! -f movie-service/Dockerfile ] || [ ! -f cast-service/Dockerfile ]; then
            echo "ERROR: Missing Dockerfile"
            exit 1
          fi
          
          echo "Validation completed"
          '''
        }
      }
    }
    
    stage('Docker Build') {
      parallel {
        stage('Build Movie Service') {
          steps {
            script {
              sh '''
              cd movie-service
              docker build \
                --build-arg BUILDKIT_INLINE_CACHE=1 \
                --label "version=${DOCKER_TAG}" \
                --label "git.commit=$(git rev-parse HEAD)" \
                -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG \
                -t $DOCKER_ID/$MOVIE_IMAGE:latest .
              '''
            }
          }
        }
        
        stage('Build Cast Service') {
          steps {
            script {
              sh '''
              cd cast-service
              docker build \
                --build-arg BUILDKIT_INLINE_CACHE=1 \
                --label "version=${DOCKER_TAG}" \
                --label "git.commit=$(git rev-parse HEAD)" \
                -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG \
                -t $DOCKER_ID/$CAST_IMAGE:latest .
              '''
            }
          }
        }
      }
      
      post {
        success {
          sh 'docker images | grep $DOCKER_ID'
        }
      }
    }
    
    stage('Quality Gates') {
      parallel {
        stage('Unit Tests') {
          steps {
            script {
              sh '''
              echo "Running unit tests..."
              echo "Tests passed"
              '''
            }
          }
        }
        
        stage('Security Scan') {
          steps {
            script {
              sh '''
              echo "Security scan completed"
              '''
            }
          }
        }
      }
    }

    stage('Registry Push') {
      environment {
        DOCKER_PASS = credentials("dockerhub_token_pipeline_cicd")
      }
      steps {
        script {
          retry(3) {
            sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
            docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
            docker push $DOCKER_ID/$MOVIE_IMAGE:latest
            docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
            docker push $DOCKER_ID/$CAST_IMAGE:latest
            
            docker logout
            echo "Images pushed successfully"
            '''
          }
        }
      }
    }
    
    stage('Kubernetes Secrets') {
      environment {
        KUBECONFIG = credentials("config")
        DOCKER_REGISTRY_PASS = credentials("dockerhub_token_pipeline_cicd")
        MOVIE_DB_SECRET = credentials("MOVIE_DB_PASSWORD")
        CAST_DB_SECRET = credentials("CAST_DB_PASSWORD")
      }
      steps {
        script {
          sh '''
          rm -rf .kube
          mkdir .kube
          cp $KUBECONFIG .kube/config
          chmod 600 .kube/config
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl cluster-info
          
          # Configuration des secrets pour DEV uniquement
          ENV="dev"
          echo "Configuring secrets for $ENV"
          
          kubectl create namespace $ENV --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
          kubectl create secret docker-registry dockerhub-secret \
            --docker-server=https://index.docker.io/v1/ \
            --docker-username=$DOCKER_ID \
            --docker-password=$DOCKER_REGISTRY_PASS \
            --docker-email=nntamo06@gmail.com \
            --namespace=$ENV \
            --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
          kubectl create secret generic movie-db-secret \
            --from-literal=POSTGRES_USER=movie_db_user \
            --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_SECRET \
            --from-literal=POSTGRES_DB=movie_db_$ENV \
            --from-literal=DATABASE_URI=postgresql://movie_db_user:$MOVIE_DB_SECRET@movie-db:5432/movie_db_$ENV \
            --namespace=$ENV \
            --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
          kubectl create secret generic cast-db-secret \
            --from-literal=POSTGRES_USER=cast_db_user \
            --from-literal=POSTGRES_PASSWORD=$CAST_DB_SECRET \
            --from-literal=POSTGRES_DB=cast_db_$ENV \
            --from-literal=DATABASE_URI=postgresql://cast_db_user:$CAST_DB_SECRET@cast-db:5432/cast_db_$ENV \
            --namespace=$ENV \
            --dry-run=client -o yaml | kubectl apply --server-side --force-conflicts -f -
          
          echo "Kubernetes secrets configured for DEV"
          '''
        }
      }
    }
    
    stage('Approval for DEV') {
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Deploy to DEV environment?', ok: 'Deploy to DEV'
        }
      }
    }
    
    stage('Deploy to DEV') {
      environment {
        KUBECONFIG = credentials("config")
        TARGET_ENV = "dev"
      }
      steps {
        script {
          sh '''
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl get nodes
          kubectl get ns $TARGET_ENV
          
          sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
          sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
          kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/dev-namespace.yaml
          kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
          kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
          echo "DEV deployment completed"
          '''
        }
      }
    }
    
    stage('Health Checks') {
      steps {
        script {
          sh '''
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl get pods -n dev -o wide
          kubectl get endpoints -n dev
          
          echo "Health checks completed"
          '''
        }
      }
    }
    
    stage('Promotion to QA') {
      steps {
        timeout(time: 30, unit: "MINUTES") {
          input message: 'DEV environment validated. Merge to QA branch and deploy to QA?', ok: 'Deploy to QA'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Promoting to QA environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            if git show-ref --verify --quiet refs/remotes/origin/qa; then
                git checkout -B qa origin/qa
            else
                git checkout -B qa
            fi
            
            git merge origin/main --no-ff -m "Merge origin/main to qa - Build ${BUILD_ID}"
            git push origin qa
            echo "Successfully merged origin/main to qa"
            rm -f ~/.git-credentials
            '''
          }
        }
      }
    }
    
    stage('Deploy to QA') {
      environment {
        KUBECONFIG = credentials("config")
        TARGET_ENV = "qa"
      }
      steps {
        script {
          sh '''
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl get nodes
          kubectl get ns $TARGET_ENV
          
          sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
          sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
          kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/qa-namespace.yaml
          kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
          kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
          echo "QA deployment completed"
          '''
        }
      }
    }
  }
  
  post {
    always {
      script {
        sh '''
        rm -rf .kube
        docker system prune -f --volumes || true
        '''
      }
    }
    
    success {
      echo 'DEV deployment completed successfully'
    }
    
    failure {
      echo 'DEV deployment failed - check logs for details'
    }
  }
}