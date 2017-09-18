### Introduction

Ansible playbook that automates the deployment of the Red Hat Summit 2016 Balloon Game in OpenShift.

### Prequisites

Note the playbook will add various imagestreams and templates to OpenShift. However, it does not update the nodejs template and this application requires that nodejs version 6 be available. Check your imagestream and make sure the nodejs imagestream has a version 6 tag, i.e.

```
oc get is nodejs -n openshift
```

If it doesn't, replace the imagestream with one that does:

```
oc project openshift
oc delete is nodejs
oc create -f https://raw.githubusercontent.com/luciddreamz/library/master/official/nodejs/imagestreams/nodejs-rhel7.json
```

### Cluster Admin Required

The playbook requires a user with cluster admin rights in order to add some of the imagestreams and templates to the ```openshift``` project.