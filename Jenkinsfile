def readProperties()
{

	def properties_file_path = "${workspace}" + "@script/properties.yml"
	def property = readYaml file: properties_file_path
	env.APP_NAME = property.APP_NAME
    env.MS_NAME = property.MS_NAME
    env.BRANCH = property.BRANCH
    env.GIT_SOURCE_URL = property.GIT_SOURCE_URL
    env.SCR_CREDENTIALS = property.SCR_CREDENTIALS
    env.SONAR_HOST_URL = property.SONAR_HOST_URL
    env.CODE_QUALITY = property.CODE_QUALITY
    env.UNIT_TESTING = property.UNIT_TESTING
    env.CODE_COVERAGE = property.CODE_COVERAGE
    env.FUNCTIONAL_TESTING = property.FUNCTIONAL_TESTING
    env.SECURITY_TESTING = property.SECURITY_TESTING
	env.PERFORMANCE_TESTING = property.PERFORMANCE_TESTING
	env.TESTING = property.TESTING
	env.QA = property.QA
	env.PT = property.PT
	env.User = property.User
    env.DOCKER_REGISTRY = property.DOCKER_REGISTRY
    env.DOCKER_REPO=property.DOCKER_REPO
	env.IMAGE_TAG = (new Date()).format("yyMMddHHmm", TimeZone.getTimeZone('UTC'))
	
    
}


podTemplate(cloud:'openshift',label: 'dotnet',
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'blrocpimpregistry:5000/jnlp-slave-dotnet:3.1',
      alwaysPullImage: true,
      privileged: true,
      envVars: [envVar(key:'http_proxy',value:''),envVar(key:'https_proxy',value:'')],
      args: '${computer.jnlpmac} ${computer.name}',
      ttyEnabled: true
    )])
{
podTemplate(cloud:'openshift',label: 'docker',
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'blrocpimpregistry:5000/shubhamasati/donotcopy:jenkinsslave',
      alwaysPullImage: true,
      privileged: true,
      envVars: [envVar(key:'http_proxy',value:''),envVar(key:'https_proxy',value:'')],
      args: '${computer.jnlpmac} ${computer.name}',
      ttyEnabled: true
    )],volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),hostPathVolume(hostPath: '/etc/docker/daemon.json', mountPath: '/etc/docker/daemon.json')] )
{
podTemplate(cloud:'openshift',label: 'zap',
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'blrocpimpregistry:5000/jenkins-slave-zap:latest',
      alwaysPullImage: true,
      privileged: true,
      envVars: [envVar(key:'http_proxy',value:''),envVar(key:'https_proxy',value:'')],
      args: '${computer.jnlpmac} ${computer.name}',
      ttyEnabled: true
    )] )
{
def PROXY_URL
def ROUTE
node 
{

    def owasp_sca = tool "owasp_sca"
   env.PATH="${env.PATH}:${owasp_sca}/bin"

   stage('Read properties')
   {
       readProperties()
       checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${SCR_CREDENTIALS}", url: "${GIT_SOURCE_URL}"]]])
       dir('./')
        {
            stash name : 'checkout' , includes : '**'
        }
   }
   
node('dotnet') 
{
   stage('Checkout')
   {
        dir('./')
        {
            unstash name : 'checkout'
        }
   }
   
   withCredentials([usernamePassword(credentialsId: "${SCR_CREDENTIALS}", usernameVariable: 'username', passwordVariable: 'password')])
   {
        PROXY_URL="http://$username:$password@ip:80"
    }    
       withEnv(["http_proxy=${PROXY_URL}","https_proxy=${PROXY_URL}"]) 
       {
        	   stage('Initial Setup')
        	   {
        	        
        			sh 'curl https://www.google.com'
        			sh 'dotnet restore'
        			sh 'dotnet tool install --global dotnet-sonarscanner '
            			
        	   }
        }	   
        	   
        	   if(env.UNIT_TESTING == 'True')
           {
           	stage('Unit Testing')
           	{
                	sh 'dotnet test'
           	}
           }
            if(env.CODE_COVERAGE == 'True')
           {
           	stage('Code Coverage')
           	{
           	    sh 'dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover'
           	}
           }
        	
            if(env.CODE_QUALITY == 'True')
        	{
    		stage('Code Quality')
    	    {
    	        
        		sh "/home/jenkins/.dotnet/tools/dotnet-sonarscanner begin  /d:sonar.host.url=${SONAR_HOST_URL} /key:${MS_NAME} /d:sonar.cs.opencover.reportsPaths='*/coverage.opencover.xml/'"
                sh 'dotnet build'
            	sh "/home/jenkins/.dotnet/tools/dotnet-sonarscanner end "
        			
    	    }
    	    }
    	    
    	    
        	
        	   
        	   
        		stage('Build')
        	   {
        	        
            		sh 'dotnet build'
            			
        	   }
        	   
        	   stage('Publish')
        	   {
        	        sh 'dotnet publish --no-build -o ./PublishOutput'
        	        dir('./PublishOutput')
        	        {
        	            stash name : 'publishoutput' , includes : '**'
        	        }
        	        
        	   }
        	   
    	
	

}

        stage('SCA')
        {
            dependencyCheck additionalArguments: '--proxyserver <ip> --proxyport 80 --proxypass <pass> --proxyuser <user>', odcInstallation: 'owasp_sca'
            dependencyCheckPublisher pattern: ''
        }
        stage('Build and Tag Image for Dev')
       {
       		
    			withCredentials([
    				usernamePassword(credentialsId: 'DockerID', usernameVariable: 'username', passwordVariable: 'password')
    			  ])
    			{  
    			    sh 'oc config use-context  ${APP_NAME}-devops'
    				 node('docker')
                    {
                    	
                    	container('jnlp')
                    	{ 
                                dir('./')
                                {
                                    unstash name : 'checkout'
                                }
            			        dir('./PublishOutput')
            			        {
            			            unstash name : 'publishoutput'
            			        }
            			        
            					sh "docker login ${DOCKER_REGISTRY} -u $username -p $password"
            					sh "docker build -t ${MS_NAME}:latest ."
            					sh 'docker tag ${MS_NAME}:latest blrocpimpregistry:5000/${DOCKER_REPO}/$MS_NAME:${APP_NAME}-dev-apps'
            					sh 'docker push blrocpimpregistry:5000/${DOCKER_REPO}/$MS_NAME:${APP_NAME}-dev-apps'
            			 }
    			        
    			     }   
    			}
    		 
        
    	   
       }
    
    stage('Dev - Deploy Application')
   {   
       FAILED_STAGE=env.STAGE_NAME
       
       
       
       
       sh 'oc set image --local=true -f Orchestration/dev/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/${DOCKER_REPO}/$MS_NAME:${APP_NAME}-dev-apps-"${IMAGE_TAG}" --dry-run -o yaml >> Orchestration/dev/deployment-"${IMAGE_TAG}".yaml' 
        sh 'chmod 777 ./Orchestration/dev/deploy.sh' 
        sh "sed -i 's/\r//' ./Orchestration/dev/deploy.sh" 
        sh './Orchestration/dev/deploy.sh deployment-"${IMAGE_TAG}".yaml'

       
       if(env.ISTIO_CHECK == 'True') {
        sh 'chmod 777 Orchestration/istio/deploy.sh'
        sh 'Orchestration/istio/deploy.sh'
      }
   }
    
        ROUTE="http://${MS_NAME}-${APP_NAME}-dev-apps.apps.admcoepaas.local"
        timeout(time:5, unit:'MINUTES') {
            input message:'Ready for DAST?'
        }
       
        node('zap')
        { 
            container('jnlp')
            {
                stage('DAST')
                {
                
                    //sleep time:5,unit:'MINUTES'
                    sh "zap-cli start --start-options '-config api.key=12345'"
                    sh "zap-cli --api-key 12345 open-url <url>"
                    sh "zap-cli --api-key 12345 spider <url>"
                    sh 'zap-cli --api-key 12345 report --output-format html --output zapreport.html'
                    stash name:'zapreport', includes:'zapreport.html'
                }
            }
        }
        
        dir('./'){ unstash name:'zapreport' }
        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'zapreport.html', reportName: 'Owasp ZAP', reportTitles: ''])

    




}

}
}
}
