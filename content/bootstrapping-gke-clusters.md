+++
date = "2018-01-03"
title = "Bootstrapping GKE Clusters for testing"
+++

I've recently started working on learning the internals of Kubernetes. I've settled on building an `operator`; which is essentially a means of extending functionality in a Kubernetes cluster by implementing a Kubernetes controller with storage backed by Custom Resource Definitions. This touches on a lot of concepts that are reused in multiple areas in the Kubernetes code base, and requires covering a large surface area of the Kubernetes golang client. It's a great way to slowly unravel the Kubernetes code base and put some hard-gained knowledge into practice.

Because this is a reasonably complicated undertaking, I wanted to build a little tool that would allow me to practice Test Driven Development, so I could build my understanding up in a directed manner. The general idea was that I would have a tool that could spin up a managed Kubernetes environment from a cloud provider (intially GKE, but I'd like to extend it later to other cloud providers), deploy the controller as a container to the cluster, and then run tests by creating, updating and deleting the custom resource. This would enable me to have a relatively fast feedback loop as I stumbled around the Kubernetes API trying to get things to work, and as I see it, perhaps create a tool that could be contributed back to the Kubernetes ecosystem at some point.

To put it simply, something like `./my-test-binary --up --test --down` should allow me to bring up a cluster, run some tests, and tear it down afterwards.

As it turns out, bringing a cluster up and tearing it down is relatively simply. Google provides an auto-generated golang client that can be used to make the API calls necessary to do this. Being auto-generated, it is a tad baroque - there are some efforts being undertaken here to provide a cleaner interface but it still lacks some of the functionality of the auto-generated API.

Firstly, you must provide a way for your application to obtain credentials that are tied to your google account. The accepted way here is to use application default credentials. As explained in the documentation, you can get application credentials that act as a proxy for your account identity by invoking `gcloud auth application-default login` (assuming you have logged in already). This will drop the credentials at `~/.config/gcloud/application_default_credentials.json`. By default, the client will check this location first, if you haven't specified another location in an environment variable (specifically, `GOOGLE_APPLICATION_CREDENTIALS)`. You could also create a service account with the necessary permissions required to create/delete a GKE cluster.

Creating a client to access the container, you must create a http client that provides the credential handling logic. This can be performed like so;

```golang
import (
    "golang.org/x/oauth2/google"
)
// client is *http.Client
client, err := google.DefaultClient(ctx, container.CloudPlatformScope)
```

The call to create a cluster with the authenticated client is relatively simple;

```golang
import (
    "google.golang.org/container/v1"
)

service, err := container.New(c.client)
if err != nil {
    return "", err
}

// I did say the API was rather baroque, didn't I?
projectsZonesClustersService := container.NewProjectsZonesClustersService(service)

createClusterRequest := &container.CreateClusterRequest{
    Cluster: &container.Cluster{
        Name:                  clusterId,
        Description:           "A cluster for e2e testing of elasticsearch-operator",
        InitialClusterVersion: "1.8.4-gke.1",
        InitialNodeCount:      3,
        EnableKubernetesAlpha: true,
        NodeConfig: &container.NodeConfig{
            DiskSizeGb:  40,
            ImageType:   "COS",
            MachineType: "f1-micro",
            OauthScopes: []string{
                "https://www.googleapis.com/auth/compute",
                "https://www.googleapis.com/auth/devstorage.read_only",
                "https://www.googleapis.com/auth/logging.write",
                "https://www.googleapis.com/auth/monitoring.write",
                "https://www.googleapis.com/auth/servicecontrol",
                "https://www.googleapis.com/auth/service.management.readonly",
                "https://www.googleapis.com/auth/trace.append",
            },
        },
    },
}

createCall := projectsZonesClustersService.Create(c.Project, c.Zone, createClusterRequest)
op, err := createCall.Do()
if err != nil {
    return "", err
}

return op.Name, nil

```

The call to create the cluster is non-blocking, so you must poll another endpoint to determine when the cluster has transitioned from 'creating' to 'created'. The creation step above returns the ID of an 'operation' which can then be used to determine the cluster creation status. I'm currently passing it to another function;

```golang
service, err := container.New(c.client)
if err != nil {
    return err
}

projectsZonesOperationsService := container.NewProjectsZonesOperationsService(service)
projectsZonesOperationsGetCall := projectsZonesOperationsService.Get(c.Project, c.Zone, operationId)
// As I'm only interested in the status of the operation, I may as well limit the fields to just 'status'
projectsZonesOperationsGetCall.Fields("status")

DoGetCall := func() (*container.Operation, error) {
    return projectsZonesOperationsGetCall.Do()
}

for op, err := DoGetCall(); op.Status != "DONE"; op, err = DoGetCall() {
    if err != nil {
        return err
    }

    time.Sleep(10 * time.Second)
}

return nil
```

Perhaps a little rough (maybe I'll eventually refactor to use a Done() channel) but it works. Deletion is implemented in a similar manner.

Assuming you have an existing cluster tied to your account, the Google Cloud documentation specifies you can get the credentials for a cluster by doing the following call;

```bash
gcloud container clusters get-credentials <name-of-cluster>
```

In getting your credentials, it auto-populates an entry into your kubeconfig file, typically locate at ~/.kube/config by default. A typically config fetched in this manner looks like the following (I've added comments to explain some of the fields).

```yaml
apiVersion: v1
clusters:
- cluster:
    # this is the HTTP certificate of the cluster endpoint so
    # the client doesn't thrown any nasty validation errors
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS...GSUNBVEUtLS0tLQo=
    # the endpoint of the cluster
    server: https://35.197.xxx.xxx
  name: gke_<project_id>_<zone>_<cluster>
# Our list of 'contexts'. Essentially user-cluster pairings that allow
# for a better experience when dealing with multiple clusters
contexts:
- context:
    cluster: gke_<project_id>_<zone>_<cluster>
    user: gke_<project_id>_<zone>_<cluster>
  name: gke_<project_id>_<zone>_<cluster>
current-context: gke_<project_id>_<zone>_<cluster>
kind: Config
preferences: {}
users:
- name: gke_<project_id>_<zone>_<cluster>
  user:
    # Auth provider determines how we authenticate with cluster
    # Auth plugins for GCP, Azure, OIDC etc can be found at
    # https://github.com/kubernetes/client-go/tree/master/plugin/pkg/client/auth
    auth-provider:
      # config denotes provider specific configuration
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /home/flash/Downloads/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      # the auth provider we are using
      name: gcp

```

Now this is all fine and dandy, but I felt like it was a clunkier user experience than I would typically like. If I am issuing a command to create a cluster, and there exists functionality in the gcloud tool to populate my kubeconfig file, surely I can have my tool populate the kubeconfig file automatically after cluster creation without the need to execute the gcloud commands and then run the tool again?

From expecting the configuration file that the gcloud tool produces, all that is needed is to;

- Fetch the CA certificate data from somewhere
- Fetch the cluster endpoint address from somewhere
- Populate the kubeconfig file with the relevant fields

It turns out is entirely possible to do this. Fetching the cluster data required (the endpoint and certificate authority data) is actually pretty easy, and be may be accomplished through a call in the container cluster API, eg;

```golang
service, err := container.New(c.client)
if err != nil {
    return nil, err
}

projectsZonesClustersService := container.NewProjectsZonesClustersService(service)
projectZonesClustersGetCall := projectsZonesClustersService.Get(c.Project, c.Zone, clusterId)
projectZonesClustersGetCall.Fields("status,endpoint,masterAuth")

cluster, err := projectZonesClustersGetCall.Do()
if err != nil {
    return nil, err
}
```

What took a bit more digging was determining how to populate a kube config file. I was familiar with the *rest.Config object, but I wasn't entirely sure how that was built from the file representation. My intuition was that there would have to at least be some deserialized structure present in client-go that represented the on-disk serialized representation; and it turns out there is - *clientcmdapi.Config. So as a first step, we need to build a clientcmdapi.Config from the information that was fetched from the preceding GCP API endpoint.

```golang

import (
    // some imports omitted for brevity
    clientcmdapi "k8s.io/client-go/tools/clientcmd/api"
)

func clientConfig(masterAuth, endpoint string) *clientcmdapi.Config {
    // fetch the path to the gcloud tool - assuming it is in your path variable
    gcloudPath, err := exec.LookPath("gcloud")
    if err != nil {
        panic(err)
    }

    // Lets start builing a new config
    config := clientcmdapi.NewConfig()

    // instantiate the cluster element of the config
    cluster := clientcmdapi.NewCluster()

    // the master auth retrieved from GCP it is base64 encoded
    // so it must be decoded first.
    caCert, err := base64.StdEncoding.DecodeString(masterAuth)
    if err != nil {
        panic(err)
    }

    cluster.CertificateAuthorityData = []byte(caCert)
    cluster.Server = fmt.Sprintf("https://%v", endpoint)

    context := clientcmdapi.NewContext()
    context.Cluster = "e2e-test-cluster"
    context.AuthInfo = "e2e-test-cluster-user"

    authInfo := clientcmdapi.NewAuthInfo()
    authInfo.AuthProvider = &clientcmdapi.AuthProviderConfig{
        Name: "gcp",
        Config: map[string]string{
            "cmd-args":   "config config-helper --format=json",
            "cmd-path":   gcloudPath,
            "expiry-key": "{.credential.token_expiry}",
            "token-key":  "{.credential.access_token}",
        },
    }

    config.Clusters["e2e-test-cluster"] = cluster
    config.Contexts["e2e-test-cluster-user"] = context
    config.AuthInfos["e2e-test-cluster-user"] = authInfo

    config.CurrentContext = "e2e-test-cluster-user"

    return config
}
```

Now that we have a *clientcmdapi.Config, we want to persist it to our on-disk kubeconfig file so that we can use it in the regular fashion.

```golang

config, err := clientConfig(endpoint, masterAuth)
if err != nil {
    panic(err)
}

// configAccess represents a way to access the on-disk kubeconfig file
// ModifyConfig takes this and merges in the changes we have added
// There are serveral other different LoadingRules objects that can be
// used to declare the location of a persisted kubeconfig file more directly.
// In my case, I'm happy to use the defaults.
configAccess := clientcmd.NewDefaultClientConfigLoadingRules()
if err != clientcmd.ModifyConfig(configAccess, *config, false); err != nil {
    panic(err)
}
```

You can find working versions at the following place in the linked [repository](https://github.com/matt-tyler/elasticsearch-operator/tree/1640cbee199e642bb907c7ca8143d97148f274a9).

As an extra tidbit, it isn't 100% necessary to provide the config parameter in the auth plugin when using the GCP auth plugin provider. If you omit it, you will force the plugin to attempt to authenticate using (once again) application default credentials. I actually have example of this occuring [here](https://github.com/matt-tyler/elasticsearch-operator/blob/1640cbee199e642bb907c7ca8143d97148f274a9/e2e/gke/gke.go#L61) - though it differs from what I have explained earlier as I instantiate my rest.Config directly, rather than bothering to persist it for later use. This configuration could still be persisted, and might even be a better way of doing things if I wanted to, for example, trigger the tests as a service hosted from somewhere else within GCP.
