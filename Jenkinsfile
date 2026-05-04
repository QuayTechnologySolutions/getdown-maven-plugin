pipeline {
    agent {
        docker {
            image 'maven:3.9.14-eclipse-temurin-11-noble' // Ubuntu 24.04 LTS ("Noble Numbat") contains git
            args '--net="host" -v /var/jenkins_home/.m2:/home/ubuntu/.m2'
        }
    }

    parameters {
        string(name: 'VERSION', defaultValue: null, description: 'Release version (Optional)')
        booleanParam(name: 'RELEASE', defaultValue: false, description: 'Perform Release')
    }

    environment {
        GITHUB_PROJECT = 'scm:git:git@github.com:QuayTechnologySolutions/getdown-maven-plugin.git'
        GIT_CREDENTIALS_ID = 'github-getdown-maven-plugin-deploy-key'
        MAVEN_ARGS = "--errors --batch-mode --settings ./maven-settings.xml"
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
    }

    stages {
        stage('Prepare Maven Settings') {
            steps {
                writeFile(file: 'maven-settings.xml', text: '''
                    <settings>
                      <servers>
                        <server>
                          <id>nexus</id>
                          <username>${env.NEXUS_CREDENTIALS_USR}</username>
                          <password>${env.NEXUS_CREDENTIALS_PSW}</password>
                        </server>
                      </servers>
                      <mirrors>
                        <mirror>
                          <id>nexus</id>
                          <mirrorOf>*</mirrorOf>
                          <url>https://nexus.quaypart.com/repository/public/</url>
                        </mirror>
                      </mirrors>
                    </settings>
                ''')
            }
        }

        stage('Determine version') {
            steps {
                script {
                    final latestTag = getLatestTag()
                    env.VERSION = params.VERSION ?: getNewVersion(latestTag)

                    echo("version: ${VERSION} (latestTag: ${latestTag})")
                }
            }
        }

        stage('Install') {
            steps {
                runMavenCommand("clean install -Drevision=${VERSION}")
            }
        }

        stage('Release') {
            when {
                expression { return params.RELEASE }
            }
            steps {
                sshagent(credentials: [GIT_CREDENTIALS_ID]) {
                    runMavenCommand("release:prepare release:perform -DreleaseVersion=${VERSION} \"-DconnectionUrl=${env.GITHUB_PROJECT}\"")
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


String getNewVersion(String latestTag) {
    if (latestTag.empty) {
        return '1.0'
    }

    def matcher = latestTag =~ /(\d+)\.(\d+)/
    if (matcher) {
        final int major = matcher[0][1].toInteger()
        final int minor = matcher[0][2].toInteger()
        return "${major}.${minor + 1}"
    }
    matcher = latestTag =~ /(\d+)/
    if (matcher) {
        final int major = matcher[0][1].toInteger()
        return "${major + 1}"
    }

    fail("Unsupported tag format [${latestTag}]")
}

String getLatestTag() {
    String latestTag = ''
    sshagent(credentials: [GIT_CREDENTIALS_ID]) {
        final command = "git for-each-ref refs/tags --sort=-taggerdate --format=\"%(refname:short)\" --count=1"
        latestTag = sh(script: command, returnStdout: true).trim()
    }

    latestTag
}

void runMavenCommand(String args) {
    sh("mvn ${MAVEN_ARGS} ${args}")
}
