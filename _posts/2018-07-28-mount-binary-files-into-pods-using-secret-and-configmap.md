---
title: "使用 secret 和 configmap 将二进制文件挂载到容器内"
excerpt: "如何使用 secret 和 configmap 将二进制文件挂载到 pod 内"
permalink: /kubernetes/mount-binary-files-into-pods-using-secret-and-configmap
last_modified_at: 2018-07-28
layout: tags
#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - secret
  - configMap
  - binary
---

### 背景

我们一个项目的开发人员提出一个需：其中一个微服务需要使用ssl方式连接外部的`kafka`服务，实现双向认证。因此需要本地配置客户端证书，在发起连接时加载。

这个证书文件，如何放到容器内？

开发人员希望我们直接将文件添加到容器内，并指定一个固定路径。  

技术上可以这么做，但有一个问题: 开发/测试/生产 不同环境的 ssl 证书必定不一样，这样就会导致不同环境要构建不同的 Docker 镜像，这不符合我们的预期。我们希望同一个Docker镜像，可以适用于所有环境。

因此，我们需要换一种方式来实现。

我们发现 Kubernetes 的 `secret` 和 `configmap` 可以做个事。 但 configmap 要 Kubernetes 1.10 开始才支持。



### 如何使用 `secret`

- 修改 Helm chart 的 Template 目录下的 `secret.yaml` 文件，增加 kafka jks 的配置：

  ```yaml
  {% raw %}
  apiVersion: v1
  kind: Secret
  type: kubernetes.io/dockercfg
  metadata:
    name: {{ .Release.Name }}-imagepullsecret
    namespace: {{ .Values.namespace }}
  data:
    .dockercfg: {{ include "dockerCfgString" . }}
    kafka.client.keystore.jks: {{ .Values.secret.data.keystore }}
    kafka.client.truststore.jks: {{ .Values.secret.data.truststore }}
  {% endraw %}
  ```

- 修改 chart 内的 `values.yaml` 文件，设置 `keystore` 和 `truststore` 的 data 和 volume 挂载点

  `keystore` 和 `truststore` 默认值为一个错误的值，避免环境互通的情况下，加载到了默认的正确值，链接到其它地方去了；
  
  同时，这里我们优化了 `volumes` 和 `volumeMounts`, 实现了多种不同类型的 volume 的挂载。

  ```yaml
  volumeMounts:
  - name: kafka-keystore
    mountPath: /home/c3user/keystore
    readOnly: true
  
  volumes:
  - name: kafka-keystore
    secret:
      secretName: tima-cons-ford-imagepullsecret
      items:
      - key: kafka.client.keystore.jks
        path: kafka.client.keystore.jks
      - key: kafka.client.truststore.jks
        path: kafka.client.truststore.jks

  secret:
    data:
      keystore: "base64 value of file kafka.client.keystore.jks"
      truststore: "base64 value of file kafka.client.truststore.jks"
  ```

- 生成 kafka jks 文件的 `base64` 编码
  ```bash
  # -w 0 // 指定输出编码不换行
  cat kafka.client.keystore.jks | base64.exe -w 0
  cat kafka.client.truststore.jks | base64.exe -w 0
  ```

- 在 `charts-deploy` 的 appvars 下 的任意 app YAML文件内，新增与上面相同的字段，但 value 值填写正确的 kafka jks 文件的 base64 编码, 这样在渲染模版时，就会覆盖 values.yaml 中的值：

  ```yaml
    secret:
      data:
        keystore: "xxxxxxxxxxxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyy"
        truststore: "yyyyyyyyyyyyyyyyyyyyyyxxxxxxxxxxxxxxxxxxxxxx"
  ```

- 在 `_helper.tpl` 模版文件内定义一段可以重用的代码，使用 `toYaml` 来直接转换 `values.yaml` 的内容为 `YAML` 格式  

  由于代码块中含有 jekyll 会主动解析的关键字：{% raw %} {{ }} {% endraw %}, 我们需要使用逃逸字符来避免 Jekyll 渲染它们。  

  用法参考这里： [Escaping double curly braces inside a markdown code block in Jekyll](https://stackoverflow.com/a/24102537/5723841)

  ```jinja
  {% raw %}
  {{- define "volume.mounts" }}
  {{ toYaml .Values.volumeMounts | trim }}
  {{- end }}
  
  {{- define "volume.sources" }}
  {{ toYaml .Values.volumes | trim }}
  {{- end }}
  {% endraw %}
  ```

- 在 Deployment 中添加 volumeMounts 和 volumes 定义

  使用优化后的模版渲染方式：在 values.yaml 中分开定义，并在 Deployment.yaml 中分开引用。

  ```jinja
  {% raw %}
    volumeMounts:
    {{- include "volume.mounts" . |indent 8 }}
  volumes:
  {{- include "volume.sources" . | indent 6 }}
  imagePullSecrets:
  - name: {{ .Release.Name }}-imagepullsecret
  {% endraw %}
  ```

- 本地渲染

  如果我们没有确定 helm 渲染出来的文件格式是否正常，就使用 `helm install` 直接安装，一旦有错就得重来，显然不方便于调试。因此我们可以在这之前使用 `helm template` 命令在本地渲染并输出 yaml 格式的文件进行修正。  
详细参考这里： https://docs.helm.sh/helm/#helm_template
  ```bash
  helm template mychart -x templates/deployment.yaml
  
  # 或
  cd mychart
  helm template ./  -x templates/deployment.yaml
  ```
  
  最后，如果没有问题，我们就可以给 Chart 打上新的 tag ，提交到 ChartMuseum 了。

- 提交成 chart
  
  编写好 Chart 之后，如果 charts-deploy（存储 appvars） 也检查无误，我们就可以将它提交到 git 仓库了。 我们的 Pipeline 会自动构建、上传并安装 Chart。

- 验证一下

  下面是我们在 Kubernetes-Dashboard 和 具体的 应用内部看到的情况。  
  
  Secret：  
  ![image](http://ws4.sinaimg.cn/large/6ee6fa3egy1fuw7ux88l0j20dh0cwq3k.jpg)  
  
  pod 内 mount情况：  
  ![image](http://wx1.sinaimg.cn/large/6ee6fa3egy1fuw7ylpb57j20oi02o0sz.jpg)
  
  
  ### 如何使用 Configmap
  
  后续补充。