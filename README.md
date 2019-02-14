# Istio Apigee Adapter

[![CircleCI](https://circleci.com/gh/apigee/istio-mixer-adapter.svg?style=shield)](https://circleci.com/gh/apigee/istio-mixer-adapter)
[![Go Report Card](https://goreportcard.com/badge/github.com/apigee/istio-mixer-adapter)](https://goreportcard.com/report/github.com/apigee/istio-mixer-adapter)
[![codecov.io](https://codecov.io/github/apigee/istio-mixer-adapter/coverage.svg?branch=master)](https://codecov.io/github/apigee/istio-mixer-adapter?branch=master)

This is the source repository for Apigee's Istio Mixer Adapter. This allows users of Istio to
incorporate Apigee Authentication, Authorization, and Analytics policies to protect and
report through the Apigee UI.

A Quick Start Tutorial continues below, but complete Apigee documentation on the concepts and usage of this adapter is also available on the [Apigee Adapter for Istio](https://docs.apigee.com/api-platform/istio-adapter/concepts) site. For more information and product support, please [contact Apigee support](https://apigee.com/about/support/portal).

---

# Installation and usage

## Version note

The current release is based on Istio 1.0.6. The included sample files and instructions below will 
automatically install the correct Istio version for you onto Kubernetes. It is recommended that
you install onto Kubernetes 1.9 or newer. See the [Istio](https://istio.io) web page for more information.  

## Prerequisite: Apigee

You must have an [Apigee Edge](https://login.apigee.com) account. If you need one, you can create one [here](https://login.apigee.com/sign_up).

## Download a Release

Istio Mixer Adapter releases can be found [here](https://github.com/apigee/istio-mixer-adapter/releases).

Download the appropriate release package for your operating system and extract it. You should a file list similar to:

    LICENSE
    README.md
    samples/apigee/authentication-policy.yaml
    samples/apigee/definitions.yaml
    samples/apigee/handler.yaml
    samples/apigee/httpapispec.yaml
    samples/apigee/rule.yaml
    samples/istio/helloworld.yaml
    samples/istio/istio-demo.yaml
    samples/istio/istio-demo-auth.yaml
    apigee-istio

`apigee-istio` (or apigee-istio.exe on Windows) is the Command Line Interface (CLI) for this project. 
You may add it to your PATH for quick access - or remember to specify the path for the commands below.

The yaml files in the samples/ directory contain the configuration for Istio and the adapter. We discuss 
these below.

## Provision Apigee for Istio

The first thing you'll need to do is provision your Apigee environment to work with the Istio adapter. 
This will install a proxy, set up a certificate, and generate some credentials for you:  

    apigee-istio -u {your username} -p {your password} -o {your organization name} -e {your environment name} provision > samples/apigee/handler.yaml

Once it completes, check your `samples/apigee/handler.yaml` file. It should look like this (with different values):

    # istio handler configuration for apigee adapter
    # generated by apigee-istio provision on 2018-06-18 15:29:31
    apiVersion: config.istio.io/v1alpha2
    kind: apigee
    metadata:
      name: apigee-handler
      namespace: istio-system
    spec:
      apigee_base: https://istioservices.apigee.net/edgemicro
      customer_base: https://myorg-myenv.apigee.net/istio-auth
      org_name: myorg
      env_name: myenv
      key: 06a40b65005d03ea24c0d53de69ab795590b0c332526e97fed549471bdea00b9
      secret: 93550179f344150c6474956994e0943b3e93a3c90c64035f378dc05c98389633

Notes:

* If you upgrading from a prior release, run the `apigee-istio provision` command with the `--forceProxyInstall` option to ensure that the latest Apigee proxy is installed for your organization.
* `apigee-istio` will automatically pick up the username and password from a 
[.netrc](https://ec.haxx.se/usingcurl-netrc.html) file in your home directory if you have an entry for 
`machine api.enterprise.apigee.com`.
* For Apigee Private Cloud (OPDK), you'll need to also specify your `--managementBase` in the command.
In this case, the .netrc entry should match this host. 

## Install Istio with Apigee mixer using Helm

Follow the official Istio [Helm Install](https://istio.io/docs/setup/kubernetes/helm-install/) instructions, but add
a `--values` parameter to the `helm template` or `helm install` command referencing this adapter's `install/mixer/helm.yaml` 
file.

Helm Template example (Option 1):

    ADAPTER=~/istio-mixer-adapter
    helm template install/kubernetes/helm/istio --name istio --namespace istio-system --values $ADAPTER/install/mixer/helm.yaml > istio.yaml

Helm Install example (Option 2):

    ADAPTER=~/istio-mixer-adapter
    helm install install/kubernetes/helm/istio --name istio --namespace istio-system --values $ADAPTER/install/mixer/helm.yaml 

## Upgrade Istio with Apigee mixer

If you have already installed Istio, you can upgrade Mixer to include the Apigee adapter by running the following commands:

    kubectl -n istio-system set image deployment/istio-telemetry mixer=gcr.io/apigee-api-management-istio/istio-mixer:1.0.6
    kubectl -n istio-system set image deployment/istio-policy mixer=gcr.io/apigee-api-management-istio/istio-mixer:1.0.6

NOTE: Change the container name to `istio-mixer-debug` if you need a Mixer container with debug tools. 
 
## Install a target service

Next, we'll install a simple [Hello World](https://github.com/istio/istio/tree/master/samples/helloworld)
service into the Istio mesh. From your Istio directory: 

    kubectl label namespace default istio-injection=enabled    
    kubectl apply -f samples/helloworld/helloworld.yaml
    
You should be able to verify two instances are running:

    kubectl get pods

And you should be able to access the service successfully:

    curl http://${GATEWAY_URL}/hello

Note: If you don't know your GATEWAY_URL, you'll need to follow [these instructions](
https://istio.io/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports) to set
the INGRESS_IP and INGRESS_PORT variables. Then, your GATEWAY_URL can be set with:
 
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

## Configure Apigee Mixer in Istio

Now that Istio is running, it's time to add Apigee policies. Apply the Apigee configuration:

        kubectl apply -f samples/apigee/definitions.yaml
        kubectl apply -f samples/apigee/handler.yaml
        kubectl apply -f samples/apigee/rule.yaml

Once done, you should no longer be able to access your `helloworld` service. If you curl it:

    curl http://${GATEWAY_URL}/hello
    
You should receive a permission denied error:

    PERMISSION_DENIED:apigee-handler.apigee.istio-system:missing authentication
    
The service is now protected by Apigee. Great! But now you've locked yourself out without a key. 
If only you had credentials. Let's fix that.

Debugging tip: If you're certain you've applied everything correctly up to this point but 
are still unable to see the `missing authentication` message, please check 
[the Github wiki](https://github.com/apigee/istio-mixer-adapter/wiki) for troubleshooting tips.  

```
ki delete po $(ki get po -l istio-mixer-type=policy -o 'jsonpath={.items[0].metadata.name}')
ki delete po $(ki get po -l istio-mixer-type=telemetry -o 'jsonpath={.items[0].metadata.name}')
```

The Mixer logs must contain a line with `started product manager` or the adapter will not work. 

### Configure your policies on Apigee via an API Product

[Create an API Product](https://apigee.com/apiproducts) in your Apigee organization:

* Give the `API Product` definition a name (`helloworld` is good).
* Select your environment(s).
* Set the Quota to `5` requests every `1` `minute`.
* Add a Path with the `+Custom Resource` button. Set the path to `/`.
* Add the service name `helloworld.default.svc.cluster.local` to `Istio Services`. 
  Note that you could also use the `apigee-istio bindings` command to associate the product with 
  an Istio service.
* Save

[Create a Developer](https://apigee.com/developers) - use any values you wish. Be creative!

[Create an App](https://apigee.com/apps):

* Give your `App` a name (`helloworld` is good).
* Select the `Developer` you just created above.
* Add your `API Product` with the `+Product` button.
* Save

Still on the App page, you should now see a `Credentials` section. Click the `Show` button 
under the `Consumer Key` heading. Copy that key!

### Access helloworld with your key

Now you should now be able to access the helloworld service in Istio by passing that key you got  
from your Apigee `App` above. Just send it as the value of the `x-api-key` header:
    
    curl http://${GATEWAY_URL}/hello -H "x-api-key: {your consumer key}"

The call should now be successful. You're back in business and your authentication policy works!

Note: There may be some latency in configuration. The API Product information refreshes from Apigee 
every two minutes. In addition, configuration changes you make to Istio will take a few moments to 
be propagated throughout the mesh. During this time, you could see inconsistent behavior as it takes
effect.

### Hit your Quota

Remember that Quota you set for 5 requests per minute? Let's max it out.

Make the same request you did above just a few more times:

    curl http://${GATEWAY_URL}/hello -H "x-api-key: {your consumer key}"

If you're in a Unix shell, you can just use repeat:

    repeat 10 curl http://${GATEWAY_URL}/hello -H "x-api-key: {your consumer key}"
    
Either way, you should see some successful calls... followed by failures that look like this:

    RESOURCE_EXHAUSTED:apigee-handler.apigee.istio-system:quota exceeded 

Did you see mixed successes and failures? That's OK. The Quota system is designed to have 
very low latency for your requests, so it uses a cache that is _eventually consistent_ with 
the remote server. Client requests don't wait for the server to respond and you could have 
inconsistent results for a second or two, but it will be worked out quickly and no clients 
have to wait in the meantime. 

### Bonus: Use JWT authentication instead of API Keys

Update the `samples/apigee/authentication-policy.yaml` file to set correct URLs for your environment:

    origins:
    - jwt:
        issuer: https://{your organization}-{your environment}.apigee.net/istio-auth/token
        jwks_uri: https://{your organization}-{your environment}.apigee.net/istio-auth/certs

The hostname (and ports) for these URLs should mirror what you used for `customer_base` config above
(adjust as appropriate if you're using OPDK).

Important: The mTLS authentication settings for your mesh and your authentication policy must match.
In other words, if you set up Istio to use mTLS (eg. you used istio-demo-auth.yaml), you must enable 
mTLS in your authentication-policy.yaml and must uncomment the mTLS the `mtls` line in the
authentication-policy.yaml file like so: 

  peers:
  - mtls:   # uncomment if you're using mTLS between services (eg. istio-demo-auth.yaml)

Conversely, if you did not enable mTLS for your mesh, do NOT enable it in your authentication policy. 
If the policies do not match, you will see errors similar to: 
`upstream connect error or disconnect/reset before headers`.

Save your changes and apply your Istio authentication policy:

    kubectl apply -f samples/apigee/authentication-policy.yaml

Now calls to `helloworld` (with or without the API Key):

    curl http://${GATEWAY_URL}/hello
    
Should receive an auth error similar to:

    Origin authentication failed.

We need a JWT token. So have `apigee-istio` get you a JWT token:

    apigee-istio token create -o {your organization} -e {your environment} -i {your key} -s {your secret}
    
Or, you can do it yourself through the API: 

    curl https://{your organization}-{your environment}.apigee.net/istio-auth/token -d '{ "client_id":"your key", "client_secret":"your secret", "grant_type":"client_credentials" }' -H "Content-Type: application/json"

Now, try again with your newly minted JWT token:

    curl http://${GATEWAY_URL}/hello -H "Authorization: Bearer {your jwt token}"

This call should now be successful.

### Check your analytics

Head back to the [Apigee Edge UI](https://apigee.com/edge).

Click the `Analyze` in the menu on the left and check out some nifty analytics information.

---

To join the Apigee pre-release program for additional documentation and support, please contact [apigee support](https://apigee.com/about/support/portal)

