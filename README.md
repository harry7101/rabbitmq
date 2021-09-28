2. Rabbitmq-peer-discovery-k8s
Github地址    https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s

参考官方给出的minikube示例即可



2.1 ConfigMap
---
# RabbitMQ Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
data:
  enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
  rabbitmq.conf: |
      cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      cluster_formation.node_cleanup.interval = 30
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      queue_master_locator=min-masters
      loopback_users.guest = false
---
2.2 RBAC
---
# RabbitMQ ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
---
# RabbitMQ Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: endpoint-reader
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get"]
---
# RabbitMQ RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: endpoint-reader
subjects:
  - kind: ServiceAccount
    name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: endpoint-reader
2.3 Service
---
# RabbitMQ Service
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-cluster
  labels:
    app: rabbitmq-cluster
    type: LoadBalancer
spec:
  type: NodePort
  selector:
    app: rabbitmq-cluster
  ports:
    - name: amqp-port
      nodePort: 30001
      port: 5672
      targetPort: 5672
      protocol: TCP
    - name: mgmt-port
      nodePort: 30002
      port: 15672
      targetPort: 15672
      protocol: TCP


2.4 StatefulSet
# RabbitMQ-Cluster StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq-cluster
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq-cluster
  serviceName: rabbitmq-cluster
  template:
    metadata:
      labels:
        app: rabbitmq-cluster
    spec:
      serviceAccountName: rabbitmq
      containers:
        - name: rabbitmq
          image: rabbitmq:3
          livenessProbe:
            exec:
              # Stage 2 check, more detail at https://www.rabbitmq.com/monitoring.html#health-checks
              command: ["rabbitmq-diagnostics", "status"]
            initialDelaySeconds: 60
            periodSeconds: 60
            timeoutSeconds: 10
          readinessProbe:
            exec:
              # Stage 2 check, more detail at https://www.rabbitmq.com/monitoring.html#health-checks
              command: ["rabbitmq-diagnostics", "status"]
            initialDelaySeconds: 60
            periodSeconds: 60
            timeoutSeconds: 10
          ports:
            - containerPort: 5672
              protocol: TCP
            - containerPort: 15672
              protocol: TCP
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # get pod.metadata.name, e.g. rabbitmq-cluster-0
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace  # get pod.metadata.namespace
            - name: RABBITMQ_DEFAULT_USER
              value: "admin"
            - name: RABBITMQ_DEFAULT_PASS
              value: "admin"
            - name: RABBITMQ_USE_LONGNAME
              value: "true"
            - name: K8S_SERVICE_NAME
              value: "rabbitmq-cluster"
            - name: RABBITMQ_NODENAME
              value: "rabbit@$(MY_POD_NAME).$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local"
            - name: K8S_HOSTNAME_SUFFIX
              value: .$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
            - name: RABBITMQ_ERLANG_COOKIE
              value: "SWvCP0Hrqv43NG7GybHC95ntCJKoW8UyNFWnBEWG8TY="    # generator by: echo $(openssl rand -base64 32)
          volumeMounts:
            - name: config-volume
              mountPath: /etc/rabbitmq
      volumes:
        - name: config-volume
          configMap:
            name: rabbitmq-config
            items:
              - key: rabbitmq.conf
                path: rabbitmq.conf
              - key: enabled_plugins
                path: enabled_plugins
3. 运行效果
部署了3个pod


 

 
查看Pod日志可以看到，RabbitMQ自动发现节点并加入集群



查看RabbitMQ Cluster状态

root@rabbitmq-cluster-0:/# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq-cluster-0.rabbitmq-internal.demo.svc.cluster.local ...
Basics

Cluster name: rabbit@rabbitmq-cluster-0.rabbitmq-internal.demo.svc.cluster.local

Disk Nodes

rabbit@rabbitmq-cluster-0.rabbitmq-internal.demo.svc.cluster.local
rabbit@rabbitmq-cluster-1.rabbitmq-internal.demo.svc.cluster.local
rabbit@rabbitmq-cluster-2.rabbitmq-internal.demo.svc.cluster.local

Running Nodes

rabbit@rabbitmq-cluster-0.rabbitmq-internal.demo.svc.cluster.local
rabbit@rabbitmq-cluster-1.rabbitmq-internal.demo.svc.cluster.local
rabbit@rabbitmq-cluster-2.rabbitmq-internal.demo.svc.cluster.local

Versions

rabbit@rabbitmq-cluster-0.rabbitmq-internal.demo.svc.cluster.local: RabbitMQ 3.8.5 on Erlang 23.0.3
rabbit@rabbitmq-cluster-1.rabbitmq-internal.demo.svc.cluster.local: RabbitMQ 3.8.5 on Erlang 23.0.3
rabbit@rabbitmq-cluster-2.rabbitmq-internal.demo.svc.cluster.local: RabbitMQ 3.8.5 on Erlang 23.0.3

Alarms

(none)

Network Partitions

(none)

Listeners
