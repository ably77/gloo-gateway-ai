# gloo-gateway-ai

In this blog we will use Gloo Gateway to create an AI API Gateway
- AI Proxy - Unified endpoint for all apps, regardless of backend LLM
- LLM Delegation - Separate LLM Routes by functionality and groups
- Prompt Templates - Templatize your prompt format
- Security - Protect your AI Gateway using api-key ExtAuth
- Observability - Track your LLM API usage using the gateway


### curl the OpenAPI LLM Directly

Description: curl the OpenAPI LLM directly, passing in the appropriate headers and body for the request

Input:
```bash
curl https://api.openai.com/v1/chat/completions   -H "Content-Type: application/json"   -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 2 minute pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Output:
```bash
{
  "id": "chatcmpl-9Crror2W84b3RmOqplwDYbTBFGYL6",
  "object": "chat.completion",
  "created": 1712854028,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Sure! Imagine you have a Kubernetes cluster with multiple microservices communicating with each other. As the number of services grow, managing the network traffic, security, and observability becomes increasingly complex. This is where a service mesh comes in.\n\nA service mesh such as Istio or Linkerd provides a dedicated infrastructure layer to handle service-to-service communication within your cluster. Here's why you should consider using a service mesh in your Kubernetes environment:\n\n1. **Traffic Management**: Service mesh allows you to easily control traffic routing, implement load balancing, and configure retries and timeouts without making changes to your application code. This helps in improving the resiliency and reliability of your services.\n\n2. **Security**: Security is paramount in a microservices architecture. Service mesh provides end-to-end encryption, mTLS authentication, and fine-grained access control policies to secure communication between services, ensuring data integrity and confidentiality.\n\n3. **Observability**: With a service mesh, you get detailed insights into the traffic flowing between services. You can monitor performance metrics, trace requests for debugging purposes, and visualize the service dependencies to identify bottlenecks or failures in your system.\n\n4. **Resilience**: Service mesh includes features like circuit breaking and fault injection to enhance the resilience of your applications. It can automatically handle failover scenarios and provide a seamless user experience even during service disruptions.\n\n5. **Consistent Policies**: Service mesh allows you to define and enforce policies consistently across all services in your cluster. Whether it's rate limiting, access control, or traffic shaping, you can centrally manage these policies without modifying individual services.\n\nIn conclusion, a service mesh simplifies the complexity of microservices communication by providing a unified platform for traffic management, security, and observability. It empowers your team to focus on building features rather than dealing with networking concerns, ultimately improving the scalability and robustness of your Kubernetes applications."
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 57,
    "completion_tokens": 378,
    "total_tokens": 435
  },
  "system_fingerprint": "fp_b28b39ffa8"
}
```

### curl the Gemini LLM Directly

Description: curl the Gemini LLM directly, passing in the appropriate headers and body for the request

Input:
```bash
curl \
  -H 'Content-Type: application/json' \
  -d '{"contents":[{"parts":[{"text":"Write a 10 word story about a magic surfboard"}]}]}' \
  -X POST 'https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key='$GEMINI_API_KEY''
```

Output:
```bash
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "Lost surfer finds enchanted board, rides perfect waves forevermore."
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0,
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HATE_SPEECH",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HARASSMENT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ]
}
```

### AI Proxy - Unified endpoint for all apps, regardless of backend LLM

Description: Configure a unified egress point for AI that can be extended to consume any public or self-hosted LLM and control, secure, and observe AI requests going through the gateway. Note the use of path rewrites from `/openai` to `/v1/chat/completions` and `/gemini` to `/v1beta/models/gemini-pro:generateContent`  to simplify the LLM egress path. The end consumer will route to the following URIs:

```
https://ai-gateway.demo.glooplatform.com/openai
https://ai-gateway.demo.glooplatform.com/gemini
```

Create an External Service for OpenAI
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: openai-chatgpt
  namespace: ai-gateway-ws-config
spec:
  hosts:
  - api.openai.com
  ports:
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
EOF
```

Create an External Service for Gemini
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: gemini-externalservice
  namespace: ai-gateway-ws-config
spec:
  hosts:
  - generativelanguage.googleapis.com
  ports:
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
EOF
```

Create a route table that routes to the OpenAI LLM ExternalService:
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-openai-routetable
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: catch-all
      matchers:
      - uri:
          prefix: /openai
      - uri:
          prefix: /v1/chat/completions
      forwardTo:
        pathRewrite: /v1/chat/completions
        hostRewrite: api.openai.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: openai-chatgpt
            namespace: ai-gateway-ws-config
EOF
```

Input:
```bash
curl https://ai-gateway.demo.glooplatform.com/openai   -H "Content-Type: application/json"  -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 50 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Output:
```bash
{
  "id": "chatcmpl-9Fl4tACpu2E2Tj7vLuqkX5TB75T8w",
  "object": "chat.completion",
  "created": 1713542915,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "A service mesh simplifies and enhances communication between microservices in your Kubernetes cluster, offering advanced features like load balancing, traffic splitting, observability, and security policies. With automatic service discovery and resilient communication, a service mesh can improve reliability, scalability, and performance of your applications without requiring code changes."
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 57,
    "completion_tokens": 60,
    "total_tokens": 117
  },
  "system_fingerprint": "fp_d9767fc5b9"
}
```

Create a route table that routes to the Gemini LLM ExternalService:
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-gemini-routetable
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: catch-all
      matchers:
      - uri:
          prefix: /gemini
      - uri:
          prefix: /v1beta/models/gemini-pro:generateContent
      forwardTo:
        pathRewrite: /v1beta/models/gemini-pro:generateContent
        hostRewrite: generativelanguage.googleapis.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: gemini-externalservice
            namespace: ai-gateway-ws-config
EOF
```

Input:
```bash
curl \
-H 'Content-Type: application/json' \
-d '{
  "contents": [
    {
      "parts": [
        {
          "text": "Write a 10 word story about a magic surfboard"
        }
      ]
    }
  ]
}' \
-X POST 'https://ai-gateway.demo.glooplatform.com/gemini?key='$GEMINI_API_KEY''
```

Output:
```bash
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "Surfer rode magical waves, defying gravity's call."
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0,
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HATE_SPEECH",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HARASSMENT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ]
}
```

### cleanup
Clean these two routes up as next we will be showing how to better organize these routes using delegations

```bash
kubectl delete routetable -n ai-gateway-ws-config direct-to-openai-routetable
kubectl delete routetable -n ai-gateway-ws-config direct-to-gemini-routetable
```

### LLM Delegation - Separate LLM Routes by functionality and groups

Description: Route table delegations allows for the decentralized management of route tables, enabling teams to independently control access to their services behind a shared host domain (in this case, multiple LLM backends). This separation of concerns enhances scalability and autonomy within the service mesh architecture.

Create a root/parent route table that the "ops team" persona will own. This route table is in control of the hostname, virtualgateways, and selected child routes that are exposed through the gateway. The following parent route table delegates to two child route tables that serve OpenAI and Gemini LLM backends
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

Now create a child route owned by the team using OpenAI. This allows the delegate team to manage the labels, matchers, and forwardTo paths of their child route table. The delegated team is not responsible for the hostname or gateway that their route is exposed on, or the routing patterns of the team using Gemini LLM
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: openai-catchall-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: openai
    prompt-template: "none"
spec:
  http:
    - name: catch-all
      matchers:
      - uri:
          prefix: /openai
      - uri:
          prefix: /v1/chat/completions
      forwardTo:
        pathRewrite: /v1/chat/completions
        hostRewrite: api.openai.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: openai-chatgpt
            namespace: ai-gateway-ws-config
EOF
```

Test the curl command again to make sure that it still works
```bash
curl https://ai-gateway.demo.glooplatform.com/openai   -H "Content-Type: application/json" -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 50 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Now create a child route owned by the team using Gemini. This allows the delegate team to manage the labels, matchers, and forwardTo paths of their child route table. The delegated team is not responsible for the hostname or gateway that their route is exposed on, or the routing patterns of the team using OpenAI LLM
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: gemini-catchall-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: gemini
    prompt-template: "none"
spec:
  http:
  - name: catch-all
    matchers:
    - uri:
        prefix: /v1beta/models/gemini-pro:generateContent
    - uri:
        prefix: /gemini
    forwardTo:
      pathRewrite: /v1beta/models/gemini-pro:generateContent
      hostRewrite: generativelanguage.googleapis.com
      destinations:
      - kind: EXTERNAL_SERVICE
        port:
          number: 443
        ref:
          name: gemini-externalservice
          namespace: ai-gateway-ws-config
EOF
```

Test the curl command again to make sure that it still works
```bash
curl \
-H 'Content-Type: application/json' \
-d '{
  "contents": [
    {
      "parts": [
        {
          "text": "Write a 10 word story about a magic surfboard"
        }
      ]
    }
  ]
}' \
-X POST 'https://ai-gateway.demo.glooplatform.com/gemini?key='$GEMINI_API_KEY''
```

Delegations allow us to provide a separation of concerns across teams as well as functions. We will continue building out our AI Gateway example using delegations to show how to implement specific policies on selected routes.

### Prompt Templates - Templatize your prompt format

#### ELI5 OpenAI Prompt Template
Description: Configure a Gloo Gateway Transformation Policy that manages inputs using custom headers, and transforms these inputs into templatized prompts. The following ELI5 template uses the `x-api-key`, `x-template`, and `x-prompt` headers to explain a topic like a 5 year old. Additionally, a user can specify an `x-model` header in order to consume a different LLM model (default is set to gpt-3.5-turbo). Configure an additional delegate route for the ELI5 prompt template to configure the request to the LLM based on these specific headers.

```bash
kubectl apply -f- <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: openai-eli5-prompt-transformation
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: "eli5"
  config:
    request:
      injaTemplate:
        headers:
          Authorization:
            text: 'Bearer {{ api_key}}'
        body:
          text: |
            {
              "model": "{% if header("x-model") != "" %}{{ llm_model }}{% else %}gpt-3.5-turbo{% endif %}",
              "messages": [
                {
                  "role": "system",
                  "content": "Explain like you are 5 years old"
                },
                {
                  "role": "user",
                  "content": "{{ prompt }}"
                }
              ]
            }
        extractors:
          # extracts an x-api-key header for the Authorization: Bearer <token>
          api_key:
            header: 'x-api-key'
            regex: '.*'
          # extracts x-model header for the body input
          llm_model:
            header: 'x-model'
            regex: '.*'
          # extracts x-prompt header for body input
          prompt:
            header: 'x-prompt'
            regex: '.*'
EOF
```
 
Configure delegate RouteTable for ELI5 Prompt Template Route for OpenAI
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: openai-eli5-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: openai
    prompt-template: "eli5"
spec:
  http:
    - name: eli5-translator
      matchers:
      - uri:
          prefix: /openai
        headers:
        - name: x-template
          value: "eli5"
        - name: x-prompt
      forwardTo:
        pathRewrite: /v1/chat/completions
        hostRewrite: api.openai.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: openai-chatgpt
            namespace: ai-gateway-ws-config
EOF
```

Modify the Parent route table to accept this delegate route
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-eli5
    labels:
      prompt-template: "eli5"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

input:
```bash
curl -X POST https://ai-gateway.demo.glooplatform.com/openai -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-template: eli5' -H 'x-prompt: star wars' -H 'Content-Type: application/json'
```

output:
```bash
{
  "id": "chatcmpl-9Flu6Vha5aT2oX7QKEGvYgEIzRVOC",
  "object": "chat.completion",
  "created": 1713546090,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "\"Star Wars is a really cool movie with lots of adventure and space battles! There are good guys called Jedi who have special powers and fight bad guys called Sith. They have lightsabers that go 'vroom vroom' and they fly in spaceships. It's so exciting!\""
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 22,
    "completion_tokens": 58,
    "total_tokens": 80
  },
  "system_fingerprint": "fp_d9767fc5b9"
}
```

#### System-User OpenAI Input Prompt Template
Description: Configure a Gloo Gateway Transformation Policy that manages inputs using custom headers, and transforms these inputs into templatized prompts. The following system-user input prompt template uses the `x-api-key`, `x-template`, `x-system-prompt`, `x-user-prompt` to take on a custom role and prompt. Additionally, a user can specify an `x-model` header in order to consume a different LLM model (default is set to gpt-3.5-turbo). Configure an additional delegate route for the ELI5 prompt template to configure the request to the LLM based on these specific headers.

```bash
kubectl apply -f- <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: openai-system-user-input-prompt-template
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: system-user-input
  config:
    request:
      injaTemplate:
        headers:
          Authorization:
            text: 'Bearer {{ api_key}}'
        body:
          text: |
            {
              "model": "{% if header("x-model") != "" %}{{ llm_model }}{% else %}gpt-3.5-turbo{% endif %}",
              "messages": [
                {
                  "role": "system",
                  "content": "{{ system_prompt }}"
                },
                {
                  "role": "user",
                  "content": "{{ user_prompt }}"
                }
              ]
            }
        extractors:
          # extracts an x-api-key header for the Authorization: Bearer <token>
          api_key:
            header: 'x-api-key'
            regex: '.*'
          # extracts x-model header for the model body input
          llm_model:
            header: 'x-model'
            regex: '.*'
          # extracts x-system-prompt header for the body input
          system_prompt:
            header: 'x-system-prompt'
            regex: '.*'
          # extracts x-system-prompt header for the body input
          user_prompt:
            header: 'x-user-prompt'
            regex: '.*'
EOF
```

Configure delegate RouteTable for system-user input prompt template route for OpenAI
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: openai-system-user-input-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: openai
    prompt-template: "system-user-input"
spec:
  http:
    - name: system-user-input
      matchers:
      - uri:
          prefix: /openai
        headers:
        - name: x-system-prompt
        - name: x-user-prompt
      forwardTo:
        pathRewrite: /v1/chat/completions
        hostRewrite: api.openai.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: openai-chatgpt
            namespace: ai-gateway-ws-config
EOF
```

Modify the Parent route table to accept this delegate route
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-eli5
    labels:
      prompt-template: "eli5"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-system-user-input
    labels:
      prompt-template: "system-user-input"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-system-user-input-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "system-user-input"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

input:
```bash
curl -X POST https://ai-gateway.demo.glooplatform.com/openai -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-system-prompt: you are a bagel expert' -H 'x-user-prompt: 10 words' -H 'Content-Type: application/json'
```

output:
```bash
{
  "id": "chatcmpl-9Fm2pBHlwdlw9WXNbixQuYlX5l4iU",
  "object": "chat.completion",
  "created": 1713546631,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Bagels are a round, chewy, and delicious bread product!"
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 19,
    "completion_tokens": 14,
    "total_tokens": 33
  },
  "system_fingerprint": "fp_d9767fc5b9"
}
```

#### Language Translator OpenAI Prompt Template

Description: Configure a Gloo Gateway Transformation Policy that manages inputs using custom headers, and transforms these inputs into templatized prompts. The following language translator prompt template uses the `x-api-key`, `x-language`, `x-prompt` to translate an input prompt into any language. Additionally, a user can specify an `x-model` header in order to consume a different LLM model (default is set to gpt-3.5-turbo). Configure an additional delegate route for the ELI5 prompt template to configure the request to the LLM based on these specific headers.

```bash
kubectl apply -f- <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: openai-language-translator-prompt-template
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: translator
  config:
    request:
      injaTemplate:
        headers:
          Authorization:
            text: 'Bearer {{ api_key}}'
        body:
          text: |
            {
              "model": "{% if header("x-model") != "" %}{{ llm_model }}{% else %}gpt-3.5-turbo{% endif %}",
              "messages": [
                {
                  "role": "system",
                  "content": "You are a translator, an expert in the {{ language }} language."
                },
                {
                  "role": "user",
                  "content": "Translate the {{ prompt }} in {{ language }}"
                }
              ]
            }
        extractors:
          # extracts an x-api-key header for the Authorization: Bearer <token>
          api_key:
            header: 'x-api-key'
            regex: '.*'
          # extracts x-model header for the model body input
          llm_model:
            header: 'x-model'
            regex: '.*'
          # extracts x-language header for the system prompt input
          language:
            header: 'x-language'
            regex: '.*'
          # extracts x-prompt header for the user prompt input
          prompt:
            header: 'x-prompt'
            regex: '.*'
EOF
```

Configure delegate RouteTable for system-user input prompt template route for OpenAI
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: openai-language-transformer-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: openai
    prompt-template: "translator"
spec:
  http:
    - name: language-translator
      matchers:
      - uri:
          prefix: /openai
        headers:
        - name: x-template
          value: translator
        - name: x-language
        - name: x-prompt
      forwardTo:
        pathRewrite: /v1/chat/completions
        hostRewrite: api.openai.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: openai-chatgpt
            namespace: ai-gateway-ws-config
EOF
```

Modify the Parent route table to accept this delegate route
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-translator
    labels:
      prompt-template: "translator"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-language-transformer-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "translator"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-eli5
    labels:
      prompt-template: "eli5"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-system-user-input
    labels:
      prompt-template: "system-user-input"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-system-user-input-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "system-user-input"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

input:
```bash
curl -X POST https://ai-gateway.demo.glooplatform.com/openai -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-template: translator' -H 'x-prompt: hello today i am here to speak about service mesh' -H 'x-language: thai' -H 'Content-Type: application/json'
```

output:
```bash
{
  "id": "chatcmpl-9F7JFFUggvrawGVD6CpZlCkdvDb14",
  "object": "chat.completion",
  "created": 1713390045,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "สวัสดีวันนี้ฉันมาที่นี่เพื่อพูดเกี่ยวกับเครือข่ายบริการในภาษาไทย"
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 36,
    "completion_tokens": 56,
    "total_tokens": 92
  },
  "system_fingerprint": "fp_c2295e73ad"
}
```




#### ELI5 Gemini Prompt Template
Description: Configure a Gloo Gateway Transformation Policy that manages inputs using custom headers, and transforms these inputs into templatized prompts. The following ELI5 template uses the `x-template`, and `x-prompt` headers to explain a topic like a 5 year old. Configure an additional delegate route for the ELI5 prompt template to configure the request to the LLM based on these specific headers.

```bash
kubectl apply -f- <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: gemini-eli5-template-transformation
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: "gemini-eli5"
  config:
    request:
      injaTemplate:
        body:
          text: |
            {
              "contents": [
                {
                  "role": "user",
                  "parts": [
                    {
                      "text": "Explain like you are 5 years old"
                    }
                  ]
                },
                {
                  "role": "model",
                  "parts": [
                    {
                      "text": "Sure I can explain any topic to you as if you were 5 years old, what would you like me to explain?"
                    }
                  ]
                },
                {
                  "role": "user",
                  "parts": [
                    {
                      "text": "{{ prompt }}"
                    }
                  ]
                }
              ]
            }
        extractors:
          # extracts x-prompt header for body input
          prompt:
            header: 'x-prompt'
            regex: '.*'
EOF
```
 
Configure delegate RouteTable for ELI5 Prompt Template Route for Gemini
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: gemini-eli5-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: gemini
    prompt-template: "gemini-eli5"
spec:
  http:
    - name: eli5-translator
      matchers:
      - uri:
          prefix: /gemini
        headers:
        - name: x-template
          value: eli5
        - name: x-prompt
      forwardTo:
        pathRewrite: /v1beta/models/gemini-pro:generateContent
        hostRewrite: generativelanguage.googleapis.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: gemini-externalservice
            namespace: ai-gateway-ws-config
EOF
```

Modify the Parent route table to accept this delegate route
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-eli5
    labels:
      prompt-template: "eli5"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-eli5
    labels:
      prompt-template: gemini-eli5
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "gemini-eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

input:
```bash
curl -X POST "https://ai-gateway.demo.glooplatform.com/gemini?key=$GEMINI_API_KEY" -H 'x-template: eli5' -H 'x-prompt: star wars' -H 'Content-Type: application/json'
```

output:
```bash
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "**Star Wars** is a story about a long time ago in a galaxy far, far away. There are good guys and bad guys, and they all have special powers.\n\nThe good guys are called the Jedi, and they use a power called the Force to help them do amazing things. The bad guys are called the Sith, and they also use the Force, but they use it for evil.\n\nThe main character in Star Wars is a young boy named Luke Skywalker. Luke lives on a desert planet called Tatooine, and he dreams of becoming a Jedi like his father. One day, Luke meets Obi-Wan Kenobi, an old Jedi Master, and Obi-Wan tells Luke about the Force and his destiny to become a Jedi.\n\nLuke joins Obi-Wan on a journey to rescue Princess Leia, a brave leader who has been captured by the evil Darth Vader. Along the way, Luke learns to use the Force and becomes a powerful Jedi.\n\nLuke and his friends fight against the Sith and the Empire, and they eventually defeat them and bring peace to the galaxy.\n\nHere is a simple explanation of the main characters in Star Wars:\n\n* **Luke Skywalker:** A young boy who dreams of becoming a Jedi.\n* **Obi-Wan Kenobi:** An old Jedi Master who teaches Luke about the Force.\n* **Princess Leia:** A brave leader who is captured by the Sith.\n* **Darth Vader:** A powerful Sith Lord who is Luke's father.\n\nI hope this helps you understand Star Wars!"
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0,
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HATE_SPEECH",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HARASSMENT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ]
}
```



#### Language Translator Gemini Prompt Template

Description: Configure a Gloo Gateway Transformation Policy that manages inputs using custom headers, and transforms these inputs into templatized prompts. The following Gemini language translator prompt template uses the `x-template`, `x-language`, and `x-prompt` to translate an input prompt into any language. Configure an additional delegate route for the ELI5 prompt template to configure the request to the LLM based on these specific headers.

```bash
kubectl apply -f- <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: gemini-language-translator-prompt-template
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: gemini-translator
  config:
    request:
      injaTemplate:
        body:
          text: |
            {
              "contents": [
                {
                  "role": "user",
                  "parts": [
                    {
                      "text": "Translate some text for me into {{ language }}"
                    }
                  ]
                },
                {
                  "role": "model",
                  "parts": [
                    {
                      "text": "Sure I can translate that for you into {{ language }}, what would you like me to translate?"
                    }
                  ]
                },
                {
                  "role": "user",
                  "parts": [
                    {
                      "text": " {{ prompt}}"
                    }
                  ]
                }
              ]
            }
        extractors:
          # extracts x-language header for the system prompt input
          language:
            header: 'x-language'
            regex: '.*'
          # extracts x-prompt header for the user prompt input
          prompt:
            header: 'x-prompt'
            regex: '.*'
EOF
```

Configure delegate RouteTable for language translator input prompt template route for Gemini
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: gemini-language-transformer-rt
  namespace: ai-gateway-ws-config
  labels:
    llm-type: gemini
    prompt-template: "gemini-translator"
spec:
  http:
    - name: language-translator
      matchers:
      - uri:
          prefix: /gemini
        headers:
        - name: x-template
          value: translator
        - name: x-language
        - name: x-prompt
      forwardTo:
        pathRewrite: /v1beta/models/gemini-pro:generateContent
        hostRewrite: generativelanguage.googleapis.com
        destinations:
        - kind: EXTERNAL_SERVICE
          port:
            number: 443
          ref:
            name: gemini-externalservice
            namespace: ai-gateway-ws-config
EOF
```

Modify the Parent route table to accept this delegate route
```bash
kubectl apply -f- <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: ai-gateway-root
  namespace: ai-gateway-ws-config
spec:
  hosts:
    - 'ai-gateway.demo.glooplatform.com'
    - 'api.openai.com'
    - 'generativelanguage.googleapis.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
  - name: openai-eli5
    labels:
      prompt-template: "eli5"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-translator
    labels:
      prompt-template: "translator"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-language-transformer-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "translator"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-system-user-input
    labels:
      prompt-template: "system-user-input"
      security: openai
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-system-user-input-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "system-user-input"
      sortMethod: ROUTE_SPECIFICITY
  - name: openai-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: openai-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: openai
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-eli5
    labels:
      prompt-template: gemini-eli5
      llm-type: gemini
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-eli5-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "gemini-eli5"
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-translator
    labels:
      prompt-template: gemini-translator
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-language-transformer-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "gemini-translator"
      # Delegates based on order of weights
      sortMethod: ROUTE_SPECIFICITY
  - name: gemini-catchall
    delegate:
      routeTables:
        # Selects tables based on name
        #- name: gemini-catchall-rt
        #  namespace: ai-gateway-ws-config
        # Selects tables based on labels
        - labels:
            llm-type: gemini
            prompt-template: "none"
      sortMethod: ROUTE_SPECIFICITY
EOF
```

input:
```bash
curl -X POST "https://ai-gateway.demo.glooplatform.com/gemini?key=$GEMINI_API_KEY" -H 'x-template: translator' -H 'x-prompt: hello today i am here to speak about service mesh' -H 'x-language: thai' -H 'Content-Type: application/json'
```

output:
```bash
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "สวัสดีครับ วันนี้ผมมาที่นี่เพื่อพูดเรื่องเซอร์วิสเมช"
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0,
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HATE_SPEECH",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_HARASSMENT",
          "probability": "NEGLIGIBLE"
        },
        {
          "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ]
}
```

### Security - Secure your gateway and mask your LLM API key(s) using custom headers

Description: Protect your AI Gateway access by using an api-key ext auth policy. This will make it so that a user-defined api-key key:value will be used to secure the gateway. Additionally, we can use the `headersFromMetadataEntry` feature in the ExtAuthPolicy to extract the actual OpenAPI LLM api-key from a secret and inject it into the request on successful auth

Create a Kubernetes secret containing the values needed for api-key auth as well as additional metadata to be used
```bash
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ai-gateway-api-key
  namespace: ai-gateway-ws-config
  labels:
    api-key: ai-gateway
type: extauth.solo.io/apikey
data:
  # value: solo.io
  # derived from the command 'echo -n solo.io | base64'
  api-key: c29sby5pbw==
  openai-api-key: <base64-encoded-value> # base64 encoded value of the OpenAI API key
EOF
```

Create an api-key ExtAuthPolicy that uses this secret
```bash
kubectl apply -f- <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: openai-gateway-api-key-auth
  namespace: ai-gateway-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        security: openai
  config:
    server:
      name: mgmt-ext-auth-server
      namespace: gloo-mesh
      cluster: mgmt
    glooAuth:
      configs:
      - apiKeyAuth:
          headerName: api-key
          headersFromMetadataEntry:
            x-api-key: 
              name: openai-api-key
          k8sSecretApikeyStorage:
            labelSelector:
              api-key: ai-gateway
EOF
```

Now you can curl the OpenAI routes that are labeled with `security: openai` to validate this use case. Instead of providing the `x-api-key: $OPENAI_API_KEY` header like before we can instead use `api-key: solo.io`
```bash
curl -X POST https://ai-gateway.demo.glooplatform.com/openai -H 'x-template: eli5' -H 'x-prompt: star wars' -H 'api-key: solo.io' -H 'Content-Type: application/json'
```

output:
```bash
' -H 'x-prompt: star wars' -H 'api-key: solo.io' -H 'Content-Type: application/json'
{
  "id": "chatcmpl-9FmzbUHjxVw1b4M3A1vfGVvrI30Dh",
  "object": "chat.completion",
  "created": 1713550275,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "\"Star Wars\" is a really cool movie about people in space who have amazing adventures. There are good guys called Jedi who have special powers, and bad guys like Darth Vader who use the dark side of the Force. They have epic battles with lightsabers and fly cool spaceships. It's a really exciting story with lots of action and fun characters!"
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 22,
    "completion_tokens": 72,
    "total_tokens": 94
  },
  "system_fingerprint": "fp_d9767fc5b9"
}
```