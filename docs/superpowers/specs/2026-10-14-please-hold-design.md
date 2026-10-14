# Please Hold — Design Spec

Date: 2026-10-14
Working title: **Please Hold**
Engine: Godot 4 (desktop, Mac first), GDScript
Audience: personal project, built to a finished standard (menu, settings, saves)
Sibling project: `~/JudgeGame` (The Bench). Reuse its look and building blocks by copying, not linking.

## 1. Concept

You are a scam-baiter with a channel, working from a dim basement lab. Scammers call you.
You play a fake victim using a persona sheet, stall them, and dig out evidence, while a
trust meter decides how long they stay on the line. Papers, Please structure: a desk full of
paper, a rule that changes each day, small decisions with knock-on consequences.
Tone is pure goofy deadpan. Scammers are cartoon incompetents and nobody is really hurt.

## 2. Core loop (one call)

- The scammer speaks. You pick one of 3-4 lines. Nothing is typed, no live AI.
- Line types:
  - **Dumb:** stalls; adds call minutes; small trust gain.
  - **Cooperate:** advances the scam script; raises trust; may expose a tell.
  - **Probe:** fishes for evidence (real name, wallet, location); costs trust.
  - **Persona answer:** the scammer asks a question your sheet answers; the lines are
    different values; the one matching the sheet is correct, others cost trust.
  - **Leak:** one option reveals your real name or street. Sets an exposure flag.
- **Trust (0-100, starts 50).** At 0 they hang up. Call minutes kept on the line is the score.
- **Scammer mood:** patient, irritated, threatening. The same line shifts trust differently
  per mood.
- **Tells** appear in captions: call-center chatter, a 3am timezone slip, a mispronounced
  company name, a reused script line. Clicking a tell logs it on the notepad. Each tell is
  worth evidence only if logged.
- **Supervisor transfer:** at set points a call may hand to a tougher second scammer with
  different tells and a stricter trust rule.
- **Hold:** the player may put the scammer on hold for a few seconds of room tone, which
  reveals background-chatter tells and costs a small amount of trust.
- A call ends by hang-up (trust 0), the scammer's script completing, or the player ending it.

## 3. Day flow (first build: one day, three calls)

1. **Morning.** Email, bills, channel stats on the CRT. Put on the headset.
2. **Call 1: gift cards.** The simplest script. Teaches trust, persona sheet, one tell.
3. **Call 2: fake tech support.** Adds the supervisor transfer and the persona check.
4. **Call 3: grandkid in jail.** A recurring rival scammer returns and tests the persona sheet.
5. **Evening.** Stamp an evidence file per call and drop it in the outbox. Pick which
   CLIP moments to post. Channel tally, bills paid or not, victims board updates.

## 4. Desk and camera

First person, seated, mouse look clamped to an arc; hover highlights clickables; lean in to
read paper, lean back to return (same as The Bench).

Objects: headset and caller-ID box, persona sheet, notepad, CRT monitor, CLIP button,
evidence stamp and outbox, desk clock, corkboard behind the desk, coffee mug.

## 5. Evidence and stamping

After each call the player assembles an evidence file from logged tells and extracted facts,
then stamps it. A stamp backed by correct tells scores well. A stamp that claims something not
actually logged costs channel reputation. Reuses the stamp and file interaction from The Bench.

## 6. Systems

- **Channel:** subscribers, money, daily bills. CLIPs pressed during calls become candidate
  videos; posting grows subs by the clip's quality (a function of call event and trust level).
  Sponsor offers (e.g. a VPN company) pay cash but require a scripted read that lowers trust.
- **Exposure:** a leak sets a flag that carries into the next day's callers, who then know
  your real details. First build records the flag and shows it in the evening summary.
- **Victims board (cartoon-light):** names from the ring's call list. Minutes you hold a ring
  keep its victims safe; the board shows who is safe at the end of the day. No graphic stakes.
- **Recurring rivals:** the first build ships one named rival (Kevin from "Microsoft") who
  calls on call 3 and remembers one lie from earlier in the day.
- **Hidden truth per call:** each scammer script has a defined set of valid tells and facts,
  so the runner can score evidence correctly.

## 7. Look and sound

- Rendering and style carry over from The Bench: soft shadows, AO, bloom on screens and
  lamps, film grain, warm grading, rounded cartoon figures with drawn faces.
- Room: a dim basement with a corkboard and red string, monitors, cables, a shelf of
  gear, a single hanging bulb.
- The scammer appears as a caller-ID card with a drawn face on the CRT, animated by the
  call mood. Text appears in a subtitle box with pitched mumble blips, one pitch per scammer.
- No voice acting. Sound: phone ring, headset crackle, hold music, room tone, stamp, CLIP chime.

## 8. Structure

| Unit | Job | Depends on |
|---|---|---|
| `GameState` (autoload) | Day, money, subs, exposure, record, flags. Save and load. | nothing |
| `CallScript` | Plain data: scammer, node graph, lines, tells, persona questions. | nothing |
| `CallRunner` | Steps a call through nodes; owns trust, mood, minutes, evidence. No visuals. | `CallScript`, `GameState` |
| `Persona` | The fake-victim sheet and answer checking. | nothing |
| `DialogueBox` | Shows scammer lines, plays blips, shows the line menu. | nothing |
| `CallerCard` | Scammer face and mood on the CRT. | nothing |
| `Desk` | Clickable objects and the evidence stamp flow. | `CallRunner` signals |
| `SeatedCamera` | Mouse look, hover, lean. Copied from The Bench. | nothing |
| `Lab` | The basement scene; wires the above. | all of the above |
| `Channel` | Clips, posting, money, bills. | `GameState` |
| `MainMenu`, `Settings` | New game, continue, volume, sensitivity. | `GameState` |

Content lives in data files (call scripts, scammer profiles, persona sheets, tells,
sponsor offers, email text), so adding a scam never touches code. `CallRunner` has no
visuals and can be tested headless.

## 9. Saving

Autosave at the start of each day and after the evening summary. One slot. A missing or
unreadable save offers New Game instead of crashing.

## 10. Testing

- **Script check:** every call script has a start node, every node a valid next, and a
  reachable end. Every persona question has exactly one matching answer.
- **Runner check:** play each call headless through win, hang-up-at-zero-trust, and leak
  paths; trust, minutes, tells and exposure update correctly.
- **Stamp check:** an over-claiming stamp is penalized; a fully backed one is not.
- **Manual:** play the full day before calling the build done.

## 11. Out of scope for the first build

Days beyond day one, romance and crypto scams, the remote-access VM minigame, sponsor read
mechanics beyond one offer, sandbox mode, live AI dialogue, voiced lines, controller support.
Live AI dialogue is a possible later swap behind the `CallRunner` line-source interface; it
needs a network key and per-call cost, so it is not planned for the first build.
