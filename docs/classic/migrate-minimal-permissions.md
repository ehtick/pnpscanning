# Migrate classic SharePoint pages to modern — with the least permission possible

Once the classic pages assessment has shown you *which* pages are ready to modernize, the next step is to actually migrate them. This guide shows a **site owner** how to convert classic pages to modern pages using **PnP PowerShell**, requesting the **smallest permission that works**, and escalating only when your tenant forces you to.

> [!NOTE]
> Migration is performed by **PnP PowerShell** (the `ConvertTo-PnPPage` cmdlet), not by the Microsoft 365 Assessment tool. The Assessment tool identifies and inventories classic pages; PnP PowerShell modernizes them. This page complements the assessment by describing the least-privilege way to run that migration.

## Key facts to understand first

- **Minimum scope = `AllSites.Manage`.** This covers everything transformation writes (create/save/publish the modern page, read the classic source, swap the page name, update navigation).
- **You sign in as yourself (delegated), not as the app.** Your effective access is *the app's scope and your own site rights*. You can only migrate pages on sites where **you already have permission**.
- **`AllSites.FullControl` is needed for exactly one optional thing:** copying a page's *item-level unique (broken-inheritance) permissions* onto the new page. Everything else works on `AllSites.Manage`.

## Which scenario are you in?

```
Register a minimal AllSites.Manage app, then Connect-PnPOnline.
│
├─ First sign-in shows an "Accept" consent button
│     → SCENARIO 1  (self-service, zero admin)
│
├─ First sign-in shows "Need admin approval"
│     → SCENARIO 2  (one-time admin consent, then self-serve forever)
│
└─ You must PRESERVE per-page unique permissions
      → SCENARIO 3  (requires AllSites.FullControl + admin consent)
```

## Prerequisites

```powershell
# PnP PowerShell (v2+ / v3+)
Install-Module PnP.PowerShell -Scope CurrentUser
```

You'll need: your tenant name (`contoso.onmicrosoft.com`), the target site URL, and the classic page name.

---

## Scenario 1 — Try self-service first (`AllSites.Manage`, user consent)

Many tenants let a normal user consent to `AllSites.Manage` by default. Try this path first — if it works, **no administrator is involved at all.**

**Step 1 — Register the minimal app (once per tenant; you can do this yourself).**

```powershell
Register-PnPEntraIDAppForInteractiveLogin `
  -ApplicationName "Classic Page Migrator" `
  -Tenant contoso.onmicrosoft.com `
  -SharePointDelegatePermissions AllSites.Manage `
  -SignInAudience AzureADMyOrg
```

- This creates a **public-client** app with the single `AllSites.Manage` delegated permission and prints a line like `App Classic Page Migrator with id <client-id> created.` — **copy that client id.**
- The cmdlet then tries to consent the app in the same run. If you are **not** an admin this last step may end in a red error (`AADSTS500027`). **That's expected and harmless — the app is already registered.** (Tell: the "App … created" line prints *before* the error.)
- *(The cmdlet also generates a self-signed certificate for app-only use. You don't need it here — you sign in interactively as yourself.)*

**Step 2 — Connect and migrate.**

```powershell
Connect-PnPOnline `
  -Url https://contoso.sharepoint.com/sites/theirsite `
  -ClientId <client-id> `
  -Tenant contoso.onmicrosoft.com `
  -Interactive

ConvertTo-PnPPage -Identity MyClassicPage.aspx -Overwrite
```

- On first connect a **consent dialog** may appear.
  - **If it shows an "Accept" button → accept it. You're done — zero admin.** Every future run is prompt-free.
  - **If it says "Need admin approval" → stop and go to Scenario 2.** (Your app is already registered; you just need it consented once.)
- `-Interactive` opens your system browser (MFA-compatible). On a headless/remote box use `-DeviceLogin` instead.

> [!NOTE]
> **Run-time note:** you must have **Edit/Full Control on the target site** — a delegated token can't exceed your own rights. If you get **403**, you're not a member of that site, not short on app permission.

---

## Scenario 2 — Consent is blocked → one-time admin consent, then self-serve forever

If Scenario 1's first sign-in said **"Need admin approval,"** your tenant disables user consent for this scope. You do **not** need to re-register anything — the app from Scenario 1 Step 1 already exists. An administrator just consents it **once, tenant-wide**, and then you (and every other site owner) use it with no prompt.

**Who can consent:** a **Cloud Application Administrator** is sufficient — **Tenant Administrator or Global Administrator are not required.** (`AllSites.Manage` is a delegated SharePoint scope, not a Graph app-role, so it doesn't force the higher role.)

**Ask your admin to do one of these:**

**Option A — Entra portal (simplest).**
Entra admin center → **Enterprise applications** → *Classic Page Migrator* → **Permissions** → **Grant admin consent for \<tenant\>**.

**Option B — Admin-consent URL** (admin opens it in a browser, signs in, clicks **Accept**):

```
https://login.microsoftonline.com/contoso.onmicrosoft.com/adminconsent?client_id=<client-id>&redirect_uri=http://localhost
```

The final redirect to `http://localhost` may show a harmless `AADSTS500027` — **the consent is still recorded.**

**Option C — Azure CLI:**

```bash
az ad app permission admin-consent --id <client-id>
```

**After consent, you just re-run Scenario 1 Step 2** — this time with **no prompt**:

```powershell
Connect-PnPOnline -Url https://contoso.sharepoint.com/sites/theirsite -ClientId <client-id> -Tenant contoso.onmicrosoft.com -Interactive
ConvertTo-PnPPage -Identity MyClassicPage.aspx -Overwrite
```

---

## Scenario 3 — You need to preserve per-page unique permissions → `AllSites.FullControl`

`ConvertTo-PnPPage` **copies a page's item-level unique (broken-inheritance) permissions to the new modern page by default.** That copy step needs the **ManagePermissions** right, which is only available through **`AllSites.FullControl`**.

**What happens with only `AllSites.Manage`:** the migration still **succeeds** — the permission-copy step is **skipped and logged** (`You don't have the needed permission level (ManagePermissions)…`). The modern page simply inherits permissions from its library instead of carrying the source page's unique ACL. For most pages this is fine; you only need FullControl if a specific page relies on unique permissions you must retain. (You can also explicitly opt out of the copy with `-SkipItemLevelPermissionCopyToClientSidePage`.)

**To actually preserve unique permissions**, register the app with `AllSites.FullControl` instead:

```powershell
Register-PnPEntraIDAppForInteractiveLogin `
  -ApplicationName "Classic Page Migrator (FullControl)" `
  -Tenant contoso.onmicrosoft.com `
  -SharePointDelegatePermissions AllSites.FullControl `
  -SignInAudience AzureADMyOrg
```

**`AllSites.FullControl` always requires an administrator to consent — a site owner can never self-consent it.** Ask a **Cloud Application Administrator** (Global Admin still not required) to consent it, exactly as in **Scenario 2**. After that:

```powershell
Connect-PnPOnline -Url https://contoso.sharepoint.com/sites/theirsite -ClientId <client-id> -Tenant contoso.onmicrosoft.com -Interactive
ConvertTo-PnPPage -Identity MyClassicPage.aspx -Overwrite   # item-level permissions now copied
```

---

## Permission reference

| Scope | Delegated permission id | Consent | Covers |
|---|---|---|---|
| `AllSites.Manage` | `b3f70a70-8a4b-4f95-9573-d71c496a53f4` | **User-consentable on many tenants** (else Cloud App Admin, once) | Full page migration: create/save/publish, read classic source, name swap, nav |
| `AllSites.FullControl` | `56680e0d-d2a3-4ae1-80d8-3c4f2100e3d0` | **Always admin** (Cloud App Admin, not Global Admin) | Everything above **+** copying item-level unique permissions |

SharePoint Online resource app id: `00000003-0000-0ff1-ce00-000000000000`.

## Good to know

- **Nothing secret is distributed.** Only the non-secret **client id** is shared. Each person signs in as themselves; access is bounded to their own sites.
- **One registration serves everyone.** Register (and, if needed, admin-consent) the app **once per tenant**; all site owners reuse the same client id.
- **`-Interactive` vs `-DeviceLogin`:** use `-Interactive` at a desktop; `-DeviceLogin` on a headless/remote host. Both support MFA.
