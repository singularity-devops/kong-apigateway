
## Spec-driven Development - A Framework For Declarative API Gateways

### Overview
In the API and microservices world, spec-driven development has been widely appraised for its design-first approach which ensures that there is a single source of truth for both systems providing and consuming services. API gateways typically support importing of OpenAPI specifications and can generate skeleton reverse proxies for underlying APIs but that's about it - just initialisation, no updates. Without updates, can it really be considered spec-driven development?

In reality, API gateways are also responsible for security, request orchestration and mediation, monitoring and reusable non-business logic. These configurations usually have their own lifecycles and fall out of sync when the specification is updated. What if these configurations could be defined as part of the specification? The entire reverse proxy configuration would live together with the single source of truth, allowing for true spec-driven development.

This article will provide a proof of concept framework for a declarative API gateway implementation covering:
 - Creation of an EKS cluster on AWS
 - Installation of the Kong Gateway (Free mode) which includes the Kong Manager UI on top of the open-source functionality
 - A sample OpenAPI specification with Kong plugin definitions
 - Inso CLI to generate Kong declarative configurations from the OpenAPI specification
 - Deck CLI to deploy, export and rollback Kong declarative configurations
 - Mockbin to inspect requests

### Pre-requisites

 - Sufficient access to an AWS account to create EKS clusters
 - [Install eksctl CLI](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) - this is used to simplify EKS cluster creation
 - [Install and configure aws CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) - this is used to authenticate and get access to AWS
 - [Install kubectl CLI](https://kubernetes.io/docs/tasks/tools/) - this is used to run commands against the Kubernetes cluster
 - [Install inso CLI](https://docs.insomnia.rest/inso-cli/install) - this is used to generate Kong declarative configurations given an OpenAPI specification
 - [Install deck CLI](https://docs.konghq.com/deck/latest/installation/) - this is used to run commands against the Kong Gateway

### Instructions

 1. Create an EKS cluster using eksctl:
``
eksctl create cluster --name kong-demo --region ap-southeast-2
``
2. Install Kong Gateway on EKS cluster. At the time of writing, it is much easier to follow the [Kong Enterprise installation instructions](https://docs.konghq.com/kubernetes-ingress-controller/latest/deployment/kong-enterprise/) and use that YAML manifest as it is more complete. Kong Gateway will be installed alongside the Kubernetes Ingress Controller. For the installation to succeed, it will be necessary to comment out all the license references in the YAML manifest - this will not provide access to enterprise features like Vitals, Dev Portal, RBAC and enterprise plugins. A working example of this YAML manifest has been included in this repository as the file **kong-ingress-enterprise.yaml** - note this example is a proof of concept and is not intended for production usage.
	

	    kubectl create namespace kong
	    kubectl create secret generic kong-enterprise-superuser-password  -n kong --from-literal=password=chooseyourownpassword
	    kubectl apply -f kong-ingress-enterprise.yaml

3. Wait a few minutes and verify the installation has succeeded:
``
kubectl get pods -n kong
``
4. Set local environment variables and patch the deployment so that Kong Manager knows where the Kong Admin URI is located:

		export KONG_PROXY_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" service -n kong kong-proxy)
		curl $KONG_PROXY_URI
		export KONG_ADMIN_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" service -n kong kong-admin)
		curl $KONG_ADMIN_URI
		export KONG_MANAGER_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" service -n kong kong-manager)
		curl $KONG_MANAGER_URI
		kubectl patch deployment -n kong ingress-kong -p "{\"spec\": { \"template\" : { \"spec\" : {\"containers\":[{\"name\":\"proxy\",\"env\": [{ \"name\" : \"KONG_ADMIN_API_URI\", \"value\": \"${KONG_ADMIN_URI}\" }]}]}}}}"

	If the curl commands return empty responses, try the following export commands instead:

		export KONG_PROXY_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-proxy)
		export KONG_ADMIN_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-admin)
		export KONG_MANAGER_URI=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-manager)

	Access the address of $KONG_MANAGER_URI in a browser and feel free to explore the UI.
5. Open the **petstore-with-plugins.yaml** file and take a moment to understand the Kong plugin syntax in the commented sections. The specification contains the following [plugins](https://docs.konghq.com/hub/):
	- Key Authentication
	- Rate Limiting
	- Correlation ID
	- CORS
	- Request Transformer

	To easily inspect the requests and see how the plugins are extending the functionality of the gateway, create a [Mockbin](https://mockbin.org/) and modify the "servers" section of the OpenAPI specification with your Mockbin URL.

	Create the Kong declarative configuration from the OpenAPI specification:

	    inso lint spec petstore-with-plugins.yaml
		inso generate config petstore-with-plugins.yaml -o declarative-petstore.yaml

	Under the hood, the inso CLI uses [openapi-2-kong](https://www.npmjs.com/package/openapi-2-kong) to convert the OpenAPI specification, including the Kong [OpenAPI extensions](https://swagger.io/docs/specification/openapi-extensions/) into a Kong declarative configuration.
6. Validate the Kong declarative configuration, dry-run the deployment and then deploy the configuration to Kong:

	    deck validate -s declarative-petstore.yaml
		deck diff -s declarative-petstore.yaml --kong-addr http://$KONG_ADMIN_URI --skip-consumers
		deck sync -s declarative-petstore.yaml --kong-addr http://$KONG_ADMIN_URI --skip-consumers
	Inspect the Kong Manager UI to see the changes. Note how the Kong Service and certain Kong Routes have been configured with plugins as per the OpenAPI specification.
7. Try send a request to create a pet:

		curl http://${KONG_PROXY_URI}/pet -H "Content-Type: application/json" -d '{
		  "id": 0,
		  "category": {
		    "id": 0,
		    "name": "string"
		  },
		  "name": "doggie",
		  "photoUrls": [
		    "string"
		  ],
		  "tags": [
		    {
		      "id": 0,
		      "name": "string"
		    }
		  ],
		  "status": "available"
		}' -v

	The response should be an error about a missing API key which means the Key Authentication plugin is working as expected.
8. Create a Consumer along with a Credential and try the request again with an API key:

	    curl -i -X POST --url http://$KONG_ADMIN_URI/consumers/ --data "username=Jason"
		curl -i -X POST --url http://$KONG_ADMIN_URI/consumers/Jason/key-auth/
		curl -X POST http://${KONG_PROXY_URI}/pet -H "apikey: YOUR_API_KEY_HERE" -H "Content-Type: application/json" -d '{
		  "id": 0,
		  "category": {
		    "id": 0,
		    "name": "string"
		  },
		  "name": "doggie",
		  "photoUrls": [
		    "string"
		  ],
		  "tags": [
		    {
		      "id": 0,
		      "name": "string"
		    }
		  ],
		  "status": "available"
		}' -v
	The request should succeed this time. Also note the CORS headers in the response which indicates the CORS plugin is working as expected:
	`Access-Control-Allow-Origin: some-whitelisted-cors-origin.com`
	
	If the request is send fast enough, it will also trigger the Rate Limiting plugin.
9. Inspect the Mockbin history to see how the `kong-request-id` header along with the other arbitrary headers are present. These were injected by the Correlation ID and Request Transformer plugins as per the OpenAPI specification configuration. Similarly, note how the `photoUrls` key name in the request body has been transformed into `urls` as per the configuration.

	Feel free to make changes to the OpenAPI specification and repeat steps 5 and 6 to update the Kong Gateway configuration. After all, the entire point is that the specification can be used for not just initialising, but also updating the API gateway configuration.
11. As a bonus, export the entire Kong Gateway configuration:
`deck dump --kong-addr http://$KONG_ADMIN_URI --skip-consumers`

	Inspect the `kong.yaml` output file. The declarative nature of the gateway makes it trivial to take snapshots of its state. This file could be stored in an artifact repository and the entire gateway can be easily rolled back by providing it as input to the `deck sync` command.

### Conclusion
This article has hopefully demonstrated how spec-driven development can be taken to the next level. Instead of just using a specification to initialise a skeleton reverse proxy and then being set aside as an afterthought, it can be made a core part of an API's ongoing lifecycle by maintaining the configuration as part of the specification.

Kong is one of the few API gateways that enables this kind of spec-driven development. Those interested may also want to have a look at some of the other tools in the Kong ecosystem such as the [Dev Portal](https://docs.konghq.com/gateway/latest/developer-portal/) or [Kuma](https://kuma.io/docs/1.7.x/) service mesh.
