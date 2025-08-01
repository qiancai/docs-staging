---
title: 为 TiDB 组件间开启 TLS
summary: 在 Kubernetes 上如何为 TiDB 集群组件间开启 TLS。
---

# 为 TiDB 组件间开启 TLS

本文主要描述了在 Kubernetes 上如何为 TiDB 集群组件间开启 TLS。TiDB Operator 从 v1.1 开始已经支持为 Kubernetes 上 TiDB 集群组件间开启 TLS。开启步骤为：

1. 为即将被创建的 TiDB 集群的每个组件生成证书：
    - 为 PD/TiKV/TiDB/Pump/Drainer/TiFlash/TiProxy/TiKV Importer/TiDB Lightning 组件分别创建一套 Server 端证书，保存为 Kubernetes Secret 对象：`${cluster_name}-${component_name}-cluster-secret`
    - 为它们的各种客户端创建一套共用的 Client 端证书，保存为 Kubernetes Secret 对象：`${cluster_name}-cluster-client-secret`

    > **注意：**
    >
    > 创建的 Secret 对象必须符合上述命名规范，否则将导致各组件部署失败。

2. 部署集群，设置 `.spec.tlsCluster.enabled` 属性为 `true`；

    > **注意：**
    >
    > - 在集群创建后，不能修改此字段，否则将导致集群升级失败，此时需要删除已有集群，并重新创建。
    > - 若集群无法重建且需要启用 TLS，请参阅[将非 TLS 集群升级为 TLS 集群](#将非-tls-集群升级为-tls-集群)。

3. 配置 `pd-ctl`，`tikv-ctl` 连接集群。

> **注意：**
>
> * TiDB v4.0.5, TiDB Operator v1.1.4 及以上版本支持 TiFlash 开启 TLS。
> * TiDB v4.0.3, TiDB Operator v1.1.3 及以上版本支持 TiCDC 开启 TLS。

其中，颁发证书的方式有多种，本文档提供两种方式，用户也可以根据需要为 TiDB 集群颁发证书，这两种方式分别为：

- 使用 `cfssl` 系统颁发证书；
- 使用 `cert-manager` 系统颁发证书；

当需要更新已有 TLS 证书时，可参考[更新和替换 TLS 证书](renew-tls-certificate.md)。

## 第一步：为 TiDB 集群各个组件生成证书

### 使用 `cfssl` 系统颁发证书

1. 首先下载 `cfssl` 软件并初始化证书颁发机构：

    
    ``` shell
    mkdir -p ~/bin
    curl -s -L -o ~/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    curl -s -L -o ~/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    chmod +x ~/bin/{cfssl,cfssljson}
    export PATH=$PATH:~/bin

    mkdir -p cfssl
    cd cfssl
    ```

2. 生成 `ca-config.json` 配置文件：

    ```shell
    cat << EOF > ca-config.json
    {
        "signing": {
            "default": {
                "expiry": "8760h"
            },
            "profiles": {
                "internal": {
                    "expiry": "8760h",
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth",
                        "client auth"
                    ]
                },
                "client": {
                    "expiry": "8760h",
                    "usages": [
                        "signing",
                        "key encipherment",
                        "client auth"
                    ]
                }
            }
        }
    }
    EOF
    ```

3. 生成 `ca-csr.json` 配置文件：

    ```shell
    cat << EOF > ca-csr.json
    {
        "CN": "TiDB",
        "CA": {
            "expiry": "87600h"
        },
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "US",
                "L": "CA",
                "O": "PingCAP",
                "ST": "Beijing",
                "OU": "TiDB"
            }
        ]
    }
    EOF
    ```

4. 使用定义的选项生成 CA：

    
    ``` shell
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
    ```

5. 生成 Server 端证书。

    这里需要为每个 TiDB 集群的组件生成一套 Server 端证书。

    - PD Server 端证书

        首先生成默认的 `pd-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > pd-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-pd",
              "${cluster_name}-pd.${namespace}",
              "${cluster_name}-pd.${namespace}.svc",
              "${cluster_name}-pd-peer",
              "${cluster_name}-pd-peer.${namespace}",
              "${cluster_name}-pd-peer.${namespace}.svc",
              "*.${cluster_name}-pd-peer",
              "*.${cluster_name}-pd-peer.${namespace}",
              "*.${cluster_name}-pd-peer.${namespace}.svc"
            ],
        ...
        ```

        > **注意：**
        >
        > PD 从 v8.0.0 版本开始支持[微服务模式](https://docs.pingcap.com/zh/tidb/dev/pd-microservices)（实验特性）。如需部署 PD 微服务，并不需要为 PD 微服务的各个组件生成证书，只需要在 `pd-server.json` 文件的 `hosts` 字段中添加微服务相关的 hosts 配置即可。以 `scheduling` 微服务为例，你需要进行以下配置：
        >
        > ``` json
        > ...
        >     "CN": "TiDB",
        >     "hosts": [
        >       "127.0.0.1",
        >       "::1",
        >       "${cluster_name}-pd",
        >       ...
        >       "*.${cluster_name}-pd-peer.${namespace}.svc",
        >       // 以下是为 `scheduling` 微服务添加的 hosts 配置
        >       "${cluster_name}-scheduling",
        >       "${cluster_name}-scheduling.${cluster_name}",
        >       "${cluster_name}-scheduling.${cluster_name}.svc",
        >       "${cluster_name}-scheduling-peer",
        >       "${cluster_name}-scheduling-peer.${cluster_name}",
        >       "${cluster_name}-scheduling-peer.${cluster_name}.svc",
        >       "*.${cluster_name}-scheduling-peer",
        >       "*.${cluster_name}-scheduling-peer.${cluster_name}",
        >       "*.${cluster_name}-scheduling-peer.${cluster_name}.svc",
        >     ],
        > ...
        > ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 PD Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal pd-server.json | cfssljson -bare pd-server
        ```

    - TiKV Server 端证书

        首先生成默认的 `tikv-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > tikv-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-tikv",
              "${cluster_name}-tikv.${namespace}",
              "${cluster_name}-tikv.${namespace}.svc",
              "${cluster_name}-tikv-peer",
              "${cluster_name}-tikv-peer.${namespace}",
              "${cluster_name}-tikv-peer.${namespace}.svc",
              "*.${cluster_name}-tikv-peer",
              "*.${cluster_name}-tikv-peer.${namespace}",
              "*.${cluster_name}-tikv-peer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiKV Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal tikv-server.json | cfssljson -bare tikv-server
        ```

    - TiDB Server 端证书

        首先生成默认的 `tidb-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > tidb-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-tidb",
              "${cluster_name}-tidb.${namespace}",
              "${cluster_name}-tidb.${namespace}.svc",
              "${cluster_name}-tidb-peer",
              "${cluster_name}-tidb-peer.${namespace}",
              "${cluster_name}-tidb-peer.${namespace}.svc",
              "*.${cluster_name}-tidb-peer",
              "*.${cluster_name}-tidb-peer.${namespace}",
              "*.${cluster_name}-tidb-peer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiDB Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal tidb-server.json | cfssljson -bare tidb-server
        ```

    - Pump Server 端证书

        首先生成默认的 `pump-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > pump-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "*.${cluster_name}-pump",
              "*.${cluster_name}-pump.${namespace}",
              "*.${cluster_name}-pump.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 Pump Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal pump-server.json | cfssljson -bare pump-server
        ```

    - Drainer Server 端证书

        首先生成默认的 `drainer-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > drainer-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "<hosts 列表请参考下面描述>"
            ],
        ...
        ```

        现在 Drainer 组件是通过 Helm 来部署的，根据 `values.yaml` 文件配置方式不同，所需要填写的 `hosts` 字段也不相同。

        如果部署的时候设置 `drainerName` 属性，像下面这样：

        ``` yaml
        ...
        # Change the name of the statefulset and pod
        # The default is clusterName-ReleaseName-drainer
        # Do not change the name of an existing running drainer: this is unsupported.
        drainerName: my-drainer
        ...
        ```

        那么就这样配置 `hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "*.${drainer_name}",
              "*.${drainer_name}.${namespace}",
              "*.${drainer_name}.${namespace}.svc"
            ],
        ...
        ```

        如果部署的时候没有设置 `drainerName` 属性，需要这样配置 `hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "*.${cluster_name}-${release_name}-drainer",
              "*.${cluster_name}-${release_name}-drainer.${namespace}",
              "*.${cluster_name}-${release_name}-drainer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，`${release_name}` 是 `helm install` 时候填写的 `release name`，`${drainer_name}` 为 `values.yaml` 文件里的 `drainerName`，用户也可以添加自定义 `hosts`。

        最后生成 Drainer Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal drainer-server.json | cfssljson -bare drainer-server
        ```

    - TiCDC Server 端证书

        首先生成默认的 `ticdc-server.json` 文件：

        
        ``` shell
        cfssl print-defaults csr > ticdc-server.json
        ```

        然后编辑这个文件，修改 `CN`，`hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-ticdc",
              "${cluster_name}-ticdc.${namespace}",
              "${cluster_name}-ticdc.${namespace}.svc",
              "${cluster_name}-ticdc-peer",
              "${cluster_name}-ticdc-peer.${namespace}",
              "${cluster_name}-ticdc-peer.${namespace}.svc",
              "*.${cluster_name}-ticdc-peer",
              "*.${cluster_name}-ticdc-peer.${namespace}",
              "*.${cluster_name}-ticdc-peer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiCDC Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal ticdc-server.json | cfssljson -bare ticdc-server
        ```

    - TiProxy Server 端证书

        首先生成默认的 `tiproxy-server.json` 文件：

        ``` shell
        cfssl print-defaults csr > tiproxy-server.json
        ```

        然后编辑这个文件，修改 `CN` 和 `hosts` 属性：

        ``` json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-tiproxy",
              "${cluster_name}-tiproxy.${namespace}",
              "${cluster_name}-tiproxy.${namespace}.svc",
              "${cluster_name}-tiproxy-peer",
              "${cluster_name}-tiproxy-peer.${namespace}",
              "${cluster_name}-tiproxy-peer.${namespace}.svc",
              "*.${cluster_name}-tiproxy-peer",
              "*.${cluster_name}-tiproxy-peer.${namespace}",
              "*.${cluster_name}-tiproxy-peer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，你也可以添加自定义 `hosts`。

        最后生成 TiProxy Server 端证书：

        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal tiproxy-server.json | cfssljson -bare tiproxy-server
        ```

    - TiFlash Server 端证书

        首先生成默认的 `tiflash-server.json` 文件：

        
        ```shell
        cfssl print-defaults csr > tiflash-server.json
        ```

        然后编辑这个文件，修改 `CN`、`hosts` 属性：

        ```json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-tiflash",
              "${cluster_name}-tiflash.${namespace}",
              "${cluster_name}-tiflash.${namespace}.svc",
              "${cluster_name}-tiflash-peer",
              "${cluster_name}-tiflash-peer.${namespace}",
              "${cluster_name}-tiflash-peer.${namespace}.svc",
              "*.${cluster_name}-tiflash-peer",
              "*.${cluster_name}-tiflash-peer.${namespace}",
              "*.${cluster_name}-tiflash-peer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiFlash Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal tiflash-server.json | cfssljson -bare tiflash-server
        ```

    - TiKV Importer Server 端证书

        如需要[使用 TiDB Lightning 恢复 Kubernetes 上的集群数据](restore-data-using-tidb-lightning.md)，则需要为其中的 TiKV Importer 组件生成如下的 Server 端证书。

        首先生成默认的 `importer-server.json` 文件：

        
        ```shell
        cfssl print-defaults csr > importer-server.json
        ```

        然后编辑这个文件，修改 `CN`、`hosts` 属性：

        ```json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-importer",
              "${cluster_name}-importer.${namespace}",
              "${cluster_name}-importer.${namespace}.svc",
              "*.${cluster_name}-importer",
              "*.${cluster_name}-importer.${namespace}",
              "*.${cluster_name}-importer.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiKV Importer Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal importer-server.json | cfssljson -bare importer-server
        ```

    - TiDB Lightning Server 端证书

        如需要[使用 TiDB Lightning 恢复 Kubernetes 上的集群数据](restore-data-using-tidb-lightning.md)，则需要为其中的 TiDB Lightning 组件生成如下的 Server 端证书。

        首先生成默认的 `lightning-server.json` 文件：

        
        ```shell
        cfssl print-defaults csr > lightning-server.json
        ```

        然后编辑这个文件，修改 `CN`、`hosts` 属性：

        ```json
        ...
            "CN": "TiDB",
            "hosts": [
              "127.0.0.1",
              "::1",
              "${cluster_name}-lightning",
              "${cluster_name}-lightning.${namespace}",
              "${cluster_name}-lightning.${namespace}.svc"
            ],
        ...
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，用户也可以添加自定义 `hosts`。

        最后生成 TiDB Lightning Server 端证书：

        
        ``` shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=internal lightning-server.json | cfssljson -bare lightning-server
        ```

6. 生成 Client 端证书。

    首先生成默认的 `client.json` 文件：

    
    ``` shell
    cfssl print-defaults csr > client.json
    ```

    然后编辑这个文件，修改 `CN`，`hosts` 属性，`hosts` 可以留空：

    ``` json
    ...
        "CN": "TiDB",
        "hosts": [],
    ...
    ```

    最后生成 Client 端证书：

    ``` shell
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
    ```

7. 创建 Kubernetes Secret 对象。

    假设你已经按照上述文档为每个组件创建了一套 Server 端证书，并为各个客户端创建了一套 Client 端证书。通过下面的命令为 TiDB 集群创建这些 Secret 对象：

    PD 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-pd-cluster-secret --namespace=${namespace} --from-file=tls.crt=pd-server.pem --from-file=tls.key=pd-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiKV 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-tikv-cluster-secret --namespace=${namespace} --from-file=tls.crt=tikv-server.pem --from-file=tls.key=tikv-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiDB 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-tidb-cluster-secret --namespace=${namespace} --from-file=tls.crt=tidb-server.pem --from-file=tls.key=tidb-server-key.pem --from-file=ca.crt=ca.pem
    ```

    Pump 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-pump-cluster-secret --namespace=${namespace} --from-file=tls.crt=pump-server.pem --from-file=tls.key=pump-server-key.pem --from-file=ca.crt=ca.pem
    ```

    Drainer 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-drainer-cluster-secret --namespace=${namespace} --from-file=tls.crt=drainer-server.pem --from-file=tls.key=drainer-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiCDC 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-ticdc-cluster-secret --namespace=${namespace} --from-file=tls.crt=ticdc-server.pem --from-file=tls.key=ticdc-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiProxy 集群证书 Secret：

    ``` shell
    kubectl create secret generic ${cluster_name}-tiproxy-cluster-secret --namespace=${namespace} --from-file=tls.crt=tiproxy-server.pem --from-file=tls.key=tiproxy-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiFlash 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-tiflash-cluster-secret --namespace=${namespace} --from-file=tls.crt=tiflash-server.pem --from-file=tls.key=tiflash-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiKV Importer 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-importer-cluster-secret --namespace=${namespace} --from-file=tls.crt=importer-server.pem --from-file=tls.key=importer-server-key.pem --from-file=ca.crt=ca.pem
    ```

    TiDB Lightning 集群证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-lightning-cluster-secret --namespace=${namespace} --from-file=tls.crt=lightning-server.pem --from-file=tls.key=lightning-server-key.pem --from-file=ca.crt=ca.pem
    ```

    Client 证书 Secret：

    
    ``` shell
    kubectl create secret generic ${cluster_name}-cluster-client-secret --namespace=${namespace} --from-file=tls.crt=client.pem --from-file=tls.key=client-key.pem --from-file=ca.crt=ca.pem
    ```

    这里给 PD/TiKV/TiDB/Pump/Drainer 的 Server 端证书分别创建了一个 Secret 供他们启动时加载使用，另外一套 Client 端证书供他们的客户端连接使用。

### 使用 `cert-manager` 系统颁发证书

1. 安装 cert-manager。

    请参考官网安装：[cert-manager installation on Kubernetes](https://docs.cert-manager.io/en/release-0.11/getting-started/install/kubernetes.html)。

2. 创建一个 Issuer 用于给 TiDB 集群颁发证书。

    为了配置 `cert-manager` 颁发证书，必须先创建 Issuer 资源。

    首先创建一个目录保存 `cert-manager` 创建证书所需文件：

    
    ``` shell
    mkdir -p cert-manager
    cd cert-manager
    ```

    然后创建一个 `tidb-cluster-issuer.yaml` 文件，输入以下内容：

    ``` yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: ${cluster_name}-selfsigned-ca-issuer
      namespace: ${namespace}
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: ${cluster_name}-ca
      namespace: ${namespace}
    spec:
      secretName: ${cluster_name}-ca-secret
      commonName: "TiDB"
      isCA: true
      duration: 87600h # 10yrs
      renewBefore: 720h # 30d
      issuerRef:
        name: ${cluster_name}-selfsigned-ca-issuer
        kind: Issuer
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: ${cluster_name}-tidb-issuer
      namespace: ${namespace}
    spec:
      ca:
        secretName: ${cluster_name}-ca-secret
    ```

    其中 `${cluster_name}` 为集群的名字，上面的文件创建三个对象：

    - 一个 SelfSigned 类型的 Issuer 对象（用于生成 CA 类型 Issuer 所需要的 CA 证书）;
    - 一个 Certificate 对象，`isCa` 属性设置为 `true`；
    - 一个可以用于颁发 TiDB 组件间 TLS 证书的 Issuer。

    最后执行下面的命令进行创建：

    
    ``` shell
    kubectl apply -f tidb-cluster-issuer.yaml
    ```

3. 创建 Server 端证书。

    在 `cert-manager` 中，Certificate 资源表示证书接口，该证书将由上面创建的 Issuer 颁发并保持更新。

    根据官网文档：[Enable TLS Authentication](https://docs.pingcap.com/zh/tidb/stable/enable-tls-between-components)，我们需要为每个组件创建一个 Server 端证书，并且为他们的 Client 创建一套公用的 Client 端证书。

    - PD 组件的 Server 端证书。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-pd-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-pd-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-pd"
          - "${cluster_name}-pd.${namespace}"
          - "${cluster_name}-pd.${namespace}.svc"
          - "${cluster_name}-pd-peer"
          - "${cluster_name}-pd-peer.${namespace}"
          - "${cluster_name}-pd-peer.${namespace}.svc"
          - "*.${cluster_name}-pd-peer"
          - "*.${cluster_name}-pd-peer.${namespace}"
          - "*.${cluster_name}-pd-peer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-pd-cluster-secret`；
        - `usages` 请添加上  `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
          - `${cluster_name}-pd`
          - `${cluster_name}-pd.${namespace}`
          - `${cluster_name}-pd.${namespace}.svc`
          - `${cluster_name}-pd-pee`
          - `${cluster_name}-pd-peer.${namespace}`
          - `${cluster_name}-pd-peer.${namespace}.svc`
          - `*.${cluster_name}-pd-peer`
          - `*.${cluster_name}-pd-peer.${namespace}`
          - `*.${cluster_name}-pd-peer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-pd-cluster-secret` 的 Secret 对象供 TiDB 集群的 PD 组件使用。

    - TiKV 组件的 Server 端证书。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-tikv-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-tikv-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-tikv"
          - "${cluster_name}-tikv.${namespace}"
          - "${cluster_name}-tikv.${namespace}.svc"
          - "${cluster_name}-tikv-peer"
          - "${cluster_name}-tikv-peer.${namespace}"
          - "${cluster_name}-tikv-peer.${namespace}.svc"
          - "*.${cluster_name}-tikv-peer"
          - "*.${cluster_name}-tikv-peer.${namespace}"
          - "*.${cluster_name}-tikv-peer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-tikv-cluster-secret`；
        - `usages` 请添加上  `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
          - `${cluster_name}-tikv`
          - `${cluster_name}-tikv.${namespace}`
          - `${cluster_name}-tikv.${namespace}.svc`
          - `${cluster_name}-tikv-peer`
          - `${cluster_name}-tikv-peer.${namespace}`
          - `${cluster_name}-tikv-peer.${namespace}.svc`
          - `*.${cluster_name}-tikv-peer`
          - `*.${cluster_name}-tikv-peer.${namespace}`
          - `*.${cluster_name}-tikv-peer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-tikv-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiKV 组件使用。

    - TiDB 组件的 Server 端证书。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-tidb-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-tidb-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-tidb"
          - "${cluster_name}-tidb.${namespace}"
          - "${cluster_name}-tidb.${namespace}.svc"
          - "${cluster_name}-tidb-peer"
          - "${cluster_name}-tidb-peer.${namespace}"
          - "${cluster_name}-tidb-peer.${namespace}.svc"
          - "*.${cluster_name}-tidb-peer"
          - "*.${cluster_name}-tidb-peer.${namespace}"
          - "*.${cluster_name}-tidb-peer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-tidb-cluster-secret`；
        - `usages` 请添加上  `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
          - `${cluster_name}-tidb`
          - `${cluster_name}-tidb.${namespace}`
          - `${cluster_name}-tidb.${namespace}.svc`
          - `${cluster_name}-tidb-peer`
          - `${cluster_name}-tidb-peer.${namespace}`
          - `${cluster_name}-tidb-peer.${namespace}.svc`
          - `*.${cluster_name}-tidb-peer`
          - `*.${cluster_name}-tidb-peer.${namespace}`
          - `*.${cluster_name}-tidb-peer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-tidb-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiDB 组件使用。

    - Pump 组件的 Server 端证书。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-pump-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-pump-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "*.${cluster_name}-pump"
          - "*.${cluster_name}-pump.${namespace}"
          - "*.${cluster_name}-pump.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-pump-cluster-secret`；
        - `usages` 请添加上  `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
          - `*.${cluster_name}-pump`
          - `*.${cluster_name}-pump.${namespace}`
          - `*.${cluster_name}-pump.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-pump-cluster-secret` 的 Secret 对象供 TiDB 集群的 Pump 组件使用。

    - Drainer 组件的 Server 端证书。

        现在 Drainer 组件是通过 Helm 来部署的，根据 `values.yaml` 文件配置方式不同，所需要填写的 `dnsNames` 字段也不相同。

        如果部署的时候设置 `drainerName` 属性，像下面这样：

        ``` yaml
        ...
        # Change the name of the statefulset and pod
        # The default is clusterName-ReleaseName-drainer
        # Do not change the name of an existing running drainer: this is unsupported.
        drainerName: my-drainer
        ...
        ```

        那么就需要这样配置证书：

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-drainer-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-drainer-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "*.${drainer_name}"
          - "*.${drainer_name}.${namespace}"
          - "*.${drainer_name}.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        如果部署的时候没有设置 `drainerName` 属性，需要这样配置 `dnsNames` 属性：

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-drainer-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-drainer-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "*.${cluster_name}-${release_name}-drainer"
          - "*.${cluster_name}-${release_name}-drainer.${namespace}"
          - "*.${cluster_name}-${release_name}-drainer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字，`${namespace}` 为 TiDB 集群部署的命名空间，`${release_name}` 是 `helm install` 时候填写的 `release name`，`${drainer_name}` 为 `values.yaml` 文件里的 `drainerName`，用户也可以添加自定义 `dnsNames`。

        - `spec.secretName` 请设置为 `${cluster_name}-drainer-cluster-secret`；
        - `usages` 请添加上  `server auth` 和 `client auth`；
        - `dnsNames` 请参考上面的描述；
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-drainer-cluster-secret` 的 Secret 对象供 TiDB 集群的 Drainer 组件使用。

    - TiCDC 组件的 Server 端证书。

      TiCDC 从 v4.0.3 版本开始支持 TLS，TiDB Operator v1.1.3 版本同步支持 TiCDC 开启 TLS 功能。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-ticdc-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-ticdc-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-ticdc"
          - "${cluster_name}-ticdc.${namespace}"
          - "${cluster_name}-ticdc.${namespace}.svc"
          - "${cluster_name}-ticdc-peer"
          - "${cluster_name}-ticdc-peer.${namespace}"
          - "${cluster_name}-ticdc-peer.${namespace}.svc"
          - "*.${cluster_name}-ticdc-peer"
          - "*.${cluster_name}-ticdc-peer.${namespace}"
          - "*.${cluster_name}-ticdc-peer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-ticdc-cluster-secret`；
        - `usages` 请添加上 `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
          - `${cluster_name}-ticdc`
          - `${cluster_name}-ticdc.${namespace}`
          - `${cluster_name}-ticdc.${namespace}.svc`
          - `${cluster_name}-ticdc-peer`
          - `${cluster_name}-ticdc-peer.${namespace}`
          - `${cluster_name}-ticdc-peer.${namespace}.svc`
          - `*.${cluster_name}-ticdc-peer`
          - `*.${cluster_name}-ticdc-peer.${namespace}`
          - `*.${cluster_name}-ticdc-peer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP，根据需要可以填写其他 IP：
          - `127.0.0.1`
          - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-ticdc-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiCDC 组件使用。

    - TiFlash 组件的 Server 端证书。

        ```yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-tiflash-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-tiflash-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-tiflash"
          - "${cluster_name}-tiflash.${namespace}"
          - "${cluster_name}-tiflash.${namespace}.svc"
          - "${cluster_name}-tiflash-peer"
          - "${cluster_name}-tiflash-peer.${namespace}"
          - "${cluster_name}-tiflash-peer.${namespace}.svc"
          - "*.${cluster_name}-tiflash-peer"
          - "*.${cluster_name}-tiflash-peer.${namespace}"
          - "*.${cluster_name}-tiflash-peer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-tiflash-cluster-secret`；
        - `usages` 请添加上 `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
            - `${cluster_name}-tiflash`
            - `${cluster_name}-tiflash.${namespace}`
            - `${cluster_name}-tiflash.${namespace}.svc`
            - `${cluster_name}-tiflash-peer`
            - `${cluster_name}-tiflash-peer.${namespace}`
            - `${cluster_name}-tiflash-peer.${namespace}.svc`
            - `*.${cluster_name}-tiflash-peer`
            - `*.${cluster_name}-tiflash-peer.${namespace}`
            - `*.${cluster_name}-tiflash-peer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
            - `127.0.0.1`
            - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-tiflash-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiFlash 组件使用。

    - TiKV Importer 组件的 Server 端证书。

      如需要[使用 TiDB Lightning 恢复 Kubernetes 上的集群数据](restore-data-using-tidb-lightning.md)，则需要为其中的 TiKV Importer 组件生成如下的 Server 端证书。

        ```yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-importer-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-importer-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-importer"
          - "${cluster_name}-importer.${namespace}"
          - "${cluster_name}-importer.${namespace}.svc"
          - "*.${cluster_name}-importer"
          - "*.${cluster_name}-importer.${namespace}"
          - "*.${cluster_name}-importer.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-importer-cluster-secret`；
        - `usages` 请添加上 `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
            - `${cluster_name}-importer`
            - `${cluster_name}-importer.${namespace}`
            - `${cluster_name}-importer.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
            - `127.0.0.1`
            - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-importer-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiKV Importer 组件使用。

    - TiDB Lightning 组件的 Server 端证书。

      如需要[使用 TiDB Lightning 恢复 Kubernetes 上的集群数据](restore-data-using-tidb-lightning.md)，则需要为其中的 TiDB Lightning 组件生成如下的 Server 端证书。

        ```yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-lightning-cluster-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-lightning-cluster-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - server auth
            - client auth
          dnsNames:
          - "${cluster_name}-lightning"
          - "${cluster_name}-lightning.${namespace}"
          - "${cluster_name}-lightning.${namespace}.svc"
          ipAddresses:
          - 127.0.0.1
          - ::1
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-lightning-cluster-secret`；
        - `usages` 请添加上 `server auth` 和 `client auth`；
        - `dnsNames` 需要填写这些 DNS，根据需要可以填写其他 DNS：
            - `${cluster_name}-lightning`
            - `${cluster_name}-lightning.${namespace}`
            - `${cluster_name}-lightning.${namespace}.svc`
        - `ipAddresses` 需要填写这两个 IP ，根据需要可以填写其他 IP：
            - `127.0.0.1`
            - `::1`
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-lightning-cluster-secret` 的 Secret 对象供 TiDB 集群的 TiDB Lightning 组件使用。

    - 一套 TiDB 集群组件的 Client 端证书。

        ``` yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: ${cluster_name}-cluster-client-secret
          namespace: ${namespace}
        spec:
          secretName: ${cluster_name}-cluster-client-secret
          duration: 8760h # 365d
          renewBefore: 360h # 15d
          subject:
            organizations:
            - PingCAP
          commonName: "TiDB"
          usages:
            - client auth
          issuerRef:
            name: ${cluster_name}-tidb-issuer
            kind: Issuer
            group: cert-manager.io
        ```

        其中 `${cluster_name}` 为集群的名字：

        - `spec.secretName` 请设置为 `${cluster_name}-cluster-client-secret`；
        - `usages` 请添加上  `client auth`；
        - `dnsNames` 和 `ipAddresses` 不需要填写；
        - `issuerRef` 请填写上面创建的 Issuer；
        - 其他属性请参考 [cert-manager API](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec)。

        创建这个对象以后，`cert-manager` 会生成一个名字为 `${cluster_name}-cluster-client-secret` 的 Secret 对象供 TiDB 组件的 Client 使用。

## 第二步：部署 TiDB 集群

在部署 TiDB 集群时，可以开启集群间的 TLS，同时可以设置 `cert-allowed-cn` 配置项（TiDB 为 `cluster-verify-cn`），用来验证集群间各组件证书的 CN (Common Name)。

> **注意：**
>
> 目前 PD 的 `cert-allowed-cn` 配置项只能设置一个值。因此所有 `Certificate` 对象的 `commonName` 都要设置成同样一个值。

在这一步中，需要完成以下操作：

- 创建一套 TiDB 集群
- 为 TiDB 组件间开启 TLS，并开启 CN 验证
- 部署一套监控系统
- 部署 Pump 组件，并开启 CN 验证

1. 创建一套 TiDB 集群（监控系统和 Pump 组件已包含在内）：

    创建 `tidb-cluster.yaml` 文件：

    
    ``` yaml
    apiVersion: pingcap.com/v1alpha1
    kind: TidbCluster
    metadata:
     name: ${cluster_name}
     namespace: ${namespace}
    spec:
     tlsCluster:
       enabled: true
     version: v8.5.2
     timezone: UTC
     pvReclaimPolicy: Retain
     pd:
       baseImage: pingcap/pd
       maxFailoverCount: 0
       replicas: 1
       requests:
         storage: "10Gi"
       config:
         security:
           cert-allowed-cn:
             - TiDB
     tikv:
       baseImage: pingcap/tikv
       maxFailoverCount: 0
       replicas: 1
       requests:
         storage: "100Gi"
       config:
         security:
           cert-allowed-cn:
             - TiDB
     tidb:
       baseImage: pingcap/tidb
       maxFailoverCount: 0
       replicas: 1
       service:
         type: ClusterIP
       config:
         security:
           cluster-verify-cn:
             - TiDB
     pump:
       baseImage: pingcap/tidb-binlog
       replicas: 1
       requests:
         storage: "100Gi"
       config:
         security:
           cert-allowed-cn:
             - TiDB
    ---
    apiVersion: pingcap.com/v1alpha1
    kind: TidbMonitor
    metadata:
     name: ${cluster_name}
     namespace: ${namespace}
    spec:
     clusters:
     - name: ${cluster_name}
     prometheus:
       baseImage: prom/prometheus
       version: v2.27.1
     grafana:
       baseImage: grafana/grafana
       version: 7.5.11
     initializer:
       baseImage: pingcap/tidb-monitor-initializer
       version: v8.5.2
     reloader:
       baseImage: pingcap/tidb-monitor-reloader
       version: v1.0.1
      prometheusReloader:
       baseImage: quay.io/prometheus-operator/prometheus-config-reloader
       version: v0.49.0
     imagePullPolicy: IfNotPresent
    ```

    然后使用 `kubectl apply -f tidb-cluster.yaml` 来创建 TiDB 集群。

    > **注意：**
    >
    > PD 从 v8.0.0 版本开始支持[微服务模式](https://docs.pingcap.com/zh/tidb/dev/pd-microservices)（实验特性），如需部署 PD 微服务，需要为各个微服务配置 `cert-allowed-cn`。以 Scheduling 服务为例，你需要进行以下配置：
    >
    > - 更新 `pd.mode` 为 `ms`
    > - 为 `scheduling` 微服务配置 `security` 字段
    >
    > ```yaml
    >   pd:
    >    baseImage: pingcap/pd
    >    maxFailoverCount: 0
    >    replicas: 1
    >    requests:
    >     storage: "10Gi"
    >    config:
    >     security:
    >       cert-allowed-cn:
    >         - TiDB
    >    mode: "ms"
    >   pdms:
    >   - name: "scheduling"
    >     baseImage: pingcap/pd
    >     replicas: 1
    >     config:
    >       security:
    >         cert-allowed-cn:
    >           - TiDB
    > ```

2. 创建 Drainer 组件并开启 TLS 以及 CN 验证。

    - 第一种方式：创建 Drainer 的时候设置 `drainerName`：

        编辑 values.yaml 文件，设置好 drainer-name，并将 TLS 功能打开：

        ``` yaml
        ...
        drainerName: ${drainer_name}
        tlsCluster:
          enabled: true
          certAllowedCN:
            - TiDB
        ...
        ```

        然后部署 Drainer 集群：

        
        ``` shell
        helm install ${release_name} pingcap/tidb-drainer --namespace=${namespace} --version=${helm_version} -f values.yaml
        ```

    - 第二种方式：创建 Drainer 的时候不设置 `drainerName`：

        编辑 values.yaml 文件，将 TLS 功能打开：

        ``` yaml
        ...
        tlsCluster:
          enabled: true
          certAllowedCN:
            - TiDB
        ...
        ```

        然后部署 Drainer 集群：

        
        ``` shell
        helm install ${release_name} pingcap/tidb-drainer --namespace=${namespace} --version=${helm_version} -f values.yaml
        ```

3. 创建 Backup/Restore 资源对象。

    - 创建 `backup.yaml` 文件：

        ``` yaml
        apiVersion: pingcap.com/v1alpha1
        kind: Backup
        metadata:
          name: ${cluster_name}-backup
          namespace: ${namespace}
        spec:
          backupType: full
          br:
            cluster: ${cluster_name}
            clusterNamespace: ${namespace}
            sendCredToTikv: true
          s3:
            provider: aws
            region: ${my_region}
            secretName: ${s3_secret}
            bucket: ${my_bucket}
            prefix: ${my_folder}
        ```

        然后部署 Backup：

        
        ``` shell
        kubectl apply -f backup.yaml
        ```

    - 创建 `restore.yaml` 文件：

        ``` yaml
        apiVersion: pingcap.com/v1alpha1
        kind: Restore
        metadata:
          name: ${cluster_name}-restore
          namespace: ${namespace}
        spec:
          backupType: full
          br:
            cluster: ${cluster_name}
            clusterNamespace: ${namespace}
            sendCredToTikv: true
          s3:
            provider: aws
            region: ${my_region}
            secretName: ${s3_secret}
            bucket: ${my_bucket}
            prefix: ${my_folder}

        ```

        然后部署 Restore：

        
        ``` shell
        kubectl apply -f restore.yaml
        ```

## 第三步：配置 `pd-ctl`、`tikv-ctl` 连接集群

1. 挂载证书。

    通过下面命令配置 `spec.pd.mountClusterClientSecret: true` 和 `spec.tikv.mountClusterClientSecret: true`：

    
    ``` shell
    kubectl patch tc ${cluster_name} -n ${namespace} --type merge -p '{"spec":{"pd":{"mountClusterClientSecret": true},"tikv":{"mountClusterClientSecret": true}}}'
    ```

    > **注意：**
    >
    > * 上面配置改动会滚动升级 PD 和 TiKV 集群。
    > * 上面配置从 TiDB Operator v1.1.5 开始支持。

2. 使用 `pd-ctl` 连接集群。

    进入 PD Pod：

    
    ``` shell
    kubectl exec -it ${cluster_name}-pd-0 -n ${namespace} sh
    ```

    使用 `pd-ctl`：

    
    ``` shell
    cd /var/lib/cluster-client-tls
    /pd-ctl --cacert=ca.crt --cert=tls.crt --key=tls.key -u https://127.0.0.1:2379 member
    ```

3. 使用 `tikv-ctl` 连接集群。

    进入 TiKV Pod：

    
    ``` shell
    kubectl exec -it ${cluster_name}-tikv-0 -n ${namespace} sh
    ```

    使用 `tikv-ctl`：

    
    ``` shell
    cd /var/lib/cluster-client-tls
    /tikv-ctl --ca-path=ca.crt --cert-path=tls.crt --key-path=tls.key --host 127.0.0.1:20160 cluster
    ```

## 将非 TLS 集群升级为 TLS 集群

本节介绍如何为现有的非 TLS TiDB 集群启用 TLS 加密通信。

> **注意：**
>
> 该操作仅适用于无法重建的现有集群。在开始操作前，请确保已充分理解每个步骤及其潜在风险。

1. 如果集群包含多个 PD 节点，需要先将 PD 节点数量缩减为 1 个。

2. 参考[第一步：为 TiDB 集群各个组件生成证书](#第一步为-tidb-集群各个组件生成证书)生成 TLS 证书，并创建 Kubernetes Secret 对象。

3. 启用 TLS：

    你可以选择以下两种方法之一启用 TLS：

    - 方法 1：执行以下命令更新 TiDB 集群配置，等待 PD Pod 完成重启后继续下一步。

        ``` shell
        kubectl patch tc ${cluster_name} -n ${namespace} --type merge -p '{
          "spec": {
            "tlsCluster": {
              "enabled": true
            }
          }
        }'
        ```

        输出示例：

        ``` shell
        tidbcluster.pingcap.com/basic patched
        ```

    - 方法 2：参考[第二步：部署 TiDB 集群](#第二步部署-tidb-集群)启用 TLS，同时设置 `cert-allowed-cn` 配置项（TiDB 为 `cluster-verify-cn`），用于验证集群间各组件证书的 CN (Common Name)。

4. 配置 PD 节点：

    1. 使用 `kubectl exec` 进入 PD Pod 并安装 etcdctl。详细安装步骤可参考 [etcdctl 安装指南](https://etcd.io/docs/v3.4/install/)。安装完成后，etcdctl 位于解压后的文件夹目录下。

    2. 查看 etcd 成员信息，此时 `peerURLs` 使用 HTTP 协议：

        ``` shell
        ./etcdctl --endpoints https://127.0.0.1:2379 --cert /var/lib/pd-tls/tls.crt --key /var/lib/pd-tls/tls.key --cacert /var/lib/pd-tls/ca.crt member list
        ```

        输出示例：

        ``` shell
        # memberID        status   name        peerURLs                                          clientURL                                          isLearner
        e94cfb12fa384e23, started, basic-pd-0, http://basic-pd-0.basic-pd-peer.pingcap.svc:2380, https://basic-pd-0.basic-pd-peer.pingcap.svc:2379, false
        ```

        记录以下信息用于下一步操作：
    
        - `memberID`：示例中为 `e94cfb12fa384e23`。
        - `peerURLs`：示例中为 `http://basic-pd-0.basic-pd-peer.pingcap.svc:2380`。

    3. 将 etcd member 的 `peerURLs` 从 HTTP 更新为 HTTPS 协议：

        ``` shell
        ./etcdctl --endpoints https://127.0.0.1:2379 --cert /var/lib/pd-tls/tls.crt --key /var/lib/pd-tls/tls.key --cacert /var/lib/pd-tls/ca.crt member update e94cfb12fa384e23 --peer-urls="https://basic-pd-0.basic-pd-peer.pingcap.svc:2380"
        ```

        输出示例：

        ``` shell
        Member e94cfb12fa384e23 updated in cluster 32ab5936d81ad54c
        ```

    4. 查看更新后的 `peerURLs`，确保已更新为 HTTPS 协议：

        ``` shell
        ./etcdctl --endpoints https://127.0.0.1:2379 --cert /var/lib/pd-tls/tls.crt --key /var/lib/pd-tls/tls.key --cacert /var/lib/pd-tls/ca.crt member list
        ```

        输出示例：

        ``` shell
        e94cfb12fa384e23, started, basic-pd-0, https://basic-pd-0.basic-pd-peer.pingcap.svc:2380, https://basic-pd-0.basic-pd-peer.pingcap.svc:2379, false
        ```

5. 如果之前进行了 PD 节点缩容，请将其扩容为原有数量。

6. 等待 TiDB 集群中的所有 Pod 完成重启。
