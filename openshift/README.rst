NOTES ABOUT OPENSHIFT
=====================

In OpenShift, some test containers (namely postgres) must be run with specific UIDs, therefore in this case the ci-jenkins service account
must be granted the 'anyuid' SCC:

.. code:: bash

  $ oc adm policy add-scc-to-user anyuid system:serviceaccount:jenkins:ci-jenkins

this must be done for every namespace in which jenkins needs to deploy.

for more information about SCCs, see official documentation here_

Setup Differences
-----------------

#) The Openshift setup uses an additional namespace ('prod'): this must be set up before creating roles and rolebindings.

.. code:: bash

  kubectl create sa ci-jenkins -n jenkins
  kubectl create sa ci-jenkins -n dev
  kubectl create sa ci-jenkins -n preprod

Roles and RoleBindings are found under the openshift/components folder:

#) Create jenkins role

.. code:: bash

  kubectl apply -f openshift/components/jenkins-role.yaml

#) Create jenkins rolebinding

.. code:: bash

  kubectl apply -f openshift/components/jenkins-rolebinding.yaml

Difference in Pipelines
-----------------------

Jenkins needs two additional plugins to manage OpenShift Clusters:

- Openshift Pipeline Plugin
- Openshift Client Plugin

Since Openshift offers the ability to run builds natively through the employment of BuildConfig objects, the Jenkins CI flow
differs slightly from the one that is run un K8S:

- **Jenkinsfile.agent-builder and Jenkinsfile.java-runner** have been replaced with **Jenkinsfile.buildconfig**: this pipeline runs and monitors buildconfig runs through the use of the Openshift Pipeline Plugin in Jenkins
- **Jenkinsfile.build-phase** now runs the image generation stage at the end of the pipeline (instead of leveraging another phase and another pipeline)
- **Jenkinsfile.app_deploy** handles rollouts in production via the Openshift Client Plugin for Jenkins

The 'oc' binary has been added to the base maven-agent image.

In the 'prod' namespace, deployment configs and other object are **persistent**, so the first deploy needs to be manually performed:

.. code:: bash

  $ kubectl kustomize openshift/deployments/pgprod/ > /tmp/prod-postgres.yaml && oc apply -f /tmp/prod-postgres.yaml
  $ kubectl kustomize openshift/deployments/prod/ > /tmp/prod-app.yaml && oc apply -f /tmp/prod-app.yaml

The rollout afterwards will be handled by the last stage of the **Jenkinsfile.app_deploy** pipeline.

Custom Templating
-----------------

Kustomize is a wonderful tool and it beats Templates hands down basically on every aspect. But as of now it does not support
Kubernetes extensions such as OCP3.11 Routes and DeploymentConfigs.

Fortunately it can be patched by adding Custom Resources Definitions (CRDs) to the templates and by writing custom transformer rules.
Look in the 'crds' folder in deployments/common and deployments/pgcommon.

For example, to let Kustomize correctly patch the VolumeClaimName in the deploymentconfig:

#) describe all needed fields in the DeploymentConfig CRD:

.. code:: json

	"github.com/mcaimi/k8s-demo-app/v1.DeploymentConfigSpec": {
		"Schema": {
			"properties": {
				"template": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "PodTemplateSpec"
				},
 		        "template/spec/volumes/secret": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "Secret"
				},
				"template/spec/containers/env/valueFrom/secretKeyRef": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "Secret"
				},
				"template/spec/volumes/configMap": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "ConfigMap"
				},
				"template/spec/volumes/persistentVolumeClaim": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "PersistentVolumeClaim",
					"x-kubernetes-object-ref-name-key": "claimName"
				},
				"template/spec/containers/resources": {
					"x-kubernetes-object-ref-api-version": "v1",
					"x-kubernetes-object-ref-kind": "ResourceRequirements"
				}
			}
		}
	}

#) Instruct Kustomize to patch the 'claimName' field defined above with an ad-hoc nameReference transformer:

.. code:: yaml

  nameReference:
  - kind: PersistentVolumeClaim
  fieldSpec:
  - path: spec/template/spec/volumes/persistentVolumeClaim/claimName
    kind: DeploymentConfig

With CRDs baked in into kustomization templates, the 'patchesStrategicMerge' directive in kustomization.yaml will not work correctly. The workaround is to define a static patch:

.. code:: yaml

  - op: add
  path: "/spec/template/spec/containers/0/resources"
  value:
    limits:
      cpu: "1"
      memory: "2Gi"
    requests:
      memory: "500Mi"
      cpu: "500m"
  
and use the 'patchesJson6902' strategy:

.. code:: yaml

  patchesJson6902:
  - path: mem-sizing.yaml
    target:
      group: apps.openshift.io
      version: v1
      kind: DeploymentConfig
      name: java-runner

directly into the kustomization.yaml file.

More information about Kustomize and CRDs can be found a this_ link and in the official kubernetes fields_ docs on GitHub.

Also have a look at this commit_ as it gives insights on how CRDs are actually implemented in kustomize

.. _here: https://docs.openshift.com/container-platform/3.11/admin_guide/manage_scc.html
.. _this: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/transformerconfigs/crd/README.md
.. _fields: https://github.com/kubernetes-sigs/kustomize/blob/master/docs/fields.md
.. _commit: https://github.com/kubernetes-sigs/kustomize/pull/105/commits/ea001347765a64bb52b1856f8f4fccec82ebcd67
