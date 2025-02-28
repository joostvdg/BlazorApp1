def podYAML = """
spec:
  serviceAccountName: jenkins-agents-sa
  nodeSelector:
    node: linux-agents
  containers:
  - name: dotnet
    image: docker-proxy.nexus-ci.agtservices.aegon.io/bitnami/dotnet-sdk:9
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
    image: docker-proxy.nexus-ci.agtservices.aegon.io/git-next-tag:1.2.0-alpine
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
                    env.baseVersion      = props.baseVersion
                    env.gitUser          = props.git.username
                    env.gitEmail         = props.git.email
                    env.gitCredentialsId = props.git.credentialsId
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
                    sh 'dotnet build --no-restore'
                }
            }
        }

        stage ('Create Stable Next Tag') {
            when { branch 'main' }
            environment {
                BASE   = "${env.baseVersion}"
                OUTPUT = 'next_tag.txt'
            }
            steps {
                container('git-next-tag') {
                    sh '''
                    git config --global safe.directory $(pwd)
                    /work/git-next-tag \
                        --baseTag ${BASE} \
                        --path $(pwd) \
                        --outputPath ${OUTPUT} \
                        -vvv
                    '''
                }
                script {
                    env.nextTag = readFile("${OUTPUT}")
                    sh 'rm ${OUTPUT}'
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
            environment {
                BASE   = "${env.baseVersion}"
                OUTPUT = 'next_tag.txt'
            }
            steps {
                container('git-next-tag') {
                    sh '''
                    git config --global safe.directory $(pwd)
                    /work/git-next-tag \
                        --baseTag ${BASE} \
                        --path $(pwd) \
                        --outputPath ${OUTPUT} \
                        --preRelease \
                        --suffix rc \
                        -vvv
                    '''
                }
                script {
                    env.nextTag = readFile("${OUTPUT}")
                    sh 'rm ${OUTPUT}'
                }
            }

        }
        stage ('Create Test Tag') {
            when { 
                branch pattern: 'test-*', comparator: "GLOB"
            }
            environment {
                BASE   = "${env.baseVersion}"
                OUTPUT = 'next_tag.txt'
            }
            steps {
                container('git-next-tag') {
                    sh '''
                    git config --global safe.directory $(pwd)
                    /work/git-next-tag \
                        --baseTag ${BASE} \
                        --path $(pwd) \
                        --outputPath ${OUTPUT} \
                        --preRelease \
                        --commit \
                        -vvv
                    '''
                }
                script {
                    env.nextTag = readFile("${OUTPUT}")
                    sh 'rm ${OUTPUT}'
                }
            }

        }

        stage ('Create Next Tag') {
            when { 
                anyOf {
                    branch pattern: 'test-.*', comparator: "GLOB"
                    branch pattern: "PR-\\d+", comparator: "REGEXP"
                    changeRequest()
                    branch 'main'
                }
            }
            steps {

                script {
                    def url = env.gitUrl
                    withCredentials([usernameColonPassword(credentialsId: env.gitCredentialsId, variable: 'GIT_CREDS')]) {
                        def alteredUrl = url.replace("https://", "https://${env.GIT_CREDS}@")
                        sh "git remote set-url origin ${alteredUrl}"
                    }
                }

                sh """
                echo "Setting git user to ${env.gitUser} <${env.gitEmail}>"
                git config --global user.email ${env.gitUser}
                git config --global user.name ${env.gitEmail}

                git remote -vv

                echo "Creating tag ${env.nextTag}"
                git tag -a  "${env.nextTag}" -m "Version bump from CloudBees CI" "${env.gitCommit}"

                echo "Pushing tag ${env.nextTag}"
                git push origin "${env.nextTag}"
                """
            }
        }
    }
}
