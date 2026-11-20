FIREWALL HEALTH CHECK - PLAYWRIGHT (browser based)
Daily Checks Automation, check 1 of the runbook

This replaces the earlier PowerShell firewall check. It is a Playwright script
(Python) that uses a real browser.


WHAT IT DOES
Every run it opens the FMC (https://10.206.226.4) in Microsoft Edge, logs in and
goes through the same pages the engineer checks by hand:

  1. Overview > Dashboards > Summary Dashboard, Status tab
     appliance status, model, software version, uptime
  2. System > Health > Monitor, time range set to "Last 24 hours", then it opens
     each unit in the tree: Firewall Management Center, MTB_FP31_Dev,
     MTB_FP42_Prod1, MTB_FP42_Prod2. For each one it reads:
       - health status (Critical / Warning / Normal) and the reason shown
       - connected or not reachable
       - HA role (Active / Standby), with a warning after a failover
       - CPU and memory (WARNING above 50%)
       - Entire Disk and critical partitions (/, /Volume, /ngfw):
         WARNING from 80%, FAIL from 90%
  3. Analysis > Intrusions > Events and Analysis > Connections > Events
     (number of events shown; any intrusion event gives a WARNING to review)

It takes a screenshot of every page, writes the report and emails it.
Result per unit and overall: PASS, WARNING or FAIL.

It is read only: it only opens pages and reads them, nothing is changed on the
FMC or the firewalls. At the end it logs out.

The email uses the same layout as the "Daily Production Checks" email:
"Good Morning All, Daily production checks have been completed, and the
results are as follows." with the Checklist | Steps | Assigned to | Completed |
Comments table, then a details table per unit. When all is fine the comment
reads "Checks have been completed. There is nothing to report." The
screenshots and a CSV are attached.

Still for the engineer: deciding whether unusual traffic or an alert needs a
deeper look (root cause). The report and screenshots show what to look at.


WHAT IS IN FirewallHealthCheck-Playwright.zip
  .gitlab-ci.yml                 the pipeline
  fmc_health_check.py            the Playwright script
  requirements.txt               the Playwright package version
  config\firewall.settings.json  the values (FMC, account, units, email, limits)
  Install Playwright (once).bat  installs Playwright on a Windows PC
  Run Firewall Check.bat         run it by hand on a PC; the browser window is
                                 shown so you can watch every step

Upload everything to a NEW GitLab project (for example
"firewall-health-check"), keeping this layout. If you already uploaded the
earlier PowerShell version, replace those files with these.


STEP 1  WHAT THE WINDOWS RUNNER NEEDS (one time)
  - Python 3.9 or newer (python.org installer, tick "Add python to PATH").
    Check in a command window:  python --version
  - Microsoft Edge (already on Windows). Playwright drives the installed Edge,
    so no browser download is needed.
  - Playwright itself: the pipeline installs it on the first run with
    "pip install playwright" if it is missing. If the runner has no internet
    access, ask your admin to install it once from your internal package
    mirror, or run "Install Playwright (once).bat" on the runner.


STEP 2  AN FMC ACCOUNT FOR THE CHECK
Create a separate FMC user for the check (System > Users), for example
"svc_fw_healthcheck":
  - read-only is enough. It needs to view Overview > Dashboards,
    Devices > Device Management, System > Health (Monitor and Events) and
    Analysis (Intrusion and Connection events). As your checklist notes say:
    a local read-only account.
  - use a SEPARATE account, not your own, so the check does not log you out
    while you are working in FMC.


STEP 3  FILL IN config\firewall.settings.json
  fmc                   "10.206.226.4"
  protocol              "https"
  username              the account from Step 2 (password goes in GitLab, Step 4)
  browser               "msedge" (or "chrome")
  show_browser          false in GitLab; Run Firewall Check.bat always shows it
  units                 the names exactly as in the Health Monitor tree
  ha_pairs              each HA pair and its two units
  ha_expected_active    which unit is normally Active
  cpu_warn_percent      50   (the checklist says CPU and memory below 50%)
  memory_warn_percent   50
  disk_warn_percent     80
  disk_fail_percent     90
  ignore_alerts         known alerts that should not fail the check. It already
                        has "Data collection for CPU module is disabled" because
                        the CPU module is switched off in your health policy,
                        which makes every unit show Critical.
  check_events          true to read the intrusion and connection event pages
  time_range            "Last 24 hours"
  settle_seconds        how long to wait for the charts to load on each page (6)
  pages                 the FMC page addresses; leave them as they are
  assigned_to           "Contractor"
  email:
    enabled             true
    send_when           "always", or "problems" (only on WARNING/FAIL)
    smtp_server         your mail relay, e.g. the Exchange server name
    port                25 (or 587)
    use_ssl             true if the relay needs TLS (usually with 587)
    username            leave "" if the relay accepts mail without a login;
                        otherwise the login, password in SMTP_PASSWORD
    from                sender address
    to / cc             recipients, e.g. the *CI-HQ-ECE CONTR list
    subject             "Daily Production Checks - Firewall {date}";
                        [WARNING] or [FAIL] is added in front when needed
    signature           the sign-off lines (\n = new line)
    pickup_folder       leave ""; used only for testing


STEP 4  PASSWORDS IN GITLAB (never in the files)
Project > Settings > CI/CD > Variables > Add variable:
  FMC_PASSWORD     the FMC account password
  SMTP_PASSWORD    only if the mail relay needs a login
Untick "Expand variable reference". Tick "Mask variable" if GitLab allows it.
Leave "Protect variable" ticked only if you run on a protected branch.


STEP 5  RUN IT
Uses the Windows runner with the tag "netscaler" (same as the NetScaler tools).
If it is a different runner, change the tag in .gitlab-ci.yml. If Python starts
with "py" instead of "python" on the runner, change PYTHON in .gitlab-ci.yml.

  By hand:   Build > Pipelines > Run pipeline
  Every day: Build > Pipeline schedules > New schedule, e.g. "0 7 * * 1-5"
             (07:00 Monday to Friday), target branch main.

In GitLab the browser runs hidden in the background (the runner has no
screen). It is the same real Edge browser doing the same steps, and the
screenshots show exactly what it saw.

The job is green for PASS, orange for WARNING, red for FAIL or if it could not
run. Click the job to see each step. The report is under the job's
Job artifacts > Browse > reports:
  firewall-<date>.html                the same report as the email
  firewall-<date>.csv                 one line per unit
  firewall-screens-<date>\*.png       a screenshot of every page
  firewall-screens-<date>\*.txt       the text read from each page


TO WATCH IT ON A PC
On a Windows PC that can reach the FMC: double-click
"Install Playwright (once).bat", then "Run Firewall Check.bat" and type the
password. The Edge window opens and you can watch it log in and go through
every page.


FIRST RUN
Please send us the reports folder of the first run (the "firewall-screens"
folder in particular). The script was built from the pages in your KT
recording; the screenshots and page text let us confirm every value is read
correctly on your FMC 7.6.5, and adjust it quickly if a page looks different.


GOOD TO KNOW
  - The check never changes anything on the FMC or the firewalls.
  - If the email cannot be sent, the job output says why and the report is
    still saved (a run that was PASS turns orange so it gets noticed).
  - If the FMC cannot be reached or the login fails, the job turns red, and
    if email works the email says the check could not run.
