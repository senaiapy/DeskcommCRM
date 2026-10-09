## ADDED Requirements

### Requirement: The diagnostic shows the update agent's last failure on one line
When the update agent's log changed in the last 120 minutes, `hostgator-setup-kit/healthcheck.sh` SHALL print the last log entry starting at its `<timestamp> [agent]` header, joining its non-empty lines into one line cut at 300 characters, instead of a fixed line offset.

#### Scenario: Proxy error body ending in a newline
- **WHEN** the agent's last POST got `no available server` followed by a blank line and the HTTP code
- **THEN** the diagnostic shows time, URL, body and code on one line instead of an empty line
