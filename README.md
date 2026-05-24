# 🔔 Salesforce → Microsoft Teams Code Change Notification System

> **Real-time automated alerts to Microsoft Teams whenever any developer makes code changes in Salesforce Sandbox — built to keep QA teams informed during testing cycles.**

---

## 📋 Table of Contents
- [Problem Statement](#-problem-statement)
- [Solution Overview](#-solution-overview)
- [System Architecture](#-system-architecture)
- [Components Created](#-components-created)
- [Prerequisites](#-prerequisites)
- [Setup Guide](#-setup-guide)
- [Apex Classes](#-apex-classes)
- [Scheduling Jobs](#-scheduling-jobs)
- [Salesforce Limitations & Workarounds](#-salesforce-limitations--workarounds)
- [Notification Format](#-notification-format)
- [Maintenance Guide](#-maintenance-guide)
- [Author](#-author)

---

## ❓ Problem Statement

During testing cycles in Salesforce Sandbox:
- Developers were making **silent code changes** without informing the QA team
- QA would find a defect → developer would quietly fix it mid-session
- Developer would then claim *"it was always working"*
- QA had **no real-time proof** of when changes occurred
- Testing results became **unreliable**

**Goal:** Build an automated system that notifies the QA team on Microsoft Teams the moment any developer changes an Apex Class, Apex Trigger, or Flow in the Sandbox.

---

## ✅ Solution Overview

A fully **Salesforce-native** monitoring system that:

- ✅ Runs **every 5 minutes** automatically via Salesforce Scheduled Jobs
- ✅ Queries **SetupAuditTrail** to detect code changes
- ✅ Sends **formatted notifications** to a Microsoft Teams channel via Incoming Webhook
- ✅ Tracks **last checked timestamp** to prevent duplicate notifications
- ✅ Requires **zero manual intervention** once set up
- ✅ Detects changes to **Apex Classes, Apex Triggers, and Flows**

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     EXECUTION FLOW                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ⏰ 12 Scheduled Jobs                                                 │
│     (every 5 minutes)                                                 │
│          │                                                            │
│          ▼                                                            │
│  CodeChangeDetector.execute()                                         │
│          │                                                            │
│          ▼                                                            │
│  CodeChangeDetector.runAsync()  ← @future(callout=true)              │
│          │                                                            │
│          ▼                                                            │
│  CodeChangeDetector.checkForChanges()                                 │
│     ├── Read last timestamp ← Code_Check_Log__c                      │
│     ├── HTTP GET → Salesforce REST API → SetupAuditTrail             │
│     ├── Filter: Section = 'Apex Class' / 'Apex Trigger' / 'Flow'    │
│     ├── Filter: CreatedDate > lastCheckTimestamp                      │
│     ├── Build message string                                          │
│     ├── TeamsNotificationService.sendNotification()                  │
│     │        └── HTTP POST → Microsoft Teams Webhook                 │
│     └── System.enqueueJob(new SaveLogQueueable(latestChange))        │
│                   └── insert Code_Check_Log__c (new timestamp)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Components Created

| Component | Type | Purpose |
|-----------|------|---------|
| `TeamsNotificationService` | Apex Class | Sends HTTP POST to Microsoft Teams webhook |
| `CodeChangeDetector` | Apex Class (Schedulable) | Main engine — queries Audit Trail, triggers notification |
| `SaveLogQueueable` | Apex Class (Queueable) | Saves last check timestamp to prevent duplicates |
| `Webhook_Config__mdt` | Custom Metadata Type | Stores Teams webhook URL securely |
| `Code_Check_Log__c` | Custom Object | Stores last processed timestamp |
| `Salesforce_Tooling_API` | Remote Site Setting | Whitelist for Salesforce REST API endpoint |
| `Teams_Webhook` | Remote Site Setting | Whitelist for Microsoft Teams webhook endpoint |
| `CodeChangeDetector_0..55` | 12 Scheduled Jobs | Fire every 5 minutes throughout the hour |

---

## 📦 Prerequisites

- Salesforce Sandbox org access
- Microsoft Teams channel with **Incoming Webhook** configured
- Salesforce Admin or Developer access
- Developer Console access

---

## 🚀 Setup Guide

### Step 1 — Microsoft Teams Incoming Webhook

1. Open Microsoft Teams → Go to target channel
2. Right click channel → **Manage Channel**
3. **Connectors** → **Incoming Webhook** → Configure
4. Give it a name (e.g. `Salesforce Alerts`) → **Create**
5. **Copy the webhook URL** — you'll need it in Step 3

---

### Step 2 — Remote Site Settings

Go to **Setup → Remote Site Settings → New** and create two entries:

**Entry 1 — Salesforce API:**
```
Name: Salesforce_Tooling_API
URL:  https://YOUR-ORG-NAME.sandbox.my.salesforce.com
Active: ✅
```

**Entry 2 — Teams Webhook:**
```
Name: Teams_Webhook
URL:  https://accenture.webhook.office.com  (or your company's domain)
Active: ✅
```

> 💡 To find your org URL, run this in Anonymous Window:
> ```apex
> System.debug(URL.getOrgDomainUrl().toExternalForm());
> ```

---

### Step 3 — Custom Metadata Type

1. **Setup → Custom Metadata Types → New**
```
Label:        Webhook Config
Plural Label: Webhook Configs
Object Name:  Webhook_Config
Visibility:   All Apex code and API can use
```

2. Add a custom field:
```
Type:   Text
Label:  Webhook URL
Name:   Webhook_URL
Length: 255
```

3. **Manage Records → New** — add your Teams webhook URL:
```
Label:       Teams Webhook
Name:        Teams_Webhook
Webhook URL: https://your-teams-webhook-url-here
```

---

### Step 4 — Custom Object

1. **Setup → Object Manager → Create → Custom Object**
```
Label:       Code Check Log
Object Name: Code_Check_Log
```

2. Add a custom field:
```
Type:  DateTime
Label: Last Check Time
Name:  Last_Check_Time
```

---

### Step 5 — Create Apex Classes

Open **Developer Console → File → New → Apex Class** and create all 3 classes below.

---

## 💻 Apex Classes

### 1. TeamsNotificationService

```apex
public class TeamsNotificationService {
    public static void sendNotification(String title, String message) {
        Webhook_Config__mdt config = [
            SELECT Webhook_URL__c 
            FROM Webhook_Config__mdt 
            LIMIT 1
        ];
        
        String body = JSON.serialize(new Map<String, Object>{
            '@type'      => 'MessageCard',
            '@context'   => 'http://schema.org/extensions',
            'themeColor' => 'FF0000',
            'summary'    => title,
            'sections'   => new List<Object>{
                new Map<String, Object>{
                    'activityTitle'    => '⚠️ Salesforce Code Change Alert!',
                    'activitySubtitle' => 'Someone changed code in Sandbox!',
                    'text'             => message
                }
            }
        });
        
        HttpRequest req = new HttpRequest();
        req.setEndpoint(config.Webhook_URL__c);
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(body);
        
        new Http().send(req);
    }
    
    @future(callout=true)
    public static void sendNotificationAsync(String title, String message) {
        sendNotification(title, message);
    }
}
```

---

### 2. CodeChangeDetector

```apex
public class CodeChangeDetector implements Schedulable {
    
    public void execute(SchedulableContext sc) {
        runAsync();
    }
    
    @future(callout=true)
    public static void runAsync() {
        checkForChanges();
    }
    
    public static void checkForChanges() {
        // Step 1: Get last check timestamp
        DateTime lastCheck;
        List<Code_Check_Log__c> logs = [
            SELECT Last_Check_Time__c 
            FROM Code_Check_Log__c 
            ORDER BY CreatedDate DESC 
            LIMIT 1
        ];
        
        if(logs.isEmpty()) {
            lastCheck = DateTime.now().addMinutes(-2);
        } else {
            lastCheck = logs[0].Last_Check_Time__c;
        }
        
        // Step 2: Query SetupAuditTrail via REST API
        String query = 'SELECT+CreatedDate,CreatedBy.Name,Action,Section,Display' +
                       '+FROM+SetupAuditTrail+ORDER+BY+CreatedDate+DESC+LIMIT+50';
        
        HttpRequest req = new HttpRequest();
        req.setEndpoint(URL.getOrgDomainUrl().toExternalForm() 
            + '/services/data/v59.0/query/?q=' + query);
        req.setMethod('GET');
        req.setHeader('Authorization', 'OAuth ' + UserInfo.getSessionId());
        req.setHeader('Content-Type', 'application/json');
        
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        if(res.getStatusCode() == 200) {
            Map<String, Object> result = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
            List<Object> records = (List<Object>) result.get('records');
            
            if(records != null && records.size() > 0) {
                String msg = '';
                DateTime latestChange = lastCheck;
                
                for(Object rec : records) {
                    Map<String, Object> record = (Map<String, Object>) rec;
                    String section = (String) record.get('Section');
                    String createdDateStr = (String) record.get('CreatedDate');
                    
                    // Step 3: Filter only Apex Class, Apex Trigger, Flow
                    if(section != 'Apex Class' && section != 'Apex Trigger' && section != 'Flow') {
                        continue;
                    }
                    
                    // Step 4: Filter only changes after last check
                    DateTime createdDate = DateTime.valueOfGmt(
                        createdDateStr.replace('T', ' ').replace('Z', '')
                    );
                    if(createdDate <= lastCheck) {
                        continue;
                    }
                    
                    if(createdDate > latestChange) {
                        latestChange = createdDate;
                    }
                    
                    Map<String, Object> createdBy = (Map<String, Object>) record.get('CreatedBy');
                    msg += '👤 **Who:** ' + createdBy.get('Name') + ' \n\n';
                    msg += '🔧 **What:** ' + record.get('Display') + ' \n\n';
                    msg += '📁 **Section:** ' + section + ' \n\n';
                    msg += '🕐 **When:** ' + createdDateStr + ' \n\n---\n\n';
                }
                
                // Step 5: Send Teams notification
                if(String.isNotBlank(msg)) {
                    TeamsNotificationService.sendNotification(
                        '⚠️ Salesforce Code Change Alert!',
                        '🔔 *Code Change Detected!*\n\n' + msg
                    );
                }
                
                // Step 6: Save latest timestamp
                System.enqueueJob(new SaveLogQueueable(latestChange));
            }
        }
    }
}
```

---

### 3. SaveLogQueueable

```apex
public class SaveLogQueueable implements Queueable {
    private DateTime latestChange;
    
    public SaveLogQueueable(DateTime latestChange) {
        this.latestChange = latestChange;
    }
    
    public void execute(QueueableContext context) {
        Code_Check_Log__c log = new Code_Check_Log__c();
        log.Last_Check_Time__c = latestChange;
        insert log;
    }
}
```

---

## ⏰ Scheduling Jobs

### Start Monitoring (run once in Anonymous Window)

```apex
for(Integer i = 0; i < 60; i += 5) {
    String cronExp = '0 ' + i + ' * * * ?';
    System.schedule('CodeChangeDetector_' + i, cronExp, new CodeChangeDetector());
}
```

This creates **12 jobs** that fire at minutes: 0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55 — every hour.

### Stop Monitoring

```apex
List<CronTrigger> jobs = [
    SELECT Id FROM CronTrigger 
    WHERE CronJobDetail.Name LIKE 'CodeChangeDetector%'
];
for(CronTrigger job : jobs) {
    System.abortJob(job.Id);
}
```

### Test Manually

```apex
CodeChangeDetector.checkForChanges();
```

---

## ⚠️ Salesforce Limitations & Workarounds

| Limitation | Why It Happens | Workaround Used |
|-----------|---------------|-----------------|
| Scheduled Apex cannot make HTTP callouts | Salesforce Governor Limit — Schedulable interface blocks direct callouts | Used `@future(callout=true)` method as bridge |
| `@future` cannot call another `@future` | Nested @future strictly prohibited in Salesforce | Removed `sendNotificationAsync()` call — used `sendNotification()` directly since we're already inside @future |
| DML + HTTP Callout in same transaction | Salesforce rule: cannot mix DML and callouts in same context | Used Queueable class for DML — runs in separate transaction |
| Custom Metadata not updatable via DML | Custom Metadata is deployment-time only — no runtime DML | Created Custom Object `Code_Check_Log__c` instead |
| Platform Events not queryable via SOQL | Platform Events follow pub/sub model — not stored as records | Dropped Platform Event approach — used SetupAuditTrail polling |
| SetupAuditTrail Section not filterable in SOQL WHERE | Salesforce limitation on this specific field | Fetched all records via REST API and filtered in Apex code |
| `getSalesforceBaseUrl()` removed in API v58+ | Method deprecated in newer API versions | Replaced with `getOrgDomainUrl()` |
| Tooling API returning HTML instead of JSON | Wrong endpoint used — `/tooling/query/` needs different auth | Used `/query/` endpoint — SetupAuditTrail available in standard REST API |

---

## 📢 Notification Format

When a code change is detected, Teams receives:

```
⚠️ Salesforce Code Change Alert!
Someone changed code in Sandbox!

🔔 Code Change Detected!

👤 Who:     Developer Name
🔧 What:    Changed XYZ_ClassName Apex Class code
📁 Section: Apex Class
🕐 When:    2026-04-05T10:30:00.000+0000

---

👤 Who:     Another Developer
🔧 What:    Changed XYZ_TriggerName Apex Trigger code
📁 Section: Apex Trigger
🕐 When:    2026-04-05T10:28:00.000+0000
```

---

## 🔧 Maintenance Guide

### If notifications stop coming
1. Check scheduled jobs: **Setup → Scheduled Jobs** — look for `CodeChangeDetector` entries
2. If missing, re-run the scheduling code in Anonymous Window
3. Verify Remote Site Settings are active
4. Check Teams webhook URL in Custom Metadata

### If Teams webhook URL changes
1. **Setup → Custom Metadata Types → Webhook Config → Manage Records**
2. Edit **Teams Webhook** record
3. Update `Webhook_URL` field
4. Update Remote Site Setting URL domain if domain changed

### Clean up old log records (run monthly)
```apex
List<Code_Check_Log__c> oldLogs = [
    SELECT Id FROM Code_Check_Log__c 
    ORDER BY CreatedDate DESC 
    LIMIT 10000 OFFSET 10
];
delete oldLogs;
```

---

## 📝 Key Learnings

- **Salesforce Governor Limits** are strict — HTTP callouts and DML cannot happen in the same transaction
- **@future methods** bridge Scheduled Apex and HTTP callouts — but cannot be nested
- **Queueable interface** solves the @future nesting problem elegantly
- **Custom Objects** are better than Custom Metadata for runtime data storage
- **SetupAuditTrail** is a powerful tool for monitoring org changes

---

## 👤 Author

**Nitish Bisht**  
Salesforce QA Automation Engineer  
Accenture | April 2026

- Salesforce Certified Administrator
- B.Tech CSE (2019-2023)

---

> ⭐ If this helped you, give it a star!
> 
> 💡 *Built to solve a real QA problem — developers making silent code changes during testing cycles.*
