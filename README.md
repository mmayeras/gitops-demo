# gitops-demo
Gitops Configuration



### GitOps installation

`$ oc apply -k 0-init/0-gitops`

**Check if pods are running in openshift-gitops :**  
`$ oc get pods -n openshift-gitops-operator`

**Get GitOps console route:**  
`$ CONSOLE_URL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath="{'https://'}{.spec.host}{'\n'}")`

_Optional: Patch argocd instance for reencrypt route_  
EDIT: Handled in argocd-instance.yml specs
`oc -n openshift-gitops patch argocd/openshift-gitops --type=merge -p='{"spec":{"server":{"route":{"enabled":true,"tls":{"insecureEdgeTerminationPolicy":"Redirect","termination":"reencrypt"}}}}}'`


### If you don't have persistent storage, you can use the LVM operator. This example assumes devices /dev/sda are configured on your nodes

`$ oc apply -k 1-apps/0-lvm/`

### Sealed secrets installation
** Secrets should not be stored as plain text in public repositories, in this example, we use https://github.com/bitnami-labs/sealed-secrets** 

`$ oc apply -k 1-apps/0-sealedsecrets/`

sealed secrets has to be created with your sealed secret key. This is out of scope of this lab.

### AAP Operator install

`$ oc apply -k 1-apps/1-aap/`

Once deployed, you can access your Ansible Automation Controller console 

`$ AAC_ROUTE=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath="{'https://'}{.spec.host}{'\n'}")`

You need to connect once to the controller Web UI to :
- Upload your subscrition manifest or fetch it using your Red Hat credentials
- Create an token for you admin user in order to manage resources with the Openshift Operators
  - Go to Users>admin>Tokens>Add 
  - Do not specify any Application and 'Write' as scope then click Save and copy the generated token

### AAP Operator Integration

Create a new secret in your ansible-automation-platform namespace 

`$ oc apply -k 1-apps/2-aap-integration`

Once again, sealed secret used in this example, you can create the secret manually following these specs :

```yaml
apiVersion: v1
data:
  host: <AAC_ROUTE>
  token: <Previously_generated_token>
kind: Secret
metadata:
  creationTimestamp: "2024-01-04T11:19:47Z"
  name: admin-token
  namespace: ansible-automation-platform
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: admin-token
    uid: 6d2dfda6-a994-4fe9-b5bb-06a41367cfdd
  resourceVersion: "808934"
  uid: 9a08752f-f7d5-412c-83d7-87502c76da22
type: Opaque
```


### Openshift Pipeline install

`$ oc apply -k 1-apps/3-tekton-operator `


### Ansible Build EE Pipeline creation

In order to fetch packages from RH repositories, we need to use valid host entitlement, we can create a secret using the same value as etc-pki-entitlement  openshift-config-managed namespace.

It will be created using Sealed Secrets too, you can create it manually with :
_Optional_ 
`$ oc get secret -n openshift-config-managed etc-pki-entitlement -o yaml | sed 's/namespace: .*/namespace: ansible-ee-build/' | oc create -f -` 

We also need an ansible config file with a valid token to fetch some collections from Red hat automation hub.

`$oc -n ansible-ee-build create secret generic ansible-cfg --from-file ansible.cfg --dry-run=client -o yaml | kubeseal -o yaml | oc create -f -`

```ini
[defaults]
host_key_checking = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s

[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url=https://console.redhat.com/api/automation-hub/content/published/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=<your_token>
[galaxy_server.galaxy]
url=https://galaxy.ansible.com/
```

These two secrets are mounted in 2-tkn-ee-build-builder-task.

The github-trigger-secret is the one used to configure your custom execution-environement repository webhook to the pipeline

`$oc -n ansible-ee-build create secret generic ansible-cfg --from-literal secretToken=123 --dry-run=client -o yaml | kubeseal -o yaml | oc create -f -`

`$ oc apply -k 1-apps/4-tekton-pipeline` 



### Sample playbooks

Playbooks use a vault password manually created using the web interface.
This vault password was used to encrypt strings in some of the example playbooks.


