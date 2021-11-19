
https://github.com/kubernetes/cloud-provider-alibaba-cloud/blob/master/docs/getting-started.md
1.创建ccm用到的cm
mkdir slb
cd slb
AccessKeyID=
AcceessKeySecret=
AccessKeyID-base64=`echo -n "$AccessKeyID" |base64`
AcceessKeySecret-base64=`echo -n "$AcceessKeySecret"|base64`
vim cloud-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-config
  namespace: kube-system
data:
  cloud-config.conf: |-
    {
        "Global": {
            "accessKeyID": "$AccessKeyID",
            "accessKeySecret": "$AcceessKeySecret-base64"
        }
    }
kubectl apply -f cloud-config.yaml
2.获取ccm用到的元数据
curl 100.100.100.200/latest/meta-data/hostname
curl 100.100.100.200/latest/meta-data/instance-id
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_CLOUD_PROVIDER_ARGS=--cloud-provider=external --hostname-override=iZj6c3ydyj9t4ztmha08rbZ --provider-id=cn-hongkong.i-j6c3ydyj9t4ztmha08rb"
Environment="--system-reserved=memory=300Mi --kube-reserved=memory=400Mi --eviction-hard=imagefs.available<15%,memory.available<300Mi,nodefs.available<10%,nodefs.inodesFree<5% --cgroup-driver=systemd"
$KUBELET_CLOUD_PROVIDER_ARGS $KUBELET_CGROUP_ARGS
systemctl daemon-reload
systemctl restart kubelet
3.修改kube-apiserver
vim /etc/kubernetes/manifests/kube-apiserver.yaml
- --cloud-provider=external
4.获取证书
cat /etc/kubernetes/pki/ca.crt|base64 -w 0
vim /etc/kubernetes/cloud-controller-manager.conf
kind: Config
contexts:
- context:
    cluster: kubernetes
    user: system:cloud-controller-manager
  name: system:cloud-controller-manager@kubernetes
current-context: system:cloud-controller-manager@kubernetes
users:
- name: system:cloud-controller-manager
  user:
    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $ca.crt
    server: https://172.16.1.193:6443
  name: kubernetes
5.创建ds
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:cloud-controller-manager
rules:
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
      - delete
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
      - update
      - patch
  - apiGroups:
      - ""
    resources:
      - services/status
    verbs:
      - update
      - patch
  - apiGroups:
    - ""
    resources:
    - serviceaccounts
    verbs:
    - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
      - list
      - watch
      - create
      - patch
      - update
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs:
      - get
      - list
      - update
      - create
  - apiGroups:
      - apiextensions.k8s.io
    resources:
      - customresourcedefinitions
    verbs:
      - get
      - update
      - create
      - delete

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:cloud-controller-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:cloud-controller-manager
subjects:
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: kube-system

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: cloud-controller-manager
    tier: control-plane
  name: cloud-controller-manager
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: cloud-controller-manager
      tier: control-plane
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: cloud-controller-manager
        tier: control-plane
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
    spec:
      serviceAccountName: cloud-controller-manager
      tolerations:
      - operator: Exists
      nodeSelector:
        node-role.kubernetes.io/master: ""
      containers:
      - name: cloud-controller-manager
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1200 
        command:
        -  /cloud-controller-manager
        - --kubeconfig=/etc/kubernetes/cloud-controller-manager.conf
        - --cloud-config=/etc/kubernetes/config/cloud-config.conf
        - --metrics-bind-addr=0

        #For terway configuration
        - --configure-cloud-routes=false

        image: registry-vpc.cn-shanghai.aliyuncs.com/acs/cloud-controller-manager-amd64:v2.0.1
        livenessProbe:
          failureThreshold: 8
          httpGet:
            host: 127.0.0.1
            path: /healthz
            port: 10258
            scheme: HTTP
          initialDelaySeconds: 15
          timeoutSeconds: 15
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        volumeMounts:
        - mountPath: /etc/kubernetes/cloud-controller-manager.conf
          name: k8s
          readOnly: true
        - name: cloud-config
          mountPath: /etc/kubernetes/config
      hostNetwork: true
      volumes:
      - hostPath:
          path: /etc/kubernetes/cloud-controller-manager.conf
          type: File
        name: k8s
      - name: cloud-config
        configMap:
          name: cloud-config
          items:
          - key: cloud-config.conf
            path: cloud-config.conf
6.验证




由于是单节点集群测试，故使用local模式的流量策略
[root@izj6c3ydyj9t4ztmha08rbz slb]# kubectl get svc nginx -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-11-18T22:36:48Z"
  finalizers:
  - service.k8s.alibaba/resources
  labels:
    app: nginx
    service.beta.kubernetes.io/hash: 73b160d328a26d99ed855f80117a95610f38768c282bc4bc5606bdc3
  name: nginx
  namespace: default
  resourceVersion: "12502"
  uid: 9c897582-0484-41b8-b983-32626599b4c1
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.98.194.116
  clusterIPs:
  - 10.98.194.116
  externalTrafficPolicy: Local
  healthCheckNodePort: 30738
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 32093
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 47.242.151.91
7.访问测试
7.1 node节点访问

7.2 pod访问

7.3 查看日志

8.ram权限
ps: 如果是ram用户需要授权如下策略
{
  "Version": "1",
  "Statement": [
    {
      "Action": [
        "ecs:Describe*",
        "ecs:AttachDisk",
        "ecs:CreateDisk",
        "ecs:CreateSnapshot",
        "ecs:CreateRouteEntry",
        "ecs:DeleteDisk",
        "ecs:DeleteSnapshot",
        "ecs:DeleteRouteEntry",
        "ecs:DetachDisk",
        "ecs:ModifyAutoSnapshotPolicyEx",
        "ecs:ModifyDiskAttribute",
        "ecs:CreateNetworkInterface",
        "ecs:DescribeNetworkInterfaces",
        "ecs:AttachNetworkInterface",
        "ecs:DetachNetworkInterface",
        "ecs:DeleteNetworkInterface",
        "ecs:DescribeInstanceAttribute"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "cr:Get*",
        "cr:List*",
        "cr:PullRepository"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "slb:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "cms:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "vpc:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "log:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "nas:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    }
  ]
}
