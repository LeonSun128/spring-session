def projectProperties = [
	[$class: 'BuildDiscarderProperty',
		strategy: [$class: 'LogRotator', numToKeepStr: '5']],
	pipelineTriggers([cron('@daily')])
]
properties(projectProperties)

def SUCCESS = hudson.model.Result.SUCCESS.toString()
currentBuild.result = SUCCESS

try {
	parallel check: {
		stage('Check') {
			node {
				checkout scm
				try {
					sh "./gradlew clean check  --refresh-dependencies --no-daemon"
				} catch(Exception e) {
					currentBuild.result = 'FAILED: check'
					throw e
				} finally {
					junit '**/build/*-results/*.xml'
				}
			}
		}
	},
	sonar: {
		stage('Sonar') {
			node {
				checkout scm
				withCredentials([string(credentialsId: 'spring-sonar.login', variable: 'SONAR_LOGIN')]) {
					try {
						sh "./gradlew clean sonarqube -PexcludeProjects='**/samples/**' -Dsonar.host.url=$SPRING_SONAR_HOST_URL -Dsonar.login=$SONAR_LOGIN --refresh-dependencies --no-daemon"
					} catch(Exception e) {
						currentBuild.result = 'FAILED: sonar'
						throw e
					}
				}
			}
		}
	},
	springio: {
		stage('Spring IO') {
			node {
				checkout scm
				try {
					sh "./gradlew clean springIoCheck -PexcludeProjects='**/samples/**' --refresh-dependencies --no-daemon --stacktrace"
				} catch(Exception e) {
					currentBuild.result = 'FAILED: springio'
					throw e
				} finally {
					junit '**/build/spring-io*-results/*.xml'
				}
			}
		}
	}

	if(currentBuild.result == 'SUCCESS') {
		parallel ossrh : {
			stage('OSSRH Deploy') {
				node {
					checkout scm
					withCredentials([file(credentialsId: 'spring-signing-secring.gpg', variable: 'SIGNING_KEYRING_FILE')]) {
						withCredentials([string(credentialsId: 'spring-gpg-passphrase', variable: 'SIGNING_PASSWORD')]) {
							withCredentials([usernamePassword(credentialsId: 'oss-token', passwordVariable: 'OSSRH_PASSWORD', usernameVariable: 'OSSRH_USERNAME')]) {
								sh "./gradlew uploadArchives -PexcludeProjects='**/samples/**' -Psigning.secretKeyRingFile=$SIGNING_KEYRING_FILE -Psigning.keyId=$SPRING_SIGNING_KEYID -Psigning.password=$SIGNING_PASSWORD -PossrhUsername=$OSSRH_USERNAME -PossrhPassword=$OSSRH_PASSWORD  --refresh-dependencies --no-daemon"
							}
						}
					}
				}
			}
		},
		docs: {
			stage('Deploy Docs') {
				node {
					checkout scm
					withCredentials([file(credentialsId: 'docs.spring.io-jenkins_private_ssh_key', variable: 'DEPLOY_SSH_KEY')]) {
						sh "./gradlew deployDocs -PdeployDocsSshKeyPath=$DEPLOY_SSH_KEY -PdeployDocsSshUsername=$SPRING_DOCS_USERNAME --refresh-dependencies --no-daemon --stacktrace"
					}
				}
			}
		}
	}
} finally {
	def buildStatus = currentBuild.result
	def buildNotSuccess =  !SUCCESS.equals(buildStatus)
	def lastBuildNotSuccess = !SUCCESS.equals(currentBuild.previousBuild?.result)

	if(buildNotSuccess || lastBuildNotSuccess) {

		stage('Notifiy') {
			node {
				final def RECIPIENTS = [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]

				def subject = "${buildStatus}: Build ${env.JOB_NAME} ${env.BUILD_NUMBER} status is now ${buildStatus}"
				def details = """The build status changed to ${buildStatus}. For details see ${env.BUILD_URL}"""

				emailext (
					subject: subject,
					body: details,
					recipientProviders: RECIPIENTS,
					to: "$SPRING_SECURITY_TEAM_EMAILS"
				)
			}
		}
	}
}
