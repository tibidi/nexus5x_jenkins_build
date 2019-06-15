int cleanUp() {
  echo 'Cleaning up environment...'
  dir(env.SOURCE_DIR) {
        def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                # Load build environment
                . build/envsetup.sh
                lunch aosp_bullhead-$BUILD_TYPE
                make clean
                make clobber
        ''')
  }
}

node('builder') {
    try {
    
        stage('Preparation') {
            echo 'Setting up environment...'
            env.ANDROID_VER='pie'
            env.DEVICE='bullhead'
            env.WORKDIR= env.WORKSPACE + '/build/'
            currentBuild.description = 'aosp_'+env.DEVICE+'-'+env.BUILD_TYPE
            env.SOURCE_DIR=env.WORKDIR + env.ANDROID_VER + '/pixel'
            env.ARCHIVE_DIR = env.SOURCE_DIR + '/archive'
            env.USE_CCACHE=1
            env.CCACHE_DIR="${env.SOURCE_DIR}/.ccache"
            env.CCACHE_NLEVELS=4
            env.ANDROID_JACK_VM_ARGS='-Xmx8g -Dfile.encoding=UTF-8 -XX:+TieredCompilation'
            env.LC_ALL='C'
            
            // Number of available cores to build
            echo 'Getting the number of available cores...'
            env.JOBS = sh (returnStdout: true, script: '''#!/usr/bin/env bash
            expr 0 + $(grep -c ^processor /proc/cpuinfo)
            ''').trim()
            echo 'Cores available: ' + env.JOBS
            if (env.CLEAN_BEFORE == 'true') {
                cleanUp()
            }
        }
        
        stage('Code syncing') {
          dir(env.SOURCE_DIR) {
            if (env.SYNC == 'true' ) {
                sh 'rm -f .repo/local_manifests/*'

                checkout poll: false, scm: [$class: 'RepoScm', currentBranch: true, destinationDir: env.SOURCE_DIR, forceSync: true, jobs: env.JOBS, manifestBranch: env.BRANCH,
                    manifestRepositoryUrl: 'https://github.com/PixelExperience/manifest', noTags: true, quiet: true,
                    localManifest: 'https://raw.githubusercontent.com/tibidi/nexus5x_jenkins_build/master/local_tibidi.xml'
                ]
            }
          }
        }
        
        stage('Build process') {
            echo 'Building ...'
            if (env.PURGE_CACHE == 'true') {
              dir(env.SOURCE_DIR) {
                sh 'rm -rf .ccache'
                }
            }
            dir(env.SOURCE_DIR) {
                def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash

                rm -rf ./out/target/product/bullhead/obj/PACKAGING/target_files_intermediates/*
                rm -f ./out/target/product/bullhead/PixelExperience_bullhead-*.zip

                export BUILD_NUMBER_JENKINS=$BUILD_NUMBER
                unset BUILD_NUMBER

                mkdir -p $ARCHIVE_DIR
                rm -rf $ARCHIVE_DIR/*

                prebuilts/misc/linux-x86/ccache/ccache -M 20G

                # Load build environment
                . build/envsetup.sh
                lunch aosp_bullhead-$BUILD_TYPE

                if [ $? -ne 0 ]
                then
                    echo "Build failed."
                    exit 2
                fi

                mka bacon -j$JOBS

                if [ $? -ne 0 ]
                then
                    echo "Build failed."
                    exit 2
                fi

                export BUILD_NUMBER=$BUILD_NUMBER_JENKINS

                mv out/target/product/bullhead/PixelExperience_bullhead-*zip $ARCHIVE_DIR
                cp out/target/product/bullhead/obj/PACKAGING/target_files_intermediates/aosp_bullhead-target_files-eng.root/SYSTEM/build.prop $ARCHIVE_DIR
        
                cp out/target/product/bullhead/system/etc/Changelog.txt $ARCHIVE_DIR
        
                cd $ARCHIVE_DIR
                touch $BUILD_TYPE.txt

                ''')
                
                echo "RC $rc"
                if ( rc != 0 )
                    error("Build failed..")
                }
        }
        
        stage('Package') {
            echo 'Pakaging ...'
            dir(env.SOURCE_DIR) {
                if (env.PATCH_BOOT_4C == 'true') {
                  def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                    cd $ARCHIVE_DIR
                    unzip PixelExperience_bullhead-* boot.img
                    mv boot.img boot.build.img
                    python /opt/misc/disable_cpu_cores.py boot.build.img boot.4c.img               
                    cp boot.4c.img boot.img
                    zip PixelExperience_bullhead-* boot.img
                    rm boot.img
                    cd ..
                  ''')
                }
				if (env.SYNC == 'true' ) {
                  def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                    export BUILD_DATE=$(grep "org.pixelexperience.build_date=" $ARCHIVE_DIR/build.prop | sed "s/.*=//")                  
                    tar cvzf kernel_$BUILD_DATE.tgz kernel/lge/bullhead
                    tar cvzf contexthub_$BUILD_DATE.tgz device/google/contexthub
                  ''')				
				}                
            }
        }
        
        stage('Archiving') {
            echo 'Archiving ...'
            dir(env.ARCHIVE_DIR) {
                archiveArtifacts allowEmptyArchive: true, artifacts: '**', excludes: '*target*', fingerprint: true, onlyIfSuccessful: true
            }
        }
        
        stage('Publishing') {
            echo 'Publishing ...'
        }
        
        if (env.CLEAN_AFER == 'true') {
            cleanUp()
        }
        
        currentBuild.result = 'SUCCESS'
        slackSend (color: 'good', message: "Jenkins Builder - Job SUCCESS: '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description}]' (${env.BUILD_URL})")
        
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        slackSend (color: 'danger', message: "Jenkins Builder - Job FAILED: '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description}]' (${env.BUILD_URL})")
    }
    echo "RESULT: ${currentBuild.result}"
}
