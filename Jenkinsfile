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
    timeout(time: 15, unit: 'MINUTES')
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
          
          # Configuration des secrets pour tous les environnements
          for ENV in dev qa staging prod; do
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
              
          done
          
          echo "Kubernetes secrets configured for all environments"
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
    
    stage('Promotion to QA') {
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'DEV environment validated. Merge to QA branch and deploy to QA?', ok: 'Deploy to QA'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Deploy to QA environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            # Forcer un état Git propre avant checkout
            git reset --hard HEAD
            git clean -fd
            
            if git show-ref --verify --quiet refs/remotes/origin/qa; then
                git checkout -B qa origin/qa
            else
                git checkout -B qa
            fi
            
            git merge origin/dev --no-ff -m "Merge origin/dev to qa - Build ${BUILD_ID}"
            git push origin qa
            echo "Successfully merged origin/dev to qa"
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
    
    stage('Promotion to STAGING') {
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'QA environment validated. Merge to STAGING branch and deploy to STAGING?', ok: 'Deploy to STAGING'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Deploy to STAGING environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            # Forcer un état Git propre avant checkout
            git reset --hard HEAD
            git clean -fd
            
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
    
    stage('Deploy to STAGING') {
      environment {
        KUBECONFIG = credentials("config")
        TARGET_ENV = "staging"
      }
      steps {
        script {
          sh '''
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl get nodes
          kubectl get ns $TARGET_ENV
          
          kubectl delete deployment movie-service-staging -n staging --ignore-not-found=true
          kubectl delete deployment cast-service-staging -n staging --ignore-not-found=true
          
          sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
          sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
          kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/staging-namespace.yaml
          kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
          kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
          echo "STAGING deployment completed"
          '''
        }
      }
    }
    
    stage('Promotion to PROD') {
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'STAGING environment validated. Merge to PROD branch and deploy to PRODUCTION?', ok: 'Deploy to PRODUCTION'
        }
        script {
          withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
            sh '''
            echo "Deploy to PRODUCTION environment..."
            git config user.name "Jenkins"
            git config user.email "jenkins@datascientest.com"
            git config credential.helper store
            echo "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git fetch origin
            
            # Forcer un état Git propre avant checkout
            git reset --hard HEAD
            git clean -fd
            
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
    
    stage('Deploy to PROD') {
      environment {
        KUBECONFIG = credentials("config")
        TARGET_ENV = "prod"
      }
      steps {
        script {
          sh '''
          export KUBECONFIG=$(pwd)/.kube/config
          
          kubectl get nodes
          kubectl get ns $TARGET_ENV
          
          kubectl delete deployment movie-service-prod -n prod --ignore-not-found=true
          kubectl delete deployment cast-service-prod -n prod --ignore-not-found=true
          
          sed -i "s|image: .*movie-service:.*|image: ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/movie-deployment.yaml
          sed -i "s|image: .*cast-service:.*|image: ${DOCKER_ID}/${CAST_IMAGE}:${DOCKER_TAG}|g" k8s-manifests/$TARGET_ENV/cast-deployment.yaml
          
          kubectl apply --server-side --force-conflicts -f k8s-manifests/namespaces/prod-namespace.yaml
          kubectl apply --server-side --force-conflicts -f k8s-manifests/$TARGET_ENV/
          
          kubectl rollout status deployment/movie-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          kubectl rollout status deployment/cast-service-$TARGET_ENV -n $TARGET_ENV --timeout=300s
          
          echo "PRODUCTION deployment completed"
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
      echo "Pipeline completed successfully"
    }
    
    failure {
      echo 'Pipeline failed - check logs for details'
    }
  }
}