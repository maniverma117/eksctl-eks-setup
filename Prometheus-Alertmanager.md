# Prometheus Alertmanager Slack Integration

This guide will help you integrate Slack notifications into Prometheus Alertmanager by setting up a Slack Incoming Webhook and configuring the Alertmanager `alertmanager.yml` configuration file.

---

## 🚀 Step 1: Create a Slack Incoming Webhook URL

1. **Go to Slack App Management**  
   Visit: [https://api.slack.com/apps](https://api.slack.com/apps)

2. **Create a New App**
   - Click **“Create New App”**
   - Select **“From scratch”**
   - Name your app (e.g., `Prometheus Alert Bot`)
   - Select the desired Slack workspace
   - Click **Create App**

3. **Enable Incoming Webhooks**
   - In the left-hand sidebar, go to **“Incoming Webhooks”**
   - Click the toggle to **Activate Incoming Webhooks**
   - Scroll down and click **“Add New Webhook to Workspace”**

4. **Authorize the App**
   - Choose the Slack channel (e.g., `#alerts`) where alerts should appear
   - Click **Allow**

5. **Copy the Webhook URL**
   - Slack will generate a webhook URL like:
     ```
     https://hooks.slack.com/services/T01ABCD2EFG/B03HIJK4LMN/xYz123456789abcdefghijkl
     ```
   - **This URL is your `slack_api_url`**

---

## ⚙️ Step 2: Configure Alertmanager

Update your `alertmanager.yml` file to include Slack configuration.

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/T01ABCD2EFG/B03HIJK4LMN/xYz123456789abcdefghijkl'

inhibit_rules:
- equal:
  - namespace
  - alertname
  source_matchers:
  - severity = critical
  target_matchers:
  - severity =~ warning|info
- equal:
  - namespace
  - alertname
  source_matchers:
  - severity = warning
  target_matchers:
  - severity = info
- equal:
  - namespace
  source_matchers:
  - alertname = InfoInhibitor
  target_matchers:
  - severity = info
- target_matchers:
  - alertname = InfoInhibitor

receivers:
- name: "null"

- name: "slack-notifications"
  slack_configs:
  - channel: '#alerts'
    send_resolved: true
    title: '{{ .CommonAnnotations.summary }}'
    text: >-
      *Alert:* {{ .CommonLabels.alertname }}  
      *Severity:* {{ .CommonLabels.severity }}  
      *Description:* {{ .CommonAnnotations.description }}  
      *Details:* {{ range .CommonLabels.SortedPairs }} • *{{ .Name }}:* {{ .Value }}  
      {{ end }}

route:
  group_by: ['namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: "null"
  routes:
  - matchers:
    - alertname = "Watchdog"
    receiver: "null"
  - matchers:
    - severity = critical
    receiver: "slack-notifications"
  - matchers:
    - severity = warning
    receiver: "slack-notifications"
  - matchers:
    - severity = info
    receiver: "slack-notifications"

templates:
- /etc/alertmanager/config/*.tmpl
