# 🏸 Room-First UX Refactor
## Badminton Club App — Architecture & Implementation Guide

---

## 1. Root Cause Analysis

### Why the Current Flow Fails

The existing system suffers from **Auth-first architecture** — every user, regardless of intent, is forced through a Google authentication funnel before they can see or do anything. This creates:

| Problem | Impact |
|---|---|
| Google popup before entering room | ~3–8s friction, popup blockers, mobile redirect loop |
| `onAuthStateChanged` → Firebase lookup (8s timeout) | Loading spinner every cold load |
| `users/{uid}` read on every session | 1 Firebase read per user per session |
| Multiple session states (admin/member/guest) | Session bugs, claim modal surprises |
| Complex claim/link modal post-login | Abandonment after Google auth |

**The mental model mismatch:** Users think "I want to join a badminton session." The app thinks "I need to know who you are before I can do anything."

---

## 2. New Architecture: Room-First

### Core Principle
> **Enter room → identify yourself (optional) → play.**
> Authentication is a feature, not a prerequisite.

### Identity Model

```
                    ┌─────────────────────────────────────────┐
                    │           User arrives at app            │
                    └─────────────────┬───────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────┐
                    │    Has valid localStorage session?       │
                    └──────┬─────────────────┬────────────────┘
                          YES               NO
                           │                 │
              ┌────────────▼──┐    ┌─────────▼──────────────┐
              │  Auto-enter   │    │   Home Screen           │
              │  (instant)    │    │   Room Code Input       │
              └───────────────┘    └─────────┬───────────────┘
                                             │
                                   ┌─────────▼──────────┐
                                   │  Room exists?       │
                                   └──────┬──────────────┘
                                          │ YES
                                ┌─────────▼──────────────┐
                                │  Members list empty?   │
                                └──┬─────────────────────┘
                                  YES          NO
                                   │            │
                            ┌──────▼──┐  ┌──────▼──────────┐
                            │Nick     │  │"Who are you?"   │
                            │Input    │  │Claim Grid +     │
                            │(new     │  │Guest option     │
                            │ player) │  └─────────────────┘
                            └──────┬──┘
                                   │
                            ┌──────▼──────────────────────┐
                            │  Firebase Anonymous auth    │
                            │  (invisible, no popup)      │
                            │  + localStorage session     │
                            └──────────────────────────────┘
```

### Auth Levels

| Role | Auth Required | What They Can Do |
|---|---|---|
| **Member** | None (Anonymous Firebase) | View rankings, RSVP, see scores |
| **Claimed Member** | None (session + claim) | Edit their own profile/icon |
| **Admin** | Google Sign-in | Everything — record matches, manage members, rules |

---

## 3. Data Structure Changes

### localStorage (per device)
```javascript
// Replaces the old bcm_* session system
bcm_last_room_code: "4831"
bcm_last_club_name: "SnowiteFC"
bcm_sessions: {
  "4831": {
    sessionId: "sid_1716123456_abc12",
    nick: "Wasarun",
    memberId: 1716000001,   // null if guest
    ts: 1716123456789       // expires after 7 days
  }
}
bcm_theme: "snowite"        // kept as before
```

### Firebase (unchanged structure — backward compatible)
```
rooms/{roomId}
  ├── createdAt
  ├── adminUid
  ├── adminName
  ├── clubName
  └── state/
      ├── members[]
      │    ├── id, name, elo, wins, losses...
      │    └── sessionId?   ← NEW: anonymous session binding
      ├── matches[]
      ├── finance[]
      ├── decayEvents[]
      └── lastUpdatedAt

users/{googleUid}           ← Only admin Google users
  ├── roomCode
  ├── role: "admin"
  └── joinedAt

adminRooms/{googleUid}: roomCode   ← O(1) admin lookup
```

### Member Document: sessionId field
```javascript
// BEFORE (Google UID binding)
{ id: 1716000001, name: "Wasarun", googleUid: "abc123...", elo: 10000 }

// AFTER (anonymous session binding)
{ id: 1716000001, name: "Wasarun", sessionId: "sid_1716_abc12", elo: 10000 }
// sessionId is ephemeral — resets when member uses a new device
// Historical match data is preserved via member id (never changes)
```

**Migration:** Existing `googleUid` fields are kept. New members use `sessionId`. The `getMyMemberId()` function checks both.

---

## 4. UX Flows

### 4.1 Member Join Flow (NEW — ultra-fast)

```
[Home Screen]
    ┌──────────────────────────┐
    │  🏸 SnowiteFC            │
    │                          │
    │  🕐 SnowiteFC · 4831    │  ← Recent room chip (tap to rejoin)
    │        แตะเพื่อเข้า →   │
    │                          │
    │  ┌─────────────────────┐ │
    │  │    4 8 3 1           │ │  ← OTP-style, numeric keyboard
    │  └─────────────────────┘ │
    │  [ 🏸 เข้าร่วม        ] │
    │  ─────── หรือ ─────────  │
    │  [ ⚙️ สร้างก๊วนใหม่  ] │  ← Admin only
    └──────────────────────────┘

↓ Enter valid code

[Who Are You? — if members exist]
    ┌──────────────────────────┐
    │  ห้อง 4831 · SnowiteFC   │
    │                          │
    │  คุณคือใคร?              │
    │                          │
    │  [🦁 Wasarun] [🐯 Min ] │
    │  [🦊 Arm    ] [👤 Guest] │
    │                          │
    │  [ ✅ เข้าร่วม         ] │
    └──────────────────────────┘

↓ One tap → inside app (< 2 seconds total)
```

### 4.2 Returning Member Flow (INSTANT)

```
[App load]
    └── localStorage session found?
        YES → Auto-enter room → start realtime sync
             → No welcome screen shown at all
             → Target: < 500ms to content
```

### 4.3 Deep Link Flow

```
URL: swfc.app/4831
    └── Extract "4831" from pathname
    └── Pre-fill room code input
    └── Show home screen with code ready
    └── User just taps "เข้าร่วม"
```

### 4.4 Admin Create Flow (Google auth required, acceptable)

```
[Home] → [สร้างก๊วนใหม่]
    └── Google Sign-in popup
    └── Enter club name
    └── See generated room code (large, prominent)
    └── Share code → members join instantly
```

### 4.5 Leave Room Flow (NO confirmation modal)

```
Profile Menu → "🚪 ออกจากห้อง"
    └── Clear localStorage session
    └── Firebase anonymous signOut (silent)
    └── Toast: "ออกจากห้องแล้ว — Undo" (3.8s)
    └── Reload → home screen
    
UNDO: restores session before reload
```

---

## 5. Session State Machine

```
States: UNKNOWN → HOME → IN_ROOM

UNKNOWN  (app load)
  → has localStorage session → IN_ROOM (instant)
  → no session, Google admin user → IN_ROOM (admin restore)
  → no session → HOME

HOME
  → enter valid room code + claim/nick → IN_ROOM
  → Google sign-in (admin) → create room → IN_ROOM

IN_ROOM
  → leave room → HOME
  → 7 days idle → HOME (session expired)
```

**Eliminated states from old system:**
- ~~`ws-step-loading`~~ (hidden by pre-restore)
- ~~`ws-step-choose`~~ (join vs create — now two separate entry points)
- ~~`ws-step-join`~~ (merged into home)
- ~~Claim modal (post-login surprise)~~ → now upfront in join flow
- ~~Guest mode~~ → every member is lightweight anonymous

---

## 6. Implementation Guide

### Step 1: Replace CSS block

**Find in `index.html`:**
```css
/* ===== WELCOME SCREEN ===== */
#welcome-screen{
  position:fixed;inset:0;z-index:9999;
```
**Replace everything** from that comment through `/* My member card highlight */` with the CSS from `room_first_ux_module.html` Section 1.

### Step 2: Replace Welcome Screen HTML

**Find:**
```html
<!-- ===== WELCOME SCREEN ===== -->
<div id="welcome-screen">
```
**Replace everything** from that comment through the closing `</div>` of the welcome screen (line ~625) with the HTML from Section 2.

### Step 3: Replace Welcome JS functions

**Find and remove** these functions (keep their surrounding structure):
- `_wFillLastRoom()` → replaced by `_rfShowRecentChip()`
- `_wShowLastRoomHint()` → stub provided
- `wGoStep()` → replaced (same name, different implementation)
- `wSelectRole()` → removed
- `wContinueWithGoogle()` → stub redirects to admin flow
- `wGoogleSignIn()` → replaced by `wAdminGoogleSignIn()`
- `wJoinNext()` → replaced by `wEnterRoom()`
- `wStartAsUser()` → replaced by `_rfFinishJoin()`
- `wStartAsAdmin()` → replaced by `wCreateClub()`
- `wFinishCreate()` → replaced by `wEnterAsAdmin()`
- `_wSetSignedIn()` → removed
- `_renderUserChip()` → removed
- `wLogout()` → replaced by `profileLeaveRoom()`

**Add** all the new JS from Sections 3 & 6 of the module.

### Step 4: Replace onAuthStateChanged callback

**Find:**
```javascript
_auth.onAuthStateChanged(async function(user) {
```
**Replace** the entire callback body with the new logic from Section 4 (the `_rfPatchAuthObserver` pattern).

### Step 5: Update Profile Menu HTML

**Find** in the profile menu dropdown (around line 705):
```javascript
onclick="profileLogout()"
```
**Replace with:**
```javascript
onclick="profileLeaveRoom()"
```
Change button text from `🚪 ออกจากระบบ` to `🚪 ออกจากห้อง`.

### Step 6: Update getMyMemberId

**Find:**
```javascript
function getMyMemberId() {
  const uid = _sGet('bcm_myUid') || '';
  if (!uid) return null;
  const m = S.members.find(m => m.googleUid === uid);
  return m ? m.id : null;
}
```
**Replace with** the updated version from Section 6 of the module (checks `sessionId`).

### Step 7: Remove obsolete welcome screen HTML elements

Remove these IDs that no longer exist:
- `#ws-step-loading` (replaced by `#ws-loading`)
- `#ws-step-main` (replaced by `#ws-home`)
- `#ws-step-choose` (removed)
- `#ws-step-join` (removed)
- `#ws-step-created` (replaced by `#ws-created`)
- `#w-room-input` (renamed to `#w-code-input`)
- `#w-google-btn` (removed — Google btn now only in admin flow)
- `#join-code-input`, `#join-next-btn` (removed)
- `#w-last-room-hint` etc. (replaced by `#w-recent-chip`)

---

## 7. Backward Compatibility

### What's preserved
- All ELO/ranking/match/finance/trophy logic — **zero changes**
- Firebase data structure — **100% backward compatible**
- Admin workflow (Google auth, create room, manage members)
- `isAdmin()`, `isMember()`, `isGuest()` functions
- `_sGet()`, `_sSet()`, `_sClear()`, `_session` object
- `_wSaveLastRoom()` — same localStorage keys
- `toast()`, `openModal()`, `closeModal()`, `tab()` — unchanged
- All existing member data (name, elo, wins, losses, etc.)

### What changes
- `googleUid` → `sessionId` for member identity
  - Migration: `getMyMemberId()` checks both fields
  - Old members with `googleUid` still work
- `profileLogout()` → `profileLeaveRoom()` (same behavior, better UX)
- Welcome screen step IDs renamed (internal only)

---

## 8. Performance Targets

| Metric | Before | After |
|---|---|---|
| Cold load → content visible | ~6–10s (auth timeout) | < 500ms (localStorage restore) |
| First-time join | ~5s (Google popup flow) | < 3s (code → claim → tap) |
| Firebase reads on entry | 2–3 (users + rooms) | 1 (rooms only, for verification) |
| Session states | 7 (loading/choose/join/created/etc.) | 3 (loading/home/in-room) |
| Google auth required | Always | Admin only |
| Realtime listeners | Starts after full auth | Starts immediately after room entry |

---

## 9. Scalability (1000 rooms / 100 concurrent)

### Firebase read patterns (after refactor)

| Operation | Firebase reads | Notes |
|---|---|---|
| Member join | 1 read (`rooms/{code}`) | Verify + load state |
| Admin return | 2 reads (`users/{uid}` + `rooms/{code}`) | Once per session |
| Realtime sync | 0 additional | Already on listener |
| Admin room lookup | O(1) via `adminRooms/{uid}` | No scan needed |

### Presence (lightweight)

For online indicators, use Firebase's built-in `.info/connected` + RTD presence pattern:
```
rooms/{roomId}/presence/{sessionId}: { nick, joinedAt, online: true }
```
With `onDisconnect().remove()` — auto-expires on tab close.

**Not implemented in this module** (add as a separate feature when needed).

---

## 10. Testing Checklist

After implementing:

- [ ] Fresh visit → Home screen appears immediately (< 300ms)
- [ ] Enter valid 4-digit code → room loads, claim grid shows
- [ ] Tap member name → enters room as that member
- [ ] Tap Guest → nick input appears
- [ ] Enter nick → inside app, rankings visible
- [ ] Reload → auto-enters same room (no welcome screen)
- [ ] Tap recent room chip → prefills code, one more tap to enter
- [ ] deep link `app.com/4831` → code pre-filled
- [ ] Profile → Leave Room → toast with Undo → home screen
- [ ] Undo within 3.8s → stays in room
- [ ] Admin flow → Google → create club → room code shown
- [ ] Admin reload → bypasses welcome screen instantly
- [ ] Old existing members (googleUid) still work for admin
- [ ] Match recording, ELO, finance — all unchanged

---

*Developed against codebase snapshot: 5,259 lines, Firebase RTDB, Kanit font, multi-theme.*
*Root cause addressed: Auth-first → Room-first architecture.*
*All ELO/game logic preserved unchanged.*
