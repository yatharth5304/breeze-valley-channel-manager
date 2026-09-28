# Security

## Secrets
Never commit OTA credentials, API keys, access tokens, webhook secrets, database passwords, or production connection strings.

Use environment/configuration mechanisms appropriate to the eventual deployment environment.

## External Integrations
Treat OTA requests, responses, webhooks, and identifiers as external input. Validate and normalize them at the adapter boundary before passing data into core services.

## Logging
Logs should support debugging and auditability without exposing credentials or unnecessary guest-sensitive data.

## Idempotency
Inbound external events must be safely repeatable. Idempotency records should prevent duplicate reservation effects when the same external event is delivered more than once.

## Dependencies
Keep dependencies minimal and review security implications before adding them.
