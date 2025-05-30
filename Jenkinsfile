pipeline {
    agent any

    environment {
        SAFE_CLI_URL = 'https://safe-releases.s3.eu-central-1.amazonaws.com/safe_cli-1.0.1.tar.gz'
        SAFE_CLI_TAR = 'safe_cli-1.0.1.tar.gz'
        SAFE_LICENSE = credentials('SAFE_LICENSE')
        SAFE_CI_CONFIG_PATH = "${WORKSPACE}/.safe/config.json"
        REBAR3_URL = 'https://s3.amazonaws.com/rebar3/rebar3'
        ERL_LIBS = '/var/jenkins_home/safe/lib'
    }

    stages {
        stage('Install Rebar3') {
            steps {
                echo 'Installing rebar3...'
                sh '''
                    curl -L -o rebar3 $REBAR3_URL
                    chmod +x rebar3
                    ./rebar3 local install
                    export PATH=/root/.cache/rebar3/bin:$PATH
                '''
            }
        }

        stage('Pre-clean up') {
            steps {
                echo 'Cleaning up previous builds...'
                sh '''
                    rm -rf _build
                    rm -rf safe
                    rm -rf /var/jenkins_home/.cache/SAFE/data_Gradualizer
                '''
            }
        }

        stage('Build with Rebar3') {
            steps {
                echo 'Compiling project with Rebar3...'
                sh './rebar3 compile'
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                echo 'Archiving build artifacts...'
                stash includes: '_build/**', name: 'build-artifact'
            }
        }

        stage('SAFE Checks') {
            parallel {
                stage('SAFE Security Check') {
                    steps {
                        echo 'Running SAFE CLI for security scanning...'
                        sh '''
                        curl -L -o $SAFE_CLI_TAR $SAFE_CLI_URL
                        mkdir -p safe
                        tar -xzf $SAFE_CLI_TAR -C safe
                        chmod +x safe/bin/safe_cli
                        ./safe/bin/safe_cli start
                        '''
                    }
                    post {
                        always {
                            echo 'Uploading SAFE report regardless of scan result...'
                            sh '''
                        curl -X POST -H "Content-Type: application/json" -d @_results/Gradualizer.safe https://safe-on-prem.fly.dev/api/reports
                    '''
                        }
                    }
                }

                stage('SAFE Dependency Check') {
                    steps {
                        echo 'Running dependency check script...'
                        sh '''
                        echo "Downloading audit script from S3..."
                        curl -L -o audit.sh https://audit-script-safe.s3.us-east-1.amazonaws.com/audit.sh
    
                        if [ $? -eq 0 ] && [ -f "./audit.sh" ]; then
                            chmod +x ./audit.sh
                            echo "Running dependency check..."
                            ./audit.sh
                        else
                            echo "ERROR: Failed to download audit.sh from S3 bucket"
                            exit 1
                        fi
                        '''
                    }
                    post {
                        always {
                            echo 'Dependency check stage completed.'
                            // Clean up downloaded script
                            sh 'rm -f ./audit.sh'
                        }
                        failure {
                            echo 'Dependency check failed - check console output for details'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Build and tests succeeded!'
        }
        failure {
            echo 'Something went wrong...'
        }
    }
}
