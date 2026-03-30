# WolfAttend
### Frictionless, Verified Attendance for In-Person Instruction

**Status:** Proposal submitted to NC State OIT and DELTA — active institutional feedback received, Canvas migration pivot in progress 

## About
This project was conceived, researched, and documented independently as a systems design exercise in identifying operational inefficiencies in institutional infrastructure and proposing buildable solutions using existing resources.


## The Problem

Attendance tracking at universities is broken in two different directions at once.

Faculty have largely stopped doing it — not because they don't believe it matters, but because every available tool forces a tradeoff between speed and integrity. Paper sign-in sheets take 3–7 minutes per session, are trivially falsified, and require manual gradebook entry. Roster call in a 200-person lecture is impractical. Moodle's existing honor-system check-in can be submitted from a dorm room, another state, or by a friend using shared credentials. No current option is simultaneously fast, low-effort, and trustworthy. So faculty stop, the early-warning signal disappears, and the feedback loop between showing up and academic outcomes is severed.

At the same time, in-person substitution — students sitting exams or attending class for each other — is structurally underdetected at scale. Large GEP and gateway courses regularly seat 100–150 students. Most NC State courses don't require a id at exam entry. The only link between a student and their submitted work is a name written on a sheet. A CS senior covering for a friend in a lower-level CS course, or an ECE major sitting a general engineering class for a classmate, is operating in familiar territory with near-zero detection risk. The enforcement mechanism is a professor recognizing an unfamiliar face in a large room — which isn't a detection mechanism at all.

These two problems compound each other. When attendance isn't tracked, there's no record to flag anomalies. When the tools are too painful to use, faculty stop building the habits that would catch fraud. WolfAttend was designed to close both gaps at once.

---

## The Solution

WolfAttend is a native Moodle plugin — `mod_wolfattend` — that gives every faculty member a verified, cheat-resistant attendance record in under 15 seconds of class time, with zero hardware and no change to how students interact with WolfWare.

The system works by combining two signals:

**Signal 1: A cryptographic one-time code (TOTP)**  
At session start, the instructor clicks Start Session in WolfWare. The plugin generates a short time-bound code using RFC 6238 TOTP — the same algorithm behind Google Authenticator. The code is unique to that session, expires after 10 minutes, and can never be reused. The instructor puts it on the projector. Students type it into WolfWare on whatever device they already have open. The entire student interaction is under 15 seconds.

**Signal 2: Passive RADIUS network validation**  
When a student submits the code, the backend queries NC State's existing Cisco ISE system via the pxGrid API — cross-referencing every device associated with that Unity ID that authenticated to an access point in or near the correct classroom within the valid session window. A student whose phone is on cellular data but whose laptop connected to campus WiFi in that building will still pass. If no associated device shows up in the right zone, the submission is flagged for instructor review — never automatically marked absent. The instructor always has the final word on any disputed record.

The key insight is that this data already exists. Cisco ISE logs every campus WiFi authentication event for every device on the network. It has been doing this for years for network management and security purposes. WolfAttend doesn't create new tracking infrastructure — it reads a log that already runs, for a purpose it was never applied to before.

---

## How It Works — Step by Step

| Step | Actor | Action |
|------|-------|--------|
| 1 | Instructor | Opens WolfAttend activity in WolfWare, clicks Start Session |
| 2 | System | Generates unique TOTP code — valid 10 minutes, this session only |
| 3 | Instructor | Displays code on screen — students must be in the room to see it |
| 4 | Student | Types code into WolfWare on phone or laptop — under 10 seconds |
| 5 | Backend | Validates code, then cross-references all devices tied to that Unity ID against Cisco ISE — was any of them authenticated to an AP in this building at this time? |
| 6 | System | Marks student present or flagged for instructor review — no student is ever automatically marked absent |

---

## Anti-Spoofing Coverage

The TOTP code handles the basic case. The RADIUS check is what makes it cheat-resistant.

| Scenario | Result |
|----------|--------|
| Student texts code to a friend at home | Blocked — friend not authenticated to campus WiFi on any device |
| Student texts code to a friend in a dorm | Blocked — wrong AP zone for the classroom across all their devices |
| Student submits via VPN from off-campus | Blocked — RADIUS checks physical WiFi auth, not IP address |
| Student photographs code and submits late | Blocked — TOTP code expired after 10 minutes |
| Student reuses a code from a previous session | Blocked — each session has a unique cryptographic secret |
| Student's phone is on cellular, not WiFi | Cross-referenced against all devices on that Unity ID — if their laptop or any other device hit a nearby AP, submission passes. If none did, flagged for instructor review, not marked absent. |

> **No student is ever automatically marked absent.** The RADIUS check produces two outcomes: present, or flagged for instructor review. The instructor makes the final call on every flagged record. Students don't need to know anything about how RADIUS authentication works — they type a code, and the validation runs silently in the background.

---

## System Architecture

WolfAttend operates across three layers, all within existing NC State infrastructure.

```
┌─────────────────────────────────────────────┐
│              MOODLE LAYER                   │
│  mod_wolfattend plugin                      │
│  Session management · Submission UI         │
│  Gradebook integration · Instructor dash    │
└────────────────────┬────────────────────────┘
                     │
┌────────────────────▼────────────────────────┐
│           VALIDATION LAYER                  │
│  TOTP validator · pxGrid client             │
│  Decision engine · AP zone lookup           │
└──────────┬─────────────────────┬────────────┘
           │                     │
┌──────────▼──────────┐ ┌────────▼────────────┐
│   Cisco ISE         │ │  AP-to-Room Map      │
│   RADIUS event log  │ │  AP_ID → Room/Zone   │
│   (already exists)  │ │  (OIT network data)  │
└─────────────────────┘ └─────────────────────┘
```



## What OIT and DELTA Need to Build This

WolfAttend requires coordination across three internal NC State teams. All dependencies are internal — no procurement, no external vendors, no new infrastructure.

**1 — OIT Network & Identity Services: Cisco ISE / pxGrid**

The load-bearing dependency. RADIUS validation requires read access to Cisco ISE authentication events via the pxGrid API.

- Team: OIT Network Operations / Identity Services
- Need: pxGrid subscriber credential scoped to RADIUS auth events — NetID, AP ID, timestamp only
- Pattern: One point-in-time query per student submission — not continuous, not session-tracked, not device data
- The data already exists. ISE logs every campus WiFi authentication. WolfAttend reads it; creates nothing new.
- OIT controls permission scope. WolfAttend sees only what it's granted.

**2 — OIT Network Infrastructure: AP-to-Room Mapping**

RADIUS returns an AP identifier, not a room name. WolfAttend needs a lookup table to translate that into a physical location.

- Team: OIT Network Infrastructure or Facilities Management
- Need: Mapping of AP_ID to building, room, and zone — infrastructure metadata, no student data
- Format: Quarterly-refreshed CSV or JSON export is sufficient. Live API preferred but not required for Phase 1.
- Already maintained internally for network management. WolfAttend reads it read-only.

**3 — DELTA LearnTech: WolfWare / Moodle Plugin Deployment**

WolfAttend is a standard Moodle activity module. DELTA's LearnTech team manages WolfWare and owns the plugin review and deployment process.

- Team: DELTA LearnTech — learntech@ncsu.edu
- Need: Plugin code review, staging environment deployment, production approval for mod_wolfattend
- Integrates with existing Moodle gradebook via standard Grades API — no custom gradebook changes
- Phase 1 can run on a DELTA-managed sandbox before any production deployment

---



## Privacy & FERPA

| Requirement | How WolfAttend Addresses It |
|-------------|----------------------------|
| Data minimization | Collects NetID, submission timestamp, and AP zone only |
| Data residency | All data on NC State OIT infrastructure — no third-party processors |
| Access control | Instructors see their own students only; students see their own record only |
| Retention | RADIUS query results purged within 30 days; attendance records per university policy |
| Student notice | Syllabus disclosure template provided to all participating faculty |
| IRB | Protocol ready to submit if required before expanded pilot |

---

## What This Is Not

- Not GPS or precise location tracking — location signal is AP zone, abstracted to building/room level
- Not continuous monitoring — one point-in-time RADIUS query per submission, nothing more
- Not a new student login or credential — students use their existing WolfWare session
- Not a hardware deployment — no beacons, no readers, nothing installed in classrooms
- Not an online course tool — explicitly scoped to in-person instruction only
- Not a third-party product — everything runs within NC State OIT infrastructure
- Not an automated absence system — no student is ever marked absent by the system. Unverified submissions are flagged for instructor review. A human makes every final attendance decision.

---

## Cost

The infrastructure cost is effectively zero — Cisco ISE, WolfWare, and campus WiFi are already paid for and running. WolfAttend reads data they already produce.

The real cost is developer time:

For context, SpotterEDU (the leading commercial alternative) charges approximately $3–6 per student per year plus Bluetooth beacon hardware at $150–300 per classroom. At NC State's scale that approaches $200,000–$500,000 annually. WolfAttend's ongoing cost is a rounding error by comparison — and the university owns the asset outright.

---

## Institutional Response

This proposal was submitted to DELTA LearnTech (Martin Dulberg, Director) and OIT in March 2026. The response confirmed the problem is real and being actively worked on:

> *"Secure attendance is definitely an issue... We are actively looking at more secure attendance solutions for Canvas."*

The feedback surfaced two legitimate technical considerations worth addressing in any future iteration:

**Canvas migration:** NC State is transitioning from Moodle/WolfWare to Canvas over the next two years. A Canvas-native version of this system — a Canvas LTI rather than a Moodle activity module — would align with the university's direction and would be a natural next iteration of this architecture.

**Cell data usage:** The feedback raised a valid question about students using cellular data instead of campus WiFi. The validation layer addresses this in two ways. First, WolfAttend cross-references all devices associated with a Unity ID against ISE logs — not just the device used to submit the code. A student whose phone is on LTE but whose laptop was connected to campus WiFi in the classroom will still pass validation. Second, no student is ever automatically marked absent. The RADIUS check produces one of two outcomes: present, or flagged for instructor review. The instructor makes the final call on every flagged record. This means false negatives are effectively eliminated — a legitimate student who happens to be flagged gets manually verified and marked present. Students don't need to know how the RADIUS check works, and the system is designed so that physical presence almost always produces a passing result across at least one of their devices.

Both are solvable design questions rather than fundamental objections to the approach.

---

## Comparison to Alternatives

| Solution | Hardware | Student App | Faculty Effort | Fraud Resistance | Annual Cost |
|----------|----------|------------|----------------|-----------------|-------------|
| Paper sign-in | None | None | High | None | ~$0 |
| Moodle honor check-in | None | None | Low | None | ~$0 |
| SpotterEDU | Beacons per room | Required | Low | Moderate | $200K–$500K+ |
| WolfAttend | None | None | Very low | High | ~$0 after build |

---

## Why This Belongs Inside OIT and DELTA

WolfAttend is not a vendor product looking for a buyer. It's a university infrastructure capability that happens to have been designed outside of OIT. The right long-term owner is OIT and DELTA:

- The two systems it depends on — WolfWare and Cisco ISE — are already managed by OIT
- Faculty adoption requires institutional endorsement that only Academic Technologies can provide
- pxGrid access, AP mapping data, and WolfWare production deployment all require OIT authorization regardless of who writes the code
- A plugin without a university owner is a maintenance liability at every LMS upgrade

The concept, architecture, and full technical documentation are provided as a starting point. OIT and DELTA build it, own it, and decide how far it goes. No outside IP, no licensing, no strings.

---


