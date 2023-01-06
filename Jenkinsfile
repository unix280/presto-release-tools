pipeline {

    agent none

    environment {
        GITHUB_CREDENTIAL_ID = 'github-personal-token-wanglinsong'
        AWS_CREDENTIAL_ID    = 'aws-jenkins'
        AWS_DEFAULT_REGION   = 'us-east-1'
        AWS_ECR              = credentials('aws-ecr-private-registry')
        AWS_ECR_PUBLIC       = 'public.ecr.aws/c0e3k9s8'
        AWS_S3_PREFIX        = 's3://oss-jenkins/artifact/presto'
        S3_URL_BASE          = 'https://oss-presto-release.s3.amazonaws.com/presto'
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '500'))
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage ('Search') {
            agent {
                kubernetes {
                    defaultContainer 'maven'
                    yamlFile 'agent-maven.yaml'
                }
            }

            stages {
                stage('Setup') {
                    steps {
                        sh 'apt update && apt install -y awscli git jq tree'
                    }
                }

                stage ('Load Presto') {
                    steps {
                        checkout $class: 'GitSCM',
                                branches: [[name: '*/master']],
                                doGenerateSubmoduleConfigurations: false,
                                extensions: [[
                                    $class: 'RelativeTargetDirectory',
                                    relativeTargetDir: 'presto'
                                ],[
                                    $class: 'CloneOption',
                                    shallow: true,
                                    noTags:  true,
                                    depth:   1,
                                    timeout: 10
                                ]],
                                submoduleCfg: [],
                                userRemoteConfigs: [[
                                    credentialsId: "${GITHUB_CREDENTIAL_ID}",
                                    url: 'https://github.com/prestodb/presto'
                                ]]
                        dir('presto') {
                            sh 'mvn versions:set -DremoveSnapshot'
                            script {
                                env.PRESTO_RELEASE_VERSION = sh(
                                    script: 'mvn org.apache.maven.plugins:maven-help-plugin:3.2.0:evaluate -Dexpression=project.version -q -DforceStdout',
                                    returnStdout: true).trim()
                            }
                            echo "Presto release version ${PRESTO_RELEASE_VERSION}"
                            sh 'git reset --hard'
                        }
                    }
                }

                stage ('Search Artifacts') {
                    steps {
                        echo "query for presto docker images with release version ${PRESTO_RELEASE_VERSION}"
                        withCredentials([[
                                $class: 'AmazonWebServicesCredentialsBinding',
                                credentialsId: "${AWS_CREDENTIAL_ID}",
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            dir('presto') {
                                sh '''#!/bin/bash -ex
                                    TAGS=($(aws ecr list-images --repository-name oss-presto/presto \
                                        | jq -r '.imageIds[].imageTag | select( . != null)' \
                                        | grep "${PRESTO_RELEASE_VERSION}-20" \
                                        | sort -r))

                                    for TAG in $TAGS
                                    do
                                        echo $TAG
                                        sha=$(echo $TAG | awk -F- '{print $3}')
                                        if git log | grep $sha
                                        then
                                            echo done
                                            echo $TAG > release-tag.txt
                                            break
                                        fi
                                    done
                                    cat release-tag.txt
                                '''
                                script {
                                    env.DOCKER_IMAGE_TAG = sh(
                                        script: 'cat release-tag.txt',
                                        returnStdout: true).trim()
                                    env.PRESTO_BUILD_VERSION = env.DOCKER_IMAGE_TAG.substring(0, env.DOCKER_IMAGE_TAG.lastIndexOf('-'));
                                    env.PRESTO_RELEASE_SHA = env.PRESTO_BUILD_VERSION.substring(env.PRESTO_BUILD_VERSION.lastIndexOf('-') + 1);
                                }
                            }
                        }
                        sh 'printenv | sort'
                        echo "${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/presto-server-${PRESTO_RELEASE_VERSION}.tar.gz"
                        echo "${AWS_ECR}/oss-presto/presto:${DOCKER_IMAGE_TAG}"
                    }
                }

                stage ('Update Presto Version') {
                    steps {
                        dir('presto') {
                            withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIAL_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh '''
                                    mvn release:branch --batch-mode  \
                                        -DautoVersionSubmodules=true \
                                        -DbranchName=release-${PRESTO_RELEASE_VERSION} \
                                        -DgenerateBackupPoms=false
                                    git checkout -b master-version-update
                                    git --no-pager log --since="40 days ago" --graph --pretty=format:'%C(auto)%h%d%Creset %C(cyan)(%cd)%Creset %C(green)%cn <%ce>%Creset %s'
                                    git branch

                                    echo "create PR..."
                                    # echo "${GITHUB_IMPLYDATA_TOKEN}" > token && gh auth login -h github.com --with-token < token && rm token
                                    # gh pr create --base master --fill --no-maintainer-edit --reviewer wanglinsong --head master-version-update
                                '''
                            }
                        }
                    }
                }

                stage ('Generate Release Notes') {
                    steps {
                        dir('presto') {
                            withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIAL_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh '''
                                    echo 'TBD'
                                    #src/release/release-notes.sh $GIT_USERNAME $GIT_PASSWORD
                                '''
                            }
                        }
                    }
                }

                stage ('Push Release Branch') {
                    steps {
                        dir('presto') {
                            withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIAL_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh '''
                                    ORIGIN="https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/prestodb/presto.git"
                                    git checkout release-${PRESTO_RELEASE_VERSION}
                                    mvn versions:set -DnewVersion="${PRESTO_RELEASE_VERSION}.1-SNAPSHOT"
                                    git add .
                                    git commit -m "Update release branch development version to ${PRESTO_RELEASE_VERSION}.1-SNAPSHOT"
                                    # git push --set-upstream "${ORIGIN}" release-0.279
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage ('Release') {
            agent {
                kubernetes {
                    defaultContainer 'dind'
                    yamlFile 'agent-dind.yaml'
                }
            }

            stages {
                stage ('Load Scripts') {
                    steps {
                        checkout $class: 'GitSCM',
                                branches: [[name: '*/master']],
                                doGenerateSubmoduleConfigurations: false,
                                extensions: [[
                                    $class: 'RelativeTargetDirectory',
                                    relativeTargetDir: 'presto-release-tools'
                                ],[
                                    $class: 'CloneOption',
                                    shallow: true,
                                    noTags:  true,
                                    depth:   1,
                                    timeout: 10
                                ]],
                                submoduleCfg: [],
                                userRemoteConfigs: [[
                                    credentialsId: "${GITHUB_CREDENTIAL_ID}",
                                    url: 'https://github.com/prestodb/presto-release-tools'
                                ]]
                    }
                }

                stage('Setup') {
                    steps {
                        sh 'apk add aws-cli bash'
                    }
                }

                stage ('Rlease Artifacts') {
                    parallel {
                        stage('Release to Maven Central') {
                            steps {
                                echo 'release all jars to Maven Central'
                                echo 'TBD'
                            }
                        }

                        stage ('Release Tarballs') {
                            steps {
                                withCredentials([[
                                        $class: 'AmazonWebServicesCredentialsBinding',
                                        credentialsId: "${AWS_CREDENTIAL_ID}",
                                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                                    script {
                                        sh '''
                                            aws s3 cp "${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/" "s3://oss-presto-release/presto/${PRESTO_RELEASE_VERSION}/" --recursive --no-progress
                                        '''
                                        echo "${S3_URL_BASE}/${PRESTO_RELEASE_VERSION}/presto-server-${PRESTO_RELEASE_VERSION}.tar.gz"
                                        echo "${S3_URL_BASE}/${PRESTO_RELEASE_VERSION}/presto-cli-${PRESTO_RELEASE_VERSION}-executable.jar"
                                    }
                                }
                            }
                        }

                        stage ('Release Docker Images') {
                            steps {
                                withCredentials([[
                                        $class: 'AmazonWebServicesCredentialsBinding',
                                        credentialsId: "${AWS_CREDENTIAL_ID}",
                                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                                    script {
                                        sh '''
                                            aws ecr get-login-password | docker login --username AWS --password-stdin ${AWS_ECR}
                                            docker pull "${AWS_ECR}/oss-presto/presto:${DOCKER_IMAGE_TAG}"
                                            docker tag "${AWS_ECR}/oss-presto/presto:${DOCKER_IMAGE_TAG}" "${AWS_ECR_PUBLIC}/presto:${PRESTO_RELEASE_VERSION}"
                                            # docker pull "${AWS_ECR}/oss-presto/presto-native:${DOCKER_IMAGE_TAG}"
                                            # docker tag "${AWS_ECR}/oss-presto/presto-native:${DOCKER_IMAGE_TAG}" "${AWS_ECR_PUBLIC}/presto-native:${PRESTO_RELEASE_VERSION}"
                                            docker image ls

                                            aws ecr-public get-login-password | docker login --username AWS --password-stdin ${AWS_ECR_PUBLIC}
                                            docker push "${AWS_ECR_PUBLIC}/presto:${PRESTO_RELEASE_VERSION}"
                                            # docker push "${AWS_ECR_PUBLIC}/presto-native:${PRESTO_RELEASE_VERSION}"
                                        '''
                                    }
                                }
                            }
                        }
                    }
                }                
            }
        }
    }
}
