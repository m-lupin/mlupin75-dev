
/* Pipeline Jenkins to build et deploy nodejs Shark App */
/* Auteur: Mejri Issam */

pipeline {

        // We will use any agent available.
        agent any

        // set a timeout of 30 minutes for this pipeline. 
        options
        {
            timeout(time: 30, unit: 'MINUTES')
	    buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
        }

        // List of parameters required to configure before the launch of the pipeline.
        parameters{
            string(name: 'BRANCH', defaultValue: 'master', description: 'Name of the branch to use.')
            string(name: 'NB_REPLICAS', defaultValue: '1', description: 'Number of replicas for deployment.')
            choice(name: 'ENVIRONMENT', choices: ['mlupin75-dev', 'mlupin75-stage'], description: 'The name of the environement where we want to deploy/build resources.')
            choice(name: 'DEPLOYMENT_TYPE', choices: ['','BUILD', 'DEPLOY'], description: 'The name of the type of deployment to process, can be either Build or Deploy.')
            string(name: 'IMAGE_TAG', defaultValue: '', description: 'tag for the build image.')
	    string(name: 'SOURCES_URL', defaultValue: 'https://github.com/m-lupin/mlupin75-dev.git', description: 'Gitlab repository source of the application.')

        }

        // List of stages used on the pipelines.
        stages
        {
            stage('Checkout.')
            {
                steps
                {
                    script{
                            if ( params.DEPLOYMENT_TYPE.isEmpty() )
                            {
                                // Use SUCCESS FAILURE or ABORTED
                                currentBuild.result = "FAILURE"
                                throw new Exception("The Parameters DEPLOYMENT_TYPE is empty, the value must be either BUILD or DEPLOY, Please set DEPLOYMENT_TYPE parameters.")
                            }
                    }
                        
                    checkout([$class: 'GitSCM', 
                            branches: [[name: "*/${env.BRANCH}"]], 
                            userRemoteConfigs:[[credentialsId:  'gitlab', url: "${env.SOURCES_URL}"]]
                    ])
                    
                } // steps
            } // stage

            stage("Build Source Code")
            {
                when {
                    // Only for BUILD Deployment type.
                    expression { params.DEPLOYMENT_TYPE == 'BUILD' }
                }
                steps
                {

                    script
                    {
                       sh """
		        oc delete bc -l build=nodejs-image
                        oc new-build --name=nodejs-image --image-stream=openshift/nodejs:16-ubi8 ${params.SOURCES_URL} --to='nodejs-image:${params.IMAGE_TAG}'
                        """
                    } // script 
                } // steps
            } // stage

            stage("Deploy resources for DEV/STAGE environments.")
            {
                 when {
                    expression { params.DEPLOYMENT_TYPE == 'DEPLOY' }
                 }
                steps
                { 
                    script
                    {
                       sh """
                        oc apply -f nodejs-image-demo-deploy.yaml -n ${params.ENVIRONMENT}
			oc set image deployment/nodejs-image-demo nodejs-image-demo=image-registry.openshift-image-registry.svc:5000/mlupin75-dev/nodejs-image:${IMAGE_TAG}
			oc scale --replicas=${params.NB_REPLICAS} deployment/nodejs-image-demo
                        oc apply -f nodejs-image-demo-svc.yaml -n ${params.ENVIRONMENT}
                        """
                    } // script
                } // steps
            } // stage

		stage("Create route to access the Application")
            	{
                 when {
                    expression { params.ENVIRONMENT == 'mlupin75-dev' && params.DEPLOYMENT_TYPE == 'DEPLOY' }
                 }
                steps
                { 
                    script
                    {
			sh """
			   oc delete route nodejs-image-demo --ignore-not-found
                       	   oc expose svc/nodejs-image-demo
			"""
                    } // script
                } // steps
            } // stage
        } // stages

        post 
        {
            always 
            {
                deleteDir() // Delete the current workspace.
                dir("${env.WORKSPACE}@script") 
                { 
                    deleteDir() // Delete the script workspace, this allow us to update the pipeline script without deleting the builconfig.
                } // dir
            } // always
        } // post 
} // pipeline
