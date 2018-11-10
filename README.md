# Authorization example using Keycloak and Istio

First we show an example of plain istio authentication and access control using JWT.
After that we try to apply the same to Knative services.
To make the example self hosted, but still realistic, we use [Keycloak](https://www.keycloak.org/).

1. Install Istio
1. Set up a sample pad
1. Block access for unauthenticated users
1. Install Keycloak
1. Set up a Realm and OpenID Connect client
1. Retrieve access token
1. Gain access to the test pod using Bearer token
1. Lock ourselves out again using Istio authorization
1. Log in as the user that is granted access

A special thanks goes to [Keycloak's blog ](https://blog.keycloak.org/2018/02/keycloak-and-istio.html)[post about Istio](http://planet.jboss.org/post/keycloak_and_istio).

## Preparations

A few things will vary depending on your cluster setup. You might be able to access the `istio-ingressgateway` over [NodePort]() or an actual public IP/FQDN. If not you can run tests in-cluster, using an alias or function.

```
function istiocurl() {
  kubectl run --restart=Never -t -i --rm --image=gcr.io/cloud-builders/curl istiocurl -- http://istio-ingressgateway.istio-system.svc.cluster.local$@
}
```

Soon enough you'll get to verify that your alias works.

## Install istio

We're using [Knative's Istio configuration](https://github.com/knative/serving/blob/v0.2.1/third_party/istio-1.0.2/download-istio.sh#L20).

```
kubectl apply -f https://github.com/knative/serving/releases/download/v0.2.1/istio-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/v0.2.1/istio-lean.yaml
```

## Sample pod

We use the Istio [Authentication Policy](https://istio.io/docs/tasks/security/authn-policy/#end-user-authentication) example,
but without the mutual TLS part.

```
kubectl apply -f ./02-istio-httpbin/
kubectl apply -f <(istioctl kube-inject -f ./02-istio-httpbin/httpbin.yaml) -n foo
```

If Istio injection worked you'll have three containers in the pod. Try `kubectl -n foo logs -l app=httpbin -c istio-proxy`.
Check that the service responds through Istio

```
$ istiocurl /headers -w '\n'
{ ... }
```

## Require authentication

Apply the [gateway and virtualservice](https://istio.io/docs/tasks/security/authn-policy/#end-user-authentication).

```
kubectl apply -f ./03-authentication
```

Note in the [Policy](./03-authentication/policy-jwt-example.yaml) that we're refering to the Keycloak service that is yet to be created.

Now, once the policy has propagated to you sidecars, access should be denied

```
$ istiocurl /headers -w '\n'
Origin authentication failed.
```

## Install Keycloak

Any installation is fine, but let's use the helm chart because it can automatically create an admin user:

```
helm install --namespace keycloak --name demo stable/keycloak \
  --set keycloak.ingress.enabled=true \
  --set keycloak.service.type=NodePort \
  --set keycloak.service.nodePort="30080" \
  --set keycloak.username=admin \
  --set keycloak.password=password
```

Use `kubectl -n keycloak get service demo-keycloak-http` to verify existence of the service that the authentication step (above) depends on.

Use appropriate means of accessing the UI in a browser, for example the NodePort enabled by the helm options above (`minikube service -n keycloak demo-keycloak-http`).
The Keycloak UI somtimes behaves erratically over `kubectl port-forward`.

You should see a login page where the username and password generated above works.

## Set up a login

 * [Create a realm](https://www.keycloak.org/docs/latest/getting_started/index.html#creating-a-realm-and-user) named `demo`.
 * Under the `Login` tab `Require SSL` select `none` to allow plain http.
 * Under ? disable password change on first login.
 * Create two [users](https://www.keycloak.org/docs/latest/getting_started/index.html#_create-new-user) `test1` and `test2` and use the `Credentials` tab to set their passwords to `test` with `Temporary` set to `OFF`.
 * Create an [OpenID Connect](https://www.keycloak.org/docs/latest/server_admin/index.html#oidc-clients) "client" named `myapp` in your demo realm.
   - Any "Root URL" is fine for this example
 * In the client make sure "Acces Type" is `public`. We want to make sure this demo setup is as insecure as possible 🙂.

## Authenticate

Finally there's some excitement! You may [test browser login](https://www.keycloak.org/docs/latest/getting_started/index.html#user-account-service) first, but we can do even better and brute force it (assuming `$(minikube ip):30080` is your keycloak host):

```
$ curl -X POST "http://$(minikube ip):30080/auth/realms/demo/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'username=test1&password=test&grant_type=password&client_id=myapp' -s \
  | jq -r '.access_token'  
# a long base64 string
```

You can use an online service like [jsonwebtoken.io](https://www.jsonwebtoken.io/) to see what it says. Try the same command for user `test2`, as you'll need to alternate between these logins shortly.

## Curl using the access token

Tokens expire after a while so you may want these two lines, together with the alias created during preparations.

```
$ token=$(<the curl + jq above>)
$ istiocurl /headers -w '\n' -H "Authorization: Bearer $token"
{ ... }
```

Now, does authentication only apply when going through Istio's gateway?

```
kubectl run --restart=Never -t -i --rm --image=gcr.io/cloud-builders/curl testcurl -- http://httpbin.foo:8000/  -w '\n' -H "Authorization: Bearer $token"
```

Indeed not.

## Require a specific claim

See your decoded token for what you can use as claims. Here we try the user (UU)ID from keycloak. Dig into the [Authorization](https://istio.io/docs/reference/config/authorization/) spec for details. And don't forget the [RbacConfig](https://istio.io/docs/reference/config/authorization/istio.rbac.v1alpha1/#RbacConfig) or your ServiceRole+binding will have zero effect.

```
kubectl apply -f ./08-authorization
# And then update the RBAC to allow one of your users in (see decoded token)
kubectl -n foo edit servicerolebinding jwt-binding
```

At this stage the Troubleshooting Authorization guide in Istio docs is a recommended read.

## Summary

You should now be blocked when using one user's token and allowed in when using the other.

You can keep editing the Keycloak User (and/or Realm) and the ServiceRoleBinding to test other [Properties](https://istio.io/docs/reference/config/authorization/constraints-and-properties/#properties) for authorization.

## Clean up

```
kubectl delete namespace keycloak
kubectl 
```