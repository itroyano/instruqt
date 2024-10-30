---
slug: 03-traffic-distribution
id: orev5ikaew0x
type: challenge
title: Traffic Distribution
tabs:
- id: 6q0nxrd0hihb
  title: Terminal 1
  type: terminal
  hostname: crc
- id: cfpjxlweovou
  title: Web Console
  type: website
  url: https://console-openshift-console.crc.${_SANDBOX_ID}.instruqt.io
  new_window: true
difficulty: intermediate
timelimit: 360
enhanced_loading: null
---
At the end of this step you will be able to:
- Provide a custom `name` to the deployment
- Configure a service to use a `blue-green` deployment pattern

Serverless services always route traffic to the **latest** revision of the deployment. In this assignment, we will learn a few different ways to split the traffic between revisions in our service.

## Revision Names
By default, Serverless generates random revision names for the service that is based on using the Serverless service’s `metadata.name` as a prefix.

The following service deployment uses the same greeter service as the last section in the tutorial except it is configured with an arbitrary revision name.

Let's deploy the greeter service again, but this time set its **revision name** to `greeter-v1` by executing:

```
kn service create greeter \
   --image quay.io/rhdevelopers/knative-tutorial-greeter:quarkus \
   --namespace serverless-tutorial \
   --revision-name greeter-v1
```

> **Note:** *The equivalent yaml for the service above can be seen by executing:*

```
cat /root/03-traffic-distribution/greeter-v1-service.yaml
```

Next, we are going to update the greeter service to add a message prefix environment variable and change the revision name to `greeter-v2`.  Do so by executing:

```
kn service update greeter \
   --image quay.io/rhdevelopers/knative-tutorial-greeter:quarkus \
   --namespace serverless-tutorial \
   --revision-name greeter-v2 \
   --env MESSAGE_PREFIX=GreeterV2
```

> **Note:** The equivalent yaml for the service above can be seen by executing:

```
cat /root/03-traffic-distribution/greeter-v2-service.yaml
```

See that the two greeter services have deployed successfully by executing

```
kn revision list
```

The last command should output two revisions, `greeter-v1` and `greeter-v2`:

```bash
NAME         SERVICE   TRAFFIC   TAGS   GENERATION   AGE    CONDITIONS   READY   REASON
greeter-v2   greeter   100%             2            4m5s   3 OK / 4     True
greeter-v1   greeter                    1            14m    3 OK / 4     True
```

> **Note:** *It is important to notice that by default the latest revision will replace the previous by receiving 100% of the traffic.*

## Blue-Green Deployment Patterns
Serverless offers a simple way of switching 100% of the traffic from one Serverless service revision (blue) to another newly rolled out revision (green).  We can rollback to a previous revision if any new revision (e.g. green) has any unexpected behaviors.

With the deployment of `greeter-v2` serverless automatically started to direct 100% of the traffic to `greeter-v2`. Now let us assume that we need to roll back `greeter-v2` to `greeter-v1` for some reason.

Update the greeter service by executing:

```
kn service update greeter \
   --traffic greeter-v1=100 \
   --tag greeter-v1=current \
   --tag greeter-v2=prev \
   --tag @latest=latest
```

The above service definition creates three sub-routes(named after traffic tags) to the existing `greeter` route.
- **current**: The revision will receive 100% of the traffic distribution
- **prev**: The previously active revision, which will now receive no traffic
- **latest**: The route pointing to the latest service deployment, here we change the default configuration to receive no traffic.

> **Note:** *Be sure to notice the special tag: `latest` in our configuration above.  In the configuration we defined 100% of the traffic be handled by `greeter-v1`.*
>
> *Using the latest tag allows changing the default behavior of services to route 100% of the traffic to the latest revision.*

We can validate the service traffic tags by executing:

```
kn route describe greeter
```

The output should be similar to:

```
Name:       greeter
Namespace:  serverless-tutorial
Age:        1m
URL:        https://greeter-serverless-tutorial.crc.9yhetjexy8dv.instruqt.io
Service:    greeter

Traffic Targets:
    0%  @latest (greeter-v2) #latest
        URL:  https://latest-greeter-serverless-tutorial.crc.9yhetjexy8dv.instruqt.io
  100%  greeter-v1 #current
        URL:  https://current-greeter-serverless-tutorial.crc.9yhetjexy8dv.instruqt.io
    0%  greeter-v2 #prev
        URL:  https://prev-greeter-serverless-tutorial.crc.9yhetjexy8dv.instruqt.io

Conditions:
  OK TYPE                      AGE REASON
  ++ Ready                      8s
  ++ AllTrafficAssigned         1m
  ++ CertificateProvisioned     1m TLSNotEnabled
  ++ IngressReady               8s
```

> **Note:** *The equivalent yaml for the service above can be seen by executing:

```
cat /root/03-traffic-distribution/service-pinned.yaml
```

The configuration change we issued above does not create any new `configuration`, `revision`, or `deployment` as there was no application update (e.g. image tag, env var, etc).  When calling the service without the sub-route, Serverless scales up the `greeter-v1`, as our configuration specifies and the service responds with the text `Hi greeter ⇒ '9861675f8845' : 1`.

We can check that `greeter-v1` is receiving 100% of the traffic now to our main route by executing:

```
APP_ROUTE=$(kn route list | awk '{print $2}' | sed -n 2p)
curl --insecure $APP_ROUTE
```

> **Challenge:** *As a test, route all of the traffic back to `greeter-v2` (green).*

Congrats! You now are able to apply a a `blue-green` deployment pattern using Serverless.  In the next assignment, we will look at `canary release` deployments.
