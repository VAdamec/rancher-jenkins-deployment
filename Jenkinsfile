#!/usr/bin/env groovy

node ('rancher'){
    def secrets = [
        [
            $class: 'VaultSecret', path: 'secret/rancher', secretValues: [
                [$class: 'VaultSecretValue', envVar: 'BETA_ACCESS_KEY', vaultKey: 'BETAAKEY'],
                [$class: 'VaultSecretValue', envVar: 'BETA_SECRET_KEY', vaultKey: 'BETASKEY'],
                [$class: 'VaultSecretValue', envVar: 'PROD_ACCESS_KEY', vaultKey: 'PRODAKEY'],
                [$class: 'VaultSecretValue', envVar: 'PROD_SECRET_KEY', vaultKey: 'PRODSKEY']
            ]
        ],
        [
            $class: 'VaultSecret', path: 'secret/sampleapp', secretValues: [
                [$class: 'VaultSecretValue', envVar: 'DB_PASSWORD', vaultKey: 'SAMPLE_WORDPRESS_DB_PASSWORD'],
                [$class: 'VaultSecretValue', envVar: 'DB_USER', vaultKey: 'SAMPLE_MYSQL_ROOT_PASSWORD'],
            ]
        ]
    ]

    def components = [
        [
            name: "wordpress",
            image: "team/sampleapp/wordpress",
            dir: "wordpress"
        ],
        [
            name: "wordpress-data",
            image: "team/sampleapp/wordpress-data",
            dir: "wordpress-data"
        ],
        [
            name: "filebeat",
            image: "team/sampleapp/filebeat",
            dir: "filebeat"
        ]
    ]

    def beta_env = [
        domain: "sample-beta.example.com",
        scale: "1",
        rancher_project: "1a1065",
        volume: "/gluster/gfs_replica/sampleapp",
        version: "${env.BUILD_NUMBER}",
        tag: "latest",
        access_key: "${env.BETA_ACCESS_KEY}",
        secret_key: "${env.BETA_SECRET_KEY}",
        db_password: "${env.DB_PASSWORD}",
        db_user: "${env.DB_USER}",
    ]

    def production_env = [
        domain: "sample.example.com.com",
        scale: "1",
        rancher_project: "1a5",
        volume: "/gluster/gfs_replica/sampleapp",
        version: "${env.BUILD_NUMBER}",
        tag: "stable",
        access_key: "${env.PROD_ACCESS_KEY}",
        secret_key: "${env.PROD_SECRET_KEY}",
        db_password: "${env.DB_PASSWORD}",
        db_user: "${env.DB_USER}",
    ]

    def environments = [
        beta: beta_env,
        production: production_env

    ]

    wrap([$class: 'VaultBuildWrapper', vaultSecrets: secrets]) {
        environments.beta.access_key = "${BETA_ACCESS_KEY}"
        environments.beta.secret_key = "${BETA_SECRET_KEY}"
        environments.beta.db_user = "${DB_USER}"
        environments.beta.db_password = "${DB_PASSWORD}"
        environments.production.access_key = "${PROD_ACCESS_KEY}"
        environments.production.secret_key = "${PROD_SECRET_KEY}"
        environments.production.db_user = "${DB_USER}"
        environments.production.db_password = "${DB_PASSWORD}"
    }

    def getRangeCommandByEnv = { env, command ->
        return "VERSION=${env.version} DOMAIN=${env.domain} SCALE=${env.scale} PERSISTENT_VOLUME=${env.volume} \
                DB_PASSWORD=${env.db_password} DB_USER=${env.db_user} \
                /bin/rancher-compose --project-name sampleapp \
                    --url https://rancher-ui.example.com/v2-beta/projects/${env.rancher_project} \
                    --access-key ${env.access_key} \
                    --secret-key ${env.secret_key} \
                    --verbose up -d " + command
    }

    def runBuild = { what, tags ->
        build job: 'graphite-event', parameters: [
                string(name: 'WHAT', value: "SampleAPP " + what), \
                string(name: 'DATA', value: "Deployed tag ${env.BUILD_NUMBER}" + what), \
                string(name: 'TAGS', value: tags) \
            ], wait: false
    }

    stage('Preparation') {
        checkout scm
        sh "sed -ie 's/build:.*//g' docker-compose.yml"
        sh "rm -f .env"
    }

    stage('Build Docker Images') {
        def stepsForParallel = [:]
        for (int i = 0; i < components.size(); i++) {
            def s = components.get(i)
            def stepName = "echoing ${s}"
            stepsForParallel[stepName] = BuildImagesParallel(s.image,s.dir,environments.beta.version,environments.beta.tag)
        }

        parallel stepsForParallel
    }

    try {
        notifyBuild('STARTED')
        stage('Deploy Beta') {
          timeout(time: 600, unit: 'SECONDS') {
            sh getRangeCommandByEnv(environments.beta, "--confirm-upgrade")
            sh getRangeCommandByEnv(environments.beta, "--force-upgrade --pull")
          }
            runBuild("Beta", "sample,build")
            currentBuild.result = 'SUCCESS'
        }
    } catch (e) {
        currentBuild.result = "FAILED"

        stage('Rollback if beta tests failed') {
            timeout(time: 600, unit: 'SECONDS') {
              sh getRangeCommandByEnv(environments.beta, "--rollback")
            }
            runBuild("Rollback", "sample,rollback")
        }
    } finally {
        // Success or failure, always send notifications
        notifyBuild(currentBuild.result)
    }

    stage('Build production - approve step') {
          input message: 'Waiting for approve', ok: 'Approve'
    }

    try {
        stage('Deploy Production') {
          timeout(time: 600, unit: 'SECONDS') {
            sh getRangeCommandByEnv(environments.production, "--confirm-upgrade")
            sh getRangeCommandByEnv(environments.production, "--force-upgrade")
          }
        }
        runBuild("Prod", "sample,build")
        currentBuild.result = 'SUCCESS'
    } catch (e) {
        currentBuild.result = "FAILED"
        stage('Rollback if prod tests failed') {
            wrap([$class: 'VaultBuildWrapper', vaultSecrets: secrets]) {
              timeout(time: 600, unit: 'SECONDS') {
               sh getRangeCommandByEnv(environments.production, "--rollback")
              }
             runBuild("Rollback", "sample,rollback")
            }
        }
    } finally {
        // Success or failure, always send notifications
        notifyBuild(currentBuild.result)
    }

    stage('Create stable tag') {
        def stepsForParallel = [:]
        for (int i = 0; i < components.size(); i++) {
            def s = components.get(i)
            def stepName = "echoing ${s}"
            stepsForParallel[stepName] = BuildImagesParallel(s.image,s.dir,environments.beta.version,environments.beta.tag)
        }

        parallel stepsForParallel
    }
}

def BuildImagesParallel(image,path,version,tag) {
  return {
    timeout(time: 600, unit: 'SECONDS') {
      docker.withRegistry('https://myregistry.example.com:5000', 'myregistry.example.com5000') {
        dir("${path}") {
          retry(3){
            def app = docker.build("${image}")
            app.push "${version}"
            app.push "${tag}"
          }
        }
      }
    }
  }
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESS'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESS') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  mattermostSend color: colorCode, message: summary
  //slackSend (color: colorCode, message: summary)
  //hipchatSend (color: color, notify: true, message: summary)
  //emailext (subject: subject,body: details,recipientProviders: [[$class: 'DevelopersRecipientProvider']])
}
