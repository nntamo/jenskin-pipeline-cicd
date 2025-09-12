pipeline {
  environment {
    DOCKER_ID = "nguetsop" // remplacez par votre docker-id
    MOVIE_IMAGE = "movie-service"
    CAST_IMAGE = "cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }
  agent any
  stages {
    stage('Docker Build') {
      steps {
        script {
          sh '''
          echo "Building Movie and Cast Services..."
          
          # Clean up existing containers
          docker rm -f movie-service || true
          docker rm -f cast-service || true
          
          # Build movie-service
          echo "Building Movie Service..."
          cd movie-service
          docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG .
          cd ..
          
          # Build cast-service
          echo "Building Cast Service..."
          cd cast-service
          docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG .
          cd ..
          
          echo "Both services built successfully"
          sleep 6
          '''
        }
      }
    }
    

    
    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials("dockerhub-token")
      }
      steps {
        script {
          retry(3) {
            sh '''
            echo "Pushing both images to Docker Hub..."
            echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            
            # Push Movie Service
            docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
            
            # Push Cast Service
            docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
            
            echo "Both images pushed successfully"
            '''
          }
        }
      }
    }
    
    stage('Create K8s Secrets') {
      environment {
        KUBECONFIG = credentials("config")
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        MOVIE_DB_PASS = credentials("MOVIE_DB_PASSWORD")
        CAST_DB_PASS = credentials("CAST_DB_PASSWORD")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          
          echo "Creating secrets for all environments..."
          
          for env in dev qa staging prod; do
            echo "Creating secrets for $env environment..."
            
            # Create Docker registry secret
            kubectl create secret docker-registry dockerhub-secret \
              --docker-server=docker.io \
              --docker-username=$DOCKER_ID \
              --docker-password=$DOCKER_PASS \
              --docker-email=nntamo06@gmail.com \
              -n $env --dry-run=client -o yaml | kubectl apply -f -
            
            # Create database secrets with environment-specific passwords
            kubectl create secret generic movie-db-secret \
              --from-literal=POSTGRES_USER=movie_db_username \
              --from-literal=POSTGRES_PASSWORD=${MOVIE_DB_PASS}_${env} \
              --from-literal=POSTGRES_DB=movie_db_${env} \
              --from-literal=DATABASE_URI=postgresql://movie_db_username:${MOVIE_DB_PASS}_${env}@movie-db:5432/movie_db_${env} \
              --namespace=$env --dry-run=client -o yaml | kubectl apply -f -
            
            kubectl create secret generic cast-db-secret \
              --from-literal=POSTGRES_USER=cast_db_username \
              --from-literal=POSTGRES_PASSWORD=${CAST_DB_PASS}_${env} \
              --from-literal=POSTGRES_DB=cast_db_${env} \
              --from-literal=DATABASE_URI=postgresql://cast_db_username:${CAST_DB_PASS}_${env}@cast-db:5432/cast_db_${env} \
              --namespace=$env --dry-run=client -o yaml | kubectl apply -f -
              
            echo "Secrets created for $env"
          done
          '''
        }
      }
    }
    
    stage('Deployment in dev') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          
          echo "Deploying to DEV environment..."
          
          # Option 1: Using Helm Charts (if you prefer)
          if [ -d "charts" ]; then
            echo "Using Helm deployment..."
            cp charts/values-dev.yaml values.yml
            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
            sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
            sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
            helm upgrade --install movie-cast-app charts --values=values.yml --namespace dev
          else
            # Option 2: Using K8s manifests (your current setup)
            echo "Using K8s manifests deployment..."
            
            # Update image tags in deployments
            sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/dev/movie-deployment.yaml
            sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/dev/cast-deployment.yaml
            
            # Apply all manifests
            kubectl apply -f k8s-manifests/namespaces/dev-namespace.yaml
            kubectl apply -f k8s-manifests/dev/
            
            # Wait for deployments
            kubectl rollout status deployment/movie-service -n dev --timeout=300s
            kubectl rollout status deployment/cast-service -n dev --timeout=300s
          fi
          
          echo "Deployed to DEV environment successfully"
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
    
    stage('Deployment in qa') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          
          echo "Deploying to QA environment..."
          
          if [ -d "charts" ]; then
            cp charts/values-qa.yaml values.yml
            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
            sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
            sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
            helm upgrade --install movie-cast-app charts --values=values.yml --namespace qa
          else
            sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/qa/movie-deployment.yaml
            sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/qa/cast-deployment.yaml
            
            kubectl apply -f k8s-manifests/namespaces/qa-namespace.yaml
            kubectl apply -f k8s-manifests/qa/
            
            kubectl rollout status deployment/movie-service -n qa --timeout=300s
            kubectl rollout status deployment/cast-service -n qa --timeout=300s
          fi
          
          echo "Deployed to QA environment successfully"
          '''
        }
      }
    }
    
    stage('Promotion to STAGING') {
      steps {
        timeout(time: 30, unit: "MINUTES") {
          input message: 'QA environment validated. Merge to STAGING branch and deploy to STAGING?', ok: 'Deploy to STAGING'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Promoting to STAGING environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            if git show-ref --verify --quiet refs/remotes/origin/staging; then
                git checkout -B staging origin/staging
            else
                git checkout -B staging
            fi
            
            git merge origin/qa --no-ff -m "Merge origin/qa to staging - Build ${BUILD_ID}"
            git push origin staging
            echo "Successfully merged origin/qa to staging"
            rm -f ~/.git-credentials
            '''
          }
        }
      }
    }
    
    stage('Deployment in staging') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          
          echo "Deploying to STAGING environment..."
          
          if [ -d "charts" ]; then
            cp charts/values-staging.yaml values.yml
            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
            sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
            sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
            helm upgrade --install movie-cast-app charts --values=values.yml --namespace staging
          else
            sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/staging/movie-deployment.yaml
            sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/staging/cast-deployment.yaml
            
            kubectl apply -f k8s-manifests/namespaces/staging-namespace.yaml
            kubectl apply -f k8s-manifests/staging/
            
            kubectl rollout status deployment/movie-service -n staging --timeout=300s
            kubectl rollout status deployment/cast-service -n staging --timeout=300s
          fi
          
          echo "Deployed to STAGING environment successfully"
          '''
        }
      }
    }
    
    stage('Promotion to PROD') {
      steps {
        timeout(time: 60, unit: "MINUTES") {
          input message: 'STAGING environment validated. Merge to PROD branch and deploy to PRODUCTION?', ok: 'Deploy to PRODUCTION'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Promoting to PRODUCTION environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            if git show-ref --verify --quiet refs/remotes/origin/prod; then
                git checkout -B prod origin/prod
            else
                git checkout -B prod
            fi
            
            git merge origin/staging --no-ff -m "Merge origin/staging to prod - Build ${BUILD_ID}"
            git push origin prod
            echo "Successfully merged origin/staging to prod"
            rm -f ~/.git-credentials
            '''
          }
        }
      }
    }
    
    stage('Deploiement en prod') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          
          echo "Deploying to PRODUCTION environment..."
          
          if [ -d "charts" ]; then
            cp charts/values-prod.yaml values.yml
            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
            sed -i "s+movieService:.*repository.*+movieService.repository: ${DOCKER_ID}/${MOVIE_IMAGE}+g" values.yml
            sed -i "s+castService:.*repository.*+castService.repository: ${DOCKER_ID}/${CAST_IMAGE}+g" values.yml
            helm upgrade --install movie-cast-app charts --values=values.yml --namespace prod
          else
            sed -i "s|image: movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/prod/movie-deployment.yaml
            sed -i "s|image: cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/prod/cast-deployment.yaml
            
            kubectl apply -f k8s-manifests/namespaces/prod-namespace.yaml
            kubectl apply -f k8s-manifests/prod/
            
            kubectl rollout status deployment/movie-service -n prod --timeout=300s
            kubectl rollout status deployment/cast-service -n prod --timeout=300s
          fi
          
          echo "Deployed to PRODUCTION environment successfully"
          '''
        }
      }
    }
  }
  
  post {
    always {
      sh '''
      # Cleanup
      docker rm -f movie-service || true
      docker rm -f cast-service || true
      docker system prune -f || true
      '''
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed!'
    }
  }
}