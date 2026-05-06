# Topic: AWS

> **Status:** stub. Will harden against a real AWS-deployed project.

## Recommended baseline

| Concern | Choice |
|---|---|
| SDK | AWS SDK v3 (modular, tree-shakeable) — never v2 |
| Lambda runtime | Node 22 (`nodejs22.x`) |
| Lambda handler shape | Single async function exporting `handler`, no framework |
| API Gateway | HTTP API (v2), not REST API (v1), unless you need Cognito auth |
| Local dev | LocalStack or `serverless-offline` for Lambda; `aws-sdk-client-mock` for tests |
| IaC | CDK (TypeScript) or Terraform — pick per project, document |
| Secrets | AWS Secrets Manager or Parameter Store, never env files in production |

## Patterns (to expand)

- One Lambda handler per concern. Don't build "monolith Lambdas".
- Cold-start matters: avoid heavy top-level imports; lazy-init clients.
- Reuse SDK clients across invocations (module-level — Lambda preserves the module between warm invocations).
- Logging: use `console.log(JSON.stringify(...))` so CloudWatch parses it as structured. Don't pull in winston/pino unless you have a reason.
- Error handling: throw at the boundary, let API Gateway map to 502/500. Catch and return business errors as 400/404.

## Open questions

- Whether to standardize on CDK or Terraform.
- Multi-region deployment patterns.
- Cost monitoring / tagging conventions.
