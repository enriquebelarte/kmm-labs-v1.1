Installing ROSA (Red Hat OpenShift on AWS)


1) Download ROSA CLI (it requires the AWS cli installed in advance)

https://console.redhat.com/openshift/create/rosa/getstarted

2) Extract binary and move it to path:

```bash
tar xfv rosa-linux.tar.gz
sudo mv rosa /usr/local/bin
```

I used the `rosa login --token XXXXX` command getting the offline access token from https://console.redhat.com/openshift/token/rosa

Then executed with default settings just running `rosa create cluster --cluster-name=myclusteroc --region=eu-west-3`.

It took 32 minutes from rosa create cluster until log msg "Install complete".

We can see the installing logs with `rosa logs install -c ebelarteoc --watch`.

```
time="2023-08-22T10:22:25Z" level=info msg="Install complete!"
time="2023-08-22T10:22:25Z" level=info msg="To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/output/auth/kubeconfig'"
time="2023-08-22T10:22:25Z" level=info msg="Access the OpenShift web-console here: https://console-openshift-console.apps.myclusteroc.79lj.p1.openshiftapps.com"
REDACTED LINE OF OUTPUT

time="2023-08-22T10:22:25Z" level=info msg="To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/output/auth/kubeconfig'"
```

I could not find the config file at /output/auth as stated at the log above.

Then created a new admin:
```bash
rosa create admin --cluster=myclusteroc
```
After that I could login to cluster like this:
```
oc login https://api.myclusteroc.79lj.p1.openshiftapps.com:6443 --username cluster-admin --password <thepassIgotabove>
```

The default OpenShift version as per August 2023 is 4.12.28 which is OK to test KMM v1.1.1 as it is in the OperatorHub in that OCP version.

Regarding AWS instances used by ROSA, default settings will fire up 3 masters/control planes on m5.2xlarge instances, 2 infra/workers on r5.xlarge instances and 2 workers in m5.xlarge instances.
