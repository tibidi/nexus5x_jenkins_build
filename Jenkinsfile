void signPackage () {
 //./build/make/tools/releasetools/sign_target_files_apks -o --default_key_mappings vendor/secure/ out/target/product/bullhead/obj/PACKAGING/target_files_intermediates/aosp_bullhead-target_files-eng.root.zip signed-target_files.zip
 //./build/make/tools/releasetools/ota_from_target_files signed-target_files.zip signed-ota_update.zip

}

void sendSms (String message) {
      sh "/opt/jenkins/misc/sendSms.sh \"$message\""
}

void resetSourceTree() {
  echo 'Reseting source tree...'
  dir(env.SOURCE_DIR) {
    def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash            
                if [ -d ".repo" ]
                then            
                          repo diff > repo.${BUILD_NUMBER}.diff
                fi
                    ''')      
    sh '''#!/usr/bin/env bash
    repo forall -c "git reset --hard"'''
  }
}

int cleanUp() {
  echo 'Cleaning up environment...'
  dir(env.SOURCE_DIR) {
        def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                # Load build environment
                . build/envsetup.sh
                lunch aosp_bullhead-$BUILD_TYPE
                make clean
                make clobber
        rm -rf ./out
        mkdir -p ./out
        ''')
  }
}

int basicCleanUp() {
    if (env.BASIC_CLEANUP == 'true') {
          echo 'Minimal cleaning up environment...'
          dir(env.SOURCE_DIR) {
            def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                    cd out
                    rm -f $(find . -name "*.log.*.gz")
                    rm -f $(find . -name "soong*.log")
                    rm -f $(find . -name "error*.log")
                    rm -f $(find . -name "build*.trace*.gz")
                    rm -f $(find . -name "PixelExperience*")
                    rm -rf target/        
            ''')
          }
    }
}


int cleanUpBuildOutput() {
  echo 'Cleaning up build result...'
  dir(env.SOURCE_DIR) {
        def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                rm -rf ./out
        mkdir ./out
        ''')
  }
}

node('builder') {
    try {
    
        stage('Preparation') {
            echo 'Setting up environment... '
            env.DEVICE='bullhead'        
            if ( ! env.ANDROID_VER ) {        
                env.ANDROID_VER='pie'
        }
            if ( ! env.WORKDIR ) {
                env.WORKDIR= env.WORKSPACE + '/build/'
            }
            if ( ! env.MANIFEST ) {
                env.MANIFEST= 'https://github.com/PixelExperience/manifest'
            }        
            currentBuild.description = env.BRANCH+'_'+env.DEVICE+'-'+env.BUILD_TYPE
            env.SOURCE_DIR=env.WORKDIR + env.ANDROID_VER + '/pixel'
            env.ARCHIVE_DIR = env.SOURCE_DIR + '/archive'
            env.ANDROID_JACK_VM_ARGS='-Xmx8g -Dfile.encoding=UTF-8 -XX:+TieredCompilation'
            env.LC_ALL='C'
            
            // Number of available cores to build
            echo 'Getting the number of available cores...'
            env.JOBS = sh (returnStdout: true, script: '''#!/usr/bin/env bash
            expr 0 + $(grep -c ^processor /proc/cpuinfo)
            ''').trim()
            echo 'Cores available: ' + env.JOBS
        
            env.USE_CCACHE=1
            env.CCACHE_DIR="${env.SOURCE_DIR}/.ccache"
            env.CCACHE_NLEVELS=4
        
            if ( ! env.CCACHE_CUSTOM_SIZE ) {
                env.CCACHE_CUSTOM_SIZE=20G
            }        
        
            if (env.CLEAN_BEFORE == 'true') {
                cleanUp()
            }
        
            if (env.RESET_SOURCE_TREE == 'true') {
                resetSourceTree()
            }
        
            if ( env.CLEAN_OUT_BEFORE && env.CLEAN_OUT_BEFORE == 'true' ) {
            echo 'Clean out folder ...'
                    cleanUpBuildOutput()
            }        
        
        }
        
        stage('Code syncing') {
          dir(env.SOURCE_DIR) {
            if (env.SYNC == 'true' ) {
            def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash            
                if [ -d ".repo" ]                
                then         
                  mkdir -p ../diff
                          repo diff > ../diff/repo.${BUILD_NUMBER}.diff
                fi
                         rm -f .repo/local_manifests/*                
                    ''')

                checkout poll: false, changelog: true, scm: [$class: 'RepoScm', currentBranch: true, destinationDir: env.SOURCE_DIR, forceSync: true, jobs: env.JOBS, manifestBranch: env.BRANCH,
                    manifestRepositoryUrl: env.MANIFEST, noTags: true, quiet: true
            ,localManifest: env.NEXUS_MANIFEST
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
        
        export TARGET_KERNEL_CONFIG=$(echo $NEXUS_MANIFEST | sed "s/.*_//" | sed "s/.xml/_defconfig/")

                rm -rf ./out/target/product/bullhead/obj/PACKAGING/target_files_intermediates/*
                rm -f ./out/target/product/bullhead/PixelExperience_*bullhead-*.zip

                export BUILD_NUMBER_JENKINS=$BUILD_NUMBER
                unset BUILD_NUMBER

                mkdir -p $ARCHIVE_DIR
                rm -rf $ARCHIVE_DIR/*

        if [ -f prebuilts/misc/linux-x86/ccache/ccache ]
        then        
                   prebuilts/misc/linux-x86/ccache/ccache -M $CCACHE_CUSTOM_SIZE
        else
           ccache -M $CCACHE_CUSTOM_SIZE
         fi

                # Load build environment
                . build/envsetup.sh
                lunch aosp_bullhead-$BUILD_TYPE

                if [ $? -ne 0 ]
                then
                    echo "Build failed."
                    exit 2
                fi

                if [ ! -z "$BUILD_JOB" ] 
        then
          mka bacon -j$BUILD_JOB
        else
                  mka bacon -j$JOBS
        fi

                if [ $? -ne 0 ]
                then
                    echo "Build failed."
                    exit 2
                fi

                export BUILD_NUMBER=$BUILD_NUMBER_JENKINS

                mv out/target/product/bullhead/PixelExperience_*bullhead-*zip $ARCHIVE_DIR
                cp out/target/product/bullhead/obj/PACKAGING/target_files_intermediates/aosp_bullhead-target_files-eng.root/SYSTEM/build.prop $ARCHIVE_DIR
        cp out/target/product/bullhead/system/build.prop $ARCHIVE_DIR
        
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
                    unzip PixelExperience_*bullhead-* boot.img
                    mv boot.img boot.build.img
                    python /opt/misc/disable_cpu_cores.py boot.build.img boot.4c.img               
                    cp boot.4c.img boot.img
                    zip PixelExperience_*bullhead-* boot.img
                    rm boot.img
                    cd ..
                  ''')
                }
        if (env.SYNC == 'true' ) {
                  def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                    export BUILD_DATE=$(grep "org.pixelexperience.build_date=" $ARCHIVE_DIR/build.prop | sed "s/.*=//")                  
                    tar czf $ARCHIVE_DIR/kernel_$BUILD_DATE.tgz $TARGET_KERNEL_FOLDER
                    tar czf $ARCHIVE_DIR/contexthub_$BUILD_DATE.tgz device/google/contexthub
            tar czf $ARCHIVE_DIR/device_$BUILD_DATE.tgz device/lge/bullhead
            tar czf $ARCHIVE_DIR/vendor_$BUILD_DATE.tgz vendor/lge
            if [ -d "hardware/qcom/audio/default" ]
            then
              tar czf $ARCHIVE_DIR/audio_$BUILD_DATE.tgz hardware/qcom/audio/default
            else
              tar czf $ARCHIVE_DIR/audio_$BUILD_DATE.tgz hardware/qcom/audio
            fi
                  ''')                
                }                
            }
        }
        
        stage('Archiving') {
            echo 'Archiving ...'
            dir(env.ARCHIVE_DIR) {
        if (env.SYNC == 'true' ) {
             sh "cp ../../diff/repo.${BUILD_NUMBER}.diff repo.diff"
        }
                archiveArtifacts allowEmptyArchive: true, artifacts: '**', excludes: '*target*', fingerprint: true, onlyIfSuccessful: true
            }
        }
        
        stage('Publishing') {
            echo 'Publishing ...'
            dir(env.ARCHIVE_DIR) {
              if (env.PUBLISH == 'true') {
                def rc = sh (returnStatus: true, script: '''#!/usr/bin/env bash
                      export BUILD_DATE=$(grep "org.pixelexperience.build_date=" build.prop | sed "s/.*=//")
                      export TARGET_KERNEL_CONFIG=$(echo $NEXUS_MANIFEST | sed "s/.*_//" | sed "s/.xml//")
                      //export folderId=$(drive folder -t ${AGENT_HOST}_${BRANCH}_${TARGET_KERNEL_CONFIG}_${BUILD_DATE} --parent $DRIVE_FOLDER | grep Id | sed "s/.* //")
                      cp boot.build.img boot.build.$BUILD_DATE.img  
                      drive add_remote --file boot.build.$BUILD_DATE.img --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o
                      drive add_remote --file PixelExperience_*bullhead-*.zip --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o
                      drive add_remote --file kernel_$BUILD_DATE.tgz --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o  
                      drive add_remote --file contexthub_$BUILD_DATE.tgz --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o    
                      drive add_remote --file device_$BUILD_DATE.tgz  --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o
                      drive add_remote --file vendor_$BUILD_DATE.tgz --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o  
                      drive add_remote --file audio_$BUILD_DATE.tgz --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o
                      drive add_remote --file repo.diff --pid 10Cu1qT__R4BrmBaEGvdtVD0m2dgXM56o
                ''')
              }
            }            
        }
        
        if (env.CLEAN_AFER == 'true') {
            cleanUp()
    } else {
        basicCleanUp()
    }
        
        currentBuild.result = 'SUCCESS'
        //sendSms ("Fabrication OK : '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description} ${env.AGENT_HOST}]'")        
        sendSms ("Fabrication Build OK : ${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description} ${env.AGENT_HOST}] ${env.PRIV_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}")
    if (env.SLACK_SEND == 'true')        
        slackSend (color: 'good', message: "${env.SLACK_NAME} Jenkins Builder - Job SUCCESS: '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description}]' (${env.BUILD_URL})")        
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        //sendSms ("Fabrication KO : '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description} ${env.AGENT_HOST}]'")        
        sendSms ("Fabrication Build KO : ${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description} ${env.AGENT_HOST}] ${env.PRIV_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}")
        if (env.SLACK_SEND == 'true')
        slackSend (color: 'danger', message: "${env.SLACK_NAME} Jenkins Builder - Job FAILED: '${env.JOB_NAME} [${env.BUILD_NUMBER} - ${currentBuild.description}]' (${env.BUILD_URL})")                
    }
    if (! env.CLEAN_OUT || env.CLEAN_OUT == 'true' ) {
        cleanUpBuildOutput()
    }    
    echo "RESULT: ${currentBuild.result}"    
}
