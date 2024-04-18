# gloo-gateway-ai

# curl the OpenAPI LLM Directly
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

# curl the OpenAPI LLM through Gloo Gateway
```bash
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-openai-routetable
  namespace: openai-ws-config
spec:
  hosts:
    - 'openai.demo.glooplatform.com'
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
          prefix: /
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
            namespace: openai-ws-config
---
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: openai-chatgpt
  namespace: openai-ws-config
spec:
  hosts:
  - api.openai.com
  ports:
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
```

curl: 
```bash
curl -v https://openai.demo.glooplatform.com   -H "Content-Type: application/json"   -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "bagel expert"
      },
      {
        "role": "user",
        "content": "5 words"
      }
    ]
  }'
```

output:
```bash
{
  "id": "chatcmpl-9F9bMgO5dETYys99al2aSowRqAJmP",
  "object": "chat.completion",
  "created": 1713398856,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Delicious, chewy, savory, versatile, satisfying."
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 16,
    "completion_tokens": 12,
    "total_tokens": 28
  },
  "system_fingerprint": "fp_c2295e73ad"
}
```

# Configure ELI5 Prompt Template
```bash
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: eli5-template-transformation
  namespace: openai-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: eli5
  config:
    request:
      injaTemplate:
        headers:
          Authorization:
            text: 'Bearer {{ api_key}}'
        body:
          text: |
            {
              "model": "{{ llm_model }}",
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
```
 
# Configure RouteTable for ELI5 Prompt Template Route
```bash
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-openai-routetable
  namespace: openai-ws-config
spec:
  hosts:
    - 'openai.demo.glooplatform.com'
    - 'api.openai.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: eli5-translator
      labels:
        prompt-template: eli5
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
        - name: x-template
          value: eli5
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
            namespace: openai-ws-config
    - name: catch-all
      matchers:
      - uri:
          prefix: /
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
            namespace: openai-ws-config
```

curl:
```bash
curl -X POST https://openai.demo.glooplatform.com -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-template: eli5' -H 'x-prompt: star wars' -H 'x-llm: openai' -H 'Content-Type: application/json'
```

output:
```bash
{
  "id": "chatcmpl-9F7Avlkoes6ZwODkXm0qr2SAEFShb",
  "object": "chat.completion",
  "created": 1713389529,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "\"Star Wars\" is a really cool movie with spaceships, robots, and awesome battles between good guys and bad guys. They use special powers called the Force and lightsabers to fight. It's a fun adventure story with lots of action and excitement!"
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 22,
    "completion_tokens": 52,
    "total_tokens": 74
  },
  "system_fingerprint": "fp_d9767fc5b9"
}
```

# Configure freeform input prompt template
```bash
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: freeform-template-transformation
  namespace: openai-ws-config
spec:
  applyToRoutes:
  - route:
      labels:
        prompt-template: freeform-input
  config:
    request:
      injaTemplate:
        headers:
          Authorization:
            text: 'Bearer {{ api_key}}'
        body:
          text: |
            {
              "model": "{{ llm_model }}",
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
```

# Configure RouteTable for Freeform Prompt Template Route
```bash
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-openai-routetable
  namespace: openai-ws-config
spec:
  hosts:
    - 'openai.demo.glooplatform.com'
    - 'api.openai.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: header-based-freeform
      labels:
        prompt-template: freeform-input
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
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
            namespace: openai-ws-config
    - name: eli5-translator
      labels:
        prompt-template: eli5
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
        - name: x-template
          value: eli5
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
            namespace: openai-ws-config
    - name: catch-all
      matchers:
      - uri:
          prefix: /
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
            namespace: openai-ws-config
```

curl:
```bash
curl -X POST https://openai.demo.glooplatform.com -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-system-prompt: you are a bagel expert' -H 'x-user-prompt: 10 words' -H 'x-llm: openai' -H 'Content-Type: application/json'
```

output:
```bash
{
  "id": "chatcmpl-9F7BBLsJgeXYjFdMzuYe98jZAhxHM",
  "object": "chat.completion",
  "created": 1713389545,
  "model": "gpt-3.5-turbo-0125",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Crunchy outside, soft inside, perfect for sandwiches or toast."
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

# Configure language translator prompt template
```bash
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: translator-template-transformation
  namespace: openai-ws-config
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
              "model": "{{ llm_model }}",
              "messages": [
                {
                  "role": "system",
                  "content": "You are a translator, an expert in the {{ language }} language"
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
```

# Configure RouteTable for Freeform Prompt Template Route
```bash
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: direct-to-openai-routetable
  namespace: openai-ws-config
spec:
  hosts:
    - 'openai.demo.glooplatform.com'
    - 'api.openai.com'
  virtualGateways:
    - name: mgmt-north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: header-based-freeform
      labels:
        prompt-template: freeform-input
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
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
            namespace: openai-ws-config
    - name: header-based-translator
      labels:
        prompt-template: translator
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
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
            namespace: openai-ws-config
    - name: eli5-translator
      labels:
        prompt-template: eli5
      matchers:
      - headers:
        - name: x-llm
          value: openai
        - name: x-api-key
        - name: x-model
        - name: x-template
          value: eli5
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
            namespace: openai-ws-config
    - name: catch-all
      matchers:
      - uri:
          prefix: /
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
            namespace: openai-ws-config
```

curl:
```bash
curl -X POST https://openai.demo.glooplatform.com -H 'x-api-key: '$OPENAI_API_KEY'' -H 'x-model: gpt-3.5-turbo' -H 'x-template: translator' -H 'x-prompt: hello today i am here to speak about service mesh' -H 'x-language: thai' -H 'x-llm: openai' -H 'Content-Type: application/json'
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