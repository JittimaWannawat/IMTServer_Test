node {
    cleanWs()
    def githubRepoName = 'imt.server'
    def registry = '041736873643.dkr.ecr.ap-southeast-1.amazonaws.com'
    def registryName = 'imt.server';
    def ns = 'development'
    def cluster = 'kollective'
    def tag
    def tagVersion

    stage('Pull Repository') {
      checkout scm
      tagVersion = "$BUILD_NUMBER"
      if (env.BRANCH_NAME == 'main') {
        ns = 'production'
      } else if (env.BRANCH_NAME == 'staging') {
        ns = 'staging'
      } else if (env.BRANCH_NAME == 'develop') {
        ns = 'develop'
      } 
      tag = "$ns-$tagVersion"
      sh "echo 'Build number by tag version:' $tagVersion"
    }

    stage('SonarQube Analysis') {
      nodejs(nodeJSInstallationName: 'node') {
        def scannerHome = tool 'SonarScanner'
        withSonarQubeEnv() {
          sh "${scannerHome}/bin/sonar-scanner"
        }
      }
    }

    stage('Unit Test') { }

    stage ('Build Image') {
      //sh "echo 'Build Image: API'"
      //if (env.BRANCH_NAME == 'develop') {
      //  sh "docker build --build-arg environment=Development -t $registry/$registryName:$tag ."
      //} else if (env.BRANCH_NAME == 'staging') {
      //  sh "docker build --build-arg environment=Staging -t $registry/$registryName:$tag ."
      //} else {
      //  sh "docker build --build-arg environment=Production -t $registry/$registryName:$tag ."
      //}
    }

    stage ('Push Image') {
        //docker.withRegistry("https://$registry", 'ecr:ap-southeast-1:Jenkins') {
        //  sh "docker push $registry/$registryName:$tag"
        //}
    }

    stage ('Deploy') {
      // Deployment
      def ymlApi = "$githubRepoName-deployment.yml"
      def dataApi = readYaml file: ymlApi
      dataApi.spec.template.spec.containers[0].image = "fissionsoftregistry.azurecr.io/influencermarkethingtool/imt.server-trunk:latest"

      sh "rm $ymlApi"
      writeYaml file: ymlApi, data: dataApi
      sh "cat $githubRepoName-deployment.yml"

      if (env.BRANCH_NAME != 'main') {
        def suffix = "$ns";
        if(ns == 'develop') {
          suffix = 'dev'
        }

        // Ingress
      if (env.BRANCH_NAME != 'main') { // Develop + Staging
        def ymlApiIngress = "$githubRepoName-ingress-${suffix}.yml"
        def dataApiIngress = readYaml file: ymlApiIngress
        dataApiIngress.spec.rules[0].host = "it-api-${suffix}.kolify.one"
        dataApiIngress.spec.tls[0].hosts[0] = "it-api-${suffix}.kolify.one"
      }else{
        def ymlApiIngress = "$githubRepoName-ingress.yml"
        def dataApiIngress = readYaml file: ymlApiIngress
        dataApiIngress.spec.rules[0].host = "internal-tool-api.kolify.one"
        dataApiIngress.spec.tls[0].hosts[0] = "internal-tool-api.kolify.one"
      }

        sh "rm $ymlApiIngress"
        writeYaml file: ymlApiIngress, data: dataApiIngress
        sh "cat $githubRepoName-ingress.yml"
      }

      withCredentials([
        [
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: "Jenkins",
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]
      ])
      {
          withEnv(["PROJECT_NAME=$githubRepoName","NAMESPACE=$ns","CLUSTER_NAME=$cluster"]) {
            sh '''
              aws eks --region ap-southeast-1 update-kubeconfig --name ${CLUSTER_NAME}  --role-arn arn:aws:iam::041736873643:role/kol-prod-admin-access
              kubectl get pods -n ${NAMESPACE}
              kubectl apply -f ${PROJECT_NAME}-configmap.yml -n ${NAMESPACE}
              kubectl apply -f ${PROJECT_NAME}-deployment.yml -n ${NAMESPACE}
              kubectl apply -f ${PROJECT_NAME}-service.yml -n ${NAMESPACE}
              kubectl apply -f ${PROJECT_NAME}-ingress.yml -n ${NAMESPACE}
            '''
          }
      }

      slackSend(color: "good", channel: "devops", message: "$githubRepoName on branch $env.BRANCH_NAME (revision $tag) has been successfully deployed.")
    }
}
