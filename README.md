# it-onboarding-agent
Quick agent created with AI for the Head of IT @ skatteguiden

IT you just ask: an onboarding agent for Entra ID
"Maria Jensen starts Monday as a data analyst" becomes an Entra ID user, the right groups, licences through group-based licensing, Slack through SCIM, a passwordless first sign-in and a device ticket. Every step follows policy and is logged, and nobody opens an admin console.

This is a small working demo I built with Claude to show how I would approach AI-native IT operations. Claude reads the request. Everything that changes the tenant is deterministic, reviewed as code and logged.

Design
Principle	How the demo does it
AI understands, policy decides	Claude only turns free text into a structured request (name, role, start date). The plan comes from policy.yaml, not from the model.
The model sees as little as possible	The model gets the request text and nothing else: no tenant data, no tokens, no admin rights.
When unsure, stop and ask	An unknown role, low confidence, open questions, a past start date or a sensitive group stops the run and lists what a human must decide.
Least privilege for the agent	The agent has its own app identity in Entra. Licences come from group-based licensing, so it never needs licence-admin rights. Scope it further with a restricted administrative unit.
No standing passwords	The new user is created disabled with a random password nobody sees. First sign-in uses a one-time Temporary Access Pass that is valid from the start date.
Auditable	Every step goes to an append-only JSON lines log (audit.jsonl) that can be sent to Sentinel or another SIEM. The pass itself is never logged.
Configuration as code	Roles and groups live in policy.yaml. You change them in a pull request, and tests run on every change.
Dry run first	Nothing changes without --apply.
Graph permissions (application)
User.ReadWrite.All: create the user (scope it with an administrative unit)
GroupMember.ReadWrite.All: add the user to the groups in the policy
UserAuthenticationMethod.ReadWrite.All: issue the Temporary Access Pass
Use a certificate or workload identity federation instead of a client secret in production.

Run it
pip install -r requirements.txt
export ANTHROPIC_API_KEY=...            # Claude reads the request
python onboard.py "Maria Jensen starts Monday as a data analyst"   # dry run
python -m pytest tests                   # offline tests of the policy engine
To apply it against a test tenant, for example from the Microsoft 365 Developer Program, set ENTRA_TENANT_ID, ENTRA_CLIENT_ID and ENTRA_CLIENT_SECRET and add --apply.

Example dry run:

Onboard Maria Jensen as data analyst, starting 2026-11-09
  1. Create Entra ID user maria.jensen@example.dk (disabled until start date)
  2. Add to group All Employees
  2. Add to group LIC-M365-E3
  2. Add to group APP-Slack
  2. Add to group Team-Data
  2. Add to group APP-PowerBI-Pro
  2. Add to group INTUNE-macOS-Standard
  3. Issue a one-time Temporary Access Pass, valid from the start date (passwordless first sign-in)
  4. Device: macbook. Ticket to IT to assign a device in Apple Business Manager / Autopilot
Next steps I would build
A scheduled job that enables the account on the start date and runs offboarding the same way: disable the account, revoke sessions, remove it from groups and wipe or retire the device in Intune or Jamf.
Request intake from Slack or Teams, with an approval button for the cases where the agent stops.
Time-bound access through Entra PIM and access packages, so access is removed automatically.
The agent's own identity, scope and lifecycle registered and reviewed like a colleague's.
