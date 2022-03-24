# Ansible role: ELK (ElasticSearch, Logstash, Kibana, Filebeat)
[![CI Molecule](https://github.com/darexsu/ansible-role-elk/actions/workflows/ci.yml/badge.svg)](https://github.com/darexsu/ansible-role-elk/actions/workflows/ci.yml)&emsp;![](https://img.shields.io/static/v1?label=idempotence&message=ok&color=success)&emsp;![Ansible Role](https://img.shields.io/ansible/role/d/58508?color=blue&label=downloads)

  - Role:
      - [platforms](#platforms)
      - [install](#install)
      - [requirements](#requirements)
      - [merge behaviour](#merge-behaviour)
  - Playbooks (merge version):
      - [install and configure: ELK (ElasticSearch, Logstash, Kibana, Filebeat), FirewallD](#install-and-configure-elk-merge-version)
          - [install:  ELK (ElasticSearch, Logstash, Kibana, Filebeat)](#install-elk-merge-version)
          - [configure:  ELK (ElasticSearch, Logstash, Kibana, Filebeat)](#configure-elk-merge-version)
  - Playbooks (full version):
      - [install and configure:  ELK (ElasticSearch, Logstash, Kibana, Filebeat), FirewallD](#install-and-configure-elk-full-version)
          - [install:  ELK (ElasticSearch, Logstash, Kibana, Filebeat)](#install-elk-full-version)
          - [configure: ELK (ElasticSearch, Logstash, Kibana, Filebeat)](#configure-elk-full-version)

### Platforms

|  Testing         | repo: elastic      |
| :--------------: | :----------------: |
| Debian 11        |  elastic.co        |
| Debian 10        |  elastic.co        |
| Ubuntu 20.04     |  elastic.co        |
| Ubuntu 18.04     |  elastic.co        |
| Oracle Linux 8   |  elastic.co        |
| Rocky Linux 8    |  elastic.co        |

### Install

```
ansible-galaxy install darexsu.elk --force
```
### Requirements

roles: [ElasticSearch](https://github.com/darexsu/ansible-role-elasticsearch), [Logstash](https://github.com/darexsu/ansible-role-logstash), [Kibana](https://github.com/darexsu/ansible-role-kibana), [Filebeat](https://github.com/darexsu/ansible-role-filebeat), [FirewallD](https://github.com/darexsu/ansible-role-firewalld) (will automatically be installed)

### Merge behaviour

Replace or Merge dictionaries (with "hash_behaviour=replace" in ansible.cfg):
```
# Replace             # Merge
---                   ---
  vars:                 vars:
    dict:                 merge:
      a: "value"            dict: 
      b: "value"              a: "value" 
                              b: "value"

# How does merge work?:
Your vars [host_vars]  -->  default vars [current role] --> default vars [include role]
  
  dict:          dict:              dict:
    a: "1" -->     a: "1"    -->      a: "1"
                   b: "2"    -->      b: "2"
                                      c: "3"
    
```

##### Install and configure: ELK (merge version)
```yaml
---
- hosts: all
  become: true

  vars:
    merge:
      # ELK
      elk:
        enabled: true
        version: "8.x"

      # ElasticSearch
      elasticsearch:
        enabled: true
      # ElasticSearch -> install
      elasticsearch_install:
        enabled: true
      # ElasticSearch -> config -> elasticsearch.yml
      elasticsearch_yml:
        enabled: true
        data: |
          path.data: /var/lib/elasticsearch
          path.logs: /var/log/elasticsearch
          xpack.security.enabled: false
          xpack.security.enrollment.enabled: true
          xpack.security.http.ssl:
            enabled: true
            keystore.path: certs/http.p12
          xpack.security.transport.ssl:
            enabled: true
            verification_mode: certificate
            keystore.path: certs/transport.p12
            truststore.path: certs/transport.p12
          http.host: [_local_, _site_]
      # ElasticSearch -> config -> jvm.options
      elasticsearch_jvm_options:
        enabled: true
        data: |
          8-13:-XX:+UseConcMarkSweepGC
          8-13:-XX:CMSInitiatingOccupancyFraction=75
          8-13:-XX:+UseCMSInitiatingOccupancyOnly
          14-:-XX:+UseG1GC
          -Djava.io.tmpdir=${ES_TMPDIR}
          -XX:+HeapDumpOnOutOfMemoryError
          9-:-XX:+ExitOnOutOfMemoryError
          -XX:HeapDumpPath=/var/lib/elasticsearch
          -XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
          -Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m

      # Kibana
      kibana:
        enabled: true
      # Kibana -> install
      kibana_install:
        enabled: true
      # Kibana -> config -> kibana.yml
      kibana_yml:
        enabled: true
        data: |
          pid.file: /run/kibana/kibana.pid
          server.host: 0.0.0.0
          server.publicBaseUrl: http://0.0.0.0:5601/
          logging:
            appenders:
              file:
                type: file
                fileName: /var/log/kibana/kibana.log
                layout:
                  type: json
            root:
              appenders:
                - default
                - file

      # Logstash
      logstash:
        enabled: true
      # Logstash -> install
      logstash_install:
        enabled: true
      # Logstash -> config -> logstash.yml
      logstash_yml:
        enabled: true
        data: |
          path.data: /var/lib/logstash
          path.logs: /var/log/logstash
      # Logstash -> config -> logstash.conf
      logstash_conf:
        input_conf:
          enabled: true
          file: "input.conf"
          src: "input_conf.j2"
          backup: false
          data:
            port: '5044'
        output_conf:
          enabled: true
          file: "output.conf"
          src: "output_conf.j2"
          backup: false
          data:
            hosts: '["http://localhost:9200"]'
            index: '"%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"'
      # Logstash -> config -> jvm.options
      logstash_jvm_options:
        enabled: true
        data: |
          -Xms1g
          -Xmx1g
          11-13:-XX:+UseConcMarkSweepGC
          11-13:-XX:CMSInitiatingOccupancyFraction=75
          11-13:-XX:+UseCMSInitiatingOccupancyOnly
          -Djava.awt.headless=true
          -Dfile.encoding=UTF-8
          -Djruby.compile.invokedynamic=true
          -Djruby.jit.threshold=0
          -Djruby.regexp.interruptible=true
          -XX:+HeapDumpOnOutOfMemoryError
          -Djava.security.egd=file:/dev/urandom
          -Dlog4j2.isThreadContextMapInheritable=true
          11-:--add-opens=java.base/java.security=ALL-UNNAMED
          11-:--add-opens=java.base/java.io=ALL-UNNAMED
          11-:--add-opens=java.base/java.nio.channels=ALL-UNNAMED
          11-:--add-opens=java.base/sun.nio.ch=ALL-UNNAMED
          11-:--add-opens=java.management/sun.management=ALL-UNNAMED

      # Filebeat
      filebeat:
        enabled: true
      # Filebeat -> install
      filebeat_install:
        enabled: true
      # Filebeat -> config -> filebeat.yml
      filebeat_yml:
        enabled: true
        data: |
          # ============================== Filebeat inputs ===============================
          filebeat.inputs:
            - type: filestream
              enabled: true
              paths:
                - /var/log/*.log
          # ============================== Filebeat modules ==============================
          filebeat.config.modules:
            path: ${path.config}/modules.d/*.yml
            reload.enabled: false
          # ======================= Elasticsearch template setting =======================
          setup.template.settings:
            index.number_of_shards: 1
          # ================================== General ===================================
          setup.kibana:
          # ---------------------------- Elasticsearch Output ----------------------------
          # output.elasticsearch:
          #   hosts: ["localhost:9200"]
          # ------------------------------ Logstash Output -------------------------------
          output.logstash:
            hosts: ["localhost:5044"]
          # ================================= Processors =================================
          processors:
            - add_host_metadata:
                when.not.contains.tags: forwarded
            - add_cloud_metadata: ~
            - add_docker_metadata: ~
            - add_kubernetes_metadata: ~
          # ================================== Logging ===================================
          # logging.level: debug
          # logging.selectors: ["*"]
          # ============================= X-Pack Monitoring ==============================
          # monitoring.enabled: false
          # monitoring.cluster_uuid:
          # monitoring.elasticsearch:
          # ============================== Instrumentation ===============================
          # instrumentation:
          # enabled: false
          # environment: ""
          # hosts:
          #   - http://localhost:8200
          # api_key:
          # secret_token:
          # ================================= Migration ==================================
          # migration.6_to_7.enabled: true

      # FirewallD
      firewalld:
        enabled: true
      # FirewallD -> rules
      firewalld_rules:
        logstash_port_5044:
          enabled: true
          zone: "public"
          state: "enabled"
          port: "5044/tcp"
          permanent: true
        kibana_port_5601:
          enabled: true
          zone: "public"
          state: "enabled"
          port: "5601/tcp"
          permanent: true

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```
##### Install: ELK (merge version)
```yaml
---
- hosts: all
  become: true

  vars:
    merge:
      # ELK
      elk:
        enabled: true
        version: "8.x"

      # ElasticSearch
      elasticsearch:
        enabled: true
      # ElasticSearch -> install
      elasticsearch_install:
        enabled: true

      # Kibana
      kibana:
        enabled: true
      # Kibana -> install
      kibana_install:
        enabled: true

      # Logstash
      logstash:
        enabled: true
      # Logstash -> install
      logstash_install:
        enabled: true

      # Filebeat
      filebeat:
        enabled: true
      # Filebeat -> install
      filebeat_install:
        enabled: true

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```
##### Configure: ELK (merge version)
```yaml
---
- hosts: all
  become: true

  vars:
    merge:
      # ELK
      elk:
        enabled: true
        version: "8.x"

      # ElasticSearch
      elasticsearch:
        enabled: true
      # ElasticSearch -> config -> elasticsearch.yml
      elasticsearch_yml:
        enabled: true
        data: |
          path.data: /var/lib/elasticsearch
          path.logs: /var/log/elasticsearch
          xpack.security.enabled: false
          xpack.security.enrollment.enabled: true
          xpack.security.http.ssl:
            enabled: true
            keystore.path: certs/http.p12
          xpack.security.transport.ssl:
            enabled: true
            verification_mode: certificate
            keystore.path: certs/transport.p12
            truststore.path: certs/transport.p12
          http.host: [_local_, _site_]
      # ElasticSearch -> config -> jvm.options
      elasticsearch_jvm_options:
        enabled: true
        data: |
          8-13:-XX:+UseConcMarkSweepGC
          8-13:-XX:CMSInitiatingOccupancyFraction=75
          8-13:-XX:+UseCMSInitiatingOccupancyOnly
          14-:-XX:+UseG1GC
          -Djava.io.tmpdir=${ES_TMPDIR}
          -XX:+HeapDumpOnOutOfMemoryError
          9-:-XX:+ExitOnOutOfMemoryError
          -XX:HeapDumpPath=/var/lib/elasticsearch
          -XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
          -Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m

      # Kibana
      kibana:
        enabled: true
      # Kibana -> config -> kibana.yml
      kibana_yml:
        enabled: true
        data: |
          pid.file: /run/kibana/kibana.pid
          server.host: 0.0.0.0
          server.publicBaseUrl: http://0.0.0.0:5601/
          logging:
            appenders:
              file:
                type: file
                fileName: /var/log/kibana/kibana.log
                layout:
                  type: json
            root:
              appenders:
                - default
                - file

      # Logstash
      logstash:
        enabled: true
      # Logstash -> config -> logstash.yml
      logstash_yml:
        enabled: true
        data: |
          path.data: /var/lib/logstash
          path.logs: /var/log/logstash
      # Logstash -> config -> logstash.conf
      logstash_conf:
        input_conf:
          enabled: true
          file: "input.conf"
          src: "input_conf.j2"
          backup: false
          data:
            port: '5044'
        output_conf:
          enabled: true
          file: "output.conf"
          src: "output_conf.j2"
          backup: false
          data:
            hosts: '["http://localhost:9200"]'
            index: '"%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"'
      # Logstash -> config -> jvm.options
      logstash_jvm_options:
        enabled: true
        data: |
          -Xms1g
          -Xmx1g
          11-13:-XX:+UseConcMarkSweepGC
          11-13:-XX:CMSInitiatingOccupancyFraction=75
          11-13:-XX:+UseCMSInitiatingOccupancyOnly
          -Djava.awt.headless=true
          -Dfile.encoding=UTF-8
          -Djruby.compile.invokedynamic=true
          -Djruby.jit.threshold=0
          -Djruby.regexp.interruptible=true
          -XX:+HeapDumpOnOutOfMemoryError
          -Djava.security.egd=file:/dev/urandom
          -Dlog4j2.isThreadContextMapInheritable=true
          11-:--add-opens=java.base/java.security=ALL-UNNAMED
          11-:--add-opens=java.base/java.io=ALL-UNNAMED
          11-:--add-opens=java.base/java.nio.channels=ALL-UNNAMED
          11-:--add-opens=java.base/sun.nio.ch=ALL-UNNAMED
          11-:--add-opens=java.management/sun.management=ALL-UNNAMED

      # Filebeat
      filebeat:
        enabled: true
      # Filebeat -> config -> filebeat.yml
      filebeat_yml:
        enabled: true
        data: |
          # ============================== Filebeat inputs ===============================
          filebeat.inputs:
            - type: filestream
              enabled: true
              paths:
                - /var/log/*.log
          # ============================== Filebeat modules ==============================
          filebeat.config.modules:
            path: ${path.config}/modules.d/*.yml
            reload.enabled: false
          # ======================= Elasticsearch template setting =======================
          setup.template.settings:
            index.number_of_shards: 1
          # ================================== General ===================================
          setup.kibana:
          # ---------------------------- Elasticsearch Output ----------------------------
          # output.elasticsearch:
          #   hosts: ["localhost:9200"]
          # ------------------------------ Logstash Output -------------------------------
          output.logstash:
            hosts: ["localhost:5044"]
          # ================================= Processors =================================
          processors:
            - add_host_metadata:
                when.not.contains.tags: forwarded
            - add_cloud_metadata: ~
            - add_docker_metadata: ~
            - add_kubernetes_metadata: ~
          # ================================== Logging ===================================
          # logging.level: debug
          # logging.selectors: ["*"]
          # ============================= X-Pack Monitoring ==============================
          # monitoring.enabled: false
          # monitoring.cluster_uuid:
          # monitoring.elasticsearch:
          # ============================== Instrumentation ===============================
          # instrumentation:
          # enabled: false
          # environment: ""
          # hosts:
          #   - http://localhost:8200
          # api_key:
          # secret_token:
          # ================================= Migration ==================================
          # migration.6_to_7.enabled: true

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```
##### Install and configure: ELK (full version)
```yaml
---
- hosts: all
  become: true

  vars:
    # ELK
    elk:
      enabled: true
      version: "8.x"

    # ElasticSearch
    elasticsearch:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # ElasticSearch -> install
    elasticsearch_install:
      enabled: true
      packages: ["elasticsearch"]
    # ElasticSearch -> config -> elasticsearch.yml
    elasticsearch_yml:
      enabled: true
      file: "elasticsearch.yml"
      src: "elasticsearch_yml.j2"
      backup: false
      data: |
        path.data: /var/lib/elasticsearch
        path.logs: /var/log/elasticsearch
        xpack.security.enabled: false
        xpack.security.enrollment.enabled: true
        xpack.security.http.ssl:
          enabled: true
          keystore.path: certs/http.p12
        xpack.security.transport.ssl:
          enabled: true
          verification_mode: certificate
          keystore.path: certs/transport.p12
          truststore.path: certs/transport.p12
        http.host: [_local_, _site_]
    # ElasticSearch -> config -> jvm.options
    elasticsearch_jvm_options:
      enabled: true
      file: "jvm.options"
      src: "jvm_options.j2"
      backup: false
      data: |
        8-13:-XX:+UseConcMarkSweepGC
        8-13:-XX:CMSInitiatingOccupancyFraction=75
        8-13:-XX:+UseCMSInitiatingOccupancyOnly
        14-:-XX:+UseG1GC
        -Djava.io.tmpdir=${ES_TMPDIR}
        -XX:+HeapDumpOnOutOfMemoryError
        9-:-XX:+ExitOnOutOfMemoryError
        -XX:HeapDumpPath=/var/lib/elasticsearch
        -XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
        -Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m

    # Kibana
    kibana:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Kibana -> install
    kibana_install:
      enabled: true
      packages: ["kibana"]
    # Kibana -> config -> kibana.yml
    kibana_yml:
      enabled: true
      file: "kibana.yml"
      src: "kibana_yml.j2"
      backup: false
      data: |
        pid.file: /run/kibana/kibana.pid
        server.host: 0.0.0.0
        server.publicBaseUrl: http://0.0.0.0:5601/
        logging:
          appenders:
            file:
              type: file
              fileName: /var/log/kibana/kibana.log
              layout:
                type: json
          root:
            appenders:
              - default
              - file

    # Logstash
    logstash:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Logstash -> install
    logstash_install:
      enabled: true
      packages: ["logstash"]
    # Logstash -> config -> logstash.yml
    logstash_yml:
      enabled: true
      file: "logstash.yml"
      src: "logstash_yml.j2"
      backup: false
      data: |
        path.data: /var/lib/logstash
        path.logs: /var/log/logstash
    # Logstash -> config -> logstash.conf
    logstash_conf:
      input_conf:
        enabled: true
        file: "input.conf"
        src: "input_conf.j2"
        backup: false
        data:
          port: '5044'
      output_conf:
        enabled: true
        file: "output.conf"
        src: "output_conf.j2"
        backup: false
        data:
          hosts: '["http://localhost:9200"]'
          index: '"%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"'
    # Logstash -> config -> jvm.options
    logstash_jvm_options:
      enabled: true
      file: "jvm.options"
      src: "jvm_options.j2"
      backup: false
      data: |
        -Xms1g
        -Xmx1g
        11-13:-XX:+UseConcMarkSweepGC
        11-13:-XX:CMSInitiatingOccupancyFraction=75
        11-13:-XX:+UseCMSInitiatingOccupancyOnly
        -Djava.awt.headless=true
        -Dfile.encoding=UTF-8
        -Djruby.compile.invokedynamic=true
        -Djruby.jit.threshold=0
        -Djruby.regexp.interruptible=true
        -XX:+HeapDumpOnOutOfMemoryError
        -Djava.security.egd=file:/dev/urandom
        -Dlog4j2.isThreadContextMapInheritable=true
        11-:--add-opens=java.base/java.security=ALL-UNNAMED
        11-:--add-opens=java.base/java.io=ALL-UNNAMED
        11-:--add-opens=java.base/java.nio.channels=ALL-UNNAMED
        11-:--add-opens=java.base/sun.nio.ch=ALL-UNNAMED
        11-:--add-opens=java.management/sun.management=ALL-UNNAMED

    # Filebeat
    filebeat:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Filebeat -> install
    filebeat_install:
      enabled: true
      packages: ["filebeat"]
    # Filebeat -> config -> filebeat.yml
    filebeat_yml:
      enabled: true
      file: "filebeat.yml"
      src: "filebeat_yml.j2"
      backup: false
      data: |
        # ============================== Filebeat inputs ===============================
        filebeat.inputs:
          - type: filestream
            enabled: true
            paths:
              - /var/log/*.log
        # ============================== Filebeat modules ==============================
        filebeat.config.modules:
          path: ${path.config}/modules.d/*.yml
          reload.enabled: false
        # ======================= Elasticsearch template setting =======================
        setup.template.settings:
          index.number_of_shards: 1
        # ================================== General ===================================
        setup.kibana:
        # ---------------------------- Elasticsearch Output ----------------------------
        # output.elasticsearch:
        #   hosts: ["localhost:9200"]
        # ------------------------------ Logstash Output -------------------------------
        output.logstash:
          hosts: ["localhost:5044"]
        # ================================= Processors =================================
        processors:
          - add_host_metadata:
              when.not.contains.tags: forwarded
          - add_cloud_metadata: ~
          - add_docker_metadata: ~
          - add_kubernetes_metadata: ~
        # ================================== Logging ===================================
        # logging.level: debug
        # logging.selectors: ["*"]
        # ============================= X-Pack Monitoring ==============================
        # monitoring.enabled: false
        # monitoring.cluster_uuid:
        # monitoring.elasticsearch:
        # ============================== Instrumentation ===============================
        # instrumentation:
        # enabled: false
        # environment: ""
        # hosts:
        #   - http://localhost:8200
        # api_key:
        # secret_token:
        # ================================= Migration ==================================
        # migration.6_to_7.enabled: true

    # FirewallD
    firewalld:
      enabled: true
    # FirewallD -> rules
    firewalld_rules:
      logstash_port_5044:
        enabled: true
        zone: "public"
        state: "enabled"
        port: "5044/tcp"
        permanent: true
      kibana_port_5601:
        enabled: true
        zone: "public"
        state: "enabled"
        port: "5601/tcp"
        permanent: true

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```
##### Install: ELK (full version)
```yaml
---
- hosts: all
  become: true

  vars:
    # ELK
    elk:
      enabled: true
      version: "8.x"

    # ElasticSearch
    elasticsearch:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # ElasticSearch -> install
    elasticsearch_install:
      enabled: true
      packages: ["elasticsearch"]

    # Kibana
    kibana:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Kibana -> install
    kibana_install:
      enabled: true
      packages: ["kibana"]

    # Logstash
    logstash:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Logstash -> install
    logstash_install:
      enabled: true
      packages: ["logstash"]

    # Filebeat
    filebeat:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Filebeat -> install
    filebeat_install:
      enabled: true
      packages: ["filebeat"]

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```
##### Configure: ELK (full version)
```yaml
---
- hosts: all
  become: true

  vars:
    # ELK
    elk:
      enabled: true
      version: "8.x"

    # ElasticSearch
    elasticsearch:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # ElasticSearch -> config -> elasticsearch.yml
    elasticsearch_yml:
      enabled: true
      file: "elasticsearch.yml"
      src: "elasticsearch_yml.j2"
      backup: false
      data: |
        path.data: /var/lib/elasticsearch
        path.logs: /var/log/elasticsearch
        xpack.security.enabled: false
        xpack.security.enrollment.enabled: true
        xpack.security.http.ssl:
          enabled: true
          keystore.path: certs/http.p12
        xpack.security.transport.ssl:
          enabled: true
          verification_mode: certificate
          keystore.path: certs/transport.p12
          truststore.path: certs/transport.p12
        http.host: [_local_, _site_]
    # ElasticSearch -> config -> jvm.options
    elasticsearch_jvm_options:
      enabled: true
      file: "jvm.options"
      src: "jvm_options.j2"
      backup: false
      data: |
        8-13:-XX:+UseConcMarkSweepGC
        8-13:-XX:CMSInitiatingOccupancyFraction=75
        8-13:-XX:+UseCMSInitiatingOccupancyOnly
        14-:-XX:+UseG1GC
        -Djava.io.tmpdir=${ES_TMPDIR}
        -XX:+HeapDumpOnOutOfMemoryError
        9-:-XX:+ExitOnOutOfMemoryError
        -XX:HeapDumpPath=/var/lib/elasticsearch
        -XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
        -Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m

    # Kibana
    kibana:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Kibana -> config -> kibana.yml
    kibana_yml:
      enabled: true
      file: "kibana.yml"
      src: "kibana_yml.j2"
      backup: false
      data: |
        pid.file: /run/kibana/kibana.pid
        server.host: 0.0.0.0
        server.publicBaseUrl: http://0.0.0.0:5601/
        logging:
          appenders:
            file:
              type: file
              fileName: /var/log/kibana/kibana.log
              layout:
                type: json
          root:
            appenders:
              - default
              - file

    # Logstash
    logstash:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Logstash -> config -> logstash.yml
    logstash_yml:
      enabled: true
      file: "logstash.yml"
      src: "logstash_yml.j2"
      backup: false
      data: |
        path.data: /var/lib/logstash
        path.logs: /var/log/logstash
    # Logstash -> config -> logstash.conf
    logstash_conf:
      input_conf:
        enabled: true
        file: "input.conf"
        src: "input_conf.j2"
        backup: false
        data:
          port: '5044'
      output_conf:
        enabled: true
        file: "output.conf"
        src: "output_conf.j2"
        backup: false
        data:
          hosts: '["http://localhost:9200"]'
          index: '"%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"'
    # Logstash -> config -> jvm.options
    logstash_jvm_options:
      enabled: true
      file: "jvm.options"
      src: "jvm_options.j2"
      backup: false
      data: |
        -Xms1g
        -Xmx1g
        11-13:-XX:+UseConcMarkSweepGC
        11-13:-XX:CMSInitiatingOccupancyFraction=75
        11-13:-XX:+UseCMSInitiatingOccupancyOnly
        -Djava.awt.headless=true
        -Dfile.encoding=UTF-8
        -Djruby.compile.invokedynamic=true
        -Djruby.jit.threshold=0
        -Djruby.regexp.interruptible=true
        -XX:+HeapDumpOnOutOfMemoryError
        -Djava.security.egd=file:/dev/urandom
        -Dlog4j2.isThreadContextMapInheritable=true
        11-:--add-opens=java.base/java.security=ALL-UNNAMED
        11-:--add-opens=java.base/java.io=ALL-UNNAMED
        11-:--add-opens=java.base/java.nio.channels=ALL-UNNAMED
        11-:--add-opens=java.base/sun.nio.ch=ALL-UNNAMED
        11-:--add-opens=java.management/sun.management=ALL-UNNAMED

    # Filebeat
    filebeat:
      enabled: true
      version: "{{ elk.version }}"
      repo: "elastic"
      service:
        enabled: true
        state: "started"
    # Filebeat -> config -> filebeat.yml
    filebeat_yml:
      enabled: true
      file: "filebeat.yml"
      src: "filebeat_yml.j2"
      backup: false
      data: |
        # ============================== Filebeat inputs ===============================
        filebeat.inputs:
          - type: filestream
            enabled: true
            paths:
              - /var/log/*.log
        # ============================== Filebeat modules ==============================
        filebeat.config.modules:
          path: ${path.config}/modules.d/*.yml
          reload.enabled: false
        # ======================= Elasticsearch template setting =======================
        setup.template.settings:
          index.number_of_shards: 1
        # ================================== General ===================================
        setup.kibana:
        # ---------------------------- Elasticsearch Output ----------------------------
        # output.elasticsearch:
        #   hosts: ["localhost:9200"]
        # ------------------------------ Logstash Output -------------------------------
        output.logstash:
          hosts: ["localhost:5044"]
        # ================================= Processors =================================
        processors:
          - add_host_metadata:
              when.not.contains.tags: forwarded
          - add_cloud_metadata: ~
          - add_docker_metadata: ~
          - add_kubernetes_metadata: ~
        # ================================== Logging ===================================
        # logging.level: debug
        # logging.selectors: ["*"]
        # ============================= X-Pack Monitoring ==============================
        # monitoring.enabled: false
        # monitoring.cluster_uuid:
        # monitoring.elasticsearch:
        # ============================== Instrumentation ===============================
        # instrumentation:
        # enabled: false
        # environment: ""
        # hosts:
        #   - http://localhost:8200
        # api_key:
        # secret_token:
        # ================================= Migration ==================================
        # migration.6_to_7.enabled: true

  tasks:
    - name: role darexsu elk
      include_role:
        name: darexsu.elk

```