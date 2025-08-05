# GraphQL Federation Gateway Template

This template is used to create GraphQL federation gateways for vCluster environments. The gateway automatically discovers and federates services that are explicitly marked for GraphQL federation.

## 🚀 Features

- **Selective Service Discovery**: Only includes services with `graphql.federation/enabled: "true"` annotation
- **OpenAPI to GraphQL**: Automatically converts REST APIs with OpenAPI specs to GraphQL
- **Real-time Updates**: Service discovery runs every 5 minutes (configurable)
- **vCluster Native**: Designed to run within vCluster environments
- **Multi-namespace Support**: Can discover services across namespaces within the vCluster

## 📋 Service Annotation Requirements

For a service to be included in the GraphQL federation, it must have:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-api-service
  annotations:
    # Required: Mark service for federation
    graphql.federation/enabled: "true"
    
    # Optional: Custom OpenAPI path (defaults to common paths)
    graphql.federation/openapi-path: "/api/v1/openapi.json"
    
    # Optional: API version
    graphql.federation/api-version: "v1"
```

## 🏗️ Architecture

```
vCluster Environment
├── GraphQL Gateway (this template)
│   ├── Service Discovery (kubectl-based)
│   ├── OpenAPI → GraphQL Schema Conversion
│   └── GraphQL Mesh Federation
├── Microservices
│   ├── webservice-1 (graphql.federation/enabled: true) ✅
│   ├── webservice-2 (graphql.federation/enabled: true) ✅
│   ├── rasa-chatbot (no annotation) ❌
│   └── realtime-lenses (no annotation) ❌
```

## 🔧 Template Variables

When this template is used by the OAM ComponentDefinition, these variables are replaced:

- `{{ GATEWAY_NAME }}`: Name of the GraphQL gateway instance
- `{{ APP_CONTAINER }}`: Name of the application container repository
- `{{ NAMESPACE }}`: Target namespace for deployment
- `{{ DOCKER_REGISTRY }}`: Docker registry URL
- `{{ DOCKER_USERNAME }}`: Docker registry username
- `{{ VERSION }}`: Version tag for the Docker image
- `{{ GITOPS_REPO }}`: GitOps repository name

## 📦 Integration with OAM

This template is triggered when you include a GraphQL gateway component in your OAM Application:

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: my-platform
spec:
  components:
  # Regular web services
  - name: user-api
    type: webservice
    properties:
      image: user-api:latest
      # Service will be included if it has federation annotation
      
  - name: order-api
    type: webservice
    properties:
      image: order-api:latest
      # Service will be included if it has federation annotation
      
  # GraphQL Gateway
  - name: api-gateway
    type: graphql-gateway
    properties:
      repository: "my-team-graphql-gateway"
      serviceSelector:
        app.kubernetes.io/managed-by: "kubevela"
        
  # These won't be included in federation
  - name: chat-support
    type: rasa-chatbot
    properties:
      repository: "support-chatbot"
      
  - name: streaming-platform
    type: realtime-platform
    properties:
      name: "event-streaming"
```

## 🛡️ Security

- RBAC: GraphQL gateway runs with limited permissions
- Service Account: Uses `graphql-gateway-sa` with cluster-wide read permissions
- Network Policies: Can be configured to restrict access

## 🔍 Debugging

Check service discovery logs:
```bash
kubectl logs -l app.kubernetes.io/component=graphql-gateway -c user-container
```

Verify service annotations:
```bash
kubectl get ksvc -o yaml | grep -A 5 "graphql.federation"
```

## 📚 API Endpoints

- `/graphql` - Main GraphQL endpoint
- `/healthz` - Health check endpoint
- `/metrics` - Prometheus metrics (if enabled)

## 🔄 Development Workflow

1. Services are created with federation annotation
2. GraphQL gateway discovers them automatically
3. OpenAPI specs are converted to GraphQL schemas
4. Schemas are federated into single endpoint

## ⚙️ Configuration

The gateway is configured via ConfigMap. Key settings:

- `discovery.interval`: How often to check for new services
- `discovery.federationAnnotation`: Annotation key to look for
- `graphql.playground`: Enable/disable GraphQL playground
- `graphql.introspection`: Enable/disable introspection