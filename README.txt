MEDQUORUM - emergency patient records that no single party controls
RMIT Hackathon 2026 | Challenge 2: "Build without blind trust" | TideCloak
=====================================================================

WHAT IT IS
Paramedics record a patient on scene. The record (details, address, video) is
encrypted on the device and sealed. After that, no administrator, no hospital
executive and no AI system can open, change or delete it alone. Access needs a
quorum of independent, TideCloak-authenticated people, and every attempt is
written to a tamper-evident (hash-chained) audit log.

The three parties from the challenge and how each is constrained:
  Administrator : cannot read records, cannot grant himself a role (TideCloak
                  QEA: a role change needs 2 of 3 admins).
  Organisation  : cannot delete or edit a record alone (needs compliance
                  officer + patient; edits are detected by the integrity seal).
  AI system     : advisory only, a doctor must confirm; has no identity and no
                  key, so bulk reads are denied and logged.

CONTENTS OF THIS PACKAGE
  README.txt              this file
  summary.doc             problem, solution, value, role of TideCloak
  docs/SYSTEM_SPEC.md     full system specification (scenarios S1-S13, API)
  setup-tidecloak.ps1     Docker + TideCloak + full Tide realm (needs Tide licence)
  setup-tideless.ps1      Docker + TideCloak realm WITHOUT Tide licence (fallback)
  demo-qea.ps1            terminal demo of the admin-cannot-self-grant scenario
  app/                    the Next.js 16 app using the TideCloak SDK
  app/init/               realm template and provisioning scripts
  app/proxy.ts            SDK createTideCloakProxy page protection
  app/lib/server/         token verification, quorum policy, audit chain, key escrow

MOCKED / SIMULATED FUNCTIONALITY (declared as required)
  REAL      TideCloak login and token verification, realm roles, QEA quorum on
            role changes (running against a local TideCloak Docker container).
  REAL*     Tide E2EE patient vault (doEncrypt/doDecrypt). *Only in a
            Tide-licensed realm. Not available in Tideless fallback mode.
  APP-LAYER Multi-party access quorum, integrity seals, hash-chained audit log,
            crypto-shred deletion. Real code, running on TideCloak-verified
            identities.
  SIMULATED Key Release Service (case-key escrow; production would use Tide
            threshold encryption + Forseti policy), AI triage (canned output,
            organisation-owned model, no external API), route map and ETA,
            dispatch calls, threat-simulator attacks.
  Note: during the event Tide's licensing service returned HTTP 500
  (TIDE-IDPEXT-VENDOR-KEYGEN_FAILED), so the demo may run in "Tideless" mode.
  The app shows a banner saying which mode is active.

REQUIREMENTS
  Windows 10/11, internet access, Docker Desktop (engine running), Node.js LTS
  (v22 recommended). Port 8080 (TideCloak) and 3000 (app) must be free.

RUN INSTRUCTIONS
  1. Open PowerShell in the project folder.

  2. Start TideCloak and create the realm. Choose ONE:

     a) No Tide licence needed (recommended if the licence step fails):
          powershell -ExecutionPolicy Bypass -File .\setup-tideless.ps1
        Creates realm "medquorum" with QEA governance (2 of 3 admins) and 7
        demo users. Password for all demo users: Medquorum1!

     b) Full Tide realm (needs Tide licence service to be working):
          powershell -ExecutionPolicy Bypass -File .\setup-tidecloak.ps1
        Follow the printed "ACTION REQUIRED" URL to create the admin's Tide
        account, then create the other users in the Tide console
        (see README.md, section 3).

  3. Install and start the app:
          cd app
          npm install
          npm run dev
     Open http://localhost:3000

  If npm fails with UNABLE_TO_VERIFY_LEAF_SIGNATURE (campus proxy):
          $env:NODE_OPTIONS="--use-system-ca"
          npm install
     or use a phone hotspot.

  UI-only preview without TideCloak (NO trust guarantees, red banner):
          cd app
          npm run dev:mock

  Reset demo data:   cd app ; npm run reset-data

DEMO ACCOUNTS (Tideless mode, password Medquorum1!)
  admin        it_admin             IT administrator
  director     hospital_director    Organisation
  compliance   compliance_officer   independent oversight
  paramedic    paramedic            captures the record
  drlee        doctor               treating doctor
  patient1     patient              data owner
  nok1         next_of_kin          proxy when patient is unconscious

2-MINUTE DEMO FLOW
  1. paramedic : Dispatch -> route -> record -> local AI triage ->
                 "Encrypt, seal & upload" (ciphertext only leaves the device).
  2. admin     : System console -> "Open record as admin" -> denied (403);
                 raw database shows ciphertext only.
  3. admin     : Governance page -> ask TideCloak to grant yourself "doctor" ->
                 202, parked. Your own approval is 1 of 2, no role yet.
     director  : approves -> TideCloak commits the change.
  4. drlee     : open case -> denied -> request next-of-kin proxy.
     nok1      : Approvals -> Approve.
  5. drlee     : opens record (decrypts in the browser), confirms the AI
                 suggestion (human decides).
  6. director  : proposes deletion -> parked, needs compliance + patient.
  7. compliance: Threat simulator: AI bulk read denied, tampered record flagged,
                 audit chain verified.
  8. patient1  : "Who touched my records" and the Tide-encrypted vault.
  The in-app list of all scenarios: http://localhost:3000/scenarios

TIDECLOAK SDK USAGE (@tidecloak/nextjs)
  TideCloakProvider, useTideCloak        app/lib/client/auth.tsx
  HasRealmRole                           app/components/Shell.tsx
  createTideCloakProxy                   app/proxy.ts
  verifyTideCloakToken                   app/lib/server/auth.ts (every API call)
  doEncrypt / doDecrypt                  app/app/patient/page.tsx
  TideCloak admin REST (QEA)             app/lib/server/kcadmin.ts, /governance

LOCAL DEV CREDENTIALS
  The TideCloak master admin (admin / password) and the demo password
  (Medquorum1!) are for the local development container only. No secrets or
  real patient data are in this repository. All patient data is fictional.
