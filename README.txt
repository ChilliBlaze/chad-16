SPHERICAL BATTLEFIELD — MULTIPLAYER

Run locally:
  node server.js
Then open http://localhost:8080

Controls:
  WASD       move
  Mouse      look (pointer lock)
  Left click attack / shoot
  Right click aim
  E          weapon wheel
  F          first/third person
  G          emote wheel
  Space      jump
  Q          crouch
  Shift      sprint
  Esc        release pointer lock

Multiplayer additions:
- Health, names and kill-streak leaderboard sync through the server.
- Rifle, minigun, sniper, knife, bat and lightsaber damage other players.
- Grenade and air strike splash damage can also hurt the caller.
- Emotes sync so other players can see them.

Performance changes:
- High-performance WebGL preference.
- Render pixel ratio capped at 1.35.
- Shadows disabled and expensive world geometry/lights reduced.

Render hosting:
Keep server.js and game.html in the same repository. Render start command: node server.js

MOVEMENT CATEGORY
-----------------
Open the weapon wheel with E and choose Movement. Movement equipment is secondary and does not replace your normal weapon.
- Grapple Hook: press R to attach/release. It can latch onto the spherical ground, tree trunks, fence posts and torches.
- Dash: press R for a fast burst; the equipped item is a glowing green baton.
- Jump Boots: Space becomes a super jump; a glowing upward-arrow device is visible in first person.
- Teleport Beacon: press R to throw the white beacon. After it lands, your next left-click attack teleports you to it instead of attacking.


ACCOUNT SAVING
--------------
Accounts are stored in accounts.json by default. Passwords are not stored directly:
the server stores a PBKDF2 password hash and random salt.

For durable accounts on Render, attach a Persistent Disk and set:
ACCOUNT_FILE=/var/data/accounts.json
(or point ACCOUNT_FILE at the path of your mounted Render disk).
Without persistent storage, Render may reset local account data after a redeploy/restart.

BATTLE ROYALE
-------------
Battle Royale is a separate 12-player queue from Quick Play. The queue starts when
12 players arrive, or after a short countdown once at least 2 players are waiting.


LATEST FIXES
------------
- Spawn protection is authoritative on the server and bullets visibly collide with the forcefield.
- The global leaderboard ranks ALL-TIME TROPHIES WON, not the current spendable balance.
- Battle Royale now has one large central planet plus 12 physically separate starting planets.
  Each of the 12 players starts on a different small planet.
- Battle Royale movement uses the nearest planet as its gravity body, so players can stand/run
  around the small planets and grapple across space toward other planets/the central planet.
- Equipped tops, hair and trousers are applied after team colouring and use independent materials
  per network player. Equipped top colour is also used on the first-person sleeve.


ACCOUNT LOGIN REBUILD
---------------------
The account handshake is now separate from lobby/game requests.
Creating an account:
1. Waits for the WebSocket server acknowledgement.
2. Sends one dedicated auth request.
3. Clears the auth retry as soon as authOk/authError is received.
4. Shows a visible error instead of silently hanging.
5. Verifies the account database is writable before confirming a new account.
6. If a configured ACCOUNT_FILE path is unavailable, the server falls back to ./accounts.json.

For permanent Render storage, a Persistent Disk is still recommended. Set ACCOUNT_FILE
to the mounted disk path (for example /var/data/accounts.json). Without a persistent
disk, accounts created in fallback/local storage can disappear on a Render redeploy.


ACCOUNT SYSTEM REBUILT + VERIFIED
---------------------------------
The login/create-account flow no longer shares the lobby retry queue.
It has its own request, timeout, retry, success and error handling.

Verified:
- fresh account creation
- database write to disk
- login after restarting the Node server
- incorrect-password rejection
- password is stored only as a PBKDF2 hash + salt
- invalid/unavailable ACCOUNT_FILE falls back to local accounts.json
- successful auth clears all auth retry timers

For permanent Render accounts, mount a Persistent Disk and point ACCOUNT_FILE
to that mounted path. If it is configured incorrectly, the server will now
fall back locally rather than leaving the login screen hanging.


ACCOUNT BUTTON V4
-----------------
The login/create button now has an independent bootstrap click handler which runs
before the 3D module. A click must immediately show CLICK RECEIVED and then a
specific status/error. It can no longer silently do nothing.

The client also requires server protocol account-v4. If an older server.js is
still deployed, the login screen explicitly says OLD SERVER DETECTED.

Diagnostic endpoint:
  /health
returns JSON including build=account-v4 and the account database filename.


ACCOUNT V5 CRASH FIX
--------------------
Fixed the startup crash:
  ReferenceError: Cannot access 'myAccount' before initialization

Cause:
The first-person hand builder read myAccount before the module-level account state
had been initialized. The myAccount declaration now occurs at module scope before
buildEmptyHand is defined or can run.

The login bootstrap remains independent and V5 requires server protocol account-v5.


PROGRESSION V6
--------------
Battle Pass:
- 1,000 steps.
- 50 XP per step.
- Winning Hotzone, Capture the Flag or Battle Royale grants a random 30-60 XP.
- New accounts begin with only Pistol, Fists and Grappling Hook.
- Weapons/utilities unlock throughout the pass; locked wheel entries are greyed out.
- Weapon skins and special effects unlock after progression milestones.

Inventory:
- Character/clothing preview + owned clothing equip controls.
- Weapon skin inventory.
- Effects inventory: bullet trails, explosion colours, forcefield colours, hand colours,
  sprint trails, jump effects and a high-tier spawn prism effect.

Gameplay:
- Server rejects damage/fire attempts from locked guns/melee weapons.
- Fists have an immediate equip-state update, larger hit reach and a follow-up hit check.
- Dash is faster and lasts longer.
- Battle Royale starting planets orbit farther from the central planet.


V6.4 FULL SYMBOL AUDIT
----------------------
Root cause found: jump() was missing its final closing brace. This accidentally placed
hundreds of lines (weapon system, health UI, account UI, multiplayer, Battle Pass,
inventory, avatar state, etc.) inside jump(), causing cascading ReferenceErrors.

The missing brace has been restored. The complete module was then checked with the
TypeScript JavaScript checker for:
- undefined identifiers (TS2304)
- use-before-declaration / temporal-dead-zone errors (TS2448 / TS2454 / TS2449)
- block-scoped redeclaration errors (TS2451)

All of those checks return zero errors in V6.4.


V6.4 FULL SYMBOL AUDIT - ROOT CAUSE FIXED
-----------------------------------------
The repeated ReferenceErrors were caused primarily by one missing closing brace in jump().
That accidentally nested hundreds of lines of otherwise-valid declarations inside jump(),
so they existed in the file but were out of scope during normal startup.

Fixes:
- Restored the missing closing brace at the real end of jump().
- Removed the old stray brace hundreds of lines later.
- Restored buildRemoteGun() from the earlier working multiplayer build.
- Moved the sprint-trail effect into the main game loop where sprint, mag, dt and fr
  are actually defined.

The complete client module was then checked with TypeScript's JavaScript checker:
- Undefined identifiers (TS2304): 0
- Use-before-declaration / TDZ (TS2448/TS2454/TS2449): 0
- Block-scoped redeclarations (TS2451): 0

Node syntax check: PASS.


PROGRESSION V6.6
----------------
- Battle Pass reduced to 100 steps and displayed as a horizontal scrolling timeline.
- The final weapon unlock is Nuke at step 100.
- Client and server use matching 1-100 weapon/effect/skin progression tables.
- Early fixed rewards include Icy Bullet Trail, Crimson Sprint Trail and Crimson Forcefield.
- Special-effect inventory cards now show category icons.
- Character inventory preview more closely resembles the in-game avatar (uneven eyes,
  overalls/bib, arms, hands, boots, equipped hair/top/trousers).
- Global leaderboard responses are ignored unless the user explicitly requested it.
  Closing the panel cancels the request, clears the panel, and prevents a late response
  from reopening it.


PROGRESSION V6.7 VISUAL + XP POLISH
-----------------------------------
- Rifle unlock moved to Battle Pass step 2.
- Match wins award 50-100 base XP.
- Consecutive wins add +15 XP per extra win, capped at +100 streak bonus.
- Kills award +10 XP and show a dedicated elimination animation.
- Bullet trails are thicker/longer and use explicit names such as ICY BULLET TRAIL.
- Pistol fire delay reduced from 430 ms to 260 ms.
- Network state send interval changed from 70 ms to 85 ms and particle/debris counts are capped.
- Login/play/lobby/profile screens and selection buttons now use embedded battlefield/world artwork.
- Waiting lobbies show mode rules and controls.


PROGRESSION V6.8 CINEMATIC MENUS
--------------------------------
- Bundles battlefield_loading.png, generated from the actual spherical battlefield look.
- Uses that artwork across the login/play screen, lobby/waiting screens and profile panels.
- Battle Pass now opens through a dedicated openBattlePass() path.
- Battle Pass has both a direct button listener and a delegated fallback listener.
- Profile panel is forced above lobby/menu layers with z-index 1000.
- renderBattlePass() explicitly forces the panel visible and scrolls to the current pass step.
