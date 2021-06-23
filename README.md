# Openshift Etcd Backup

This document describe the process to automate the backup of etcd for an OPenshift 4.X clusters.

This procedure is usefull in the recovery of one or more Master Nodes Cluster on Openshift 4. 



### Prerequisites

So you should have an OpenShift 4.6+ cluster (or a Kubernetes one but it's trickier to setup) with the different features enabled:

> If you don't have cli on your machine please go to those urls : 
    <br>
      ` oc : ` [oc-cli](https://mirror.openshift.com/pub/openshift-v4/clients/oc/4.6/)

# ECTD Backup :
You will need to execute those commands :
```
$ cat etcd-openshift-backup.yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: ocp-etcd-backup
  labels:
    app: openshift-etcd-backup

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openshift-etcd-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-etcd-backup

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-etcd-backup
rules:
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
      - create
      - delete
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-etcd-backup
  labels:
    app: openshift-etcd-backup
subjects:
  - kind: ServiceAccount
    name: openshift-etcd-backup
    namespace: ocp-etcd-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-etcd-backup

---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-etcd-backup
spec:
  schedule: "00 00 * * 5" # Every Friday at Midnight. Change here if you want another schedule.
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      labels:
        app: openshift-etcd-backup
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            app: openshift-etcd-backup
        spec:
          containers:
            - name: etcd-backup
              image: "registry.redhat.io/openshift4/ose-cli"
              command:
                - "/bin/bash"
                - "-c"
                - oc get no -l node-role.kubernetes.io/master --no-headers -o name | grep -i master-0 | xargs -I {} --  oc debug {} -- bash -c 'chroot /host sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup/'
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "openshift-etcd-backup"
          serviceAccount: "openshift-backup"
      
---
````

```
# Excute this command to create the different ressource.
$ oc create -f etcd-openshift-backup.yml

namespace/ocp-etcd-backup created
serviceaccount/openshift-etcd-backup created
clusterrole.rbac.authorization.k8s.io/cluster-etcd-backup created
clusterrolebinding.rbac.authorization.k8s.io/openshift-etcd-backup created
cronjob.batch/openshift-backup created
```
      
