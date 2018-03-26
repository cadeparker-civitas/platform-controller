#!/usr/bin/env groovy

/*
 * Copyright (c) 2017, Civitas Learning Incorporated.
 * All Rights Reserved.
 *
 * This software is protected by U.S. Copyright Law and International Treaties.
 * Unauthorized use, duplication, reverse engineering, any form of redistribution,
 * or use in part or in whole other than by prior, express, printed and signed license
 * for use is subject to civil and criminal prosecution.
 */

properties([disableConcurrentBuilds()])

pipeline {
  parameters {
    string(name: 'gradleLogging', defaultValue: '--info --stacktrace',
        description: 'Gradle logging flags?')
    string(name: 'releaseScope', defaultValue: 'minor',
        description: 'What type of version bump (patch, minor, or major)?')
    string(name: 'releaseStage', defaultValue: 'SNAPSHOT',
        description: 'What release stage (SNAPSHOT, alpha, beta, rc, final)?')
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    skipDefaultCheckout()
  }

  environment {
    // GitHub credentials for GR Git so release tagging can be done
    GRGIT_CREDENTIALS = credentials('cc8eb3ff-7556-454b-8541-f7489d8d31c7')
    GRGIT_USER = "${GRGIT_CREDENTIALS_USR}"
    GRGIT_PASS = "${GRGIT_CREDENTIALS_PSW}"
    // Nexus credentials for publishing
    CIVITAS_NEXUS_CREDENTIALS = credentials('647767a4-4803-4f3c-93a9-9c9850c3f8cc')
    CIVITAS_NEXUS_USERNAME = "${CIVITAS_NEXUS_CREDENTIALS_USR}"
    CIVITAS_NEXUS_PASSWORD = "${CIVITAS_NEXUS_CREDENTIALS_PSW}"
    // Shared gradle arguments
    SHARED_GRADLE_ARGUMENTS =
        "${params.gradleLogging} -Prelease.scope=${params.releaseScope} -Prelease.stage=${params.releaseStage}"
  }

  agent any

  stages {
    stage('Checkout Code') {
      steps {
        // Since the default checkout does not allow cleaning, do it manually here with the CleanBeforeCheckout
        //   and CleanCheckout extensions enabled.  Note that the scm.* methods, used to copy existing SCM info,
        //   required explicit whitelisting in [jenkinsServerBaseUrl]/scriptApproval/.
        // This issue tracks adding the "clean" extensions to a more standard "checkout scm" block:
        //   https://issues.jenkins-ci.org/browse/JENKINS-37658
        checkout([
            $class                           : 'GitSCM',
            branches                         : scm.branches,
            doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
            extensions                       : scm.extensions +
                [[$class: 'CleanBeforeCheckout'], [$class: 'CleanCheckout'],
                 [$class: 'CloneOption', noTags: false, shallow: false, reference: '']],
            submoduleCfg                     : scm.submoduleCfg,
            userRemoteConfigs                : scm.userRemoteConfigs,
        ])
      }
    }

    stage('Build') {
      steps {
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} -x test -x integrationTest build"
      }
      post {
        always {
          // Archive checkstyle and findbugs reports for later
          archiveArtifacts '**/build/reports/checkstyle/**/*.xml,**/build/reports/findbugs/**/*.xml'
        }
      }
    }

    stage('Test') {
      steps {
        // The 'build' step needs to compile the test classes for checkstyle/findbugs/etc.
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} -x classes -x testClasses test"
      }
      post {
        always {
          junit allowEmptyResults: true, keepLongStdio: true, testResults: '**/build/test-results/**/*.xml'
        }
      }
    }

    stage('Integration Test') {
      steps {
        // The 'build' step needs to compile the test classes for checkstyle/findbugs/etc.
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} -x classes -x testClasses integrationTest"
      }
      post {
        always {
          junit allowEmptyResults: true, keepLongStdio: true, testResults: '**/build/test-results/**/*.xml'
        }
      }
    }

    stage('Build Docker') {
      steps {
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} -x build dockerBuildImage"
      }
    }

    // Tags Git repo and pushes to origin
    stage('Release') {
      steps {
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} release"
      }
    }

    stage('Publish Nexus & ECR (Snapshot)') {
      when {
        expression {
          'SNAPSHOT' == params.releaseStage
        }
      }
      steps {
        sh "./gradlew ${SHARED_GRADLE_ARGUMENTS} -x jar -x classes publish"
      }
    }

    stage('Publish Nexus & ECR (Release)') {
      when {
        expression {
          'SNAPSHOT' != params.releaseStage
        }
      }
      steps {
        // In order to get the same version as we just tagged, use the 'rebuild' release stage
        //   to publish by not passing any release.* properties.
        sh "./gradlew ${params.gradleLogging} -x jar -x classes publish"
      }
    }
    
    stage('Notify Logstash') {
      steps {
        logstashSend failBuild: true, maxLines: 100
      }
    }

    stage('Publish Code Coverage') {
      steps {
        step([$class: 'JacocoPublisher',
          execPattern:'**/**.exec',
          classPattern: '**/classes',
          sourcePattern: '**/src/main/java'])
      }
    }
  }
}
