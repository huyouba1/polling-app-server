def label = "slave-${UUID.randomUUID().toString()}"


podTemplate(label: label, serviceAccount: 'jenkins',containers: [
  containerTemplate(name: 'slave', image: 'inbound-agent:4.11-1-jdk11', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'maven', image: 'maven:3.6-alpine', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'kubectl', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'helm:latest', command: 'cat', ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: '/root/.m2', hostPath: '/var/run/m2'),
  hostPathVolume(mountPath: '/home/jenkins/.kube', hostPath: '/root/.kube'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    def dockerRegistryUrl = "docker.io"
    def imageEndpoint = "ist/polling-app-server"
    def image = "${dockerRegistryUrl}/${imageEndpoint}"

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
      try {
        container('maven') {
          echo "2. 代码编译打包阶段"
          sh "mvn clean package -Dmaven.test.skip=true -s  settings.xml"
        }
      } catch (exc) {
        println "构建失败 - ${currentBuild.fullDisplayName}"
        throw(exc)
      }
    }
    stage('构建 Docker 镜像') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'dockerAuth',
        usernameVariable: 'dockerHubUser',
        passwordVariable: 'dockerHubPassword']]) {
          container('docker') {
            echo "3. 构建 Docker 镜像阶段"
            sh """
              docker login ${dockerRegistryUrl} -u ${dockerHubUser} -p Harbor12345
              docker build -t ${image}:${imageTag} .
              docker push ${image}:${imageTag}
              """
          }
      }
    }
    stage('运行 Kubectl') {
      container('kubectl') {
        echo "查看 K8S 集群 Pod 列表"
        sh "kubectl get pods"
        sh """
          sed -i "s#<IMAGE>#${image}#g" k8s.yaml
          sed -i "s#<IMAGE_TAG>#${imageTag}#g" k8s.yaml
          kubectl apply -f k8s.yaml
        """
      }
    }
    stage('运行 Helm') {
      container('helm') {
        echo "查看 Helm Release 列表"
        sh "helm list"
      }
    }
  }
}
