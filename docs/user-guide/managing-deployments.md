---
---
You've deployed your application and exposed it via a service. Now what? Kubernetes provides a number of tools to help you manage your application deployment, including scaling and updating. Among the features we'll discuss in more depth are [configuration files](/docs/user-guide/configuring-containers/#configuration-in-kubernetes) and [labels](/docs/user-guide/deploying-applications/#labels).

* TOC
{:toc}

## Organizing resource configurations

Many applications require multiple resources to be created, such as a Replication Controller and a Service. Management of multiple resources can be simplified by grouping them together in the same file (separated by `---` in YAML). For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Multiple resources can be created the same way as a single resource:

```shell
$ kubectl create -f ./nginx-app.yaml
services/my-nginx-svc
replicationcontrollers/my-nginx
```

The resources will be created in the order they appear in the file. Therefore, it's best to specify the service first, since that will ensure the scheduler can spread the pods associated with the service as they are created by the replication controller(s).

`kubectl create` also accepts multiple `-f` arguments:

```shell
$ kubectl create -f ./nginx-svc.yaml -f ./nginx-rc.yaml
```

And a directory can be specified rather than or in addition to individual files:

```shell
$ kubectl create -f ./nginx/
```

`kubectl` will read any files with suffixes `.yaml`, `.yml`, or `.json`.

It is a recommended practice to put resources related to the same microservice or application tier into the same file, and to group all of the files associated with your application in the same directory. If the tiers of your application bind to each other using DNS, then you can then simply deploy all of the components of your stack en masse.

A URL can also be specified as a configuration source, which is handy for deploying directly from configuration files checked into github:

```shell
$ kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/kubernetes/master/docs/user-guide/pod.yaml
pods/nginx
```

## Bulk operations in kubectl

Resource creation isn't the only operation that `kubectl` can perform in bulk. It can also extract resource names from configuration files in order to perform other operations, in particular to delete the same resources you created:

```shell
$ kubectl delete -f ./nginx/
replicationcontrollers "my-nginx" deleted
services "my-nginx-svc" deleted
```

In the case of just two resources, it's also easy to specify both on the command line using the resource/name syntax:

```shell
$ kubectl delete replicationcontrollers/my-nginx services/my-nginx-svc
```

For larger numbers of resources, one can use labels to filter resources. The selector is specified using `-l`:

```shell
$ kubectl delete all -lapp=nginx
replicationcontrollers "my-nginx" deleted
services "my-nginx-svc" deleted
```

Because `kubectl` outputs resource names in the same syntax it accepts, it's easy to chain operations using `$()` or `xargs`:

```shell
$ kubectl get $(kubectl create -f ./nginx/ -o name | grep my-nginx)
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR    REPLICAS
my-nginx     nginx          nginx      app=nginx   2
NAME           LABELS      SELECTOR    IP(S)          PORT(S)
my-nginx-svc   app=nginx   app=nginx   10.0.152.174   80/TCP
```

## Using labels effectively

The examples we've used so far apply at most a single label to any resource. There are many scenarios where multiple labels should be used to distinguish sets from one another.

For instance, different applications would use different values for the `app` label, but a multi-tier application, such as the [guestbook example](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/guestbook/), would additionally need to distinguish each tier. The frontend could carry the following labels:

```yaml
     labels:
        app: guestbook
        tier: frontend
```

while the Redis master and slave would have different `tier` labels, and perhaps even an additional `role` label:

```yaml
     labels:
        app: guestbook
        tier: backend
        role: master
```

and

```yaml
     labels:
        app: guestbook
        tier: backend
        role: slave
```

The labels allow us to slice and dice our resources along any dimension specified by a label:

```shell
$ kubectl create -f ./guestbook-fe.yaml -f ./redis-master.yaml -f ./redis-slave.yaml
replicationcontrollers "guestbook-fe" created
replicationcontrollers "guestbook-redis-master" created
replicationcontrollers "guestbook-redis-slave" created
$ kubectl get pods -Lapp -Ltier -Lrole
NAME                           READY     STATUS    RESTARTS   AGE       APP         TIER       ROLE
guestbook-fe-4nlpb             1/1       Running   0          1m        guestbook   frontend   <none>
guestbook-fe-ght6d             1/1       Running   0          1m        guestbook   frontend   <none>
guestbook-fe-jpy62             1/1       Running   0          1m        guestbook   frontend   <none>
guestbook-redis-master-5pg3b   1/1       Running   0          1m        guestbook   backend    master
guestbook-redis-slave-2q2yf    1/1       Running   0          1m        guestbook   backend    slave
guestbook-redis-slave-qgazl    1/1       Running   0          1m        guestbook   backend    slave
my-nginx-divi2                 1/1       Running   0          29m       nginx       <none>     <none>
my-nginx-o0ef1                 1/1       Running   0          29m       nginx       <none>     <none>
$ kubectl get pods -lapp=guestbook,role=slave
NAME                          READY     STATUS    RESTARTS   AGE
guestbook-redis-slave-2q2yf   1/1       Running   0          3m
guestbook-redis-slave-qgazl   1/1       Running   0          3m
```

## Canary deployments

Another scenario where multiple labels are needed is to distinguish deployments of different releases or configurations of the same component. For example, it is common practice to deploy a *canary* of a new application release (specified via image tag) side by side with the previous release so that the new release can receive live production traffic before fully rolling it out. For instance, a new release of the guestbook frontend might carry the following labels:

```yaml
     labels:
        app: guestbook
        tier: frontend
        track: canary
```

and the primary, stable release would have a different value of the `track` label, so that the sets of pods controlled by the two replication controllers would not overlap:

```yaml
     labels:
        app: guestbook
        tier: frontend
        track: stable
```

The frontend service would span both sets of replicas by selecting the common subset of their labels, omitting the `track` label:

```yaml
  selector:
     app: guestbook
     tier: frontend
```

## Updating labels

Sometimes existing pods and other resources need to be relabeled before creating new resources. This can be done with `kubectl label`. For example:

```shell
$ kubectl label pods -lapp=nginx tier=fe
pod "my-nginx-v4-9gw19" labeled
pod "my-nginx-v4-hayza" labeled
pod "my-nginx-v4-mde6m" labeled
pod "my-nginx-v4-sh6m8" labeled
pod "my-nginx-v4-wfof4" labeled
$ kubectl get pods -lapp=nginx -Ltier
NAME                READY     STATUS    RESTARTS   AGE       TIER
my-nginx-v4-9gw19   1/1       Running   0          15m       fe
my-nginx-v4-hayza   1/1       Running   0          14m       fe
my-nginx-v4-mde6m   1/1       Running   0          18m       fe
my-nginx-v4-sh6m8   1/1       Running   0          19m       fe
my-nginx-v4-wfof4   1/1       Running   0          16m       fe
```

For more information, please see [labels](/docs/user-guide/labels/) and [kubectl label](/docs/user-guide/kubectl/kubectl_label/) document.

## Updating annotations

Sometimes you want to attach annotations to resources. Annotations are arbitrary non-identifying metadata for retrieval by API clients such as tools, libraries, etc. This can be done with `kubectl annotate`. For example:

```shell
$ kubectl annotate pods my-nginx-v4-9gw19 description='my frontend running nginx'
$ kubectl get pods my-nginx-v4-9gw19 -o yaml
apiversion: v1
kind: pod
metadata:
  annotations:
    description: my frontend running nginx
...
```

For more information, please see [annotations](/docs/user-guide/annotations/) and [kubectl annotate](/docs/user-guide/kubectl/kubectl_annotate/) document.

## Scaling your application

When load on your application grows or shrinks, it's easy to scale with `kubectl`. For instance, to increase the number of nginx replicas from 2 to 3, do:

```shell
$ kubectl scale rc my-nginx --replicas=3
replicationcontroller "my-nginx" scaled
$ kubectl get pods -lapp=nginx
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-1jgkf   1/1       Running   0          3m
my-nginx-divi2   1/1       Running   0          1h
my-nginx-o0ef1   1/1       Running   0          1h
```

To have the system automatically choose the number of nginx replicas as needed, range from 1 to 3, do:

```shell 
$ kubectl autoscale rc my-nginx --min=1 --max=3
replicationcontroller "my-nginx" autoscaled
$ kubectl get pods -lapp=nginx
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-1jgkf   1/1       Running   0          3m
my-nginx-divi2   1/1       Running   0          3m
$ kubectl get horizontalpodautoscaler 
NAME      REFERENCE                           TARGET    CURRENT     MINPODS   MAXPODS   AGE
nginx     ReplicationController/nginx/scale   80%       <waiting>   1         3         1m
```

For more information, please see [kubectl scale](/docs/user-guide/kubectl/kubectl_scale/), [kubectl autoscale](/docs/user-guide/kubectl/kubectl_autoscale/) and [horizontal pod autoscaler](/docs/user-guide/horizontal-pod-autoscaling/) document.

## Updating your application without a service outage

At some point, you'll eventually need to update your deployed application, typically by specifying a new image or image tag, as in the canary deployment scenario above. `kubectl` supports several update operations, each of which is applicable to different scenarios.

To update a service without an outage, `kubectl` supports what is called ['rolling update'?](/docs/user-guide/kubectl/kubectl_rolling-update), which updates one pod at a time, rather than taking down the entire service at the same time. See the [rolling update design document](https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/docs/design/simple-rolling-update.md) and the [example of rolling update](/docs/user-guide/update-demo/) for more information.

Let's say you were running version 1.7.9 of nginx:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

To update to version 1.9.1, you can use [`kubectl rolling-update --image`](https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/docs/design/simple-rolling-update.md):

```shell
$ kubectl rolling-update my-nginx --image=nginx:1.9.1
Created my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
```

In another window, you can see that `kubectl` added a `deployment` label to the pods, whose value is a hash of the configuration, to distinguish the new pods from the old:

```shell
$ kubectl get pods -lapp=nginx -Ldeployment
NAME                                              READY     STATUS    RESTARTS   AGE       DEPLOYMENT
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-k156z   1/1       Running   0          1m        ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-v95yh   1/1       Running   0          35s       ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-divi2                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-o0ef1                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-q6all                                    1/1       Running   0          8m        2d1d7a8f682934a254002b56404b813e
```

`kubectl rolling-update` reports progress as it progresses:

```shell
Scaling up my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 from 0 to 3, scaling down my-nginx from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 1
Scaling my-nginx down to 2
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 2
Scaling my-nginx down to 1
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 3
Scaling my-nginx down to 0
Update succeeded. Deleting old controller: my-nginx
Renaming my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 to my-nginx
replicationcontroller "my-nginx" rolling updated
```

If you encounter a problem, you can stop the rolling update midway and revert to the previous version using `--rollback`:

```shell
$ kubectl rolling-update my-nginx --rollback
Setting "my-nginx" replicas to 1
Continuing update with existing controller my-nginx.
Scaling up nginx from 1 to 1, scaling down my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 from 1 to 0 (keep 1 pods available, don't exceed 2 pods)
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 down to 0
Update succeeded. Deleting my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
replicationcontroller "my-nginx" rolling updated
```

This is one example where the immutability of containers is a huge asset.

If you need to update more than just the image (e.g., command arguments, environment variables), you can create a new replication controller, with a new name and distinguishing label value, such as:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx-v4
spec:
  replicas: 5
  selector:
    app: nginx
    deployment: v4
  template:
    metadata:
      labels:
        app: nginx
        deployment: v4
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.2
        args: [“nginx”,”-T”]
        ports:
        - containerPort: 80
```

and roll it out:

```shell
$ kubectl rolling-update my-nginx -f ./nginx-rc.yaml
Created my-nginx-v4
Scaling up my-nginx-v4 from 0 to 5, scaling down my-nginx from 4 to 0 (keep 4 pods available, don't exceed 5 pods)
Scaling my-nginx-v4 up to 1
Scaling my-nginx down to 3
Scaling my-nginx-v4 up to 2
Scaling my-nginx down to 2
Scaling my-nginx-v4 up to 3
Scaling my-nginx down to 1
Scaling my-nginx-v4 up to 4
Scaling my-nginx down to 0
Scaling my-nginx-v4 up to 5
Update succeeded. Deleting old controller: my-nginx
replicationcontroller "my-nginx-v4" rolling updated
```

You can also run the [update demo](/docs/user-guide/update-demo/) to see a visual representation of the rolling update process.

## In-place updates of resources

Sometimes it's necessary to make narrow, non-disruptive updates to resources you've created. For instance, you might want to update the container's image of your pod.

### kubectl patch

Suppose you want to fix a typo of the container's image of a pod. One way to do that is with `kubectl patch`:

```shell
# Suppose you have a pod with a container named "nginx" and its image "nignx" (typo), 
# use container name "nginx" as a key to update the image from "nignx" (typo) to "nginx"
$ kubectl get pod my-nginx-1jgkf -o yaml
```

```yaml
apiversion: v1
kind: pod
...
spec:
  containers:
  -image: nignx
   name: nginx
...
```

```shell
$ kubectl patch pod my-nginx-1jgkf -p '{"spec":{"containers":[{"name":"nginx","image":"nginx"}]}}'
"my-nginx-1jgkf" patched
$ kubectl get pod my-nginx-1jgkf -o yaml
```

```yaml
apiversion: v1
kind: pod
...
spec:
  containers:
  -image: nginx
   name: nginx
...
```

The patch is specified using json.

The system ensures that you don’t clobber changes made by other users or components by confirming that the `resourceVersion` doesn’t differ from the version you edited. If you want to update regardless of other changes, remove the `resourceVersion` field when you edit the resource. However, if you do this, don’t use your original configuration file as the source since additional fields most likely were set in the live state.

For more information, please see [kubectl patch](/docs/user-guide/kubectl/kubectl_patch/) document.

### kubectl edit

Alternatively, you may also update resources with `kubectl edit`:

```shell
$ kubectl edit pod my-nginx-1jgkf
```

This is equivalent to first `get` the resource, edit it in text editor, and then `replace` the resource with the updated version:

```shell
$ kubectl get pod my-nginx-1jgkf -o yaml > /tmp/nginx.yaml
$ vi /tmp/nginx.yaml
# do some edit, and then save the file
$ kubectl replace -f /tmp/nginx.yaml
pod "my-nginx-1jgkf" replaced
$ rm /tmp/nginx.yaml
```

This allows you to do more significant changes more easily. Note that you can specify the editor with your `EDITOR` or `KUBE_EDITOR` environment variables.

For more information, please see [kubectl edit](/docs/user-guide/kubectl/kubectl_edit/) document.

## Using configuration files

A more disciplined alternative to patch and edit is `kubectl apply`.

With apply, you can keep a set of configuration files in source control, where they can be maintained and versioned along with the code for the resources they configure. Then, when you're ready to push configuration changes to the cluster, you can run `kubectl apply`.

This command will compare the version of the configuration that you're pushing with the previous version and apply the changes you've made, without overwriting any automated changes to properties you haven't specified.

```shell
$ kubectl apply -f ./nginx-rc.yaml
replicationcontroller "my-nginx-v4" configured
```

As shown in the example above, the configuration used with `kubectl apply` is the same as the one used with `kubectl replace`. However, instead of deleting the existing resource and replacing it with a new one, `kubectl apply` modifies the configuration of the existing resource.

Note that `kubectl apply` attaches an annotation to the resource in order to determine the changes to the configuration since the previous invocation. When it's invoked, `kubectl apply` does a three-way diff between the previous configuration, the provided input and the current configuration of the resource, in order to determine how to modify the resource.

Currently, resources are created without this annotation, so the first invocation of `kubectl apply` will fall back to a two-way diff between the provided input and the current configuration of the resource. During this first invocation, it cannot detect the deletion of properties set when the resource was created. For this reason, it will not remove them.

All subsequent calls to `kubectl apply`, and other commands that modify the configuration, such as `kubectl replace` and `kubectl edit`, will update the annotation, allowing subsequent calls to `kubectl apply` to detect and perform deletions using a three-way diff.

## Disruptive updates

In some cases, you may need to update resource fields that cannot be updated once initialized, or you may just want to make a recursive change immediately, such as to fix broken pods created by a replication controller. To change such fields, use `replace --force`, which deletes and re-creates the resource. In this case, you can simply modify your original configuration file:

```shell
$ kubectl replace -f ./nginx-rc.yaml --force
replicationcontrollers "my-nginx-v4" replaced
```

## What's next?

- [Learn about how to use `kubectl` for application introspection and debugging.](/docs/user-guide/introspection-and-debugging/)
- [Configuration Best Practices and Tips](/docs/user-guide/config-best-practices/)
