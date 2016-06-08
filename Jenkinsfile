node {
   // If deploying images to gcr - your project name here
  def project = 'engineering-devops'
  def appName = 'openidm'
  def feSvcName = "${appName}"
  // Generated image tag - adjust for your environment
  def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
  def templateImage = "forgerock/${appName}:template"

  checkout scm

  stage 'Build image'
  sh("docker build --no-cache -t ${imageTag} openidm")


  stage 'Push image to registry'
  sh("gcloud docker push ${imageTag}")

  stage "Deploy Application"

 // Create prod namespace if it doesn't exist
 sh("kubectl get ns production || kubectl create ns production")

  switch (env.BRANCH_NAME) {
    // canary deployment to production
    // TOOD: Revist this once we have multi-node deployment

    // this doesnt do anything different right now - but when we
    // get multi-instance  working on Kube, it will
    // deploy a single canary node to a N node production cluster.
    case "canary":
        // Change deployed image to the one we just built
        sh("sed -i.bak 's#${templateImage}#${imageTag}#' ./k8s/canary/*.yaml")
        // note we depliy the canary to the *production* namespace
        sh("kubectl --namespace=production apply -f k8s/services/")
        sh("kubectl --namespace=production apply -f k8s/canary/")
        //sh("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out to production
    case "production":
        // Change deployed image in staging to the one we just built
        sh("sed -i.bak 's#${templateImage}#${imageTag}#' ./k8s/production/*.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/production/")
        // For prod we want an ingress
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/ingress/")
        //sh("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out a dev (master) or feature branch environment. Each env gets its own namespace
    default:
        // Create dev branch namespace if it doesn't exist
        sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/openidm-svc.yaml")
        sh("sed -i.bak 's#${templateImage}#${imageTag}#' ./k8s/dev/*.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
        echo 'To access your environment run `kubectl proxy or kubectl port-forward`'
        echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
  }
}
