FIREWALL HEALTH CHECK  (Cisco Secure Firewall Management Center)
Daily Checks Automation, check 1 of the runbook

WHAT IT DOES
Every run it logs in to the FMC (10.206.226.4) through its built-in connection,
reads the health of the FMC and every firewall it manages, writes a report and
emails it. It is read only: nothing is changed on the FMC or the firewalls.

It checks, for the FMC and each firewall (MTB_FP31_Dev, MTB_FP42_Prod1,
MTB_FP42_Prod2):
  1. Firewall connected to the FMC
  2. HA pair MTB_FP42_Prod_HA: one unit Active, the other Standby, config in
     sync, and a warning if a failover happened (Prod2 Active instead of Prod1)
  3. Health alerts from the last 24 hours (critical = FAIL, warning = WARNING)
  4. CPU and memory usage, latest and peak (WARNING above 50%)
  5. Disk usage and critical partitions such as /, /Volume, /ngfw
     (WARNING from 80%, FAIL from 90%)
Result per unit and overall: PASS, WARNING or FAIL.

The email uses the same layout as the "Daily Production Checks" email:
"Good Morning All, Daily production checks have been completed, and the
results are as follows." with the Checklist | Steps | Assigned to |
Completed | Comments table, then a details table per firewall. When all is
fine the comment reads "Checks have been completed. There is nothing to
report."

What stays manual: connection and intrusion events. Cisco does not offer them
through the FMC connection, so the report reminds the engineer to look at
Analysis > Connections / Intrusions in FMC when something needs a closer look.


WHAT IS IN FirewallHealthCheck.zip
  .gitlab-ci.yml                 the pipeline
  FirewallHealthCheck.ps1        the check
  config\firewall.settings.json  the values (FMC, account, email, limits)
  Run Firewall Check.bat         run it by hand on a Windows PC (asks for
                                 the password), handy for a first try

Upload everything to a NEW GitLab project (for example "firewall-health-check"),
keeping this layout. Do not put it in the NetScaler tools project: both have
their own .gitlab-ci.yml and one would replace the other.


STEP 1  AN FMC ACCOUNT FOR THE CHECK
Create a separate FMC user for the check (System > Users), for example
"svc_fw_healthcheck":
  - read-only is enough. It needs to view Devices > Device Management and
    System > Health (Health Monitor and Health Events). The built-in
    "Maintenance User" role may cover it; if not, make a custom read-only
    role with those two.
  - use a SEPARATE account, not your own: Cisco logs a user out of the web
    page when the same user logs in through the built-in connection.
The built-in connection (REST API) is on by default. To check:
System (gear) > Configuration > REST API Preferences > Enable REST API.


STEP 2  FILL IN config\firewall.settings.json
  fmc                   "10.206.226.4"
  username              the account from Step 1 (password goes in GitLab, Step 3)
  protocol              "https" - leave it (so the password is always encrypted)
  devices               [] = all firewalls in the FMC, or list names,
                        e.g. ["MTB_FP31_Dev", "MTB_FP42_Prod1", "MTB_FP42_Prod2"]
  alert_hours           how far back to read health alerts (24)
  cpu_warn_percent      50   (the checklist says CPU and memory below 50%)
  memory_warn_percent   50
  disk_warn_percent     80
  disk_fail_percent     90
  ha_expected_active    which unit is normally Active in each HA pair;
                        a warning is raised after a failover
  ignore_alerts         known alerts that should not fail the check. Already
                        contains "Data collection for CPU module is disabled"
                        because the KT recording shows the CPU module is
                        switched off in your health policy, which marks every
                        unit Critical. Add more text here the same way after
                        the first run if an alert is known and accepted.
  assigned_to           "Contractor"
  email:
    enabled             true
    send_when           "always", or "problems" (only on WARNING/FAIL)
    smtp_server         your mail relay, e.g. the Exchange server name
    port                25 (or 587)
    use_ssl             true if the relay needs TLS (usually with 587)
    username            leave "" if the relay accepts mail without a login;
                        otherwise the login, password in SMTP_PASSWORD
    use_windows_login   true to log in to the relay as the runner's Windows
                        account instead
    from                sender address
    to / cc             recipients, e.g. the *CI-HQ-ECE CONTR list
    subject             "Daily Production Checks - Firewall {date}";
                        [WARNING] or [FAIL] is added in front when needed
    signature           the sign-off lines (\n = new line)
    pickup_folder       leave ""; used only for testing (writes the email as
                        a file instead of sending it)


STEP 3  PASSWORDS IN GITLAB (never in the files)
Project > Settings > CI/CD > Variables > Add variable:
  FMC_PASSWORD     the FMC account password
  SMTP_PASSWORD    only if the mail relay needs a login
Untick "Expand variable reference". Tick "Mask variable" if GitLab allows it.
Leave "Protect variable" ticked only if you run on a protected branch.


STEP 4  RUN IT
Uses the same Windows runner (tag "netscaler") as the NetScaler tools. If it
is a different runner, change the tag in .gitlab-ci.yml.

  By hand:   Build > Pipelines > Run pipeline
  Every day: Build > Pipeline schedules > New schedule, e.g. "0 7 * * 1-5"
             (07:00 Monday to Friday), target branch main.

The job is green for PASS, orange for WARNING, red for FAIL or if it could not
run. Click the job to see the full output. The report files are under the job's
Job artifacts > Browse > reports:
  firewall-<date>.html          the same report as the email
  firewall-<date>.csv           one line per unit
  firewall-raw-<date>\*.json    exactly what the FMC returned


FIRST RUN
Please send us the reports folder of the first run (the "firewall-raw" folder
in particular). The FMC connection was built from Cisco's documentation, and
the raw files let us confirm every value against your FMC 7.6.5.


GOOD TO KNOW
  - The check never changes anything on the FMC or the firewalls.
  - If the email cannot be sent, the job output says why and the report is
    still saved (a run that was PASS turns orange so it gets noticed).
  - If the FMC cannot be reached or the login fails, the job turns red and,
    if email works, the email says the check could not run.
