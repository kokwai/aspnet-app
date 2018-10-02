pipeline {
    agent {
        label "jenkins-jx-base"
    }
    environment {
      ORG               = 'kokwai'
      APP_NAME          = 'aspnet-app'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    DEPLOY_NAMESPACE = "jx-staging"
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('jx-base') {
            sh "cat /home/jenkins/.docker/config.json"
            sh "docker build -t $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION ."
            sh "docker push $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

          dir ('./charts/preview') {
           container('jx-base') {
             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."
           }
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('jx-base') {
            // ensure we're not on a detached head
            sh "git checkout master"
            // until we switch to the new kubernetes / jenkins credential implementation use git credentials store
            sh "git config --global credential.helper store"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
          }
          dir ('./charts/aspnet-app') {
            container('jx-base') {
              sh "jx step git credentials"
              sh "make tag"
            }
          }
          container('jx-base') {
            sh "cat /home/jenkins/.docker/config.json"
            sh "docker build -t $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION) ."
            sh "docker push $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/aspnet-app') {
            container('jx-base') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

                sh '''
                    curl -kLo helm-linux-amd64-v2.9.1.tar.gz https://9.30.118.226:8443/api/cli/helm-linux-amd64.tar.gz
                    tar -xvf helm-linux-amd64-v2.9.1.tar.gz
                    rm helm-linux-amd64-v2.9.1.tar.gz
                    cp -f linux-amd64/helm /usr/bin/helm
                    rm -rf linux-amd64
                    wget https://gist.githubusercontent.com/kokwai/fe352638f8f2495176e337a9c7a1efa3/raw/103e34e0a66fe3cc53b8738c67a4b9896ce9cae3/helm_wrapper.sh
                    chmod 755 helm_wrapper.sh
                    which helm
                    helm init --client-only --skip-refresh
                    curl -kLo cloudctl https://9.30.118.226:8443/api/cli/cloudctl-linux-amd64
                    chmod 755 cloudctl
                    ./cloudctl login --skip-ssl-validation -a https://9.30.118.226:8443 -u admin -p admin -c id-mycluster-account -n default
                    cp /home/jenkins/.cloudctl/clusters/mycluster/cert.pem /home/jenkins/.helm/cert.pem
                    cp /home/jenkins/.cloudctl/clusters/mycluster/key.pem /home/jenkins/.helm/key.pem
                    rm cloudctl
                    ls -R
                    helm version --tls
                    ./helm_wrapper.sh version
                    jx edit helmbin helm_wrapper.sh
                '''
              // release the helm chart
              sh 'jx step helm release'
              sh 'jx step helm apply'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }
