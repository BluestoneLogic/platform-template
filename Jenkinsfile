def branchVariables = [
    "origin/testing": [
        "serverURL": "https://dmp-kd-test.kinops.io/",
        "slugName": "dmp-kd-test",
        "spaceName": "DMP-KD Test",
    ],
    "origin/dev": [
        "serverURL": "https://bluestone-dev.kinops.io/",
        "slugName": "bluestone-logic",
        "spaceName": "Bluestone Logic",
    ]
]

pipeline {
    agent any
    environment {
        FPRNAME = sh(script: 'date +"%Y%m%d%H%M%S".fpr', , returnStdout: true).trim()
    }
    stages {
        stage('Fortify') {
            agent {
                docker {
                    image 'docker-registry.toolchain.c2il.org/factory/fortify-sca:latest'
                    args '''
                    -v ${workspace}:/opt/kinetic-configuration
                    -e PROJECT_NAME="JBOX DMP Platform Template"
                    -e PROJECT_VERSION="0.1"
                    -e PATH="$PATH:/app/sca/bin:/app/scatools/bin:/usr/local/bin"
                    -u root
                    '''
                    reuseNode true
                }
            }
            steps {
                echo "Beginning Fortify scan..."
                withCredentials([string(credentialsId: 'fortify_authtoken', variable: 'AUTHTOKEN')]) {
                    sh '''
                    cd /opt/kinetic-configuration
                    fortifyupdate
                    sourceanalyzer -verbose -b srcbuild -exclude "/opt/kinetic-configuration/build/static/js/*.chunk.js" /opt/kinetic-configuration/**/* -Dcom.fortify.sca.limiters.MaxPassthroughChainDepth=8 -Dcom.fortify.sca.limiters.MaxChainDepth=8 -Dcom.fortify.sca.EnableDOMModeling=true
                    sourceanalyzer -verbose -b srcbuild -scan -f /opt/kinetic-configuration/$FPRNAME
                    fortifyclient -debug -url https://fortify.toolchain.c2il.org/ -authtoken $AUTHTOKEN uploadFPR -file /opt/kinetic-configuration/$FPRNAME -project "$PROJECT_NAME" -applicationVersion "$PROJECT_VERSION"
                    '''
                }
            }
        }
        stage('Docker Build and Deploy') {
            agent {
                docker{
                    image 'docker-registry.toolchain.c2il.org/factory/jbox/ubi8-metacop:latest'
                    args '''
                        -v ${workspace}:/opt/kinetic-configuration
                        -e PATH="$PATH:/opt/node/node-v16.13.2-linux-x64/bin:/usr/sbin"
                        -e NODE_OPTIONS="--max-old-space-size=4096"
                        --network host
                        -u root
                    '''
                    reuseNode true
                }
            }

            stages{
                stage('Build') {
                    steps {
                        script {
                            def branch = env.GIT_BRANCH
                            env.serverURL = branchVariables?.get(branch)?.serverURL
                            env.slugName = branchVariables?.get(branch)?.slugName
                            env.spaceName = branchVariables?.get(branch)?.spaceName
                        }
                        echo "Running a build..."
                        sh '''
                        cd /opt/kinetic-configuration
                        yum install @ruby:3.1 gettext -y
                        gem install bundler
                        gem install rexml
                        bundle install
                        '''
                        withCredentials([usernamePassword(credentialsId: 'kd-dev-credentials', usernameVariable: "USERNAME", passwordVariable:"B64PASSWORD")]) {
                            sh("export serviceUsername=${USERNAME}")
                            sh("export servicePassword=${B64PASSWORD}")
                            sh("envsubst < config/servername_environment_export_config.yml > config/export.yml")
                        }

                        sh '''
                        ruby ./export.rb -c config/export.yml
                        chmod -R 777 .
                        tar cvf $BUILDNAME .
                        '''
                    }
                }
                stage('Deploy') {
                    steps{
                        script {
                            env.BUCKET_NAME = "s3://disa-marketplace-kd-config"
                        }
                        echo "Deploying..."
                        // install aws cli
                        sh '''
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
                        unzip /tmp/awscliv2.zip -d /tmp
                        /tmp/aws/install
                        '''
                        // aws config setup
                        withCredentials([usernamePassword(credentialsId: 'srv-dmp-aws', usernameVariable: 'ACCESS_KEY', passwordVariable: 'SECRET_KEY')]) {
                            sh("aws configure set aws_access_key_id $ACCESS_KEY")
                            sh("aws configure set aws_secret_access_key $SECRET_KEY")
                            sh("aws configure set default.region us-gov-west-1")
                        }
                        // upload
                        sh '''
                         aws s3 cp --recursive /opt/kinetic-configuration/export ${BUCKET_NAME} --region us-gov-west-1
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                docker.image('docker-registry.toolchain.c2il.org/factory/jbox/ubi8-metacop:latest').inside("-u root") {
                // sh 'find . -user root -name * | xargs chmod ugo+rw || true'
                sh '''
                chmod -R ugo+rw /data/workspace/|| true
                '''
                }
                docker.image('docker-registry.toolchain.c2il.org/factory/fortify-sca:latest').inside("-u root") {
                // sh 'find . -user root -name * | xargs chmod ugo+rw || true'
                sh '''
                chmod -R ugo+rw /data/workspace/ || true
                '''
                }
            }
        }
    }
}