
node
{
    stage('SCM Checkout')
        {
        git credentialsId: 'gitvasanth', url: 'https://github.com/vasanthanilavan/my-app.git'
        }

    stage ('MVN package')
        {
        def mvnHome = tool name: 'maven-3', type: 'maven'
        def mvnCMD = "${mvnHome}/bin/mvn"
        sh "${mvnCMD} clean package"
        }
    
    stage ('Build Docker image')
        {
       sh 'docker build -t vasanthanilavan/myapp:1.0 .' 
        }  
    
    stage ('Push docker image to hub')
        {
        withCredentials([string(credentialsId: 'Dockerhub', variable: 'DockerHubPwd')])
            {
                sh "docker login -u vasanthanilavan -p ${DockerHubPwd}"
            }
        sh  'docker push vasanthanilavan/myapp:1.0'
        }
    
    
    stage (' Run Docker in Dev server')
        {
        def dockerRun= 'docker run -d --name myapp -p 8080:8080 vasanthanilavan/myapp:1.0'   
            sshagent(['dockerDevServer']) 
            {
                sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.43.81 ${dockerRun}"
            }
        }
    
    stage (' Install and Run Docker in Test server')
        {
        def dockerRun= 'docker run -d --name myapp -p 8080:8080 vasanthanilavan/myapp:1.0'
        def jdkinstall= 'sudo amazon-linux-extras install java-openjdk11 -y'
        def dockerInstall= 'sudo yum install docker -y'
        def dockeruserAdd= 'sudo usermod -a -G docker ec2-user'
        def startDocker= 'sudo systemctl start docker'   
            sshagent(['dockerTestServerID']) 
            {
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.44.228 ${jdkinstall}"
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.44.228 ${dockerInstall}"
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.44.228 ${dockeruserAdd}"
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.44.228 ${startDocker}"
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.44.228 ${dockerRun}"
            }
        }   


    stage (' Run Docker in Prod server')
        {
        def dockerRun= 'docker run -d --name myapp -p 8080:8080 vasanthanilavan/myapp:1.0'   
            sshagent(['dockerProdServerID']) 
            {
            sh " ssh -o StrictHostKeyChecking=no ec2-user@172.31.46.214 ${dockerRun}"
            }
        }
}
