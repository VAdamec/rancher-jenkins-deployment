# Build and deploy sample app (wordpress)
* using wordpress, jenkins pipelines and rancher-compose

# Vault
- https://wiki.jenkins-ci.org/display/JENKINS/HashiCorp+Vault+Plugin
* policy '''jenkins''' need to contain read access to '''secret/sampleapp'''
```bash
vault write secret/sampleapp SAMPLE_WORDPRESS_DB_PASSWORD=example SAMPLE_MYSQL_ROOT_PASSWORD=example
```

# Jenkins
- multibranch job + deploy key
- ```git@gitlab.example.com:rancher/sampleapp.git```
- deploy to PROD waiting for manual approve

## BuildImagesParallel
- run docker build in parallel to speed up pipeline

## graphite-event
- simple job which writes events to Graphite

## notifyBuild
- send selected notification, default is  [Mattermost](https://www.mattermost.org/)

# Jenkins slave
* standard Jenkins swarm container with added rancher-compose

## Variables

```bash
${env.BUILD_TAG} - take from Jenkins envvars
domain=${env.DOMAIN} - defined in Jenkinsfile
scale=${env.COLLECTOR_SCALE} - defined in Jenkinsfile (static for Beta, read from actuall setup for PROD)
dbpassword=${SAMPLE_WORDPRESS_DB_PASSWORD} - read from Vault
root_password=${SAMPLE_MYSQL_ROOT_PASSWORD} - read from Vault

${RANCHER_URL}
${BETA_ACCESS_KEY}
${BETA_SECRET_KEY}
${PROD_ACCESS_KEY}
${PROD_SECRET_KEY}
```

## Registry - images

 - team/sampleapp/wordpress
 - team/sampleapp/wordpress-data
 - team/sampleapp/wordpress-filebeat

## Ojeby
- removing build definition from docker-compose via sed in Jenkinspipeline to allow local build and also deploy via rancher-cli
- to keep KISS no testing steps, rollback are made just if prev step/stage failed
