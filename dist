#!/usr/bin/env groovy

pipeline {
    options {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
    }
    agent {
        label 'build-linux-x64'
    }
    parameters {
	    string(name: 'ULTIMA_MAJOR_VERSION', defaultValue: '0')
	    string(name: 'ULTIMA_MINOR_VERSION', defaultValue: '0')
	    string(name: 'ULTIMA_ROOT_REVISION', defaultValue: '0')
	    string(name: 'GITHASH_ULTIMA_DIST', defaultValue: 'master')   
        string(name: 'GITHASH_ULTIMA_NGINX', defaultValue: 'master')
    }
    environment {
        TARGET="vmc"
        RESULT_MODEL="NeyrossPlatform"
        RESULT_VERSION="${ULTIMA_MAJOR_VERSION}.${ULTIMA_MINOR_VERSION}.${ULTIMA_ROOT_REVISION}"
        RESULT_FILENAME="${RESULT_MODEL}-${RESULT_VERSION}.sh"
        storage_root="releases/ultima-${TARGET}/last-build"
        storage_dir="${storage_root}/ultima-${TARGET}-${ULTIMA_MAJOR_VERSION}.${ULTIMA_MINOR_VERSION}"
        ftp_host="10.1.31.1"
        ftp_username="robot"
        ftp_password="5CacgPzIx"
        ftp_begin="set ftp:passive-mode off"
    }
    stages {
        stage('Cloning ultima-dist, ultima-nginx repositories') {
            steps {
                dir ('git/ultima-dist') {
                    git(url: 'git@git.itrium-spb.ru:ultima/ultima-dist.git', credentialsId: '19a6d233-4e91-4567-a67e-5a1211786063')
                    sh 'git checkout ${GITHASH_ULTIMA_DIST}'
                }
                dir ('git/ultima-nginx') {
                    git(url: 'git@git.itrium-spb.ru:ultima/ultima-nginx.git', credentialsId: '19a6d233-4e91-4567-a67e-5a1211786063')
                    sh 'git checkout ${GITHASH_ULTIMA_NGINX}'
                }
            }
        }
        stage('Cleaning obsolete debs and nginx dir and debug dir') {
            steps {
                sh 'rm -rf target && mkdir -p target/debs target/nginx'
                sh 'rm -rf debug && mkdir -p debug'
            }
        }
        stage('Copying nginx config and add making tar-archive') {
            steps {
                sh """
                mkdir -p ultima-nginx
                mkdir -p ultima-nginx/bin
                cp -rf git/ultima-nginx/default ultima-nginx
                cp -rf git/ultima-nginx/systemd ultima-nginx
                cp -rf git/ultima-nginx/bin/config-nginx.sh ultima-nginx/bin
                if [ -f git/ultima-nginx/bin/keygen.sh ]; then
                cp -rf git/ultima-nginx/bin/keygen.sh ultima-nginx/bin
                fi
                tar -czf target/nginx/ultima-nginx.tar.gz ultima-nginx
                rm -rf ultima-nginx
                """
            }
        }
        stage ('Copying artifacts from ultima-play') {
            steps {
                copyArtifacts filter: '**/*.deb', flatten: true, projectName: 'ultima-play', selector: lastSuccessful(), target: 'target/debs'
            }
        }
        stage ('Copying artifacts from ultima-media-server (.deb)') {
            steps {
                copyArtifacts filter: '**/*.deb', flatten: true, projectName: 'ultima-media-server', selector: lastSuccessful(), target: 'target/debs'
            }
        }
        stage ('Copying artifacts from neyross-embed (.deb)') {
            steps {
                copyArtifacts filter: '**/*.deb', flatten: true, projectName: 'neyross-embed', selector: lastSuccessful(), target: 'target/debs'
            }
        }
        stage ('Copying artifacts from ultima-webapp') {
            steps {
                copyArtifacts filter: '**/*.deb', flatten: true, projectName: 'ultima-webapp', selector: lastSuccessful(), target: 'target/debs'
            }
        }
        stage ('Copying artifacts from ultima-nginx') {
            steps {
                copyArtifacts filter: '**/*.deb', flatten: true, fingerprintArtifacts: true, projectName: 'ultima-nginx', selector: lastSuccessful(), target: 'target/debs'
            }
        }
        stage ('Copying artifacts from ultima-plugins/argus-rrhid') {
            steps {
                copyArtifacts filter: '**/*.upf', flatten: true, fingerprintArtifacts: true, projectName: 'ultima-plugins/argus-rrhid', selector: lastSuccessful(), target: 'target/upfs'
            }
        }
        stage ('Copying artifacts from neyross-embed (debug)') {
            steps {
                copyArtifacts filter: '**/*.debug', flatten: true, fingerprintArtifacts: true, projectName: 'neyross-embed', selector: lastSuccessful(), target: 'debug'
            }
        }
        stage ('Copying artifacts from ultima-media-server (debug)') {
            steps {
                copyArtifacts filter: '**/*.debug', flatten: true, fingerprintArtifacts: true, projectName: 'ultima-media-server', selector: lastSuccessful(), target: 'debug'
            }
        }
        stage('Putting files to $storage_dir') {
            steps {
                sh 'lftp -u ${ftp_username}:${ftp_password} -e "${ftp_begin};rm -r ${storage_root};mkdir -p ${storage_dir};cd ${storage_dir};mput debug/*;mput target/debs/*;mput target/nginx/*;mput target/upfs/*;quit" ${ftp_host}'
            }
        }
        stage('Copying dependencies to target directory') {
            steps {
                sh """
                    cp git/ultima-dist/dependencies/libjpeg8_8d-1+deb7u1_amd64.deb target
                    cp git/ultima-dist/dependencies/libgtk2.0-common_2.24.32-1ubuntu1_all.deb target
                    cp git/ultima-dist/dependencies/libgtk2.0-0_2.24.32-1ubuntu1_amd64.deb target
                """
            }
        }
        stage('Making NeyrossPlatform-*.sh script') {
            steps {
                sh """
                    cd target
                    tar -czf ultima-${TARGET}.tar.gz *
                    cp ../git/ultima-dist/${TARGET}/install.sh .
                    cat install.sh ultima-${TARGET}.tar.gz > ${RESULT_FILENAME}
                    chmod +x ${RESULT_FILENAME}
                    rm -Rf ultima-${TARGET}.tar.gz
                """
            }
        }
        stage('Copying script to ftp-server') {
            steps {
                sh 'lftp -u $ftp_username:$ftp_password -e "$ftp_begin;cd $storage_dir;put target/${RESULT_FILENAME};quit" $ftp_host'
            }
        }
        stage('Archiving the artifacts') {
            steps {
                archiveArtifacts artifacts: 'target/*.sh', followSymlinks: false
            }
        }
    }
}
