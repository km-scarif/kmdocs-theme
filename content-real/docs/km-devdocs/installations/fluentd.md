---
title: "Fluentd"
date: 2022-10-26T11:30:02-04:00
summary: "Instructions for running Fluentd in Kubernetes"
test: "sfjasdiofjj"
tags:
  - install
  - fluentd
  - kubernetes
---

# Fluentd Configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: fluentd
data:
  fluent.conf: |-
    ################################################################
    # This source gets all logs from local docker host
    @include pods-kind-fluent.conf
    @include elastic-fluent.conf
  pods-kind-fluent.conf: |-
    <source>
      @type tail
      format json
      read_from_head true
      tag kubernetes.*
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      exclude_path [
        "/var/log/containers/fluent*",
        "/var/log/containers/local-path-provisioner*",
        "/var/log/containers/svclb-traefik*",
        "/var/log/containers/traefik*",
        "/var/log/containers/coredns*",
        "/var/log/containers/metrics-server*"
      ]
      <parse>
        @type regexp
        #https://regex101.com/r/ZkOBTI/1
        expression ^(?<time>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.[^Z]*Z)\s(?<stream>[^\s]+)\s(?<character>[^\s])\s(?<message>.*)$
        #time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
      skip_labels "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_LABELS'] || 'false'}"
      skip_container_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_CONTAINER_METADATA'] || 'false'}"
      skip_master_url "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_MASTER_URL'] || 'false'}"
      skip_namespace_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_NAMESPACE_METADATA'] || 'false'}"
    </filter>
  elastic-fluent.conf: |-
    <match **>
      @type elasticsearch
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST'] || 'elasticsearch-svc.elastic-kibana'}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT'] || '9200'}"
      index_name fluentd-k8s
      type_name fluentd
    </match>
```
