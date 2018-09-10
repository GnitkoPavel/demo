pipeline {
	agent any 
	environment { 
        WAR_NAME = 'AllocationsService.war'
        WAR_PATH = 'target'
        CONFIG_FOLDER_NAME = 'allocations'
        WILDFLY_SERVER_GROUP = 'EAM'
        DEPLOYMENT_PATH='/opt/oraclebase/deployments' 
        DEPLOYMENT_CONFIG_PATH = '/opt/oraclebase/EAM/config'
        BRANCH_TO_DEPLOY = 'develop'
        ZIP_NAME = 'AllocationsService_Full_Package-$BUILD_NUMBER.zip'
    }
	options {
		gitLabConnection('njros1lz539')
	}
	triggers {
		gitlab(
			triggerOnPush: true,
			triggerOnMergeRequest: true,
			triggerOnNoteRequest: false,
			noteRegex: ".*[Jj]enkins rebuild.*",
			setBuildDescription: true,
			addNoteOnMergeRequest: false,
			addCiMessage: true,
			branchFilterType: 'All'
		)
	}
	stages {
		stage("build") {
			steps {
				withMaven(maven: 'maven') { 
				    echo "-----------------------------------------------------------------------------------------------"
					echo "Building WAR"
					sh "mvn clean package -P wildfly" 
				}
			}
		}
		stage ('archive artifact') {
            steps {
                echo "-----------------------------------------------------------------------------------------------"
                echo "Archiving all necessary artifacts of $WAR_PATH/* to $ZIP_NAME" 
                sh "cd $WAR_PATH && mkdir artifacts && cp -r $WAR_NAME artifacts && cp -r classes/$CONFIG_FOLDER_NAME artifacts"
                sh "cd  $WAR_PATH/artifacts && zip -r ../../$ZIP_NAME *" 
                sh "rm -r $WAR_PATH/artifacts"
            }
        }
		stage("deploy") {
		    when { branch "$BRANCH_TO_DEPLOY" }
			steps {
				echo "-----------------------------------------------------------------------------------------------"
				echo "Deploying WAR to Dev Wildfly..."
                    sshagent (credentials: ['ssh_key']) {
					sh "scp $WAR_PATH/$WAR_NAME oracle@fsawildflyd1:$DEPLOYMENT_PATH"
					sh "scp -r $WAR_PATH/classes/$CONFIG_FOLDER_NAME oracle@fsawildflyd1:$DEPLOYMENT_CONFIG_PATH"
					sh '''
                        ssh oracle@fsawildflyd1 "export JAVA_HOME=/usr/jdk1.8.0_144/ && /opt/oraclebase/wildfly/wildfly-12.0.0.Final/bin/jboss-cli.sh <<EOF
                        connect
                        if (outcome==\\"success\\") of /deployment=$WAR_NAME:query
                            deploy $DEPLOYMENT_PATH/$WAR_NAME --force
                        else
                            deploy $DEPLOYMENT_PATH/$WAR_NAME --server-groups=$WILDFLY_SERVER_GROUP
                        end-if
                        EOF"
                    '''
				}
			}
		} 
	}
	
	post {
		failure {
			updateGitlabCommitStatus name: 'build', state: 'failed'
		}
		success {
			updateGitlabCommitStatus name: 'build', state: 'success'
			echo "Publishing artifacts of $WAR_PATH/* - WAR and configs"
            archiveArtifacts artifacts: "$WAR_PATH/$WAR_NAME,$ZIP_NAME", fingerprint: true, excludes: "*SNAPSHOT*"

		}
	}
}
