# Deploy an gRPC service in Okteto Cloud

This example demonstrates how to route traffic to a gRPC service through Okteto Cloud.

This sample is based on https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/grpc/README.md

## Prerequisites

1. A free [Okteto Cloud](https://cloud.okteto.com) account 
1. `grpcurl` installed in your machine

### Step 1: Download your Okteto Cloud credentials

Log in to your Okteto Cloud account and click on the `Credentials` button on the left. Point your `KUBECONFIG` environment variable to the credentials file you just downloaded.

More information is available at https://okteto.com/docs/cloud/credentials.html

### Step 2: Launch the Application

```sh
$ kubectl create -f k8s.yaml
```

The `k8s.yaml` file is a standard kubernetes manifest. It includes a deployment, a service, and an ingress. It is running a grpc service
listening on port `50051`.

The sample application
[fortune-teller-app](https://github.com/kubernetes/ingress-nginx/tree/master/images/grpc-fortune-teller)
is a grpc server implemented in go. Here's the stripped-down implementation:

```go
func main() {
	grpcServer := grpc.NewServer()
	fortune.RegisterFortuneTellerServer(grpcServer, &FortuneTeller{})
	lis, _ := net.Listen("tcp", ":50051")
	grpcServer.Serve(lis)
}
```

A few things to note:

1. The application is not doing TLS. Instead, the ingress is terminating it , and grpc traffic will travel unencrypted to the pod. If you prefer to forward encrypted traffic to your POD and terminate TLS at the gRPC server itself, add the ingress annotation `nginx.ingress.kubernetes.io/backend-protocol: "GRPCS"` and provide your own certificate.
1. We've tagged the ingress with the annotation `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`.  This is the magic ingredient that sets up the appropriate nginx configuration to route http/2 traffic to our service.
1. Okteto Cloud is automatically providing the TLS at the ingress.
1. We've  also tagged the ingress with the annotation `dev.okteto.com/generate-host: "true"`.  This tells Okteto Cloud to replace the host with the one create by Okteto Cloud on deploy time.


### Step 3: Test the App

Once we've applied our configuration to kubernetes, it's time to test that we can actually talk to the backend.  To do this, we'll use the
[grpcurl](https://github.com/fullstorydev/grpcurl) utility.


Get the endpoint of your app by going to the Okteto Cloud UI, or by running:

```sh
export URL=$(kubectl get ing fortune-ingress -o=go-template='{{range .spec.rules}}{{.host}}{{end}}')
```


Call the app:

```sh
$ grpcurl $URL:443 build.stack.fortune.FortuneTeller/Predict
{
  "message": "Let us endeavor so to live that when we come to die even the undertaker will be sorry.\n\t\t-- Mark Twain, \"Pudd'nhead Wilson's Calendar\""
}
```