# HackCon 2026 - Secure PowerShell with GenAI
This repository includes the Presentation slides &amp; tools from HackCon 2026 talk -<br>
<b>_'Automating Securely with AI - Tips &amp; Insights for PowerShell Professionals'._</b><br>
### Scripts/tools used in this talk: ###
1. Simple function calling gpt directly from PowerShell using REST API:<br>
```
# First, get API key - e.g. https://platform.openai.com/settings/organization/api-keys, or https://platform.openai.com/account/api-keys

# then save it into an environment variable
$env:OPENAI_API_KEY = 'sk...2CyFz2'

function Ask-GPT ([string]$prompt, [int]$MaxTokens) {
$apiKey = $env:OPENAI_API_KEY
$headers = @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer $apiKey"
}
$body = @{
    "model" = "gpt-4o-mini"
    "prompt" = $prompt
    "max_tokens" = $MaxTokens
} | ConvertTo-Json

$uri = "https://api.openai.com/v1/completions"

Invoke-RestMethod -Uri $uri -Method POST -Headers $headers -Body $body
}

# example
$results = Ask-GPT -prompt $prompt -MaxTokens 100000
$Results.choices[0].text # | clip
```
<br>

2. Windows Memory Threat Analysis - Comprehensive memory threat analysis for process memory regions. Scan process(es) for suspicious patterns (protection+hex/strings), map memory regions to threads; then analyze for threats & produce a detailed report:<br>
https://github.com/YossiSassi/WindowsMemoryThreatAnalysis
<br>

3. PowerGuard Cloud - an open-source, security monitoring solution designed for Entra ID-joined environments. It fills a critical detection gap by providing immutable, cloud-based analysis of PowerShell activity-independent of local EDR agents:<br>
https://github.com/TenRoot/PowerGuard
<br>

4. PSAI - PowerShell AI module. Think ChatGPT meets PowerShell - Includes Autonomous Agents:<br>
https://github.com/dfinke/PSAI
<br>

5. AI implementation plan:<br>
[Example AI implementation plan](./AI%20Implementation%20Plan.md)
<br>

6. Other code snippets used throughout the presentation can be seen in the slides:
https://github.com/YossiSassi/HackCon_2026_PowerShell_AI/blob/main/presentation_hackcon2026_powershell_ai.pdf
