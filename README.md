# Integrating MITMProxy into OpenShift Authentication Pod

Below are steps on how to integrate the [MITMProxy](https://mitmproxy.org) with OpenShift 4.x. This can be a useful guide for troubleshooting issues when communicating with external authentication providers.

Download the files used in this article [here](https://github.com/bostrt/ocp4-auth-mitm/raw/master/ocp-auth-mitm.zip):

~~~
$ wget https://github.com/bostrt/ocp4-auth-mitm/raw/master/ocp-auth-mitm.zip
~~~

### Create Project, Pod, and Service
Create the following resources and initiate port forward so you can access the mitmproxy web interface:

~~~
$ oc new-project mitmproxy; oc project mitmproxy
$ oc create -f resources/mitm-pod.yaml
$ oc create -f resources/mitm-service.yaml
$ oc port-forward svc/mitmproxy --address 127.0.0.1 8888:8081
~~~

At this point, you should be able to open <http://127.0.0.1:8888> in your local web browser.

### Extract CA Certificate from MITMProxy

~~~
$ oc exec -n mitmproxy mitmproxy cat /root/.mitmproxy/mitmproxy-ca-cert.pem > /tmp/mitmproxy-ca-cert.pem
$ oc create cm user-ca-bundle -n openshift-config --from-file=ca-bundle.crt=/tmp/mitmproxy-ca-cert.pem
~~~

### Configure OpenShift to trust MITMProxy CA Certificate
##### *DO NOT* continue if you have configured a Proxy already.
Use the following to check if you have configured a Proxy or a trusted CA:

~~~
$ oc get proxy cluster -o yaml
~~~

##### Continue *ONLY IF* you do not have a trustedCA configured already.

~~~
$ oc patch proxy cluster --type=json -p '[{"op":"add","path":"/spec/trustedCA","value":{"name":"user-ca-bundle"}}]'
~~~

### Disable the OpenShift Authentication Operator
The following steps must be done to prevent the Operator from overwriting upcoming changes. The very last step in this guide will re-enable the Operator.

~~~
$ oc patch clusterversion version --type json -p "$(cat resources/disable-operator-patch.yaml)"
$ oc scale --replicas=0 deploy/authentication-operator -n openshift-authentication-operator
~~~

### Configure Proxy Environment Variables for Authentication

~~~
$ oc set env deployment/oauth-openshift -n openshift-authentication HTTP_PROXY=http://mitmproxy.mitmproxy.svc:8080 HTTPS_PROXY=http://mitmproxy.mitmproxy.svc:8080 NO_PROXY=10.0.0.0/8,172.0.0.0/8,192.0.0.0/8,localhost
~~~

### Cleanup
The following step will re-enable management of the OpenShift Authentication Operator. This includes scaling the operator back up and restoring all original settings you have set in the OAuth configuration.

~~~
$ oc patch clusterversion version --type json -p '[{"op":"remove", "path":"/spec/overrides"}]'
~~~
