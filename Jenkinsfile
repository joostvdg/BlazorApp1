def podYAML = """
spec:
  serviceAccountName: jenkins-agents-sa
  nodeSelector:
    node: linux-agents
  containers:
  - name: dotnet
    image: bitnami/dotnet-sdk:9
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: "4096Mi"
        cpu: "1500m"
      limits:
        memory: "4096Mi"
  - name: git-next-tag
    image: ghcr.io/joostvdg/git-next-tag:1.2.0-alpine
    command: ['cat']
    tty: true
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
  imagePullSecrets:
    - name: nexus-cred
"""

pipeline {
    agent {
        kubernetes {
            yaml podYAML
        }
    }
    libraries {
      lib('cloudbees-shared-library@main')
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    scmVars       = checkout scmGit(branches: [[name: '*/main']], extensions: [checkoutOption(1), cloneOption(depth: 1, noTags: false, reference: '', shallow: true)], userRemoteConfigs: [[credentialsId: 'svcgtsdcatpipeline-token', url: 'https://git.us.aegon.com/jgriendt1/BlazorApp1.git']])
                    setEnv.setURLS()
                    env.gitUrl    = scmVars.GIT_URL
                    env.gitCommit = scmVars.GIT_COMMIT
                    echo "scmVars=${scmVars}"
                }
            }
        }
        stage('Read Properties') {
            steps {
                script {
                    props = readYaml file: "./ci.yaml"
                    env.baseVersion                        = props.baseVersion
                    env.gitUser                            = props.git.username
                    env.gitEmail                           = props.git.email
                    env.gitCredentialsId                   = props.git.credentialsId
                    env.ArtifactPublish                    = props.ArtifactPublish
                    env.ArtifactPublishCommandExtraOptions = props.ArtifactPublishCommandExtraOptions
                    env.ArtifactPath                       = "${env.WORKSPACE}"
                    env.ArtifactPublushRepo                = props.ArtifactPublushRepo
                }
            }
        }

        stage ('Create Stable Next Tag') {
            when { branch 'main' }
            steps {
                script {
                    version.nextStable("${env.baseVersion}")
                }
            }
        }
        stage ('Create Pre-release Next Tag') {
            when { 
                anyOf {
                    branch pattern: "PR-\\d+", comparator: "REGEXP"
                    changeRequest()
                }
            }
            steps {
                script {
                    version.nextReleaseCandidate("${env.baseVersion}")
                }
            }
        }
        stage ('Create Test Tag') {
            when { 
                branch pattern: 'test-*', comparator: "GLOB"
            }
            steps {
                script {
                    version.nextPreReleaseCommit("${env.baseVersion}")
                }
            }

        }
        stage ('Restore') {
            steps {
                script {
                    // write the NuGet.Config file with the Nexus Repositories configured
                    // have to ensure we're on the correct agent, so have to write it at this time
                    writeNugetConfig("ci.yaml", 'nexus-cred')
                    container('dotnet') {
                        sh 'dotnet restore'
                    }
                }
            }
        }
        stage ('Build') {
            steps {
                container('dotnet') {
                    sh """
                    dotnet build --no-restore /p:PackageVersion=${env.nextTag} --no-restore -c Release
                    """
                }
            }
        }
        stage ('Create Next Tag') {
            when { 
                anyOf {
                    branch pattern: 'test-*', comparator: "GLOB"
                    branch pattern: "PR-\\d+", comparator: "REGEXP"
                    changeRequest()
                    branch 'main'
                }
            }
            steps {
                script {
                    version.createGitTag(env.gitUrl, env.gitCredentialsId, env.gitUser, env.gitEmail, env.gitCommit, env.nextTag) 
                }
            }
        }

        stage ('Make release package') {
            when { expression { env.ArtifactPublish } }
            steps {
                container('dotnet') {
                    sh """
                    cat BlazorApp1/BlazorApp1.csproj
                    dotnet pack  -c Release --no-restore --no-build /p:PackageVersion=${env.nextTag} -o ${env.ArtifactPath}
                    """
                }

            }
        }
        stage ('Publish Artifact') {
            when { expression { env.ArtifactPublish } }
            environment {
                NUGET_API_KEY      = credentials('jgriendt1-nexus-nuget-token')
                NUGET_PACKAGE_REPO = "${NEXUS_URL}/repository/${env.ArtifactPublushRepo}"
            }
            steps {
                container('dotnet') {
                    sh """
                    dotnet nuget push ${env.ArtifactPath}/*.nupkg --source ${NUGET_PACKAGE_REPO}  --api-key ${NUGET_API_KEY}
                    """
                }
            }
        }
    }
}
