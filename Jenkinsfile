pipeline {
    agent any
    
    triggers {
        cron('H/30 * * * *')
    }
    
    environment {
        UPSTREAM_REPO = 'https://github.com/microsoft/mimalloc.git'
        FORK_REPO = 'https://github.com/Skullians/mimalloc.git'
        TARGET_BRANCH = 'main'
    }
    
    stages {
        stage('Checkout Fork') {
            steps {
                git branch: "${TARGET_BRANCH}",
                    url: "${FORK_REPO}",
                    credentialsId: 'malloc'
            }
        }
        
        stage('Add Upstream Remote') {
            steps {
                sh '''
                    git remote remove upstream || true
                    git remote add upstream ${UPSTREAM_REPO}
                    git fetch upstream
                '''
            }
        }
        
        stage('Check for Updates') {
            steps {
                script {
                    def upstreamCommit = sh(
                        script: 'git rev-parse upstream/${TARGET_BRANCH}',
                        returnStdout: true
                    ).trim()
                    
                    def currentCommit = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                    
                    env.NEEDS_UPDATE = (upstreamCommit != currentCommit).toString()
                    echo "Needs update: ${env.NEEDS_UPDATE}"
                    echo "Upstream ${TARGET_BRANCH}: ${upstreamCommit}"
                    echo "Current HEAD: ${currentCommit}"
                }
            }
        }
        
        stage('Merge Upstream') {
            when {
                environment name: 'NEEDS_UPDATE', value: 'true'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'malloc', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh '''
                        git merge upstream/${TARGET_BRANCH} --no-edit
                        
                        git config user.name "${GIT_USERNAME}"
                        git config user.email "${GIT_USERNAME}@users.noreply.github.com"
                        
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Skullians/jemalloc.git ${TARGET_BRANCH}
                    '''
                }
            }
        }
        
        stage('Build') {
            when {
                environment name: 'NEEDS_UPDATE', value: 'true'
            }
            steps {
                sh '''
                    mkdir build
                    cd build
                    cmake ../
                    make
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                cd lib
                ln -sf libmimalloc.so.2.2 libmimalloc.so || true
            '''
            archiveArtifacts artifacts: 'lib/libmimalloc.so', allowEmptyArchive: true
        }
        success {
            script {
                if (env.NEEDS_UPDATE == 'true') {
                    echo 'Fork updated and build completed successfully'
                }
            }
        }
    }
}