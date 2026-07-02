# stackiq-infra
Infrastructure for deploying the stackiq project on your own server.



Generate secrets: 
kubectl delete secret stackiq-secrets -n stackiq
kubectl create secret generic stackiq-secrets `
  -n stackiq `
  --from-literal=DATABASE_URL= `
  --from-literal=POSTGRES_USER= `
  --from-literal=POSTGRES_PASSWORD= `
  --from-literal=POSTGRES_DB= `
  --from-literal=REDIS_URL= `
  --from-literal=BULLMQ_QUEUE_NAME= `
  --from-literal=OPENAI_API_KEY= `
  --from-literal=GITHUB_API_TOKEN= `
  --from-literal=GITHUB_MINER_TIMEOUT_MS=20000 `
  --from-literal=ISSUES_MINING_TIMEOUT_MS=300000 `
  --from-literal=ISSUES_MINING_LOOKBACK_DAYS=60 `
  --from-literal=ISSUES_MINING_MAX_ISSUES=80 `
  --from-literal=ISSUES_MINING_TIMELINE_ITEMS=15 `
  --from-literal=ISSUES_MINING_MAX_TIMELINE_PAGES=1 `
  --from-literal=ISSUES_MINING_INCLUDE_DEV_DEPENDENCIES=true `
  --from-literal=DEPENDENCY_CACHE_VERSION=v1 `
  --from-literal=DEPENDENCY_CACHE_TTL_DAYS=14 `
  --from-literal=PARTIAL_DEPENDENCY_CACHE_TTL_DAYS=1