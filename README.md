# UDS Package Actions Runner Controller

This Zarf package installs [actions-runner-controller](https://github.com/actions/actions-runner-controller) (ARC) and deploys a self hosted GitHub Actions runner.

A GitHub App is installed to authenticate to GitHub and register runners. This is preferable to using Personal Access Tokens, which are tied to a specific user's identity.

The runner is installed as a [scale set](https://github.com/actions/actions-runner-controller/blob/master/docs/preview/gha-runner-scale-set-controller/README.md). This is a new (preview) GitHub feature for self hosted runner which is more secure and more efficient. Scale sets support scaling to zero when not in use. Also, the runner is significantly more secure, because the (org) runner registration token is not exposed to runners, instead only a per-workflow JIT token is used.

The runners are installed using the [`containerMode: kubernetes` mode of ARC](https://github.com/actions/actions-runner-controller/blob/master/docs/deploying-alternative-runners.md#runner-with-k8s-jobs). This uses the k8s hooks from [runner-container-hooks](https://github.com/actions/runner-container-hooks) to run each job of the pipeline as a Kubernetes Pod with the workflow specified container image. Note that all workflow jobs MUST [specify a `container:`](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#example-using-a-dockerfile-in-your-repository). Also, [docker container actions that use `runs.image: Dockerfile`](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#example-using-a-dockerfile-in-your-repository) are NOT compatible (if they use a public Docker image instead of `Dockerfile` they are compatible though). Also, `docker` will not be usable within the workflow steps and `docker build` will not work.

There is an alternative `containerMode: dind` which removes these restrictions but it uses a privileged Docker sidecar container to execute containers and run `docker` commands. Due to it's use of privileged it provides significantly less security isolation.

Note that the k8s container hooks do not currently support configuring the Pod spec of workflow pods and resource limits or custom volume mounts are not currently possible, see https://github.com/actions/runner-container-hooks/pull/50#issuecomment-1551874570. If you require this immediately, a hacky solution is to use a mutating admission webhook such as Kyverno to mutate Pods named `*-workflow`.

The workflow Pods are run in the `arc-runners` Kubernetes namespace. This namespace has a [`zarf.dev/agent: ignore` label](https://github.com/defenseunicorns/kibbles-AND-bits/blob/main/runner-scale-set/namespace.yaml#L5) to exclude it from Zarf Agent image rewriting and allow workflows to use public images not available in the zarf registry.

## When should actually you use self hosted runners?

Public repos get completely free (infinite, within reason) hours of GitHub.com hosted Actions on public repos:  
https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration
>GitHub Actions usage is free for standard GitHub-hosted runners in public repositories, and for self-hosted runners.

Generally GitHub's hosted runners will be more secure (mostly: more ephemeral) than anything you can ever self host. They're basically (free!) Azure VMs they run the fly for you. With self hosted runners, it is potentially possible for the the runner host to be persistently compromised.

Use cases for self hosted runners:

* Self hosted GitHub Enterprise: you don't get any GitHub-hosted runners at all
* On prem runners (a self hosted runner has access to the local network and can easily interact with local resources)
* Bigger runners than the free GitHub-hosted runners
  * Note, larger runners are available by paying GitHub: https://docs.github.com/en/actions/using-github-hosted-runners/using-larger-runners
  * Or, you can instead self host ARC in AWS with a larger instance size and pay AWS for EC2 or EKS
* Private repos which do not have infinite free GitHub-hosted runner minutes
  * Note, additional minutes are available by paying GitHub: https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions
  * Or, you can self host ARC in AWS and pay AWS for EC2 or EKS
* Running a runner inside the dev/staging VPC or inside the staging k8s cluster (in a VPC) to allow easy access to (maybe private) AWS resources
  * Note, please compare to using some type of VPN solution and maybe [authenticate with the GitHub Actions OIDC JWT ID token](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
* Running a runner inside AWS to minimize network transfer for faster transfers and minimizing billing for ingress/egress

## Installation

The following instructions describe how to create and install a GitHub App onto a GitHub _org_. A similar process can be used to install the GitHub App onto a single repository only. Note, that you can also install the GitHub App onto an org and limit it's "Repository access" to "Only select repositories".

See the upstream ARC documentation for full installation steps:
https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md#deploying-using-github-app-authentication

1. **Create an GitHub App on your GitHub org**
   * Contrary to the ARC documentation, "Administration (read / write)" permissions are not required.
   * You may use the following URL to create the application on an org. Replace `:org` in the following URL with the name of your org:

     ```
     https://github.com/organizations/:org/settings/apps/new?url=http://github.com/actions/actions-runner-controller&webhook_active=false&public=false&organization_self_hosted_runners=write&actions=read&checks=read
     ```

2. **Record the "App ID" in the App's settings**
    * This is the `GITHUB_APP_ID` Zarf variable
3. **In the App's settings, generate and download a private key file by clicking "Generate a private key"**
    * This is `GITHUB_APP_PRIVATE_KEY` Zarf variable
    * Due to a Zarf variable issue, you will need to include literal `\n` to create newlines in the variable
4. **In the App's settings, click "Install App" on the left, and install the application on the org**
    * Record the installation ID (it is at the end of URL), this is the `GITHUB_APP_INSTALLATION_ID` Zarf variable
    * If desired, you may limit the installation's "Repository access" to "Only select repositories"
5. **The URL of the org the application was installed on is `GITHUB_CONFIG_URL`**
6. **Create a `zarf-config.yaml` with the variables configured as desired:**

   ```yaml
   package:
     deploy:
       set:

         # The URL of the GitHub org that the GitHub App is installed on
         github_config_url: "https://github.com/octocat"

         # The id of the GitHub App
         github_app_id: "123"

         # The id of the installation of the GitHub App on the org
         # This id can be found in the URL of the installation's settings
         github_app_installation_id: "456"

         # The file containing the private key of the GitHub App
         github_app_private_key: deploy-github-app-private-key.pem

         # Optional. The name of the runner group to register the scale set in
         # You must first create the runner group in the GitHub UI manually before setting it here
         #github_runner_group: "default"

         # The name of the runner scale set
         # This value is effectively a GitHub runs-on label for the runners. Note the runners will not have an "on-prem" label
         github_runner_scaleset_name: "arc"

         # The storage class name used for the Kubernetes ephemeral volume (temporary PVC/PV) used for the working directory of workflow execution.
         # If using k3s/k3d you can use the `local-path` storage class.
         github_runner_storage_class: "local-path"

         # Optional. The size of the ephemeral volume
         # Note, local-path does not enforce storage limits.
         #github_runner_storage_size: "1Gi"

         # Optional. The maximum number of runners to run concurrently.
         #github_runner_max: 5

         # Optional. The minimum number of runners to scale down to.
         # By default it will scale to zero.
         #github_runner_min: 0
   ```

7. **Install the actions-runner-controller Zarf package**

   ```bash
   zarf package deploy oci://ghcr.io/defenseunicorns/packages/actions-runner-controller:0.0.3-amd64
   ```

### Big Bang values needed to deploy

If deploying onto a cluster with Big Bang including kyverno, the `restrict-image-registries` policy will reject non Iron Bank images and the `disallow-image-tags` policy will reject the `latest` tag used by `ghcr.io/actions/actions-runner:latest`.

The following Big Bang values may be used to add an exception for the ARC namespaces:

```yaml
kyvernoPolicies:
  values:
    policies:
      disallow-image-tags:
        exclude:
          any:
          - resources:
              namespaces:
              - arc-runners
              - arc-systems
      restrict-image-registries:
        exclude:
          any:
          - resources:
              namespaces:
              - arc-runners
              - arc-systems
```
