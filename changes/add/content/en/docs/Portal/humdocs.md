---
title: "Human User Study"
linkTitle: "Human User Study"
description: >
    How to run a human user study in SPHERE.
---

# User Guide

The SPHERE Study Portal is a website for running human user studies on the SPHERE testbed. There are two types of users: **researchers** who set up and manage studies, and **participants** who take part in them.

---

## For Researchers: Running a Study

### What you need before you start

You need an account on SPHERE. If you don't have one, create one [here](https://launch.sphere-testbed.net/registration). Once you have an account, make sure you are logged in at [https://launch.sphere-testbed.net](https://launch.sphere-testbed.net) before doing anything below - the study portal reads your login automatically.

---

### Step 1 - Open the admin dashboard

Go to [https://hum.sphere-testbed.net/admin](https://hum.sphere-testbed.net/admin) in your browser.

If you see a message saying you are not logged in, click the link to the Launch portal, sign in, and then come back to this page.

You will see a table (which will be empty at first) and, further down the page, two sections called **Organizations** and **Projects**.

---

### Step 2 - Create a study link

Scroll down to the **Organizations** or **Projects** section. These list the research groups and projects you are associated with.

Find the one you want to run the study under and click **Create Link**. The portal generates a unique invitation link for that organization or project and adds it to the **Participant Links** section below.

Click **Copy link** next to the new link. This is the URL you will send to your participants. It looks like:

```
https://hum.sphere-testbed.net/start?link=XXXXXXXXXXXX
```

> You can create multiple links - for example, one per recruiting platform or one per batch of participants. Each link is tracked separately.

---

### Step 3 - Recruit participants

Paste the link into your Prolific study, your MTurk HIT, or however you are recruiting. Participants open the link in their browser to begin.

---

### Step 4 - Monitor participants

Back at [https://hum.sphere-testbed.net/admin](https://hum.sphere-testbed.net/admin), the main table shows every participant who has clicked your link and started the study. It updates in real time - click **Refresh** to reload it.

Key columns in the table:

| Column | What it means |
|---|---|
| **Participant ID** | The ID the participant entered (e.g. their Prolific ID) |
| **Platform** | Prolific, MTurk, or Other |
| **Checklist Status** | Three checkboxes showing where the participant is: Onboarding done, Tasks done, Submission done |
| **Status** | In Progress or Completed |
| **Submission Key** | The code the participant is shown at the end - they paste this back into Prolific/MTurk to confirm completion |
| **Started / Completed** | Timestamps in UTC |

Click **any row** to open the detail view for that participant. You can see:

- Their full information (IP address, link they used, organization, researcher)
- A complete **event log** - every action they took, with timestamps and details
- Payment status controls

Use the **filter bar** at the top of the table to narrow down by status, platform, date range, or participant ID.

---

### Step 5 - Mark payments

Once you have confirmed a participant completed the study and should be paid:

1. Find their row in the table.
2. Check the **Payable** box.
3. After you have sent payment, check the **Paid** box.

Both checkboxes save immediately - there is no separate save button.

You can also do this from the participant detail page, where you can choose between **Not Payable**, **Payable**, and **Paid** using radio buttons.

---

### Step 6 - Export data

**All participants (Combined CSV):** Click the **Download Combined CSV** button in the top-right corner of the admin page. This exports every participant currently shown in the table, including any filters you have applied.

**One participant (Individual CSV):** Click the **⋯** button on any row, then click **Download CSV**. This downloads just that participant's full event log.

---

## For Participants: Taking a Study

### What you need

Nothing except a web browser and the link your researcher sent you. You do not need an account or a password.

---

### Step 1 - Open your study link

Click the link you received from the researcher (or from Prolific/MTurk). It will open a page called **Start Study** in your browser.

> If you see a red message saying "This page must be opened from your unique study link," it means the link is missing or incomplete. Go back to Prolific or MTurk and click the link from there.

---

### Step 2 - Fill in your details and consent

You will see two fields and a checkbox:

1. **Platform** - select where you found this study (Prolific, MTurk, or Other).
2. **Participant ID (PID)** - enter your participant ID. On Prolific, this is your Prolific ID. On MTurk, this is your Worker ID. If you are unsure where to find it, check the instructions in your Prolific/MTurk task.
3. **Consent checkbox** - check the box to confirm you agree to participate.

Once all three are filled in, the **Continue** button will become active. Click it.

---

### Step 3 - Complete the study tasks

You will be taken to the **Task** page. This page shows your progress checklist:

- ⬜ Onboarding & Consent
- ⬜ Complete Required Tasks
- ⬜ Record Submission Key

Follow the instructions shown on the page to complete the required tasks. As you progress, the checklist items will check off.

When you have finished everything, click the **Finish** button.

---

### Step 4 - Copy your submission key

The final page shows your **Submission Key** - a short code that proves you completed the study. It looks something like:

```
ABC12345
```

Click **Copy Submission Key** (or copy it manually). Then go back to Prolific or MTurk and paste it into the completion box. **You must do this to receive payment.** If you close the page without copying the key, contact the researcher.

---

## Common Questions

**I got an error when I tried to open the study link.**
Make sure you are clicking the exact link the researcher sent. Do not modify the URL. If the problem continues, contact the researcher.

**The Continue button on the Start page is greyed out.**
All three fields must be filled in: Platform, Participant ID, and the consent checkbox. Check that none are missing.

**I finished but the Done page shows "No submission found."**
This usually means you navigated away and came back. Contact the researcher and give them your Participant ID - they can look up your submission key from the admin dashboard.

**I am a researcher and I do not see my organization in the list.**
Your Launch portal account may not be associated with a research organization. Contact your testbed administrator.

**I am a researcher and I cannot access the admin page.**
Make sure you are logged into the Launch portal first at [https://launch.sphere-testbed.net](https://launch.sphere-testbed.net), then return to the admin page.
