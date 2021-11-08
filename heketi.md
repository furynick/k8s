[[_TOC_]]

# Installer le serveur glusterfs sur le deux noeuds (vm1 et vm2)
```bash
aptitude -t buster-backports -R install glusterfs-server
systemctl enable --now glusterd
groupadd -g 889 glusterfs
useradd -g glusterfs -d /var/lib/glusterd -u 889 -s /bin/bash glusteradm
install -o glusteradm -g glusterfs -m 0700 -d /var/lib/glusterd/.ssh
chown glusteradm: /var/lib/glusterd
cat >/etc/sudoers.d/heketi <<EoF
Defaults:glusteradm        !requiretty

glusteradm ALL = (root) NOPASSWD: /usr/sbin/mkfs.xfs
glusteradm ALL = (root) NOPASSWD: /usr/sbin/gluster
glusteradm ALL = (root) NOPASSWD: /usr/sbin/lvm
glusteradm ALL = (root) NOPASSWD: /usr/bin/systemctl status glusterd
glusteradm ALL = (root) NOPASSWD: /usr/bin/udevadm
glusteradm ALL = (root) NOPASSWD: /usr/bin/umount
glusteradm ALL = (root) NOPASSWD: /usr/bin/mount
glusteradm ALL = (root) NOPASSWD: /usr/bin/mkdir
glusteradm ALL = (root) NOPASSWD: /usr/bin/chown
glusteradm ALL = (root) NOPASSWD: /usr/bin/chmod
glusteradm ALL = (root) NOPASSWD: /usr/bin/rmdir
glusteradm ALL = (root) NOPASSWD: /usr/bin/lsof
glusteradm ALL = (root) NOPASSWD: /usr/bin/awk
glusteradm ALL = (root) NOPASSWD: /usr/bin/sed
EoF
cat >.bash_aliases <<EoF
function heketi-cli () {
  [ -z "\$HEKETI_CLI_KEY" ] && HEKETI_CLI_KEY=\$(kubectl get secret heketi-admin-secret -n heketi -o=custom-columns=PASS:.data.key --no-headers=true | base64 -d)
  POD=\$(kc get pods -n heketi -o json | jq -r .items[0].metadata.name)
  if [ -n "\$POD" ]; then
    OPTS="--user admin --secret '\$HEKETI_CLI_KEY' --server http://localhost:8080"
    CMD="kubectl exec '\$POD' -n heketi -it -- heketi-cli \$OPTS \$@"
    eval \$CMD
  else
    echo "Pod heketi non trouve"
  fi
}
EoF
```

# Créer les resources PCS
```bash
pcs resource create vip-kglrctvma ocf:heartbeat:IPaddr2 "ip=$(gethostip -d kglrctvma)" op monitor interval=10s --group GlusterStorage
pcs resource ban vip-kglrctvma kmrctvm3
pcs resource ban vip-kglrctvma kmrctvm4
```

# Déployer Heketi
cf. https://computingforgeeks.com/configure-kubernetes-dynamic-volume-provisioning-with-heketi-glusterfs/  
cf. https://raw.githubusercontent.com/heketi/heketi/master/etc/heketi.json
```bash
HEKETI_ADMIN_PASS=$(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12})
HEKETI_USER_PASS=$(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12})
ssh-keygen -t ed25519 -f access_key -N ''
install -o glusteradm -g glusterfs -m 0644 access_key.pub ~glusteradm/.ssh/authorized_keys
  # Copier la cle publique sur le second noeud
cat >heketi.json <<EoF
{
  "port": "8080",
  "enable_tls": false,
  "cert_file": "",
  "key_file": "",
  "use_auth": true,
  "jwt": {
    "admin": {
      "key": "$HEKETI_ADMIN_PASS"
    },
    "user": {
      "key": "$HEKETI_USER_PASS"
    }
  },
  "backup_db_to_kube_secret": false,
  "profiling": false,
  "glusterfs": {
    "executor": "ssh",
    "sshexec": {
      "keyfile": "/etc/heketi/access_key",
      "user": "glusteradm",
      "sudo": true,
      "debug_umount_failures": true
    },
    "db": "/var/lib/heketi/heketi.db",
    "refresh_time_monitor_gluster_nodes": 120,
    "start_time_monitor_gluster_nodes": 10,
    "loglevel": "debug",
    "auto_create_block_hosting_volume": true,
    "block_hosting_volume_size": 500,
    "block_hosting_volume_options": "group gluster-block",
    "pre_request_volume_options": "",
    "post_request_volume_options": ""
  }
}
EoF
cat >deploy-heketi.yaml <<EoF
---
apiVersion: v1
kind: Namespace
metadata:
  name: heketi
---
apiVersion: v1
kind: Secret
metadata:
  name: heketi-admin-secret
  namespace: heketi
data:
  key: $(base64 <<<"$HEKETI_ADMIN_PASS")
type: kubernetes.io/glusterfs
---
apiVersion: v1
kind: Secret
metadata:
  name: heketi-user-secret
  namespace: heketi
data:
  key: $(base64 <<<"$HEKETI_USER_PASS")
type: kubernetes.io/glusterfs
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heketi-service-account
  namespace: heketi
---
kind: Secret
apiVersion: v1
type: Opaque
metadata:
  name: heketi-config-secret
  namespace: heketi
data:
  access_key: $(base64 -w 0 <access_key)
  access_key.pub: $(base64 -w 0 <access_key.pub)
  heketi.json: $(base64 -w 0 <heketi.json)
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: heketi
  namespace: heketi
  labels:
    team: pf3wi
    app: heketi
  annotations:
    description: Defines how to deploy Heketi
spec:
  replicas: 1
  selector:
    matchLabels:
      team: pf3wi
      app: heketi
  template:
    metadata:
      name: heketi
      namespace: heketi
      labels:
        team: pf3wi
        app: heketi
    spec:
      serviceAccountName: heketi-service-account
      containers:
      - image: heketi/heketi:latest
        imagePullPolicy: IfNotPresent
        name: heketi
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: db
          mountPath: "/var/lib/heketi"
        - name: config
          mountPath: /etc/heketi
        readinessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 3
          httpGet:
            path: "/hello"
            port: 8080
        livenessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 30
          httpGet:
            path: "/hello"
            port: 8080
      volumes:
      - name: db
        iscsi:
          targetPortal: "$(gethostip -d kisrctvma):3260"
          iqn: "iqn.2021-06.k8srct.storage:iscsivg"
          lun: 1
          fsType: ext4
          readOnly: false
      - name: config
        secret:
          secretName: heketi-config-secret
---
kind: Service
apiVersion: v1
metadata:
  name: heketi
  namespace: heketi
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    team: pf3wi
    app: heketi
  type: ClusterIP
EoF
kubectl apply -f deploy-heketi.yaml
```

# Initialisation de heketi
```bash
heketi-cli cluster create
CL=$(heketi-cli cluster list --json | jq -r ".clusters[0]")
heketi-cli node add \
  --zone=1 \
  --cluster $CL \
  --management-host-name=kglrctvm1 \
  --storage-host-name=$(gethostip -d kglrctvm1)
heketi-cli node add \
  --zone=2 \
  --cluster $CL \
  --management-host-name=kglrctvm2 \
  --storage-host-name=$(gethostip -d kglrctvm2)
heketi-cli cluster info $CL --json | jq -r .nodes[] | while read n; do
  eval "$(heketi-cli node info $n --json | jq -r .hostnames.manage[0])=$n"
heketi-cli device add \
  --name=/dev/sdc \
  --node=$kglrctvm1
heketi-cli device add \
  --name=/dev/sdc \
  --node=$kglrctvm2
```

# Création de la classe de stockage
```bash
cat >gluster-storageclass.yaml <<EoF
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: glusterfs
  namespace: heketi
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/glusterfs
allowVolumeExpansion: true
parameters:
#  resturl: "http://heketi.heketi.k8srct.pf3wi:8080"
  resturl: "http://$(kubectl get svc -n heketi heketi -o custom-columns=IP:.spec.clusterIP --no-headers):8080"
  restuser: "user"
  restuserkey: "$HEKETI_USER_PASS"
  restauthenabled: "true"
#  secretName: heketi-admin-secret
#  secretNamespace: heketi
  clusterid: $CL
  volumetype: replicate:2
  volumenameprefix: k8s
EoF
```
