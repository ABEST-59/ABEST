# 🔄 Google Cloud Directory Sync (GCDS) Configuration

![GCDS Version](https://img.shields.io/badge/GCDS-v5.0.50-blue.svg)
![Status](https://img.shields.io/badge/Status-Validated-success.svg)
![Protocol](https://img.shields.io/badge/Protocol-LDAPS%20%2F%20Port%20636-informational.svg)
![Auth](https://img.shields.io/badge/Auth-OAuth%202.0-green.svg)

Technical documentation for the synchronization architecture between **Active Directory (LDAP)** and **Google Workspace**.

---

## 📌 Technical Overview

| Scope | Details |
| :--- | :--- |
| **Primary Google Domain** | `abest.ovh` |
| **LDAP Server** | `pldc1.ldap.abest.ovh` |
| **Connection Type** | LDAPS (Port `636`) |
| **Auth Type** | Simple (`ABEST\gcds`) |
| **Base DN** | `OU=ABEST,DC=ldap,DC=abest,DC=ovh` |

---

## ⚙️ Configuration Details

### 1. Google Domain Configuration
* **Primary Domain**: `abest.ovh`
* **Read Timeout**: `20` seconds
* **Authorization Status**: Authorized via OAuth 2.0

### 2. General Sync Scope
* [x] **Organizational Units**
* [x] **User Accounts**
* [x] **Groups**

---

### 3. Mappings & Search Rules

#### 🏢 Organizational Units (OUs)
* **OU Mapping**: `OU=ABEST,DC=ldap,DC=abest,DC=ovh` ➔ `ABEST`
* **Filter**: `(objectClass=organizationalUnit)`
* **Scope**: `SUBTREE`

#### 👤 User Accounts
* **Primary Email Attribute**: `userPrincipalName`
* **Unique Identifier**: `objectGUID`
* **Aliases Attribute**: `proxyAddresses`
* **Name Attributes**: `givenName` (First), `sn` (Last), `displayName` (Full Name)

> **User Search Rule (Active Users Only):**
> ```ldap
> (&(objectCategory=person)(objectClass=user)(userPrincipalName=*)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
> ```

> 💡 **User Lifecycle Policy:**
> - Users removed from LDAP are **suspended** in Google (not deleted).
> - Google Workspace Super Admins are **protected** from deletion/suspension.

#### 👥 Groups
* **Standard Groups Attribute**: `mail`
* **Dynamic Groups Attribute**: `msExchDynamicDistributionList`
* **Member / Owner Attributes**: `member` / `managedBy`
* **Manager Role Policy**: Maintain managers (do not remove on sync)

---

## 🚦 Sync Validation Status

| Module | Status | Notes |
| :--- | :---: | :--- |
| **Google Domain Config** | ✅ | Connected via OAuth 2.0 |
| **LDAP Configuration** | ✅ | Connection successful via SSL |
| **General / OUs / Users / Groups** | ✅ | All rules validated |
| **Notifications** | ⚠️ | SMTP server setup required for email reports |

---

## 🛠️ Operating Instructions

### Manual Execution (UI)
1. Open **Google Cloud Directory Sync** (`v5.0.50`).
2. Load the XML configuration file.
3. Navigate to **Sync** tab.
4. Click **Simulate sync** and verify the change log.
5. Click **Sync & apply changes** to execute production synchronization.
6. Save AS en fichier .xml (PLDC1_Config.xml et PLDC2_Config.xml)

### Scheduled Task Execution (CLI)
To run GCDS in headless / batch mode using Windows Task Scheduler:

```cmd
"C:\Program Files\Google Cloud Directory Sync\sync-cmd.exe" -apply -c "C:\path\to\config.xml"
"FAIRE AVEC AWX"
```
