# Use Dockerfile-based builds with Supply Chain Choreographer

This topic explains how you can use Dockerfile-based builds with Supply Chain Choreographer.

For any source-based supply chains, when you specify the new `dockerfile`
parameter in a workload, the builds switch from using Kpack to using Kaniko.
Source-based supply chains are supply chains that don't take a pre-built image.
Kaniko is an open-source tool for building container images from a Dockerfile
without running Docker inside a container.

<table>
  <tr>
    <th>parameter name</th>
    <th>meaning</th>
    <th>example</th>
  </tr>

  <tr>
    <td><code>dockerfile<code></td>
    <td>relative path to the Dockerfile file in the build context</td>
    <td><pre>./Dockerfile</pre></td>
  </tr>

  <tr>
    <td><code>docker_build_context<code></td>
    <td>relative path to the directory where the build context is</td>
    <td><pre>.</pre></td>
  </tr>

  <tr>
    <td><code>docker_build_extra_args<code></td>
    <td>
      list of flags to pass directly to Kaniko (such as providing arguments,
      and so on to a build)
    </td>
    <td><pre>- --build-arg=FOO=BAR</pre></td>
  </tr>
</table>


For example, assuming that you want to build a container image our of a
repository named `github.com/foo/bar` whose Dockerfile resides in the root of
that repository, you can switch from using Kpack to building from that
Dockerfile by passing the `dockerfile` parameter:

```console
$ tanzu apps workload create foo \
  --git-repo https://github.com/foo/bar \
  --git-branch dev \
  --param dockerfile=./Dockerfile \
  --type web

🔎 Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    apps.tanzu.vmware.com/workload-type: web
      7 + |  name: foo
      8 + |  namespace: dev
      9 + |spec:
     10 + |  params:
     11 + |  - name: dockerfile
     12 + |    value: ./Dockerfile
     13 + |  source:
     14 + |    git:
     15 + |      ref:
     16 + |        branch: dev
     17 + |      url: https://github.com/foo/bar
```

Similarly, if the context to be used for the build must be set to a different
directory within the repository, you can make use of the `docker_build_context`
to change that:

```console
$ tanzu apps workload create foo \
  --git-repo https://github.com/foo/bar \
  --git-branch dev \
  --param dockerfile=MyDockerfile \
  --param docker_build_context=./src
```

> **Important** This feature has no platform operator configurations to be passed
> through `tap-values.yaml`, but if `ootb-supply-chain-*.registry.ca_cert_data` or
`shared.ca_cert_data` is configured in `tap-values`, the certificates
> are considered when pushing the container image.

## OpenShift

Despite that Kaniko can perform container image builds without
needing either a Docker daemon or privileged containers, it does
require the use of:

- Capabilities usually dropped from the more restrictive
  SecurityContextContraints enabled by default in OpenShift.
- The root user.

To overcome such limitations imposed by the default unprivileged
SecurityContextConstraints (SCC), Tanzu Application Platform installs:

- `SecurityContextConstraints/ootb-templates-kaniko-restricted-v2-with-anyuid` with enough extra privileges for
  Kaniko to operate.
- `ClusterRole/ootb-templates-kaniko-restricted-v2-with-anyuid` to permit the use of such SCC to any actor binding to
  that cluster role.

Each developer namespace needs a role binding that binds the role to an actor: `ServiceAccount`.
For more information, see [Set up developer namespaces to use your installed packages ](../install-online/set-up-namespaces.hbs.md).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workload-kaniko-scc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ootb-templates-kaniko-restricted-v2-with-anyuid
subjects:
  - kind: ServiceAccount
    name: default
```

With the SCC created and the ServiceAccount bound to the role that permits the
use of the SCC, OpenShift accepts the pods created to run Kaniko to build
the container images.


> **Note** Such restrictions are due to well-known limitations in how Kaniko
> performs the image builds, and there is currently no solution. For more information, see [kaniko#105].

## TKG 1.26 and above / Clusters with PSA enabled

Clusters with the [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) enabled and 
set to `enforce` will also not be able to run `kaniko` without changes to their configuration.  This is because the webhook 
requires containers to be run as "non-root" and `kaniko` needs to be "root" to function.

A workaround is to label the namespace that `kaniko` runs as `privileged`

```yaml
pod-security.kubernetes.io/enforce: privileged 
```

> **Note** Such restrictions are due to well-known limitations in how Kaniko
> performs the image builds, and there is currently no solution. For more information, see [kaniko#105].

[kaniko#105]: https://github.com/GoogleContainerTools/kaniko/issues/105

[SecurityContextConstraint]: https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html

