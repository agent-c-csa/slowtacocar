pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          securityContext:
            runAsUser: 0
          serviceAccountName: jenkins-agent
          containers:
          - name: docker
            image: docker:27.2-dind
            volumeMounts:
            - name: cert-volume
              mountPath: /etc/ssl/certs
              readOnly: true
            securityContext:
              privileged: true
          - name: kubectl
            image: bitnami/kubectl:1.27
            command:
            - cat
            tty: true
          volumes:
          - name: cert-volume
            hostPath:
              path: /etc/ssl/certs
              type: Directory
        '''
    }
  }

  environment {
    HARBOR = credentials('harbor')
  }

  stages {
    stage('Build') {
      steps {
        container('docker') {
          sh 'docker login cme-harbor.int.bobbygeorge.dev -u $HARBOR_USR -p $HARBOR_PSW'
          sh 'docker build -t bobbygeorge.dev --cache-to type=inline --cache-from type=registry,ref=cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev:$GIT_BRANCH --cache-from type=registry,ref=cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev:latest .'
          sh '[ "$GIT_BRANCH" = "master" ] && docker tag bobbygeorge.dev cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev:latest'
          sh 'docker tag bobbygeorge.dev cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev:$GIT_BRANCH'
          sh 'docker tag bobbygeorge.dev cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev:$GIT_COMMIT'
          sh 'docker push -a cme-harbor.int.bobbygeorge.dev/bobbygeorge.dev/bobbygeorge.dev'
        }
      }
    }

    stage('Deploy Preview') {
      when {
        not {
          branch 'master'
        }
      }
      steps {
        container('kubectl') {
          sh 'TAG=$GIT_COMMIT NAMESPACE=bobbygeorge-dev-$GIT_BRANCH DOMAIN=$GIT_BRANCH.www.bobbygeorge.dev envsubst \'$TAG:$NAMESPACE:$DOMAIN\' < kubernetes.yaml | kubectl apply -f -'
        }
      }
    }
    stage('Deploy Prod') {
      when {
        branch 'master'
      }
      steps {
        container('kubectl') {
          sh 'TAG=$GIT_COMMIT NAMESPACE=bobbygeorge-dev DOMAIN=www.bobbygeorge.dev envsubst \'$TAG:$NAMESPACE:$DOMAIN\' < kubernetes.yaml | kubectl apply -f -'
          sh 'NAMESPACE=bobbygeorge-dev envsubst \'$NAMESPACE\' < kubernetes-redirect.yaml | kubectl apply -f -'
        }
      }
    }
  }
}
