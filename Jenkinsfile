pipeline {
    agent any

    stages {
        stage('Collect') {
            steps {
                stash name: 'flac-source', includes: '**/*'
                git url: 'https://gitlab.xiph.org/steils/flac.git', branch: 'master'
            }
        }

        stage('Build-setup') {
            steps {
                unstash 'flac-source'
                sh '''
                    docker build -t flac:builder -f Dockerfile.builder .
                '''
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'flac:builder'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                unstash 'flac-source'
                sh '''
                    ./autogen.sh
                    ./configure
                    make
                '''
                stash name: 'flac-build'
            }
        }

        stage('Test-setup') {
            steps {
                unstash 'flac-source'
                sh '''
                    docker build -t flac:tester -f Dockerfile.tester .
                '''
            }
        }
        
        stage('Test') {
            agent {
                docker {
                    image 'flac:tester'
                }
            }
            steps {
                unstash 'flac-build'
                sh '''
                    # Usuwanie starych logów
                    rm -f test/*.log
                    
                    # Testy
                    make check
                '''
            }
            post {
                always {
                    script {
                        def buildNum = env.BUILD_NUMBER
                        // Zmiana nazw na numerowane buildem
                        sh """
                            for log in test/*.log; do
                                if [ -f "\$log" ]; then
                                    case "\$log" in
                                        *_build_*) ;;  # skip already renamed logs
                                        *)
                                            base=\$(basename "\$log" .log)
                                            mv "\$log" "test/\${base}_build_${buildNum}.log"
                                            ;;
                                    esac
                                fi
                            done
                        """
                        // Archiwizacja
                        archiveArtifacts artifacts: "test/*_build_${buildNum}.log", allowEmptyArchive: true
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'flac:builder'
                    args '-u root'
                }
            }
            steps {
                unstash 'flac-build'
                sh """
                    make install
                    ldconfig
                    flac --version
                """
            }
        }
        
        stage('Publish') {
            steps {
                unstash 'flac-build'
                archiveArtifacts artifacts: 'src/libFLAC/.libs/libFLAC.so.*.*.*, src/libFLAC++/.libs/libFLAC++.so.*.*.*, src/flac/flac, src/metaflac/metaflac',
                                   fingerprint: true
            }
            post {
                success {
                    echo 'Biblioteki FLAC zapisane'
                }
                failure {
                    echo 'Nie udało się zapisać bibliotek FLAC'
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
