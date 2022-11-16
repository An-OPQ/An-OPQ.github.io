---
title: K8S部署Kuboard
cover: https://w.wallhaven.cc/full/k7/wallhaven-k7mww1.jpg
tag: K8S
---

# K8S部署Kuboard

ps：使用的是Gitlab版哦，若不是可以直接注释对应的模块yaml就行

https://kuboard.cn/install/v3/install-in-k8s.html#%E5%AE%89%E8%A3%85

官方网站:⬆︎⬆︎⬆︎

以下文章由于个人学习与记录，若有不懂 请阅读官方手册

- 使用hostPath持久化

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: kuboard

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kuboard-v3-config
  namespace: kuboard
data:
  # 关于如下参数的解释，请参考文档 https://kuboard.cn/install/v3/install-built-in.html
  # [common]
  KUBOARD_SERVER_NODE_PORT: '30080'
  KUBOARD_AGENT_SERVER_UDP_PORT: '30081'
  KUBOARD_AGENT_SERVER_TCP_PORT: '30081'
  KUBOARD_SERVER_LOGRUS_LEVEL: info  # error / debug / trace
  # KUBOARD_AGENT_KEY 是 Agent 与 Kuboard 通信时的密钥，请修改为一个任意的包含字母、数字的32位字符串，此密钥变更后，需要删除 Kuboard Agent 重新导入。
  KUBOARD_AGENT_KEY: 32b7d6572c6255211b4eec9009e4a816
  KUBOARD_AGENT_IMAG: eipwork/kuboard-agent
  KUBOARD_QUESTDB_IMAGE: questdb/questdb:6.0.5
  KUBOARD_DISABLE_AUDIT: 'false' # 如果要禁用 Kuboard 审计功能，将此参数的值设置为 'true'，必须带引号。

  # 关于如下参数的解释，请参考文档 https://kuboard.cn/install/v3/install-gitlab.html
  # [gitlab login]
  #KUBOARD_LOGIN_TYPE: "gitlab"
  #KUBOARD_ROOT_USER: "your-user-name-in-gitlab" # kuboard——root用户，添加自己的gitlab账户，非邮箱哦！
  #GITLAB_BASE_URL: "gitlab——URL"
  #GITLAB_APPLICATION_ID: "ak"
  #GITLAB_CLIENT_SECRET: "sk"
  #SSO_REDIRECT_URI: "http://公网or内网ip:30080/sso/callback"
  
  # 关于如下参数的解释，请参考文档 https://kuboard.cn/install/v3/install-github.html
  # [github login]
  # KUBOARD_LOGIN_TYPE: "github"
  # KUBOARD_ROOT_USER: "your-user-name-in-github"
  # GITHUB_CLIENT_ID: "17577d45e4de7dad88e0"
  # GITHUB_CLIENT_SECRET: "ff738553a8c7e9ad39569c8d02c1d85ec19115a7"

  # 关于如下参数的解释，请参考文档 https://kuboard.cn/install/v3/install-ldap.html
  # [ldap login]
  # KUBOARD_LOGIN_TYPE: "ldap"
  # KUBOARD_ROOT_USER: "your-user-name-in-ldap"
  # LDAP_HOST: "ldap-ip-address:389"
  # LDAP_BIND_DN: "cn=admin,dc=example,dc=org"
  # LDAP_BIND_PASSWORD: "admin"
  # LDAP_BASE_DN: "dc=example,dc=org"
  # LDAP_FILTER: "(objectClass=posixAccount)"
  # LDAP_ID_ATTRIBUTE: "uid"
  # LDAP_USER_NAME_ATTRIBUTE: "uid"
  # LDAP_EMAIL_ATTRIBUTE: "mail"
  # LDAP_DISPLAY_NAME_ATTRIBUTE: "cn"
  # LDAP_GROUP_SEARCH_BASE_DN: "dc=example,dc=org"
  # LDAP_GROUP_SEARCH_FILTER: "(objectClass=posixGroup)"
  # LDAP_USER_MACHER_USER_ATTRIBUTE: "gidNumber"
  # LDAP_USER_MACHER_GROUP_ATTRIBUTE: "gidNumber"
  # LDAP_GROUP_NAME_ATTRIBUTE: "cn"

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kuboard-boostrap
  namespace: kuboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-boostrap-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kuboard-boostrap
  namespace: kuboard

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s.kuboard.cn/name: kuboard-etcd
  name: kuboard-etcd
  namespace: kuboard
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s.kuboard.cn/name: kuboard-etcd
  template:
    metadata:
      labels:
        k8s.kuboard.cn/name: kuboard-etcd
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/master
                    operator: Exists
              - matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: Exists
              - matchExpressions:
                  - key: k8s.kuboard.cn/role
                    operator: In
                    values:
                      - etcd
      containers:
        - env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
            - name: HOSTIP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.hostIP
          image: 'eipwork/etcd-host:3.4.16-2'
          imagePullPolicy: Always
          name: etcd
          ports:
            - containerPort: 2381
              hostPort: 2381
              name: server
              protocol: TCP
            - containerPort: 2382
              hostPort: 2382
              name: peer
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /health
              port: 2381
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          volumeMounts:
            - mountPath: /data
              name: data
      dnsPolicy: ClusterFirst
      hostNetwork: true
      restartPolicy: Always
      serviceAccount: kuboard-boostrap
      serviceAccountName: kuboard-boostrap
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
      volumes:
        - hostPath:
            path: /usr/share/kuboard/etcd
          name: data
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate


---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    k8s.kuboard.cn/name: kuboard-v3
  name: kuboard-v3
  namespace: kuboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s.kuboard.cn/name: kuboard-v3
  template:
    metadata:
      labels:
        k8s.kuboard.cn/name: kuboard-v3
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/master
                    operator: Exists
              weight: 100
            - preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: Exists
              weight: 100
      containers:
        - env:
            - name: HOSTIP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.hostIP
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          envFrom:
            - configMapRef:
                name: kuboard-v3-config
          image: 'eipwork/kuboard:v3'
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /kuboard-resources/version.json
              port: 80
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          name: kuboard
          ports:
            - containerPort: 80
              name: web
              protocol: TCP
            - containerPort: 443
              name: https
              protocol: TCP
            - containerPort: 10081
              name: peer
              protocol: TCP
            - containerPort: 10081
              name: peer-u
              protocol: UDP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /kuboard-resources/version.json
              port: 80
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources: {}
          # startupProbe:
          #   failureThreshold: 20
          #   httpGet:
          #     path: /kuboard-resources/version.json
          #     port: 80
          #     scheme: HTTP
          #   initialDelaySeconds: 5
          #   periodSeconds: 10
          #   successThreshold: 1
          #   timeoutSeconds: 1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      serviceAccount: kuboard-boostrap
      serviceAccountName: kuboard-boostrap
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists

---
apiVersion: v1
kind: Service
metadata:
  annotations: {}
  labels:
    k8s.kuboard.cn/name: kuboard-v3
  name: kuboard-v3
  namespace: kuboard
spec:
  ports:
    - name: web
      nodePort: 30080
      port: 80
      protocol: TCP
      targetPort: 80
    - name: tcp
      nodePort: 30081
      port: 10081
      protocol: TCP
      targetPort: 10081
    - name: udp
      nodePort: 30081
      port: 10081
      protocol: UDP
      targetPort: 10081
  selector:
    k8s.kuboard.cn/name: kuboard-v3
  sessionAffinity: None
  type: NodePort

```

> - Kuboard V3 依赖于 etcd 提供数据的持久化服务，在当前的安装方式下，kuboard-etcd 的存储卷被映射到宿主机节点的 hostPath (/usr/share/kuboard/etcd 目录)；
> - 为了确保每次重启，etcd 能够加载到原来的数据，以 DaemonSet 的形式部署 kuboard-etcd，并且其容器组将始终被调度到 master 节点，因此，您有多少个 master 节点，就会调度多少个 kuboard-etcd 的实例；
> - 某些情况下，您的 master 节点只有一个或者两个，却仍然想要保证 kubuoard-etcd 的高可用，此时，您可以通过为一到两个 worker 节点添加 `k8s.kuboard.cn/role=etcd` 的标签，来增加 kuboard-etcd 的实例数量；



> 打开浏览器访问： http://your-node-ip-address:30080
>
> 会需要你登录Gitlab然后进行跳转到Kuboard上