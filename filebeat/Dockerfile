FROM debian:jessie

MAINTAINER Vaclav Adamec <vaclav.adamec@suchy-zleb.cz>

ENV FILEBEAT_VERSION=5.2.1 \
    FILEBEAT_SHA1=694fe12e56ebf8e4c4b11b590cfb46c662e7a3c1

RUN set -x && \
  apt-get update && \
  apt-get install -y wget && \
  wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.1-linux-x86_64.tar.gz -O /opt/filebeat.tar.gz && \
  cd /opt && \
  echo "${FILEBEAT_SHA1}  filebeat.tar.gz" | sha1sum -c - && \
  tar xzvf filebeat.tar.gz && \
  cd filebeat-* && \
  cp filebeat /bin && \
  cd /opt && \
  rm -rf filebeat* && \
  apt-get purge -y wget && \
  apt-get autoremove -y && \
  apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ADD filebeat.yml /filebeat.yml
ADD filebeat.template.json /filebeat.template.json
ADD filebeat.template-es2x.json /filebeat.template-es2x.json

CMD [ "filebeat", "-e" ]
