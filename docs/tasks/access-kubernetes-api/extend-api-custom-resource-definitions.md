---
title: Extend the Kubernetes API with CustomResourceDefinitions
approvers:
- deads2k
- enisoc
---

{% capture overview %}
This page shows how to install a [custom resource](/docs/concepts/api-extension/custom-resources/)
into the Kubernetes API by creating a CustomResourceDefinition.
{% endcapture %}

{% capture prerequisites %}
* Read about [custom resources](/docs/concepts/api-extension/custom-resources/).
* Make sure your Kubernetes cluster has a master version of 1.7.0 or higher.
{% endcapture %}

{% capture steps %}
## Create a CustomResourceDefinition

When you create a new *CustomResourceDefinition* (CRD), the Kubernetes API Server
reacts by creating a new RESTful resource path, either namespaced or cluster-scoped,
as specified in the CRD's `scope` field. As with existing built-in objects, deleting a
namespace deletes all custom objects in that namespace.
CustomResourceDefinitions themselves are non-namespaced and are available to all namespaces.

For example, if you save the following CustomResourceDefinition to `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

And create it:

```shell
kubectl create -f resourcedefinition.yaml
```

Then a new namespaced RESTful API endpoint is created at:

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

This endpoint URL can then be used to create and manage custom objects.
The `kind` of these objects will be `CronTab` from the spec of the
CustomResourceDefinition object you created above.


## Create custom objects

After the CustomResourceDefinition object has been created, you can create
custom objects. Custom objects can contain custom fields. These fields can
contain arbitrary JSON.
In the following example, the `cronSpec` and `image` custom fields are set in a
custom object of kind `CronTab`.  The kind `CronTab` comes from the spec of the
CustomResourceDefinition object you created above.

If you save the following YAML to `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

and create it:

```shell
kubectl create -f my-crontab.yaml
```

You can then manage your CronTab objects using kubectl. For example:

```shell
kubectl get crontab
```

Should print a list like this:

```console
NAME                 KIND
my-new-cron-object   CronTab.v1.stable.example.com
```

Note that resource names are not case-sensitive when using kubectl,
and you can use either the singular or plural forms defined in the CRD,
as well as any short names.

You can also view the raw YAML data:

```shell
kubectl get ct -o yaml
```

You should see that it contains the custom `cronSpec` and `image` fields
from the yaml you used to create it:

```console
apiVersion: v1
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    clusterName: ""
    creationTimestamp: 2017-05-31T12:56:35Z
    deletionGracePeriodSeconds: null
    deletionTimestamp: null
    name: my-new-cron-object
    namespace: default
    resourceVersion: "285"
    selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
    uid: 9423255b-4600-11e7-af6a-28d2447dc82b
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
{% endcapture %}

{% capture discussion %}
## Advanced topics

### Finalizers

*Finalizers* allow controllers to implement asynchronous pre-delete hooks.
Custom objects support finalizers just like built-in objects.

You can add a finalizer to a custom object like this:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```

The first delete request on an object with finalizers merely sets a value for the
`metadata.deletionTimestamp` field instead of deleting it.
This triggers controllers watching the object to execute any finalizers they handle.

Each controller then removes its finalizer from the list and issues the delete request again.
This request only deletes the object if the list of finalizers is now empty,
meaning all finalizers are done.

### Validation

Validation of custom objects is possible via [OpenAPI v3 schema](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#schemaObject).
Additionally, the following restrictions are applied to the schema:

- The fields `default`, `nullable`, `discriminator`, `readOnly`, `writeOnly`, `xml` and
`deprecated` cannot be set.
- The field `uniqueItems` cannot be set to true.
- The field `additionalProperties` cannot be set to false.

This feature is __alpha__ in v1.8 and may change in backward incompatible ways.
Enable this feature using the `CustomResourceValidation` feature gate on
the [kube-apiserver](/docs/admin/kube-apiserver):

```
--feature-gates=CustomResourceValidation=true
```

The schema is defined in the CustomResourceDefinition. In the following example, the
CustomResourceDefinition applies the following validations on the custom object:

- `spec.cronSpec` must be a string and must be of the form described by the regular expression.
- `spec.replicas` must be an integer and must have a minimum value of 1 and a maximum value of 10.

Save the CustomResourceDefinition to `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

And create it:

```shell
kubectl create -f resourcedefinition.yaml
```

A request to create a custom object of kind `CronTab` will be rejected if there are invalid values in its fields.
In the following example, the custom object contains fields with invalid values:

- `spec.cronSpec` does not match the regular expression.
- `spec.replicas` is greater than 10.

If you save the following YAML to `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * *"
  image: my-awesome-cron-image
  replicas: 15
```

and create it:

```shell
kubectl create -f my-crontab.yaml
```

you will get an error:

```console
The CronTab "my-new-cron-object" is invalid: []: Invalid value: map[string]interface {}{"apiVersion":"stable.example.com/v1", "kind":"CronTab", "metadata":map[string]interface {}{"name":"my-new-cron-object", "namespace":"default", "deletionTimestamp":interface {}(nil), "deletionGracePeriodSeconds":(*int64)(nil), "creationTimestamp":"2017-09-05T05:20:07Z", "uid":"e14d79e7-91f9-11e7-a598-f0761cb232d1", "selfLink":"", "clusterName":""}, "spec":map[string]interface {}{"cronSpec":"* * * *", "image":"my-awesome-cron-image", "replicas":15}}:
validation failure list:
spec.cronSpec in body should match '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
spec.replicas in body should be less than or equal to 10
```

If the fields contain valid values, the object creation request is accepted.

Save the following YAML to `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 5
```

And create it:

```shell
kubectl create -f my-crontab.yaml
crontab "my-new-cron-object" created
```
{% endcapture %}

{% capture whatsnext %}
* Learn how to [Migrate a ThirdPartyResource to CustomResourceDefinition](/docs/tasks/access-kubernetes-api/migrate-third-party-resource/).
{% endcapture %}

{% include templates/task.md %}
