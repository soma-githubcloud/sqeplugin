# Plugin: Allure Server

**Activation**: Set `plugins.allureServer.enabled: true` in `configs/integrations.yml`.

## Capabilities

### Publish Reports to Allure Server
After tests complete (or any test run that generates `allure-results/`), if this plugin
is enabled, publish the results to a hosted Allure TestOps or self-hosted Allure Server:

1. Verify `allure-results/` directory exists with at least one `.json` result file
2. Run the publish command via shell MCP:
   ```
   npx allure-commandline send \
     --server-url <allureServerUrl> \
     --project-id <projectId> \
     allure-results
   ```
3. Return the report URL to the user:
   `Report published → <allureServerUrl>/projects/<projectId>/launches/latest`

### Auto-Publish on CI
When `autoPublish: true`, the `ci-github.yml.tmpl` template includes the Allure publish step
automatically (the `{{#if publishToAllureServer}}` block in the template).

## Configuration (in configs/integrations.yml)
```yaml
allureServer:
  enabled: true
  serverUrl: ${ALLURE_SERVER_URL}
  projectId: ${ALLURE_PROJECT_ID}
  autoPublish: false
```

## Required Environment Variables
- `ALLURE_SERVER_URL` — Base URL of your Allure Server (e.g., `https://allure.mycompany.com`)
- `ALLURE_PROJECT_ID` — Project identifier on the Allure Server

## Security Note
Never commit `ALLURE_SERVER_URL` tokens or credentials. Always reference as `${ENV_VAR}` in
`integrations.yml`. If the server requires authentication, set the token via:
```bash
export ALLURE_TOKEN=<your-token>
```
and reference it as `${ALLURE_TOKEN}` in your CI secrets.

## MCP Used
Uses `mcp__shell__execute_command` to run the `allure-commandline send` command.
Reads `allure-results/` via filesystem MCP to verify results exist before publishing.

## Error Handling
- If `allure-results/` is empty or missing → warn the user, do NOT attempt publish
- If server is unreachable → log the error URL and suggest checking `ALLURE_SERVER_URL`
- If publish succeeds → confirm with the report URL
