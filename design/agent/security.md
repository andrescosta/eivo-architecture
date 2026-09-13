https://vercel.com/blog/building-secure-ai-agents
https://github.com/protectai/llm-guard

Agreed — sidecar per service, centralized config and monitoring. Fits naturally into the existing K8s setup:

Sidecar intercepts all LLM traffic for that service — input and output scanning
Centralized config — rules, thresholds, and scanner profiles managed from one place, applied consistently across all services (EiBot, Assistance, Editorial)
Centralized monitoring — all injection attempts, flagged outputs, and scan results flow to the same observability stack you already have

Every service gets the same protection without embedding any scanning logic in the application code. And you can tighten or loosen rules globally without touching deployments.

https://github.com/DataDog/dd-trace-js/issues/7357

https://github.com/superagent-ai/superagent
https://ai-sdk.dev/tools-registry/superagent