# Please Hold First Build Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One complete, polished day of Please Hold: morning inbox, three scam calls (gift cards, fake tech support, grandkid-in-jail) played through a line menu with a trust meter, evidence stamping, and an evening channel tally.

**Architecture:** All rules live in headless `RefCounted` scripts under `scripts/core/` (`GameState`, `CallRunner`, `Evidence`, `Channel`, `CallValidator`) that are test-driven with a SceneTree test runner. Calls are JSON data files under `data/calls/`. The visual layer (basement lab room, HUD, CRT caller card, controller, menu) is built afterwards from code in the style of the sibling Godot project `~/JudgeGame`, copying its room kit, character builder, seated camera and sound synth.

**Tech Stack:** Godot 4.7.1 (`/opt/homebrew/bin/godot`), GDScript, no external assets (all geometry and sound generated in code), JSON content, git.

## Global Constraints

Copied from `docs/superpowers/specs/2026-10-14-please-hold-design.md`:

- Engine: Godot 4, desktop, Mac first, GDScript. Viewport 1920x1080, stretch mode `canvas_items`, aspect `expand`.
- Pure goofy deadpan tone. Scammers are cartoon incompetents; nobody is really hurt. No graphic stakes.
- No live AI and no typed input: the player picks from 3-4 hand-written lines per turn.
- Content lives in data files (call scripts, persona, tells, email text); adding a scam never touches code.
- `CallRunner` holds the rules and has no visuals, so it is testable headless.
- Trust meter is 0-100, starts at 50, and the scammer hangs up at 0. Score is call minutes kept on the line.
- Persona question: the scammer's question has several value lines; the one matching the persona sheet is correct, the others cost trust.
- Evidence: a stamp backed by logged tells scores well; claiming something not logged costs channel reputation.
- First build is one day, three calls, one recurring rival (Kevin) on call 3 who remembers one lie from call 2.
- No voice acting: subtitle box with pitched mumble blips, one pitch per scammer. No audio files; all sound is synthesized.
- One save slot. Autosave at the start of the day (after the morning decision) and after the evening summary. A missing or unreadable save must offer New Game, never crash.
- Reuse the Judge game by copying files from `~/JudgeGame/scripts/`, not by linking.
- Out of scope: days beyond day one, romance and crypto scams, the remote-access VM minigame, sandbox mode, live AI dialogue, voiced lines, controller support.

## File Structure

```
project.godot                      engine config (autoloads added in Task 6)
.gitignore
data/persona.json                  the fake-victim sheet
data/calls/gift_cards.json         call 1
data/calls/tech_support.json       call 2 (supervisor transfer)
data/calls/grandkid_jail.json      call 3 (rival Kevin)
scripts/core/game_state.gd         day, clock, money, subs, rep, flags, queue, save/load (no visuals)
scripts/core/call_runner.gd        one call's rules: trust, mood, tells, persona, evidence (no visuals)
scripts/core/evidence.gd           scores a stamped evidence file
scripts/core/channel.gd            clips, posting, sponsor, daily settle
scripts/core/call_validator.gd     checks a call script's data
scripts/core/data.gd               JSON loading and text helpers
scripts/game.gd                    autoload: state, settings, saving, scene changes
scripts/sfx.gd                     autoload: synthesized sound (copied from Judge, extended)
scripts/room_kit.gd                copied from Judge: materials, boxes, lights, film look
scripts/character.gd               copied from Judge: one animated person
scripts/character_builder.gd       copied from Judge: builds a person from an appearance dict
scripts/seated_camera.gd           copied from Judge: mouse look, hover, click
scripts/ui_kit.gd                  fonts, panel boxes, labels, buttons
scripts/hud.gd                     all 2D interface
scripts/lab_room.gd                builds the basement and its desk objects
scripts/caller_card.gd             SubViewport that renders the scammer onto the CRT
scripts/lab.gd                     scene controller: morning, calls, stamping, evening
scripts/main_menu.gd               title screen and settings
scenes/lab.tscn, scenes/main_menu.tscn
tests/run.sh, tests/run_tests.gd   headless rule tests
tests/shot.gd, tests/shot.tscn     screenshot a scene
tests/autoplay.sh                  plays the whole day headless
```

Design notes that differ from the spec's table, deliberately:

- `Persona` is a data file plus `Data.persona()`, not a class; `DialogueBox` is folded into `hud.gd`; `Desk` is folded into `lab_room.gd` as clickable bodies.
- During a call the mouse is free, so HOLD, CLIP and HANG UP are HUD buttons (keys H and C also work), and the notepad and persona sheet are HUD panels. The 3D desk carries the headset, persona sheet, CRT, outbox, corkboard, clock and caller-ID box.

---

### Task 1: Scaffold, test harness, GameState

**Files:**
- Create: `project.godot`, `.gitignore`, `tests/run.sh`, `tests/run_tests.gd`, `scripts/core/game_state.gd`

**Interfaces:**
- Produces: `GameState` (preload path `res://scripts/core/game_state.gd`) with constants `START_MONEY=30`, `BILLS=60`, `OPEN=1140`; vars `day, minutes, money, subs, rep, flags, exposed, stage, call_index, calls_today, clips, results`; methods `add_time(mins:int)`, `clock_text()->String`, `change_rep(delta:int)`, `record_result(r:Dictionary)`, `has_next_call()->bool`, `current_call_id()->String`, `advance_call()`, `start_next_day()`, `to_dict()`, `static from_dict(d)`, `save(path)->bool`, `static load_from(path)` (returns a GameState or `null`).
- Produces: test harness. Any function named `test_*` in `tests/run_tests.gd` is discovered and run. Helper `check(cond: bool, msg: String)`.

- [ ] **Step 1: Create the project files**

```bash
cd "/Users/alexanderbogert/Please Hold"
mkdir -p scripts/core data/calls scenes tests shots
```

`project.godot`:

```ini
config_version=5

[application]

config/name="Please Hold"
config/features=PackedStringArray("4.7", "Forward Plus")

[display]

window/size/viewport_width=1920
window/size/viewport_height=1080
window/stretch/mode="canvas_items"
window/stretch/aspect="expand"

[rendering]

anti_aliasing/quality/screen_space_aa=1
```

`.gitignore`:

```
.godot/
shots/
```

`tests/run.sh` (a script error would leave Godot idling, so it is capped at 40 seconds):

```sh
#!/bin/sh
# Runs the rule tests headless and reports.
cd "$(dirname "$0")/.."
LOG="${TMPDIR:-/tmp}/pleasehold_tests.log"
godot --headless --path . -s tests/run_tests.gd > "$LOG" 2>&1 &
PID=$!
for i in $(seq 1 40); do
	kill -0 $PID 2>/dev/null || break
	sleep 1
done
kill $PID 2>/dev/null
grep -E "FAIL|ERROR|passed" "$LOG" | sort -u | head -40
grep -q " 0 failed" "$LOG" && ! grep -q "SCRIPT ERROR" "$LOG"
```

```bash
chmod +x tests/run.sh
```

- [ ] **Step 2: Write the harness and the failing GameState tests**

`tests/run_tests.gd`:

```gdscript
extends SceneTree
## Headless test runner: godot --headless --path . -s tests/run_tests.gd

const GameState = preload("res://scripts/core/game_state.gd")

var _failures := 0
var _current := ""


func _init() -> void:
	var passed := 0
	var failed := 0
	for m in get_method_list():
		var n: String = m["name"]
		if not n.begins_with("test_"):
			continue
		_current = n
		var before := _failures
		call(n)
		if _failures == before:
			passed += 1
			print("PASS ", n)
		else:
			failed += 1
	print("%d passed, %d failed" % [passed, failed])
	quit(1 if failed > 0 else 0)


func check(cond: bool, msg: String) -> void:
	if not cond:
		_failures += 1
		print("FAIL ", _current, ": ", msg)


# --- GameState ---

func test_clock_text() -> void:
	var s = GameState.new()
	check(s.clock_text() == "7:00 PM", "starts at 7:00 PM, got " + s.clock_text())
	s.add_time(75)
	check(s.clock_text() == "8:15 PM", "75 minutes later is 8:15 PM, got " + s.clock_text())
	s.minutes = 1440 + 30
	check(s.clock_text() == "12:30 AM", "past midnight wraps, got " + s.clock_text())


func test_rep_clamps() -> void:
	var s = GameState.new()
	s.change_rep(500)
	check(s.rep == 100, "clamps high")
	s.change_rep(-500)
	check(s.rep == 0, "clamps low")


func test_record_result() -> void:
	var s = GameState.new()
	s.record_result({"id": "x", "minutes": 9, "leaked": true, "clips": [{"quality": 2}], "evidence": []})
	check(s.minutes == GameState.OPEN + 9, "call minutes land on the clock")
	check(s.exposed, "a leak exposes you")
	check(s.clips.size() == 1 and s.results.size() == 1, "clip and result are kept")


func test_call_queue() -> void:
	var s = GameState.new()
	check(s.has_next_call() and s.current_call_id() == "gift_cards", "first call is gift cards")
	s.advance_call()
	s.advance_call()
	check(s.has_next_call() and s.current_call_id() == "grandkid_jail" and s.stage != "evening", "third call is the grandkid")
	s.advance_call()
	check(not s.has_next_call() and s.stage == "evening", "after the third call it is evening")


func test_save_round_trip() -> void:
	var s = GameState.new()
	s.money = 123
	s.subs = 250
	s.flags["said_toaster"] = true
	s.exposed = true
	s.stage = "evening"
	s.call_index = 2
	s.clips = [{"call": "gift_cards", "quality": 3}]
	check(s.save("user://test_save.json"), "save succeeds")
	var l = GameState.load_from("user://test_save.json")
	check(l != null, "load succeeds")
	if l == null:
		return
	check(l.money == 123 and l.subs == 250, "numbers survive")
	check(l.flags.get("said_toaster", false) == true and l.exposed, "flags survive")
	check(l.stage == "evening" and l.call_index == 2, "stage and queue survive")
	check(int(l.clips[0]["quality"]) == 3, "clips survive")


func test_bad_saves() -> void:
	check(GameState.load_from("user://does_not_exist.json") == null, "missing file gives null")
	var f := FileAccess.open("user://corrupt.json", FileAccess.WRITE)
	f.store_string("this is not json")
	f.close()
	check(GameState.load_from("user://corrupt.json") == null, "corrupt file gives null")


func test_next_day() -> void:
	var s = GameState.new()
	s.flags["sponsor"] = true
	s.flags["said_toaster"] = true
	s.exposed = true
	s.record_result({"id": "x", "minutes": 5, "leaked": false, "clips": [{"quality": 1}], "evidence": []})
	s.advance_call()
	s.start_next_day()
	check(s.day == 2 and s.minutes == GameState.OPEN and s.stage == "morning", "a fresh morning")
	check(s.call_index == 0 and s.results.is_empty() and s.clips.is_empty(), "the queue and clips reset")
	check(not s.flags.has("sponsor") and s.flags.has("said_toaster") and s.exposed, "sponsor clears, memory and exposure stay")
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `tests/run.sh`
Expected: errors such as `Could not preload resource file "res://scripts/core/game_state.gd"` (the file does not exist yet); exit status non-zero.

- [ ] **Step 4: Implement GameState**

`scripts/core/game_state.gd`:

```gdscript
extends RefCounted
## Everything that persists: day, clock, money, channel, flags. No visuals.

const START_MONEY := 30
const BILLS := 60
const OPEN := 1140   # 7:00 PM, in minutes

var day := 1
var minutes := OPEN
var money := START_MONEY
var subs := 100
var rep := 50                # channel reputation, 0..100
var flags := {}              # story flags: sponsor, said_toaster, ...
var exposed := false         # a caller has your real details
var stage := "morning"       # morning | call | evening
var call_index := 0
var calls_today: Array = ["gift_cards", "tech_support", "grandkid_jail"]
var clips: Array = []        # clips pressed today and not posted yet
var results: Array = []      # finished calls today


func add_time(mins: int) -> void:
	minutes += mins


func clock_text() -> String:
	var h := (minutes / 60) % 24
	var m := minutes % 60
	var suffix := "AM" if h < 12 else "PM"
	var h12 := h % 12
	if h12 == 0:
		h12 = 12
	return "%d:%02d %s" % [h12, m, suffix]


func change_rep(delta: int) -> void:
	rep = clampi(rep + delta, 0, 100)


## Called by CallRunner when a call ends.
func record_result(r: Dictionary) -> void:
	add_time(int(r["minutes"]))
	if r.get("leaked", false):
		exposed = true
	clips.append_array(r.get("clips", []))
	results.append(r)


func has_next_call() -> bool:
	return call_index < calls_today.size()


func current_call_id() -> String:
	return calls_today[call_index]


## Call once the current call has been stamped and filed.
func advance_call() -> void:
	call_index += 1
	if not has_next_call():
		stage = "evening"


func start_next_day() -> void:
	day += 1
	minutes = OPEN
	stage = "morning"
	call_index = 0
	clips = []
	results = []
	flags.erase("sponsor")


func to_dict() -> Dictionary:
	return {
		"day": day, "minutes": minutes, "money": money, "subs": subs, "rep": rep,
		"flags": flags, "exposed": exposed, "stage": stage, "call_index": call_index,
		"calls_today": calls_today, "clips": clips, "results": results,
	}


static func from_dict(d: Dictionary):
	var s = new()
	s.day = int(d.get("day", 1))
	s.minutes = int(d.get("minutes", OPEN))
	s.money = int(d.get("money", START_MONEY))
	s.subs = int(d.get("subs", 100))
	s.rep = int(d.get("rep", 50))
	s.flags = d.get("flags", {})
	s.exposed = bool(d.get("exposed", false))
	s.stage = str(d.get("stage", "morning"))
	s.call_index = int(d.get("call_index", 0))
	s.calls_today = d.get("calls_today", s.calls_today)
	s.clips = d.get("clips", [])
	s.results = d.get("results", [])
	return s


func save(path: String) -> bool:
	var f := FileAccess.open(path, FileAccess.WRITE)
	if f == null:
		return false
	f.store_string(JSON.stringify(to_dict()))
	return true


## Returns a GameState, or null if the file is missing or unreadable.
static func load_from(path: String):
	if not FileAccess.file_exists(path):
		return null
	var json := JSON.new()  # instance parse: a corrupt file is expected, not an engine error
	if json.parse(FileAccess.get_file_as_string(path)) != OK:
		return null
	if not (json.data is Dictionary) or not json.data.has("day"):
		return null
	return from_dict(json.data)
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `tests/run.sh`
Expected: `7 passed, 0 failed`; exit status 0.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "Scaffold project, test harness and GameState" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 2: CallRunner core (trust, mood, minutes, hang-up, facts)

**Files:**
- Create: `scripts/core/call_runner.gd`
- Modify: `tests/run_tests.gd` (add preload, `_mini()`, tests)

**Interfaces:**
- Consumes: `GameState` from Task 1 (`record_result`, `flags`).
- Produces: `CallRunner` (preload path `res://scripts/core/call_runner.gd`).
  - Signals: `line(who: String, text: String, mood: String)`, `options_ready(options: Array)` (each `{text, type}`), `tells_changed(tells: Array)`, `mood_changed(mood: String)`, `scammer_changed(scammer: Dictionary)`, `finished(result: Dictionary)`.
  - Vars: `data, persona, state, scammer, node_id, node, trust, irritation, minutes, evidence, revealed, held, leaked, clips, done, result`.
  - Methods: `start(call_data: Dictionary, persona_sheet: Dictionary, game_state)`, `options() -> Array`, `say_text() -> String`, `fill(text) -> String`, `mood_index() -> int`, `mood() -> String`, `choose(index: int)`.
  - `result` dictionary: `{id, outcome ("complete"|"hangup"|"quit"), minutes, evidence, leaked, clips, victims_safe, victims_total}`.
- Call script data shape (used by every later task):

```
{ "id", "title", "start": node_id,
  "scammer": {"name","pitch","face":{appearance}, "decay"?: int},
  "ring": {"name","victims":[...],"minutes_per_victim":int},
  "facts": {fact_id: note text}, "charges": [{"id","text","needs":[evidence ids]}],
  "nodes": { id: {"say", "variants"?:[{"if_flag","say"}], "tells"?:[{"id","text","on_hold"?,"if_flag"?}],
                  "transfer"?: scammer dict, "end"?: true,
                  "options"?:[{"text","type":dumb|cooperate|probe|persona|leak,"trust":int,"minutes":int,
                               "next":node_id,"gives_fact"?,"set_flag"?,"value_of"?,"value"?}] } } }
```

- [ ] **Step 1: Write the failing tests**

In `tests/run_tests.gd`, add under the existing preload:

```gdscript
const CallRunner = preload("res://scripts/core/call_runner.gd")
```

Append:

```gdscript
# --- CallRunner ---

func _mini() -> Dictionary:
	return {
		"id": "mini", "start": "a",
		"scammer": {"name": "Tester", "pitch": 1.0, "face": {}},
		"ring": {"name": "R", "victims": ["x", "y"], "minutes_per_victim": 2},
		"facts": {"f1": "A fact"},
		"charges": [{"id": "c1", "text": "C1", "needs": ["t1"]}],
		"nodes": {
			"a": {"say": "Hi {name}.", "tells": [{"id": "t1", "text": "tell one"}, {"id": "t2", "text": "held tell", "on_hold": true}], "options": [
				{"text": "dumb", "type": "dumb", "trust": 4, "minutes": 2, "next": "b"},
				{"text": "coop", "type": "cooperate", "trust": 6, "minutes": 1, "next": "b"},
				{"text": "probe", "type": "probe", "trust": -10, "minutes": 1, "gives_fact": "f1", "next": "b"},
				{"text": "rude", "type": "probe", "trust": -60, "minutes": 1, "gives_fact": "f1", "next": "b"}]},
			"b": {"say": "Bye.", "end": true},
		},
	}


func test_runner_opens() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	var said := []
	var shown := []
	r.line.connect(func(who, text, mood): said.append([who, text, mood]))
	r.options_ready.connect(func(opts): shown.append(opts))
	r.start(_mini(), {"name": "Dot"}, s)
	check(said.size() == 1 and said[0][0] == "Tester" and said[0][1] == "Hi Dot." and said[0][2] == "patient", "opening line is filled and patient, got %s" % [said])
	check(shown.size() == 1 and shown[0].size() == 4 and shown[0][1]["type"] == "cooperate", "four options shown in order")
	check(r.trust == 50 and r.minutes == 0 and not r.done, "starts at trust 50, zero minutes")


func test_runner_trust_and_minutes() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	r.choose(1)   # cooperate +6, one minute, then the script ends
	check(r.trust == 56, "cooperate adds 6, got %d" % r.trust)
	check(r.done and r.result["outcome"] == "complete" and r.result["minutes"] == 1, "the script ran out")
	check(s.minutes == GameState.OPEN + 1 and s.results.size() == 1, "the result lands on the state")
	check(r.result["victims_safe"] == 0 and r.result["victims_total"] == 2, "one minute saves nobody")
	var s2 = GameState.new()
	var r2 = CallRunner.new()
	r2.start(_mini(), {"name": "Dot"}, s2)
	r2.choose(0)   # dumb: two minutes
	check(r2.result["minutes"] == 2 and r2.result["victims_safe"] == 1, "two minutes saves one victim")


func test_runner_hangup() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	r.choose(3)   # -60
	check(r.done and r.result["outcome"] == "hangup" and r.trust == 0, "trust zero is a hang-up")
	check(r.result["evidence"].is_empty(), "no fact is given on the turn they hang up")
	check(r.result["minutes"] == 1, "the minute still counts")


func test_runner_probe_fact() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	r.choose(2)   # -10 but gives f1
	check(r.result["evidence"] == ["f1"] and r.trust == 40, "the probe pays in evidence and costs trust")
	check(r.irritation == 3, "a probe adds extra irritation, got %d" % r.irritation)


func test_runner_mood() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	r.irritation = 4
	check(r.mood() == "irritated", "irritation 4 is irritated")
	r.choose(0)   # dumb +4, halved when irritated
	check(r.trust == 52, "dumb gains halve when irritated, got %d" % r.trust)
	var r2 = CallRunner.new()
	r2.start(_mini(), {"name": "Dot"}, GameState.new())
	r2.irritation = 8
	check(r2.mood() == "threatening", "irritation 8 is threatening")
	r2.choose(0)
	check(r2.trust == 50, "dumb gains nothing when threatening, got %d" % r2.trust)
	var r3 = CallRunner.new()
	r3.start(_mini(), {"name": "Dot"}, GameState.new())
	r3.irritation = 8
	r3.choose(1)   # cooperate +6, doubled when threatening
	check(r3.trust == 62, "cooperate doubles when threatening, got %d" % r3.trust)
	var r4 = CallRunner.new()
	r4.start(_mini(), {"name": "Dot"}, GameState.new())
	r4.irritation = 4
	r4.choose(2)   # probe -10, x1.5 when irritated
	check(r4.trust == 35, "probe losses grow when irritated, got %d" % r4.trust)


func test_runner_bad_input() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	r.choose(99)
	r.choose(-1)
	check(not r.done and r.trust == 50, "out of range choices do nothing")
	r.choose(0)
	var trust := r.trust
	r.choose(1)
	check(r.trust == trust and s.results.size() == 1, "a finished call ignores further choices")
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `tests/run.sh`
Expected: error `Could not preload resource file "res://scripts/core/call_runner.gd"`.

- [ ] **Step 3: Implement the CallRunner core**

`scripts/core/call_runner.gd`:

```gdscript
extends RefCounted
## Steps one scam call. Holds the rules, no visuals.

signal line(who: String, text: String, mood: String)   # the scammer speaks
signal options_ready(options: Array)                    # [{text, type}] in display order
signal tells_changed(tells: Array)                      # tells the player can note right now
signal mood_changed(mood: String)
signal scammer_changed(scammer: Dictionary)
signal finished(result: Dictionary)

const MOODS := ["patient", "irritated", "threatening"]
## Trust multipliers by option type and mood index (patient, irritated, threatening).
const GAIN := {"dumb": [1.0, 0.5, 0.0], "cooperate": [1.0, 1.5, 2.0]}
const LOSS := {"probe": [1.0, 1.5, 2.0], "persona": [1.0, 1.5, 2.0]}
const START_TRUST := 50
const SPONSOR_TRUST := -10   # reading the sponsor's ad has already put them off
const HOLD_COST := 3
const CLIP_COST := 2
const IRRITATED_AT := 4
const THREATENING_AT := 8

var data: Dictionary
var persona: Dictionary
var state
var scammer: Dictionary
var node_id := ""
var node: Dictionary
var trust := START_TRUST
var irritation := 0
var minutes := 0
var evidence: Array = []
var revealed: Array = []
var held := false
var leaked := false
var clips: Array = []
var done := false
var result: Dictionary = {}


func start(call_data: Dictionary, persona_sheet: Dictionary, game_state) -> void:
	data = call_data
	persona = persona_sheet
	state = game_state
	scammer = data["scammer"]
	trust = START_TRUST + (SPONSOR_TRUST if state.flags.get("sponsor", false) else 0)
	irritation = 0
	minutes = 0
	evidence = []
	clips = []
	leaked = false
	done = false
	result = {}
	scammer_changed.emit(scammer)
	_enter(data["start"])


func mood_index() -> int:
	if irritation >= THREATENING_AT:
		return 2
	return 1 if irritation >= IRRITATED_AT else 0


func mood() -> String:
	return MOODS[mood_index()]


## Replace {field} with the persona sheet's value.
func fill(text: String) -> String:
	for k in persona:
		text = text.replace("{%s}" % k, str(persona[k]))
	return text


func say_text() -> String:
	for v in node.get("variants", []):
		if state.flags.get(v["if_flag"], false):
			return fill(v["say"])
	return fill(node["say"])


func options() -> Array:
	var out: Array = []
	for o in node.get("options", []):
		out.append({"text": fill(o["text"]), "type": o["type"]})
	return out


func choose(index: int) -> void:
	if done:
		return
	var opts: Array = node.get("options", [])
	if index < 0 or index >= opts.size():
		return
	var o: Dictionary = opts[index]
	var kind: String = o["type"]
	var delta := float(o.get("trust", 0))
	if delta > 0.0:
		delta *= GAIN.get(kind, [1.0, 1.0, 1.0])[mood_index()]
	elif delta < 0.0:
		delta *= LOSS.get(kind, [1.0, 1.0, 1.0])[mood_index()]
	var rude := delta < 0.0
	trust = clampi(trust + int(round(delta)) - int(scammer.get("decay", 0)), 0, 100)
	minutes += int(o.get("minutes", 1))
	if trust <= 0:
		_finish("hangup")
		return
	if o.has("gives_fact"):
		_add_evidence(o["gives_fact"])
	var before := mood_index()
	irritation += 1 + (2 if kind == "probe" or (kind == "persona" and rude) else 0)
	if mood_index() != before:
		mood_changed.emit(mood())
	_enter(o["next"])


func _enter(id: String) -> void:
	node_id = id
	node = data["nodes"][id]
	held = false
	_refresh_tells()
	line.emit(scammer["name"], say_text(), mood())
	if node.get("end", false):
		_finish("complete")
	else:
		options_ready.emit(options())


func _refresh_tells() -> void:
	revealed = []
	tells_changed.emit(revealed)


func _add_evidence(id: String) -> void:
	if not (id in evidence):
		evidence.append(id)


func _finish(outcome: String) -> void:
	if done:
		return
	done = true
	var ring: Dictionary = data["ring"]
	var victims: Array = ring["victims"]
	@warning_ignore("integer_division")
	var safe := mini(victims.size(), minutes / int(ring["minutes_per_victim"]))
	result = {
		"id": data["id"], "outcome": outcome, "minutes": minutes,
		"evidence": evidence.duplicate(), "leaked": leaked, "clips": clips.duplicate(true),
		"victims_safe": safe, "victims_total": victims.size(),
	}
	state.record_result(result)
	finished.emit(result)
```

Note: `_refresh_tells()` is a deliberate stub for this task (it shows nothing). Task 3 gives it the real body and the `hold`, `log_tell`, `clip`, `end_call`, flags, leak and transfer behaviour.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `tests/run.sh`
Expected: `13 passed, 0 failed`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add CallRunner core: trust, mood, minutes, hang-up, facts" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 3: CallRunner extras (tells, hold, persona, leak, flags, transfer, clip, sponsor, quit)

**Files:**
- Modify: `scripts/core/call_runner.gd` (replace `_refresh_tells`, add `hold`, `log_tell`, `clip`, `end_call`; leak, flags and transfer already work through the shared code from Task 2 except where noted below)
- Modify: `tests/run_tests.gd` (add `_mini2()` and tests)

**Interfaces:**
- Consumes: Task 2's `CallRunner`.
- Produces: `hold() -> bool` (false if already held or done), `log_tell(id: String) -> bool`, `clip() -> Dictionary` (empty if done; else `{call, node, quality, minutes}`), `end_call()`. Leaking sets `result["leaked"]`. A node with `"transfer"` swaps `scammer` and emits `scammer_changed` before the node's line. Tells with `"on_hold": true` only show after `hold()`; tells with `"if_flag"` only show when that state flag is set.

- [ ] **Step 1: Write the failing tests**

Append to `tests/run_tests.gd`:

```gdscript
func _mini2() -> Dictionary:
	return {
		"id": "mini2", "start": "a",
		"scammer": {"name": "First", "pitch": 1.0, "face": {}},
		"ring": {"name": "R", "victims": ["x"], "minutes_per_victim": 5},
		"facts": {},
		"charges": [],
		"nodes": {
			"a": {"say": "Birthday?", "options": [
				{"text": "It's {birthday}.", "type": "persona", "trust": 6, "minutes": 1, "value_of": "birthday", "next": "b"},
				{"text": "It's yesterday.", "type": "persona", "trust": -12, "minutes": 1, "value": "yesterday", "next": "b"}]},
			"b": {"say": "Tell me something.", "variants": [{"if_flag": "lied", "say": "You lied before."}], "options": [
				{"text": "A lie", "type": "dumb", "trust": 2, "minutes": 1, "set_flag": "lied", "next": "c"},
				{"text": "My real address", "type": "leak", "trust": 8, "minutes": 1, "next": "c"}]},
			"c": {"say": "Let me get my boss.", "tells": [{"id": "late", "text": "flagged tell", "if_flag": "lied"}], "options": [
				{"text": "ok", "type": "dumb", "trust": 2, "minutes": 1, "next": "d"}]},
			"d": {"transfer": {"name": "Boss", "pitch": 0.7, "decay": 2, "face": {}}, "say": "Boss here.", "options": [
				{"text": "hi", "type": "dumb", "trust": 4, "minutes": 1, "next": "e"}]},
			"e": {"say": "Bye.", "end": true},
		},
	}


func test_tells_and_hold() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	var seen := []
	r.tells_changed.connect(func(t): seen.append(t.map(func(x): return x["id"])))
	r.start(_mini(), {"name": "Dot"}, s)
	check(seen.back() == ["t1"], "only the plain tell shows at first, got %s" % [seen.back()])
	check(r.log_tell("t1") and "t1" in r.evidence, "a visible tell can be noted")
	check(not r.log_tell("t2"), "a hold-only tell cannot be noted before holding")
	check(r.hold(), "hold works")
	check(r.trust == 47 and seen.back() == ["t1", "t2"], "hold costs 3 trust and shows the room, got %d %s" % [r.trust, seen.back()])
	check(r.log_tell("t2"), "now it can be noted")
	check(not r.hold(), "only one hold per moment")
	r.choose(1)
	check(r.result["evidence"] == ["t1", "t2"], "noted tells end up in the result")


func test_hold_can_hang_up() -> void:
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, GameState.new())
	r.trust = 2
	r.hold()
	check(r.done and r.result["outcome"] == "hangup", "holding with no trust left ends the call")


func test_persona_answers() -> void:
	var r = CallRunner.new()
	r.start(_mini2(), {"birthday": "March 3"}, GameState.new())
	check(r.options()[0]["text"] == "It's March 3.", "the sheet fills the line, got %s" % r.options()[0]["text"])
	r.choose(1)   # the wrong value
	check(r.trust == 38 and r.irritation == 3, "a wrong persona answer costs 12 and adds irritation, got %d %d" % [r.trust, r.irritation])
	var r2 = CallRunner.new()
	r2.start(_mini2(), {"birthday": "March 3"}, GameState.new())
	r2.choose(0)
	check(r2.trust == 56 and r2.irritation == 1, "the right value gains trust, got %d %d" % [r2.trust, r2.irritation])


func test_leak_and_transfer() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	var names := []
	r.scammer_changed.connect(func(sc): names.append(sc["name"]))
	r.start(_mini2(), {"birthday": "March 3"}, s)
	r.choose(0)   # persona right: 56
	r.choose(1)   # leak +8: 64
	r.choose(0)   # dumb +2: 66, now they fetch the boss
	check(r.scammer["name"] == "Boss" and names == ["First", "Boss"], "the transfer swaps the scammer, got %s" % [names])
	r.choose(0)   # dumb +4, minus the boss's decay of 2
	check(r.trust == 68, "the boss's decay applies, got %d" % r.trust)
	check(r.result["outcome"] == "complete" and r.result["leaked"], "the leak is in the result")
	check(s.exposed, "and on the state")


func test_flags_and_variants() -> void:
	var s = GameState.new()
	var r = CallRunner.new()
	var said := []
	var shown := []
	r.line.connect(func(who, text, mood): said.append(text))
	r.tells_changed.connect(func(t): shown.append(t.size()))
	r.start(_mini2(), {"birthday": "March 3"}, s)
	r.choose(0)
	check(said.back() == "Tell me something.", "plain line without the flag")
	r.choose(0)   # tell the lie
	check(s.flags.get("lied", false), "the lie is remembered")
	check(shown.back() == 1, "the flagged tell shows once the lie is told")
	var s2 = GameState.new()
	s2.flags["lied"] = true
	var r2 = CallRunner.new()
	var said2 := []
	r2.line.connect(func(who, text, mood): said2.append(text))
	r2.start(_mini2(), {"birthday": "March 3"}, s2)
	r2.choose(0)
	check(said2.back() == "You lied before.", "a remembered lie changes the line, got %s" % said2.back())


func test_clip() -> void:
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, GameState.new())
	var c: Dictionary = r.clip()
	check(c["quality"] == 1 and r.trust == 48 and r.clips.size() == 1, "a calm early clip is worth 1 and costs 2 trust")
	r.trust = 80
	r.irritation = 4
	check(r.clip()["quality"] == 3, "irritated plus high trust is worth 3")
	r.trust = 2
	r.clip()
	check(r.trust == 1 and not r.done, "a clip never hangs the call up")
	r.end_call()
	check(r.clip().is_empty(), "no clipping after the call")


func test_sponsor_and_quit() -> void:
	var s = GameState.new()
	s.flags["sponsor"] = true
	var r = CallRunner.new()
	r.start(_mini(), {"name": "Dot"}, s)
	check(r.trust == 40, "the sponsor read starts you at 40, got %d" % r.trust)
	r.end_call()
	check(r.done and r.result["outcome"] == "quit", "hanging up yourself is a quit")
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `tests/run.sh`
Expected: FAIL lines (tells never appear, `hold`/`clip`/`end_call` missing script errors).

- [ ] **Step 3: Implement the extras**

In `scripts/core/call_runner.gd` replace the stub `_refresh_tells` with:

```gdscript
func _refresh_tells() -> void:
	revealed = []
	for t in node.get("tells", []):
		if t.has("if_flag") and not state.flags.get(t["if_flag"], false):
			continue
		if t.get("on_hold", false) and not held:
			continue
		revealed.append(t)
	tells_changed.emit(revealed)
```

Add the node-entry transfer. In `_enter`, after `held = false`, insert:

```gdscript
	if node.has("transfer"):
		scammer = node["transfer"]
		scammer_changed.emit(scammer)
```

Mark leaks. In `choose`, after the `var rude := ...` line, add:

```gdscript
	if kind == "leak":
		leaked = true
	if o.has("set_flag"):
		state.flags[o["set_flag"]] = true
```

Add these public methods (before `_enter`):

```gdscript
## Put the scammer on hold: you hear the room behind them. Costs a little trust.
func hold() -> bool:
	if done or held:
		return false
	held = true
	trust = maxi(trust - HOLD_COST, 0)
	if trust <= 0:
		_finish("hangup")
		return true
	_refresh_tells()
	return true


## Write a visible tell on the notepad.
func log_tell(id: String) -> bool:
	for t in revealed:
		if t["id"] == id:
			_add_evidence(id)
			return true
	return false


## Mark this moment for the channel. Costs a beat of attention, never the call.
func clip() -> Dictionary:
	if done:
		return {}
	trust = maxi(trust - CLIP_COST, 1)
	var c := {"call": data["id"], "node": node_id, "minutes": minutes,
		"quality": 1 + mood_index() + (1 if trust >= 70 else 0)}
	clips.append(c)
	return c


func end_call() -> void:
	_finish("quit")
```

Check the clip numbers against the tests: the first clip at trust 50 gives quality `1 + 0 + 0 = 1` and trust 48. With trust 80 and irritation 4, the clip first lowers trust to 78, then `quality = 1 + 1 + 1 = 3`. In `clip()` the quality is computed after the trust cost, which matches both tests.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `tests/run.sh`
Expected: `20 passed, 0 failed`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "CallRunner: tells, hold, persona, leak, flags, transfer, clip" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Evidence scoring and Channel

**Files:**
- Create: `scripts/core/evidence.gd`, `scripts/core/channel.gd`
- Modify: `tests/run_tests.gd`

**Interfaces:**
- Consumes: `GameState` (`clips`, `results`, `rep`, `subs`, `money`, `flags`).
- Produces:
  - `Evidence.is_backed(charge: Dictionary, evidence: Array) -> bool`
  - `Evidence.score(call_data: Dictionary, evidence: Array, claims: Array) -> Dictionary` → `{backed:int, unbacked:int, rep_delta:int}` (`backed*3 - unbacked*6`)
  - `Channel.accept_sponsor(state)`, `Channel.post(state, picked: Array) -> {subs:int, cash:int}`, `Channel.settle(state) -> {minutes, pay, sponsor, bills, net}`

- [ ] **Step 1: Write the failing tests**

Add preloads to `tests/run_tests.gd`:

```gdscript
const Evidence = preload("res://scripts/core/evidence.gd")
const Channel = preload("res://scripts/core/channel.gd")
```

Append:

```gdscript
# --- Evidence and Channel ---

func test_evidence_scoring() -> void:
	var d := {"charges": [
		{"id": "a", "text": "A", "needs": ["t1"]},
		{"id": "b", "text": "B", "needs": ["t1", "t2"]},
		{"id": "c", "text": "C", "needs": ["t3"]}]}
	var sc: Dictionary = Evidence.score(d, ["t1", "t2"], ["a", "b", "c"])
	check(sc["backed"] == 2 and sc["unbacked"] == 1 and sc["rep_delta"] == 0, "two backed, one over-claim nets 0, got %s" % [sc])
	var none: Dictionary = Evidence.score(d, [], [])
	check(none["backed"] == 0 and none["unbacked"] == 0 and none["rep_delta"] == 0, "claiming nothing is neutral")
	var over: Dictionary = Evidence.score(d, [], ["a"])
	check(over["unbacked"] == 1 and over["rep_delta"] == -6, "an unbacked claim costs 6, got %s" % [over])
	var part: Dictionary = Evidence.score(d, ["t1"], ["b"])
	check(part["unbacked"] == 1, "a charge needs ALL its evidence")
	check(Evidence.score(d, ["t1"], ["nonsense"])["backed"] == 0, "unknown claim ids are ignored")


func test_channel_post() -> void:
	var s = GameState.new()
	s.clips = [{"call": "x", "quality": 2}, {"call": "y", "quality": 1}]
	var got: Dictionary = Channel.post(s, [0])
	check(got["subs"] == 20 and got["cash"] == 10, "a quality 2 clip at rep 50 is 20 subs and $10, got %s" % [got])
	check(s.subs == 120 and s.money == GameState.START_MONEY + 10, "state is updated")
	check(s.clips.size() == 1 and int(s.clips[0]["quality"]) == 1, "the unposted clip stays")
	var s2 = GameState.new()
	s2.rep = 0
	s2.clips = [{"call": "x", "quality": 2}]
	check(Channel.post(s2, [0])["subs"] == 10, "a bad reputation halves growth")


func test_channel_settle() -> void:
	var s = GameState.new()
	s.results = [{"minutes": 10}, {"minutes": 12}]
	var t: Dictionary = Channel.settle(s)
	check(t["pay"] == 44 and t["net"] == -16 and s.money == GameState.START_MONEY - 16, "22 minutes pays $44 against $60 bills, got %s" % [t])
	var s2 = GameState.new()
	s2.results = [{"minutes": 10}, {"minutes": 12}]
	Channel.accept_sponsor(s2)
	check(Channel.settle(s2)["net"] == 24, "the sponsor adds $40")
	var s3 = GameState.new()
	check(Channel.settle(s3)["net"] == -60, "an idle day just costs the bills")
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `tests/run.sh`
Expected: `Could not preload resource file "res://scripts/core/evidence.gd"`.

- [ ] **Step 3: Implement**

`scripts/core/evidence.gd`:

```gdscript
extends RefCounted
## Scores a stamped evidence file. A claim is backed only when every piece of
## evidence it needs was logged on the call.

const BACKED_REP := 3
const UNBACKED_REP := 6


static func is_backed(charge: Dictionary, evidence: Array) -> bool:
	for need in charge["needs"]:
		if not (need in evidence):
			return false
	return true


## `claims` are the charge ids the player ticked before stamping.
static func score(call_data: Dictionary, evidence: Array, claims: Array) -> Dictionary:
	var backed := 0
	var unbacked := 0
	for charge in call_data["charges"]:
		if charge["id"] in claims:
			if is_backed(charge, evidence):
				backed += 1
			else:
				unbacked += 1
	return {"backed": backed, "unbacked": unbacked, "rep_delta": backed * BACKED_REP - unbacked * UNBACKED_REP}
```

`scripts/core/channel.gd`:

```gdscript
extends RefCounted
## Money and subscribers. Pure rules, no visuals.

const GameState = preload("res://scripts/core/game_state.gd")

const MINUTE_PAY := 2    # donations per call minute
const CLIP_CASH := 5     # per quality point of a posted clip
const CLIP_SUBS := 10    # per quality point, scaled by reputation
const SPONSOR_PAY := 40


static func accept_sponsor(state) -> void:
	state.flags["sponsor"] = true


## Post the clips at `picked` (indices into state.clips). Returns what they earned.
static func post(state, picked: Array) -> Dictionary:
	var subs := 0
	var cash := 0
	var keep: Array = []
	for i in state.clips.size():
		var c: Dictionary = state.clips[i]
		if i in picked:
			var q := int(c["quality"])
			subs += int(round(q * CLIP_SUBS * (0.5 + state.rep / 100.0)))
			cash += q * CLIP_CASH
		else:
			keep.append(c)
	state.clips = keep
	state.subs += subs
	state.money += cash
	return {"subs": subs, "cash": cash}


## End of the day: donations for time on the line, the sponsor, then the bills.
static func settle(state) -> Dictionary:
	var minutes := 0
	for r in state.results:
		minutes += int(r["minutes"])
	var pay := minutes * MINUTE_PAY
	var sponsor := SPONSOR_PAY if state.flags.get("sponsor", false) else 0
	var net := pay + sponsor - GameState.BILLS
	state.money += net
	return {"minutes": minutes, "pay": pay, "sponsor": sponsor, "bills": GameState.BILLS, "net": net}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `tests/run.sh`
Expected: `23 passed, 0 failed`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add evidence scoring and channel economy" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Data helpers, call validator, persona and the three call scripts

**Files:**
- Create: `scripts/core/data.gd`, `scripts/core/call_validator.gd`, `data/persona.json`, `data/calls/gift_cards.json`, `data/calls/tech_support.json`, `data/calls/grandkid_jail.json`
- Modify: `tests/run_tests.gd`

**Interfaces:**
- Consumes: `CallRunner`, `GameState` (tests play every script headless).
- Produces:
  - `Data.read_json(path: String)`, `Data.call_script(id: String) -> Dictionary`, `Data.persona() -> Dictionary`, `Data.persona_lines(p: Dictionary) -> String`, `Data.note_text(call_data: Dictionary, id: String) -> String`
  - `CallValidator.problems(call_data: Dictionary, persona: Dictionary) -> Array` of strings; empty means playable.
  - Persona fields: `name, birthday, bank, balance, grandkid, pet, street`.

- [ ] **Step 1: Write the failing tests**

Add preloads to `tests/run_tests.gd`:

```gdscript
const Data = preload("res://scripts/core/data.gd")
const CallValidator = preload("res://scripts/core/call_validator.gd")
```

Append:

```gdscript
# --- Call scripts ---

const CALL_IDS := ["gift_cards", "tech_support", "grandkid_jail"]


func test_validator_accepts_minis() -> void:
	check(CallValidator.problems(_mini(), {}).is_empty(), "mini is valid, got %s" % [CallValidator.problems(_mini(), {})])
	check(CallValidator.problems(_mini2(), {"birthday": "x"}).is_empty(), "mini2 is valid, got %s" % [CallValidator.problems(_mini2(), {"birthday": "x"})])


func test_validator_catches_problems() -> void:
	var broken := _mini()
	broken["nodes"]["a"]["options"][0]["next"] = "nowhere"
	check(not CallValidator.problems(broken, {}).is_empty(), "a dead-end option is caught")
	var no_end := _mini()
	no_end["nodes"]["b"].erase("end")
	no_end["nodes"]["b"]["options"] = [
		{"text": "x", "type": "dumb", "trust": 1, "minutes": 1, "next": "a"},
		{"text": "y", "type": "dumb", "trust": 1, "minutes": 1, "next": "a"}]
	check(not CallValidator.problems(no_end, {}).is_empty(), "a script with no end is caught")
	var two_right := _mini2()
	two_right["nodes"]["a"]["options"][1]["value_of"] = "birthday"
	check(not CallValidator.problems(two_right, {"birthday": "x"}).is_empty(), "two correct persona answers are caught")
	check(not CallValidator.problems(_mini2(), {}).is_empty(), "a persona field the sheet lacks is caught")
	var bad_charge := _mini()
	bad_charge["charges"][0]["needs"] = ["never"]
	check(not CallValidator.problems(bad_charge, {}).is_empty(), "a charge nothing can prove is caught")
	var orphan := _mini()
	orphan["nodes"]["z"] = {"say": "lost", "end": true}
	check(not CallValidator.problems(orphan, {}).is_empty(), "an unreachable node is caught")


func test_call_scripts_valid() -> void:
	var persona: Dictionary = Data.persona()
	for id in CALL_IDS:
		var probs: Array = CallValidator.problems(Data.call_script(id), persona)
		check(probs.is_empty(), "%s has problems: %s" % [id, probs])


func _play(call_data: Dictionary, pick: Callable, flags := {}):
	var s = GameState.new()
	s.flags = flags.duplicate()
	var r = CallRunner.new()
	r.start(call_data, Data.persona(), s)
	var guard := 0
	while not r.done and guard < 60:
		r.choose(pick.call(r.node["options"]))
		guard += 1
	return r


func test_call_scripts_playable() -> void:
	for id in CALL_IDS:
		var d: Dictionary = Data.call_script(id)
		var first = _play(d, func(opts): return 0)
		check(first.done and first.result["outcome"] == "complete", "%s: always picking the first line ends with the script finishing, got %s" % [id, first.result.get("outcome")])
		var worst = _play(d, func(opts):
			var w := 0
			for i in opts.size():
				if opts[i]["trust"] < opts[w]["trust"]:
					w = i
			return w)
		check(worst.done, "%s: always picking the worst line ends" % id)
		var best = _play(d, func(opts):
			var w := 0
			for i in opts.size():
				if opts[i]["trust"] > opts[w]["trust"]:
					w = i
			return w)
		check(best.done and best.result["outcome"] == "complete", "%s: always picking the best-trust line completes" % id)


func test_rival_remembers() -> void:
	var d: Dictionary = Data.call_script("grandkid_jail")
	var s = GameState.new()
	var r = CallRunner.new()
	var said := []
	r.line.connect(func(who, text, mood): said.append(text))
	r.start(d, Data.persona(), s)
	check(not said[0].contains("toaster"), "Kevin does not mention a toaster by default")
	var s2 = GameState.new()
	s2.flags["said_toaster"] = true
	var r2 = CallRunner.new()
	var said2 := []
	r2.line.connect(func(who, text, mood): said2.append(text))
	r2.start(d, Data.persona(), s2)
	check(said2[0].contains("toaster"), "Kevin brings up the toaster when you told Greg about it")
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `tests/run.sh`
Expected: `Could not preload resource file "res://scripts/core/data.gd"`.

- [ ] **Step 3: Implement the helpers and validator**

`scripts/core/data.gd`:

```gdscript
extends RefCounted
## JSON loading and small text helpers for call data.

const PERSONA_LABELS := [
	["name", "Name"], ["birthday", "Born"], ["bank", "Bank"], ["balance", "Balance"],
	["grandkid", "Grandchild"], ["pet", "Pet"], ["street", "Street"],
]


static func read_json(path: String):
	var json := JSON.new()
	if json.parse(FileAccess.get_file_as_string(path)) != OK:
		push_error("bad json: " + path)
		return {}
	return json.data


static func call_script(id: String) -> Dictionary:
	return read_json("res://data/calls/%s.json" % id)


static func persona() -> Dictionary:
	return read_json("res://data/persona.json")


static func persona_lines(p: Dictionary) -> String:
	var out: Array = []
	for pair in PERSONA_LABELS:
		if p.has(pair[0]):
			out.append("%s: %s" % [pair[1], p[pair[0]]])
	return "\n".join(out)


## What the notepad shows for an evidence id: the tell's text, or the fact's note.
static func note_text(call_data: Dictionary, id: String) -> String:
	for n in call_data["nodes"].values():
		for t in n.get("tells", []):
			if t["id"] == id:
				return t["text"]
	return call_data.get("facts", {}).get(id, id)
```

`scripts/core/call_validator.gd`:

```gdscript
extends RefCounted
## Checks a call script's data. Returns a list of problems; empty means it is playable.

const TYPES := ["dumb", "cooperate", "probe", "persona", "leak"]


static func problems(d: Dictionary, persona: Dictionary) -> Array:
	var out: Array = []
	for key in ["id", "start", "scammer", "ring", "charges", "nodes"]:
		if not d.has(key):
			out.append("missing key " + key)
	if not out.is_empty():
		return out
	var nodes: Dictionary = d["nodes"]
	if not nodes.has(d["start"]):
		out.append("start node missing")
		return out
	_check_scammer(d["scammer"], "scammer", out)
	var ring: Dictionary = d["ring"]
	if ring.get("victims", []).is_empty() or int(ring.get("minutes_per_victim", 0)) < 1:
		out.append("ring needs victims and minutes_per_victim")
	var facts: Dictionary = d.get("facts", {})
	var obtainable := {}
	var has_end := false
	for id in nodes:
		var n: Dictionary = nodes[id]
		if not n.has("say"):
			out.append("%s: no say" % id)
		if n.has("transfer"):
			_check_scammer(n["transfer"], id + ".transfer", out)
		for t in n.get("tells", []):
			obtainable[t["id"]] = true
		if n.get("end", false):
			has_end = true
			continue
		var opts: Array = n.get("options", [])
		if opts.size() < 2 or opts.size() > 4:
			out.append("%s: needs 2-4 options" % id)
		var persona_count := 0
		var value_of := 0
		for o in opts:
			if not (o.get("type", "") in TYPES):
				out.append("%s: bad option type" % id)
			if not nodes.has(o.get("next", "")):
				out.append("%s: option goes nowhere" % id)
			if o.has("gives_fact"):
				obtainable[o["gives_fact"]] = true
				if not facts.has(o["gives_fact"]):
					out.append("%s: fact %s has no note in facts" % [id, o["gives_fact"]])
			if o.get("type", "") == "persona":
				persona_count += 1
			if o.has("value_of"):
				value_of += 1
				if not persona.has(o["value_of"]):
					out.append("%s: persona sheet has no %s" % [id, o["value_of"]])
		if persona_count > 0 and (persona_count != opts.size() or value_of != 1):
			out.append("%s: a persona question needs only persona options and exactly one value_of" % id)
	if not has_end:
		out.append("no end node")
	for c in d["charges"]:
		for need in c["needs"]:
			if not obtainable.has(need):
				out.append("charge %s needs %s, which nothing provides" % [c["id"], need])
	var seen := {d["start"]: true}
	var queue := [d["start"]]
	while not queue.is_empty():
		var cur: String = queue.pop_front()
		for o in nodes[cur].get("options", []):
			if nodes.has(o.get("next", "")) and not seen.has(o["next"]):
				seen[o["next"]] = true
				queue.append(o["next"])
	for id in nodes:
		if not seen.has(id):
			out.append("%s: unreachable" % id)
	return out


static func _check_scammer(s: Dictionary, label: String, out: Array) -> void:
	for key in ["name", "pitch", "face"]:
		if not s.has(key):
			out.append("%s: missing %s" % [label, key])
```

- [ ] **Step 4: Write the persona and the three call scripts**

`data/persona.json`:

```json
{
	"name": "Dorothy Plimpton",
	"birthday": "March 3, 1941",
	"bank": "First Bank of Pickett",
	"balance": "$412",
	"grandkid": "Tyler",
	"pet": "Mr. Whiskers",
	"street": "14 Maple Lane"
}
```

`data/calls/gift_cards.json`:

```json
{
	"id": "gift_cards",
	"title": "The Gift Card Man",
	"scammer": {"name": "Agent Dave", "pitch": 0.9, "face": {"skin": 2, "hair_style": 1, "hair_color": 0, "build": 1.0, "height": 1.0, "outfit": 0, "outfit_color": 0, "glasses": false, "tie": true, "mustache": true}},
	"ring": {"name": "Boiler Room 7", "victims": ["Mildred", "Earl", "Bo"], "minutes_per_victim": 3},
	"facts": {
		"real_name": "Supervisor's real name: Dale Pruitt",
		"callback_number": "Callback number: 555-0147 (a pizza place)"
	},
	"charges": [
		{"id": "impersonation", "text": "Pretending to be the IRS", "needs": ["gift_cards"]},
		{"id": "call_center", "text": "Running a call center", "needs": ["chatter"]},
		{"id": "script", "text": "Reading from a printed script", "needs": ["policy"]},
		{"id": "overseas", "text": "Calling from overseas, not Washington", "needs": ["clock"]},
		{"id": "identity", "text": "Real name of the caller identified", "needs": ["real_name"]},
		{"id": "callback", "text": "Real callback number identified", "needs": ["callback_number"]}
	],
	"start": "b1",
	"nodes": {
		"b1": {
			"say": "Hello ma'am, this is Agent Dave of the Internal Revenue Spaghetti Service. You are in very big trouble.",
			"tells": [{"id": "chatter", "text": "[behind him, forty people are all saying 'Hello sir, this is Microsoft']"}],
			"options": [
				{"text": "Oh my! Is this about my bird feeder?", "type": "dumb", "trust": 4, "minutes": 2, "next": "b2"},
				{"text": "Yes, hello, I'm listening.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "b2"},
				{"text": "Which office are you calling from, dear?", "type": "probe", "trust": -6, "minutes": 1, "next": "p1"}
			]
		},
		"p1": {
			"say": "Washington. Very big building. Many flags.",
			"tells": [{"id": "clock", "text": "[a clock behind him strikes three. It is three in the morning in Washington]", "on_hold": true}],
			"options": [
				{"text": "Lovely. Say hello to the flags for me.", "type": "dumb", "trust": 3, "minutes": 2, "next": "b2"},
				{"text": "And your supervisor's real name, sweetie?", "type": "probe", "trust": -8, "minutes": 1, "gives_fact": "real_name", "next": "b2"}
			]
		},
		"b2": {
			"say": "You owe four thousand dollars in back taxes. But we can fix this today. You must go to Target and buy gift cards.",
			"tells": [{"id": "gift_cards", "text": "[he is demanding payment in gift cards]"}],
			"options": [
				{"text": "Gift cards! Do they have the ones with kittens?", "type": "dumb", "trust": 4, "minutes": 2, "next": "b3"},
				{"text": "Gift cards. Yes. Of course. Whatever you say.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "b3"},
				{"text": "Since when does the government take gift cards?", "type": "probe", "trust": -7, "minutes": 1, "next": "p2"}
			]
		},
		"p2": {
			"say": "Since... this year. New policy. Very modern.",
			"tells": [{"id": "policy", "text": "[he is reading from a printed script and loses his place]"}],
			"options": [
				{"text": "Oh, modern! I do love modern.", "type": "dumb", "trust": 3, "minutes": 2, "next": "b3"},
				{"text": "Well, if it's the new policy, I suppose.", "type": "cooperate", "trust": 3, "minutes": 1, "next": "b3"}
			]
		},
		"b3": {
			"say": "First I must verify your identity. What is your date of birth?",
			"options": [
				{"text": "It's January 12, 1952.", "type": "persona", "trust": -12, "minutes": 1, "value": "January 12, 1952", "next": "b4"},
				{"text": "It's {birthday}.", "type": "persona", "trust": 6, "minutes": 1, "value_of": "birthday", "next": "b4"},
				{"text": "It's the Fourth of July, 1776.", "type": "persona", "trust": -12, "minutes": 1, "value": "July 4, 1776", "next": "b4"}
			]
		},
		"b4": {
			"say": "Good. Now: four cards, one thousand dollars each, from Target. Hurry.",
			"options": [
				{"text": "Four cards... like the kind for playing bridge?", "type": "dumb", "trust": 4, "minutes": 2, "next": "b5"},
				{"text": "Four cards, yes, at Target. I'm on my way.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "b5"},
				{"text": "What number should I call you back on, dear?", "type": "probe", "trust": -7, "minutes": 1, "gives_fact": "callback_number", "next": "b5"}
			]
		},
		"b5": {
			"say": "Now read me the numbers on the back of each card. Slowly. I have all night. I do not have all night.",
			"options": [
				{"text": "Let me find my glasses. They were here a second ago.", "type": "dumb", "trust": 3, "minutes": 3, "next": "b6"},
				{"text": "One, two, three... oh, I've lost my place.", "type": "cooperate", "trust": 5, "minutes": 2, "next": "b6"},
				{"text": "Sure, honey. I'm Chad Baker, 42 Birch Street. Want my cards too?", "type": "leak", "trust": 10, "minutes": 1, "next": "b6"}
			]
		},
		"b6": {
			"say": "Ma'am, I think we have a bad connection. I will call you back. Maybe.",
			"end": true
		}
	}
}
```

`data/calls/tech_support.json`:

```json
{
	"id": "tech_support",
	"title": "Windows Support",
	"scammer": {"name": "Greg", "pitch": 1.15, "face": {"skin": 3, "hair_style": 4, "hair_color": 1, "build": 1.1, "height": 1.0, "outfit": 2, "outfit_color": 7, "glasses": true, "tie": false, "mustache": false}},
	"ring": {"name": "Windows Support Pod 12", "victims": ["Gladys", "Norm", "Pat"], "minutes_per_victim": 3},
	"facts": {
		"company_slip": "He said 'Windows Pod 12' when asked his company, not Microsoft"
	},
	"charges": [
		{"id": "wrong_company", "text": "Not actually from Microsoft", "needs": ["company_slip"]},
		{"id": "script", "text": "Reading from a printed script", "needs": ["script_page"]},
		{"id": "farm", "text": "Calling from a farm-adjacent call center", "needs": ["rooster"]},
		{"id": "remote", "text": "Demanding remote access to your computer", "needs": ["remote"]},
		{"id": "fake_boss", "text": "Pretending his boss is a different person", "needs": ["steve_voice"]}
	],
	"start": "t1",
	"nodes": {
		"t1": {
			"say": "Hello, this is Greg from Windows Technical Support. Your computer has sent us a very urgent virus.",
			"tells": [{"id": "rooster", "text": "[a rooster crows behind him, then someone yells 'Greg, lunch!']"}],
			"options": [
				{"text": "A virus? Should I give it soup?", "type": "dumb", "trust": 4, "minutes": 2, "next": "t2"},
				{"text": "Oh dear. What do I do?", "type": "cooperate", "trust": 6, "minutes": 1, "next": "t2"},
				{"text": "How did my computer call you, exactly?", "type": "probe", "trust": -6, "minutes": 1, "next": "tp1"}
			]
		},
		"tp1": {
			"say": "It... has a special button. Very special. Only we know.",
			"tells": [{"id": "script_page", "text": "[he flips a page of a printed script, loudly]"}],
			"options": [
				{"text": "A button! Which one is the button?", "type": "dumb", "trust": 3, "minutes": 2, "next": "t2"},
				{"text": "I'll look for the special button.", "type": "cooperate", "trust": 3, "minutes": 1, "next": "t2"}
			]
		},
		"t2": {
			"say": "Please go to your computer and tell me what you see.",
			"options": [
				{"text": "I see a toaster. Wait, is this the toaster?", "type": "dumb", "trust": 5, "minutes": 2, "set_flag": "said_toaster", "next": "t3"},
				{"text": "I see a screen. It has a picture of a cat.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "t3"},
				{"text": "First, what company did you say you work for?", "type": "probe", "trust": -7, "minutes": 1, "gives_fact": "company_slip", "next": "t3"}
			]
		},
		"t3": {
			"say": "To verify you, I need to know your bank. Which bank do you use?",
			"options": [
				{"text": "The Piggy Bank, on the shelf.", "type": "persona", "trust": -12, "minutes": 1, "value": "The Piggy Bank", "next": "t4"},
				{"text": "Bank of Nowhere.", "type": "persona", "trust": -12, "minutes": 1, "value": "Bank of Nowhere", "next": "t4"},
				{"text": "{bank}, dear.", "type": "persona", "trust": 6, "minutes": 1, "value_of": "bank", "next": "t4"}
			]
		},
		"t4": {
			"say": "Now install a very small program called remote access. It will let me see your screen. And your bank.",
			"tells": [{"id": "remote", "text": "[he is asking for remote access to her computer]"}],
			"options": [
				{"text": "A remote? Like for the television?", "type": "dumb", "trust": 4, "minutes": 2, "next": "t5"},
				{"text": "Okay, I'm clicking things. Click click.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "t5"},
				{"text": "Can I speak to your boss about this, dear?", "type": "probe", "trust": -4, "minutes": 1, "next": "t4t"}
			]
		},
		"t4t": {
			"transfer": {"name": "Supervisor Steve", "pitch": 0.75, "decay": 2, "face": {"skin": 3, "hair_style": 4, "hair_color": 1, "build": 1.1, "height": 1.0, "outfit": 0, "outfit_color": 1, "glasses": true, "tie": true, "mustache": true}},
			"say": "This is Supervisor Steve. Greg is... new. What is the problem?",
			"tells": [{"id": "steve_voice", "text": "[Steve sounds exactly like Greg, only lower]"}],
			"options": [
				{"text": "Oh, Steve! Are you related to Greg?", "type": "dumb", "trust": 4, "minutes": 2, "next": "t5"},
				{"text": "Good, you sound very senior.", "type": "cooperate", "trust": 5, "minutes": 1, "next": "t5"}
			]
		},
		"t5": {
			"say": "Now tell me your bank balance, to confirm there is no virus in it.",
			"options": [
				{"text": "{balance}, I think. Or was that my age?", "type": "persona", "trust": 6, "minutes": 1, "value_of": "balance", "next": "t6"},
				{"text": "Twelve million dollars.", "type": "persona", "trust": -12, "minutes": 1, "value": "$12,000,000", "next": "t6"},
				{"text": "Eleven dollars and a button.", "type": "persona", "trust": -12, "minutes": 1, "value": "$11", "next": "t6"}
			]
		},
		"t6": {
			"say": "Last step. Please read me your full name and address so I can mail you your free antivirus.",
			"options": [
				{"text": "Free antivirus! Let me find a pen.", "type": "dumb", "trust": 4, "minutes": 3, "next": "t7"},
				{"text": "{name}, {street}.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "t7"},
				{"text": "Chad Baker, 42 Birch Street. Is that enough for the antivirus?", "type": "leak", "trust": 10, "minutes": 1, "next": "t7"}
			]
		},
		"t7": {
			"say": "I am getting a strange signal. I will call you back. Do not turn off your toaster.",
			"end": true
		}
	}
}
```

`data/calls/grandkid_jail.json`:

```json
{
	"id": "grandkid_jail",
	"title": "Kevin, Your Favorite Grandson",
	"scammer": {"name": "Kevin", "pitch": 1.3, "face": {"skin": 0, "hair_style": 2, "hair_color": 3, "build": 0.95, "height": 1.0, "outfit": 2, "outfit_color": 3, "glasses": false, "tie": false, "mustache": false, "cap": true}},
	"ring": {"name": "Grandkid Hotline", "victims": ["Walt", "Ethel", "Marge"], "minutes_per_victim": 3},
	"facts": {
		"name_fail": "He did not know Grandma's name",
		"account_name": "The wire account belongs to 'Totally Real Bail LLC'"
	},
	"charges": [
		{"id": "not_grandson", "text": "Not actually your grandson", "needs": ["name_fail"]},
		{"id": "same_ring", "text": "Same ring as the tech support call", "needs": ["same_ring"]},
		{"id": "typing", "text": "Running a call center", "needs": ["typing"]},
		{"id": "pressure", "text": "Demanding secrecy and speed", "needs": ["urgent"]},
		{"id": "wire", "text": "Demanding a wire to a stranger", "needs": ["wire"]},
		{"id": "account", "text": "Account holder identified", "needs": ["account_name"]}
	],
	"start": "g1",
	"nodes": {
		"g1": {
			"say": "Grandma? It's me, your favorite grandson. I'm in jail and you can't tell Mom.",
			"variants": [{"if_flag": "said_toaster", "say": "Grandma? It's me. Greg told me about the toaster. Anyway, I'm in jail and you can't tell Mom."}],
			"tells": [
				{"id": "typing", "text": "[you hear keyboards clacking and someone laughing]"},
				{"id": "same_ring", "text": "[he mentions Greg. This is the same ring as the toaster call]", "if_flag": "said_toaster"}
			],
			"options": [
				{"text": "Kevin! Is that you? Which grandson are you?", "type": "dumb", "trust": 4, "minutes": 2, "next": "g2"},
				{"text": "Oh, my poor baby! Of course I won't tell your mother.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "g2"},
				{"text": "What's my name, dear?", "type": "probe", "trust": -7, "minutes": 1, "next": "gp1"}
			]
		},
		"gp1": {
			"say": "...Grandma. Is this a test? It's, uh. Nana.",
			"options": [
				{"text": "Nana! Yes, that's me.", "type": "dumb", "trust": 3, "minutes": 2, "next": "g2"},
				{"text": "Nana is not my name, young man.", "type": "probe", "trust": -8, "minutes": 1, "gives_fact": "name_fail", "next": "g2"}
			]
		},
		"g2": {
			"say": "I was arrested for... very unfair parking. I need bail money. Right now.",
			"tells": [{"id": "urgent", "text": "[he keeps saying 'right now' and 'don't tell anyone']"}],
			"options": [
				{"text": "Parking! In my day we walked everywhere.", "type": "dumb", "trust": 5, "minutes": 3, "next": "g3"},
				{"text": "How much do you need, sweetheart?", "type": "cooperate", "trust": 6, "minutes": 1, "next": "g3"},
				{"text": "Put your lawyer on, I'd like a word.", "type": "probe", "trust": -6, "minutes": 1, "next": "g3"}
			]
		},
		"g3": {
			"say": "Grandma, say my name so the guard knows you're family.",
			"options": [
				{"text": "Brad, you silly goose.", "type": "persona", "trust": -12, "minutes": 1, "value": "Brad", "next": "g4"},
				{"text": "Napoleon.", "type": "persona", "trust": -12, "minutes": 1, "value": "Napoleon", "next": "g4"},
				{"text": "{grandkid}, my darling.", "type": "persona", "trust": 6, "minutes": 1, "value_of": "grandkid", "next": "g4"}
			]
		},
		"g4": {
			"say": "They only take wire transfers. I need you to send eight thousand dollars to this account.",
			"tells": [{"id": "wire", "text": "[he wants a wire transfer to a stranger's account]"}],
			"options": [
				{"text": "Wire? Like for the fence? My fence is fine.", "type": "dumb", "trust": 4, "minutes": 3, "next": "g5"},
				{"text": "Tell me the account number, sweetie.", "type": "cooperate", "trust": 6, "minutes": 1, "next": "g5"},
				{"text": "Whose account is it, Kevin? Say the full name.", "type": "probe", "trust": -8, "minutes": 1, "gives_fact": "account_name", "next": "g5"}
			]
		},
		"g5": {
			"say": "Grandma, they are making me hang up. Send it now. Please. I love you.",
			"options": [
				{"text": "I need to find my checkbook. It's in the oven.", "type": "dumb", "trust": 4, "minutes": 3, "next": "g6"},
				{"text": "I'm at the bank now. They're asking questions.", "type": "cooperate", "trust": 5, "minutes": 2, "next": "g6"},
				{"text": "I'm Chad Baker at 42 Birch Street, not your grandma. Surprise!", "type": "leak", "trust": 10, "minutes": 1, "next": "g6"}
			]
		},
		"g6": {
			"say": "...You know what? I have to go. Bye, Grandma.",
			"end": true
		}
	}
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `tests/run.sh`
Expected: `28 passed, 0 failed`. If `test_call_scripts_valid` or `test_call_scripts_playable` fails, the message names the script and the problem: fix the JSON (never the validator) until clean.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "Add call validator, persona and the three call scripts" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Engine plumbing: Game and Sfx autoloads, copied Judge pieces

**Files:**
- Create: `scripts/game.gd`, `scripts/ui_kit.gd`
- Copy: `scripts/room_kit.gd`, `scripts/character.gd`, `scripts/character_builder.gd`, `scripts/seated_camera.gd`, `scripts/sfx.gd` (from `~/JudgeGame/scripts/`)
- Modify: `scripts/sfx.gd` (add clip chime and hold music), `project.godot` (autoloads and main scene)

**Interfaces:**
- Consumes: `GameState`.
- Produces:
  - Autoload `Game`: vars `state, save_path, mouse_sensitivity, volume, fullscreen, quality, autoplay`; constants `LAB := "res://scenes/lab.tscn"`, `MENU := "res://scenes/main_menu.tscn"`; methods `new_game()`, `has_save()->bool`, `continue_game()->bool`, `autosave()`, `goto(scene_path)` (awaitable fade), `load_settings()`, `save_settings()`, `apply_settings()`.
  - Autoload `Sfx`: `play(sound, pitch=1.0, volume_db=0.0)`, `blip(pitch)`, `ambience(on)`, `hold_music(on)`; sound names: `gavel, stamp, paper, phone, door, click, blip, clip`.
  - `UI` kit (`res://scripts/ui_kit.gd`): constants `PAPER, INK, AMBER, BRASS, GOOD, BAD`; static `serif()`, `typed()`, `box(color, radius, margin=0, border=Color.TRANSPARENT, border_width=0)`, `label(text, size, color, font)`, `button(text, size=28)`.
  - Copied files keep their Judge APIs: `Kit.box/cylinder/sphere/capsule/mesh/clickable/pendant_light/environment_1987/film_overlay/flat/wood/plaster/glow/brass`, `Builder.build(appearance) -> Node3D` (characters face +Z), `Character` (`set_mood`, `set_tell`, `talk`, `pitch`, `look_point`), `SeatedCamera` (`set_look`, `clicked(id, target)`, `hovered(prompt)`, `lean_to`, `lean_back`, `look_enabled`).

- [ ] **Step 1: Copy the Judge files**

```bash
cd "/Users/alexanderbogert/Please Hold"
for f in room_kit character character_builder seated_camera sfx; do cp ~/JudgeGame/scripts/$f.gd scripts/$f.gd; done
ls scripts
```
Expected: the five `.gd` files are listed.

- [ ] **Step 2: Extend Sfx with the clip chime and hold music**

In `scripts/sfx.gd`, replace the line `"blip": _wav(_blip()),` with:

```gdscript
		"blip": _wav(_blip()), "clip": _wav(_chime()),
```

Add a member next to `var _ambience: AudioStreamPlayer`:

```gdscript
var _hold: AudioStreamPlayer
```

In `_ready()`, directly after the line `add_child(_ambience)`, add:

```gdscript
	_hold = AudioStreamPlayer.new()
	var tune := _wav(_hold_music())
	tune.loop_mode = AudioStreamWAV.LOOP_FORWARD
	tune.loop_end = RATE * 4
	_hold.stream = tune
	_hold.volume_db = -14.0
	add_child(_hold)
```

Add these functions after `ambience()`:

```gdscript
func hold_music(on: bool) -> void:
	if on and not _hold.playing:
		_hold.play()
	elif not on:
		_hold.stop()
```

and at the end of the file:

```gdscript
func _chime() -> PackedFloat32Array:
	var b := _buf(0.5)
	for i in b.size():
		var t := float(i) / RATE
		var second := sin(TAU * 1320.0 * (t - 0.12)) if t > 0.12 else 0.0
		b[i] = (sin(TAU * 880.0 * t) + second) * exp(-t * 7.0) * 0.4
	return b


## Eight-note elevator arpeggio, exactly four seconds so the loop is seamless.
func _hold_music() -> PackedFloat32Array:
	var notes := [261.6, 329.6, 392.0, 329.6, 293.7, 349.2, 440.0, 349.2]
	var b := _buf(4.0)
	for i in b.size():
		var t := float(i) / RATE
		var n := int(t * 2.0) % notes.size()
		var local := fmod(t * 2.0, 1.0)
		b[i] = sin(TAU * notes[n] * t) * exp(-local * 3.0) * 0.35
	return b
```

- [ ] **Step 3: Write the UI kit**

`scripts/ui_kit.gd`:

```gdscript
extends RefCounted
## Shared look for the 2D interface: fonts, panel boxes, labels, buttons.

const PAPER := Color(0.93, 0.89, 0.78)
const INK := Color(0.13, 0.11, 0.09)
const AMBER := Color(0.95, 0.72, 0.35)
const BRASS := Color(0.62, 0.46, 0.2, 0.8)
const GOOD := Color(0.55, 0.85, 0.5)
const BAD := Color(0.9, 0.4, 0.35)

static var _serif: SystemFont
static var _typed: SystemFont


static func serif() -> SystemFont:
	if _serif == null:
		_serif = SystemFont.new()
		_serif.font_names = PackedStringArray(["Georgia", "Times New Roman", "serif"])
	return _serif


static func typed() -> SystemFont:
	if _typed == null:
		_typed = SystemFont.new()
		_typed.font_names = PackedStringArray(["American Typewriter", "Courier New", "monospace"])
	return _typed


static func box(color: Color, radius: int, margin := 0, border := Color.TRANSPARENT, border_width := 0) -> StyleBoxFlat:
	var s := StyleBoxFlat.new()
	s.bg_color = color
	s.set_corner_radius_all(radius)
	s.set_content_margin_all(margin)
	if border_width > 0:
		s.border_color = border
		s.set_border_width_all(border_width)
	return s


static func label(text: String, size: int, color: Color, font: Font) -> Label:
	var l := Label.new()
	l.text = text
	l.add_theme_font_override("font", font)
	l.add_theme_font_size_override("font_size", size)
	l.add_theme_color_override("font_color", color)
	return l


static func button(text: String, size := 28) -> Button:
	var b := Button.new()
	b.text = text
	b.add_theme_font_override("font", serif())
	b.add_theme_font_size_override("font_size", size)
	b.add_theme_stylebox_override("normal", box(Color(0.1, 0.08, 0.06, 0.94), 5, 14, BRASS, 1))
	b.add_theme_stylebox_override("hover", box(Color(0.35, 0.22, 0.1, 0.97), 5, 14, AMBER, 2))
	b.add_theme_stylebox_override("pressed", box(Color(0.5, 0.32, 0.14), 5, 14, AMBER, 2))
	b.add_theme_stylebox_override("focus", StyleBoxEmpty.new())
	b.add_theme_stylebox_override("disabled", box(Color(0.1, 0.08, 0.06, 0.3), 5, 14))
	b.add_theme_color_override("font_color", PAPER)
	b.pressed.connect(func(): Sfx.play("click", 1.0, -8.0))
	return b
```

- [ ] **Step 4: Write the Game autoload**

`scripts/game.gd`:

```gdscript
extends Node
## Autoload. Owns the current GameState, settings, saving and scene changes.

const GameState = preload("res://scripts/core/game_state.gd")
const Kit = preload("res://scripts/room_kit.gd")
const LAB := "res://scenes/lab.tscn"
const MENU := "res://scenes/main_menu.tscn"
## 3D render scale per quality level; the interface always draws at full resolution.
const RENDER_SCALE := [0.5, 0.67, 1.0]
const SETTINGS_PATH := "user://settings.cfg"

var save_path := "user://save.json"
var state = GameState.new()
var mouse_sensitivity := 1.0
var volume := 0.8
var fullscreen := false
var quality := 1   # 0 low, 1 medium, 2 high
var autoplay := false   # --autoplay: skip all waiting, for end-to-end checks

var _fade: ColorRect


func _ready() -> void:
	process_mode = Node.PROCESS_MODE_ALWAYS
	var layer := CanvasLayer.new()
	layer.layer = 100
	add_child(layer)
	_fade = ColorRect.new()
	_fade.color = Color(0, 0, 0, 0)
	_fade.set_anchors_preset(Control.PRESET_FULL_RECT)
	_fade.mouse_filter = Control.MOUSE_FILTER_IGNORE
	layer.add_child(_fade)
	load_settings()
	if "--autoplay" in OS.get_cmdline_user_args():
		autoplay = true
		save_path = "user://autoplay_save.json"   # never touch the player's save
		Engine.time_scale = 12.0


func new_game() -> void:
	state = GameState.new()
	autosave()


## True only for a save that loads; a missing or corrupt file counts as no save.
func has_save() -> bool:
	return GameState.load_from(save_path) != null


func continue_game() -> bool:
	var loaded = GameState.load_from(save_path)
	if loaded == null:
		return false
	state = loaded
	return true


func autosave() -> void:
	state.save(save_path)


func goto(scene_path: String) -> void:
	var t := create_tween()
	t.tween_property(_fade, "color:a", 1.0, 0.5)
	await t.finished
	get_tree().paused = false
	get_tree().change_scene_to_file(scene_path)
	t = create_tween()
	t.tween_interval(0.25)
	t.tween_property(_fade, "color:a", 0.0, 0.7)


func load_settings() -> void:
	var cfg := ConfigFile.new()
	if cfg.load(SETTINGS_PATH) == OK:
		mouse_sensitivity = cfg.get_value("game", "mouse_sensitivity", 1.0)
		volume = cfg.get_value("game", "volume", 0.8)
		fullscreen = cfg.get_value("game", "fullscreen", false)
		quality = clampi(cfg.get_value("game", "quality", 1), 0, 2)
	apply_settings()


func save_settings() -> void:
	var cfg := ConfigFile.new()
	cfg.set_value("game", "mouse_sensitivity", mouse_sensitivity)
	cfg.set_value("game", "volume", volume)
	cfg.set_value("game", "fullscreen", fullscreen)
	cfg.set_value("game", "quality", quality)
	cfg.save(SETTINGS_PATH)
	apply_settings()


func apply_settings() -> void:
	AudioServer.set_bus_volume_db(0, linear_to_db(maxf(volume, 0.0001)))
	Kit.quality = quality
	var view := get_tree().root
	view.scaling_3d_mode = Viewport.SCALING_3D_MODE_FSR if quality < 2 else Viewport.SCALING_3D_MODE_BILINEAR
	view.scaling_3d_scale = RENDER_SCALE[quality]
	view.msaa_3d = Viewport.MSAA_2X if quality >= 2 else Viewport.MSAA_DISABLED
	view.screen_space_aa = Viewport.SCREEN_SPACE_AA_FXAA
	if DisplayServer.get_name() != "headless":
		var mode := DisplayServer.WINDOW_MODE_FULLSCREEN if fullscreen else DisplayServer.WINDOW_MODE_WINDOWED
		if DisplayServer.window_get_mode() != mode:
			DisplayServer.window_set_mode(mode)
```

- [ ] **Step 5: Register the autoloads and main scene**

Create the placeholder scene `scenes/main_menu.tscn` so the main scene resolves (Task 10 gives it a script):

```
[gd_scene format=3]

[node name="Root" type="Node3D"]
```

In `project.godot`, replace the `[application]` section and add `[autoload]` after it:

```ini
[application]

config/name="Please Hold"
run/main_scene="res://scenes/main_menu.tscn"
config/features=PackedStringArray("4.7", "Forward Plus")

[autoload]

Game="*res://scripts/game.gd"
Sfx="*res://scripts/sfx.gd"
```

- [ ] **Step 6: Verify the project boots with no script errors, and the rule tests still pass**

Run:
```bash
godot --headless --path . --quit 2>&1 | grep -E "ERROR|SCRIPT" ; echo "exit: done"
tests/run.sh
```
Expected: no `ERROR`/`SCRIPT` lines from the first command (only `exit: done`); `28 passed, 0 failed` from the second. If `character_builder.gd` or `seated_camera.gd` report a missing identifier, fix by checking that `Game` and `Sfx` are registered in `project.godot`.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "Add Game and Sfx autoloads, UI kit and copied Judge room/character/camera code" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 7: The basement lab room and screenshot harness

**Files:**
- Create: `scripts/lab_room.gd`, `scenes/lab.tscn`, `tests/shot.gd`, `tests/shot.tscn`, temporary minimal `scripts/lab.gd`

**Interfaces:**
- Consumes: `Kit` (room_kit.gd).
- Produces: `LabRoom` (`res://scripts/lab_room.gd`): constants `TOP := 0.78`, `SEAT := Vector3(0, 1.3, -0.3)`, `FRONT_Z := -2.3`, `BACK_Z := 3.2`, `HALF_W := 3.6`, `H := 2.7`; `static environment() -> Environment`; `static build(root: Node3D) -> Dictionary` returning `{"screen": MeshInstance3D (CRT screen, has a StandardMaterial3D material_override), "caller_id": Label3D, "clock": Label3D, "cards": Array of 9 Label3D (corkboard victim cards, three rows of three)}`. Clickable ids: `headset`, `persona`, `crt`, `outbox`, `corkboard`.
- Produces: `tests/shot.tscn` run as `godot --path . res://tests/shot.tscn -- <scene> <name> <frames> [extra user args]` writing `shots/<name>.png`. This needs a real window (do not pass `--headless`).

- [ ] **Step 1: Write the room**

`scripts/lab_room.gd`:

```gdscript
extends RefCounted
## Builds the basement lab. The player sits at SEAT looking down -Z at the desk.

const Kit = preload("res://scripts/room_kit.gd")

const TOP := 0.78       # desk surface height
const SEAT := Vector3(0, 1.3, -0.3)
const FRONT_Z := -2.3   # the wall behind the desk
const BACK_Z := 3.2
const HALF_W := 3.6
const H := 2.7


## Dim, cool, a little foggy. The warm 1987 grain from the Judge game still applies.
static func environment() -> Environment:
	var e := Kit.environment_1987(0.02)
	e.background_color = Color(0.02, 0.02, 0.03)
	e.ambient_light_color = Color(0.35, 0.4, 0.45)
	e.ambient_light_energy = 0.5
	e.volumetric_fog_albedo = Color(0.6, 0.65, 0.7)
	return e


static func build(root: Node3D) -> Dictionary:
	var refs := {"cards": []}
	_shell(root)
	_desk(root, refs)
	_board(root, refs)
	_dressing(root)
	_lights(root)
	return refs


static func _flat_label(parent: Node, text: String, pos: Vector3, pixel: float, color: Color, tilt := -75.0) -> Label3D:
	var l := Label3D.new()
	l.text = text
	l.font_size = 48
	l.pixel_size = pixel
	l.modulate = color
	l.outline_size = 0
	l.position = pos
	l.rotation_degrees = Vector3(tilt, 0, 0)
	parent.add_child(l)
	return l


static func _shell(root: Node3D) -> void:
	var wall := Kit.plaster(Color(0.5, 0.52, 0.48))
	var depth := BACK_Z - FRONT_Z
	var mid := (BACK_Z + FRONT_Z) * 0.5
	Kit.box(root, Vector3(HALF_W * 2, 0.2, depth), Vector3(0, -0.1, mid), Kit.flat(Color(0.32, 0.31, 0.3), 0.95))
	Kit.box(root, Vector3(HALF_W * 2, 0.2, depth), Vector3(0, H + 0.1, mid), Kit.flat(Color(0.22, 0.21, 0.2), 0.95))
	Kit.box(root, Vector3(HALF_W * 2, H, 0.3), Vector3(0, H * 0.5, FRONT_Z - 0.15), wall)
	Kit.box(root, Vector3(HALF_W * 2, H, 0.3), Vector3(0, H * 0.5, BACK_Z + 0.15), wall)
	Kit.box(root, Vector3(0.3, H, depth), Vector3(-HALF_W - 0.15, H * 0.5, mid), wall)
	Kit.box(root, Vector3(0.3, H, depth), Vector3(HALF_W + 0.15, H * 0.5, mid), wall)
	for x in [-2.4, -1.6, 2.2]:
		Kit.cylinder(root, 0.06, depth, Vector3(x, H - 0.12, mid), Kit.flat(Color(0.5, 0.5, 0.52), 0.4, 0.8)).rotation.x = PI / 2


static func _desk(root: Node3D, refs: Dictionary) -> void:
	var wood := Kit.wood(Color(0.36, 0.2, 0.1), false)
	var cz := FRONT_Z + 0.7   # desk spans z -2.2 .. -1.0
	Kit.box(root, Vector3(2.9, 0.06, 1.2), Vector3(0, TOP - 0.03, cz), wood)
	for lx in [-1.35, 1.35]:
		for lz in [FRONT_Z + 0.2, FRONT_Z + 1.15]:
			Kit.box(root, Vector3(0.07, TOP - 0.06, 0.07), Vector3(lx, (TOP - 0.06) * 0.5, lz), wood)

	# CRT: the scammer's face is projected on its screen by caller_card.gd
	var plastic := Kit.flat(Color(0.72, 0.69, 0.6), 0.6)
	Kit.box(root, Vector3(0.56, 0.46, 0.48), Vector3(-0.3, TOP + 0.23, -1.95), plastic)
	Kit.box(root, Vector3(0.3, 0.1, 0.3), Vector3(-0.3, TOP + 0.05, -1.95), plastic)
	var quad := QuadMesh.new()
	quad.size = Vector2(0.4, 0.34)
	var screen := Kit.mesh(root, quad, Vector3(-0.3, TOP + 0.25, -1.709), null)
	var sm := StandardMaterial3D.new()
	sm.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	sm.albedo_color = Color(0.85, 1.0, 0.9)
	screen.material_override = sm
	refs["screen"] = screen
	Kit.clickable(root, Vector3(0.6, 0.5, 0.5), Vector3(-0.3, TOP + 0.25, -1.95), "crt", "Check the channel")

	# caller-ID box
	Kit.box(root, Vector3(0.26, 0.07, 0.18), Vector3(0.2, TOP + 0.035, -1.45), Kit.flat(Color(0.12, 0.12, 0.13), 0.5))
	refs["caller_id"] = _flat_label(root, "--", Vector3(0.2, TOP + 0.075, -1.45), 0.0013, Color(0.5, 1.0, 0.5))

	# desk clock
	Kit.box(root, Vector3(0.24, 0.1, 0.1), Vector3(1.15, TOP + 0.05, -1.9), Kit.flat(Color(0.1, 0.1, 0.1), 0.5))
	refs["clock"] = _flat_label(root, "7:00 PM", Vector3(1.15, TOP + 0.08, -1.845), 0.0011, Color(1.0, 0.35, 0.2), -20.0)

	# headset
	var steel := Kit.flat(Color(0.15, 0.15, 0.17), 0.4, 0.6)
	Kit.cylinder(root, 0.055, 0.04, Vector3(-1.1, TOP + 0.04, -1.2), steel).rotation.z = PI / 2
	Kit.cylinder(root, 0.055, 0.04, Vector3(-0.9, TOP + 0.04, -1.2), steel).rotation.z = PI / 2
	Kit.box(root, Vector3(0.22, 0.015, 0.03), Vector3(-1.0, TOP + 0.1, -1.2), steel)
	Kit.clickable(root, Vector3(0.34, 0.2, 0.2), Vector3(-1.0, TOP + 0.08, -1.2), "headset", "Answer the phone")

	# persona sheet
	Kit.box(root, Vector3(0.22, 0.004, 0.3), Vector3(-0.95, TOP + 0.002, -1.65), Kit.flat(Color(0.93, 0.89, 0.78), 0.9))
	_flat_label(root, "PERSONA", Vector3(-0.95, TOP + 0.006, -1.65), 0.0009, Color(0.15, 0.1, 0.08))
	Kit.clickable(root, Vector3(0.26, 0.05, 0.34), Vector3(-0.95, TOP + 0.03, -1.65), "persona", "Read the persona sheet")

	# outbox tray
	var tray := Kit.flat(Color(0.2, 0.2, 0.22), 0.5)
	Kit.box(root, Vector3(0.34, 0.03, 0.26), Vector3(1.0, TOP + 0.015, -1.4), tray)
	Kit.box(root, Vector3(0.34, 0.06, 0.02), Vector3(1.0, TOP + 0.03, -1.27), tray)
	Kit.box(root, Vector3(0.26, 0.012, 0.2), Vector3(1.0, TOP + 0.036, -1.4), Kit.flat(Color(0.93, 0.89, 0.78), 0.9))
	Kit.clickable(root, Vector3(0.38, 0.12, 0.3), Vector3(1.0, TOP + 0.05, -1.4), "outbox", "File your evidence")

	# coffee mug
	Kit.cylinder(root, 0.04, 0.09, Vector3(0.55, TOP + 0.045, -1.15), Kit.flat(Color(0.8, 0.78, 0.7), 0.5))


## Corkboard on the wall behind the desk: one row of three index cards per scam ring.
static func _board(root: Node3D, refs: Dictionary) -> void:
	var z := FRONT_Z + 0.03
	Kit.box(root, Vector3(2.4, 1.1, 0.05), Vector3(0, 1.85, z), Kit.flat(Color(0.55, 0.38, 0.2), 1.0))
	Kit.clickable(root, Vector3(2.4, 1.1, 0.2), Vector3(0, 1.85, z + 0.05), "corkboard", "Look at the board")
	var paper := Kit.flat(Color(0.93, 0.9, 0.82), 0.9)
	for row in 3:
		for col in 3:
			var p := Vector3(-0.75 + col * 0.75, 2.17 - row * 0.32, z + 0.03)
			Kit.box(root, Vector3(0.62, 0.22, 0.008), p, paper)
			var l := Label3D.new()
			l.text = ""
			l.font_size = 44
			l.pixel_size = 0.0016
			l.modulate = Color(0.15, 0.1, 0.08)
			l.outline_size = 0
			l.position = p + Vector3(0, 0, 0.007)
			root.add_child(l)
			refs["cards"].append(l)
	var red := Kit.flat(Color(0.7, 0.08, 0.06), 0.6)
	for s in [[Vector3(0, 1.95, z + 0.05), 0.35, 1.9], [Vector3(0.1, 1.7, z + 0.05), -0.2, 1.6], [Vector3(-0.2, 2.0, z + 0.05), 0.9, 1.0]]:
		var string := Kit.box(root, Vector3(s[2], 0.006, 0.006), s[0], red)
		string.rotation.z = s[1]


static func _dressing(root: Node3D) -> void:
	# shelf of gear on the right wall
	var metal := Kit.flat(Color(0.2, 0.2, 0.22), 0.5, 0.4)
	Kit.box(root, Vector3(0.4, 0.04, 2.0), Vector3(HALF_W - 0.2, 1.4, -0.6), Kit.wood(Color(0.3, 0.17, 0.09)))
	for i in 5:
		var h := 0.12 + (i % 3) * 0.06
		Kit.box(root, Vector3(0.3, h, 0.25), Vector3(HALF_W - 0.2, 1.42 + h * 0.5, -1.4 + i * 0.4), metal)
	# a basement window high on the left wall, dusk blue
	Kit.box(root, Vector3(0.04, 0.4, 1.0), Vector3(-HALF_W + 0.02, 2.1, -0.4), Kit.glow(Color(0.35, 0.5, 0.8), 1.4))
	# cables on the floor
	for i in 3:
		Kit.box(root, Vector3(0.03, 0.03, 1.4), Vector3(-0.6 + i * 0.5, 0.02, -0.7), Kit.flat(Color(0.05, 0.05, 0.05), 0.7)).rotation.y = 0.2 * (i - 1)


static func _lights(root: Node3D) -> void:
	Kit.pendant_light(root, Vector3(0.2, 2.0, -1.2), 0.7, 2.4, true)
	var crt := OmniLight3D.new()
	crt.position = Vector3(-0.3, TOP + 0.3, -1.4)
	crt.light_color = Color(0.5, 0.9, 0.8)
	crt.light_energy = 0.8
	crt.omni_range = 3.0
	root.add_child(crt)
	var window := OmniLight3D.new()
	window.position = Vector3(-HALF_W + 0.6, 2.1, -0.4)
	window.light_color = Color(0.45, 0.6, 0.9)
	window.light_energy = 1.0
	window.omni_range = 5.0
	root.add_child(window)
```

- [ ] **Step 2: Write a temporary lab controller, the lab scene, and the screenshot harness**

`scripts/lab.gd` (temporary: Task 9 replaces it completely):

```gdscript
extends Node3D

const Kit = preload("res://scripts/room_kit.gd")
const LabRoom = preload("res://scripts/lab_room.gd")
const SeatedCamera = preload("res://scripts/seated_camera.gd")

var cam: Camera3D


func _ready() -> void:
	LabRoom.build(self)
	var env := WorldEnvironment.new()
	env.environment = LabRoom.environment()
	add_child(env)
	Kit.film_overlay(self)
	cam = SeatedCamera.new()
	cam.position = LabRoom.SEAT
	add_child(cam)
	cam.make_current()
```

`scenes/lab.tscn`:

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/lab.gd" id="1"]

[node name="Lab" type="Node3D"]
script = ExtResource("1")
```

`tests/shot.tscn`:

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://tests/shot.gd" id="1"]

[node name="Shot" type="Node"]
script = ExtResource("1")
```

`tests/shot.gd`:

```gdscript
extends Node
## Screenshot a scene: godot --path . res://tests/shot.tscn -- <scene> <name> <frames> [user args for the scene]
## Needs a real window. Writes shots/<name>.png


func _ready() -> void:
	var a := OS.get_cmdline_user_args()
	Game.save_path = "user://shot_save.json"   # scenes autosave: never let a screenshot touch the player's save
	var scene: String = a[0]
	var shot: String = a[1] if a.size() > 1 else scene
	var frames := int(a[2]) if a.size() > 2 else 60
	var node: Node = load("res://scenes/%s.tscn" % scene).instantiate()
	add_child(node)
	for i in frames:
		await get_tree().process_frame
	DirAccess.make_dir_recursive_absolute(ProjectSettings.globalize_path("res://shots"))
	get_viewport().get_texture().get_image().save_png("res://shots/%s.png" % shot)
	get_tree().quit()
```

- [ ] **Step 3: Take a screenshot and inspect it**

Run:
```bash
godot --path . res://tests/shot.tscn -- lab lab_idle 90
```
Then view `shots/lab_idle.png` with the Read tool.
Expected: a dim basement seen from the chair: a wooden desk in the lower half, a beige CRT (dark screen) left of centre, a pendant bulb glowing, a corkboard with red string on the wall above the desk, cables on the floor, cool window light on the left. No crash, no pink missing-material surfaces.

Iterate on `lab_room.gd` until it reads clearly as a lab: if it is too dark raise `e.ambient_light_energy` in `environment()`; if the desk objects are clipped by the bottom of the frame, lower `SEAT.y` or raise the objects' positions; the camera can look down with the mouse so a slightly cropped desk edge is acceptable.

- [ ] **Step 4: Confirm headless boot and tests are clean, then commit**

```bash
godot --headless --path . --quit 2>&1 | grep -E "ERROR|SCRIPT"; tests/run.sh
git add -A
git commit -m "Build the basement lab room and screenshot harness" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```
Expected: no error lines, `28 passed, 0 failed`.

---

### Task 8: HUD and the CRT caller card

**Files:**
- Create: `scripts/hud.gd`, `scripts/caller_card.gd`

**Interfaces:**
- Consumes: `UI` (ui_kit.gd), `Data` (`persona_lines`), `Game.autoplay`, `Sfx.blip`, `Character` API via `Builder`.
- Produces `Hud` (CanvasLayer):
  - Signals: `choice_made(index: int)`, `tell_clicked(id: String)`, `hold_pressed`, `clip_pressed`, `hangup_pressed`.
  - Var `cam: Camera3D` (set by the scene; modals turn its mouse look off and restore it).
  - Methods: `set_prompt(text)`, `set_hint(text)`, `toast(text)`, `set_clock(text)`, `call_ui(on: bool)`, `set_persona(p: Dictionary)`, `set_notes(lines: Array)`, `set_trust(trust: int, mood: String, minutes: int)`, `show_line(who: String, text: String, pitch: float)`, `show_options(options: Array)`, `show_tells(tells: Array, noted: Array)`, and the awaitables `info(title, body, button="Continue")`, `choose_one(title, body, buttons: Array) -> int`, `pick_many(title, body, labels: Array, button="Done") -> Array` (indices ticked). All three return at once when `Game.autoplay` is on (`choose_one` returns 0, `pick_many` returns `[0]`).
- Produces `CallerCard` (SubViewport): `show_scammer(s: Dictionary)`, `set_mood(m: String)`, `talk(seconds: float)`, `clear()`.

- [ ] **Step 1: Write the HUD**

`scripts/hud.gd`:

```gdscript
extends CanvasLayer
## All 2D interface. Every awaitable returns at once when Game.autoplay is on.

const UI = preload("res://scripts/ui_kit.gd")
const Data = preload("res://scripts/core/data.gd")

signal choice_made(index: int)
signal tell_clicked(id: String)
signal hold_pressed
signal clip_pressed
signal hangup_pressed
signal _modal_done(value)

var cam: Camera3D

var _dot: Panel
var _prompt: Label
var _hint: Label
var _toast: Label
var _clock: Label
var _call: Control
var _trust: ProgressBar
var _mood: Label
var _minutes: Label
var _who: Label
var _text: Label
var _choices: VBoxContainer
var _tells: VBoxContainer
var _sheet: Label
var _notes: Label
var _modal: Control
var _was_looking := false
var _typing := false
var _shown := 0.0
var _blip_at := 0
var _pitch := 1.0
var _pending: Array = []


func _ready() -> void:
	layer = 10
	process_mode = Node.PROCESS_MODE_ALWAYS
	_dot = Panel.new()
	_dot.add_theme_stylebox_override("panel", UI.box(Color(1, 1, 1, 0.75), 4))
	_dot.size = Vector2(8, 8)
	_dot.set_anchors_and_offsets_preset(Control.PRESET_CENTER)
	_dot.mouse_filter = Control.MOUSE_FILTER_IGNORE
	add_child(_dot)

	_prompt = UI.label("", 26, Color.WHITE, UI.serif())
	_prompt.set_anchors_and_offsets_preset(Control.PRESET_CENTER)
	_prompt.position.y += 34
	_prompt.grow_horizontal = Control.GROW_DIRECTION_BOTH
	_prompt.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	_prompt.add_theme_color_override("font_outline_color", Color.BLACK)
	_prompt.add_theme_constant_override("outline_size", 8)
	_prompt.mouse_filter = Control.MOUSE_FILTER_IGNORE
	add_child(_prompt)

	_hint = UI.label("", 30, UI.AMBER, UI.typed())
	_hint.set_anchors_and_offsets_preset(Control.PRESET_CENTER_TOP)
	_hint.position.y += 28
	_hint.grow_horizontal = Control.GROW_DIRECTION_BOTH
	_hint.add_theme_color_override("font_outline_color", Color.BLACK)
	_hint.add_theme_constant_override("outline_size", 8)
	_hint.mouse_filter = Control.MOUSE_FILTER_IGNORE
	add_child(_hint)

	_toast = UI.label("", 44, UI.PAPER, UI.typed())
	_toast.set_anchors_and_offsets_preset(Control.PRESET_CENTER)
	_toast.position.y -= 180
	_toast.grow_horizontal = Control.GROW_DIRECTION_BOTH
	_toast.add_theme_color_override("font_outline_color", Color.BLACK)
	_toast.add_theme_constant_override("outline_size", 10)
	_toast.modulate.a = 0.0
	_toast.mouse_filter = Control.MOUSE_FILTER_IGNORE
	add_child(_toast)

	var plate := _plate(self, Control.PRESET_TOP_RIGHT, Vector2(-30, 24))
	plate.grow_horizontal = Control.GROW_DIRECTION_BEGIN
	_clock = UI.label("", 28, UI.PAPER, UI.typed())
	plate.add_child(_clock)

	_call = Control.new()
	_call.set_anchors_preset(Control.PRESET_FULL_RECT)
	_call.mouse_filter = Control.MOUSE_FILTER_IGNORE
	_call.visible = false
	add_child(_call)
	_build_call()


func _plate(parent: Node, preset: int, offset: Vector2) -> PanelContainer:
	var p := PanelContainer.new()
	p.add_theme_stylebox_override("panel", UI.box(Color(0.06, 0.05, 0.04, 0.84), 5, 14, UI.BRASS, 1))
	p.set_anchors_and_offsets_preset(preset)
	p.position += offset
	p.mouse_filter = Control.MOUSE_FILTER_IGNORE
	parent.add_child(p)
	return p


func _build_call() -> void:
	# trust, mood and minutes, top left
	var trust_plate := _plate(_call, Control.PRESET_TOP_LEFT, Vector2(30, 24))
	var tv := VBoxContainer.new()
	trust_plate.add_child(tv)
	tv.add_child(UI.label("TRUST", 22, UI.AMBER, UI.typed()))
	_trust = ProgressBar.new()
	_trust.min_value = 0
	_trust.max_value = 100
	_trust.value = 50
	_trust.show_percentage = false
	_trust.custom_minimum_size = Vector2(300, 22)
	tv.add_child(_trust)
	_mood = UI.label("", 24, UI.GOOD, UI.typed())
	_minutes = UI.label("", 24, UI.PAPER, UI.typed())
	tv.add_child(_mood)
	tv.add_child(_minutes)

	# persona sheet and notepad, right column
	var col := VBoxContainer.new()
	col.set_anchors_and_offsets_preset(Control.PRESET_TOP_RIGHT)
	col.position += Vector2(-30, 90)
	col.grow_horizontal = Control.GROW_DIRECTION_BEGIN
	col.add_theme_constant_override("separation", 12)
	col.mouse_filter = Control.MOUSE_FILTER_IGNORE
	_call.add_child(col)
	var sheet_plate := PanelContainer.new()
	sheet_plate.add_theme_stylebox_override("panel", UI.box(UI.PAPER, 4, 14))
	_sheet = UI.label("", 22, UI.INK, UI.typed())
	sheet_plate.add_child(_sheet)
	col.add_child(sheet_plate)
	var notes_plate := PanelContainer.new()
	notes_plate.add_theme_stylebox_override("panel", UI.box(Color(0.95, 0.9, 0.55), 4, 14))
	_notes = UI.label("", 20, UI.INK, UI.typed())
	_notes.autowrap_mode = TextServer.AUTOWRAP_WORD_SMART
	_notes.custom_minimum_size = Vector2(360, 0)
	notes_plate.add_child(_notes)
	col.add_child(notes_plate)

	# tells you can note, left middle
	_tells = VBoxContainer.new()
	_tells.set_anchors_and_offsets_preset(Control.PRESET_CENTER_LEFT)
	_tells.position += Vector2(30, 0)
	_tells.grow_vertical = Control.GROW_DIRECTION_BOTH
	_tells.add_theme_constant_override("separation", 10)
	_tells.mouse_filter = Control.MOUSE_FILTER_IGNORE
	_call.add_child(_tells)

	# the scammer's line, bottom centre
	var dialogue := PanelContainer.new()
	dialogue.add_theme_stylebox_override("panel", UI.box(Color(0.06, 0.05, 0.04, 0.9), 6, 24, UI.BRASS, 2))
	dialogue.set_anchors_and_offsets_preset(Control.PRESET_CENTER_BOTTOM)
	dialogue.offset_left = -600
	dialogue.offset_right = 600
	dialogue.offset_top = -230
	dialogue.offset_bottom = -50
	dialogue.grow_vertical = Control.GROW_DIRECTION_BEGIN
	dialogue.mouse_filter = Control.MOUSE_FILTER_IGNORE
	var dv := VBoxContainer.new()
	dialogue.add_child(dv)
	_who = UI.label("", 26, UI.AMBER, UI.serif())
	_text = UI.label("", 32, UI.PAPER, UI.serif())
	_text.autowrap_mode = TextServer.AUTOWRAP_WORD_SMART
	_text.custom_minimum_size = Vector2(1140, 90)
	dv.add_child(_who)
	dv.add_child(_text)
	_call.add_child(dialogue)

	# your lines, above it
	_choices = VBoxContainer.new()
	_choices.set_anchors_and_offsets_preset(Control.PRESET_CENTER_BOTTOM)
	_choices.offset_left = -600
	_choices.offset_right = 600
	_choices.offset_top = -560
	_choices.offset_bottom = -250
	_choices.grow_vertical = Control.GROW_DIRECTION_BEGIN
	_choices.alignment = BoxContainer.ALIGNMENT_END
	_choices.add_theme_constant_override("separation", 10)
	_choices.mouse_filter = Control.MOUSE_FILTER_IGNORE
	_call.add_child(_choices)

	# hold, clip, hang up, bottom right
	var row := HBoxContainer.new()
	row.set_anchors_and_offsets_preset(Control.PRESET_BOTTOM_RIGHT)
	row.position += Vector2(-30, -60)
	row.grow_horizontal = Control.GROW_DIRECTION_BEGIN
	row.grow_vertical = Control.GROW_DIRECTION_BEGIN
	row.add_theme_constant_override("separation", 10)
	_call.add_child(row)
	for spec in [["HOLD (H)", hold_pressed], ["CLIP (C)", clip_pressed], ["HANG UP", hangup_pressed]]:
		var b := UI.button(spec[0], 24)
		b.pressed.connect(func(): spec[1].emit())
		row.add_child(b)


func _input(event: InputEvent) -> void:
	if not _call.visible or _modal != null:
		return
	if _typing and event is InputEventMouseButton and event.pressed:
		_finish_typing()
	elif event is InputEventKey and event.pressed and not event.echo:
		if event.keycode == KEY_H:
			hold_pressed.emit()
		elif event.keycode == KEY_C:
			clip_pressed.emit()


func _process(delta: float) -> void:
	if not _typing:
		return
	_shown += delta * 38.0
	var n := int(_shown)
	if n >= _blip_at + 2:
		Sfx.blip(_pitch)
		_blip_at = n
	_text.visible_characters = n
	if n >= _text.text.length():
		_finish_typing()


func _finish_typing() -> void:
	_typing = false
	_text.visible_characters = -1
	_flush_choices()


# --- simple setters ---

func set_prompt(text: String) -> void:
	_prompt.text = text


func set_hint(text: String) -> void:
	_hint.text = text


func toast(text: String) -> void:
	_toast.text = text
	_toast.modulate.a = 1.0
	var t := create_tween()
	t.tween_interval(0.6)
	t.tween_property(_toast, "modulate:a", 0.0, 0.6)


func set_clock(text: String) -> void:
	_clock.text = text


func call_ui(on: bool) -> void:
	_call.visible = on
	_dot.visible = not on
	if not on:
		_hide_choices()
		_clear_tells()


func set_persona(p: Dictionary) -> void:
	_sheet.text = "PERSONA SHEET\n" + Data.persona_lines(p)


func set_notes(lines: Array) -> void:
	_notes.text = "NOTEPAD\n" + ("\n".join(lines) if not lines.is_empty() else "(nothing yet)")


func set_trust(trust: int, mood: String, minutes: int) -> void:
	_trust.value = trust
	_mood.text = "Mood: " + mood
	_mood.add_theme_color_override("font_color", {"patient": UI.GOOD, "irritated": UI.AMBER, "threatening": UI.BAD}.get(mood, UI.PAPER))
	_minutes.text = "On the line: %d min" % minutes


# --- the call ---

func show_line(who: String, text: String, pitch: float) -> void:
	_who.text = who
	_text.text = text
	_pitch = pitch
	_hide_choices()
	_pending = []
	_shown = 0.0
	_blip_at = 0
	if Game.autoplay:
		_text.visible_characters = -1
		_typing = false
	else:
		_text.visible_characters = 0
		_typing = true


func show_options(options: Array) -> void:
	_pending = options
	if not _typing:
		_flush_choices()


func _flush_choices() -> void:
	_hide_choices()
	for i in _pending.size():
		var b := UI.button(_pending[i]["text"], 26)
		b.autowrap_mode = TextServer.AUTOWRAP_WORD_SMART
		b.custom_minimum_size = Vector2(1200, 0)
		b.alignment = HORIZONTAL_ALIGNMENT_LEFT
		b.pressed.connect(_pick.bind(i))
		_choices.add_child(b)
	_pending = []


func _pick(i: int) -> void:
	_hide_choices()
	choice_made.emit(i)


func _hide_choices() -> void:
	for c in _choices.get_children():
		c.queue_free()


func _clear_tells() -> void:
	for c in _tells.get_children():
		c.queue_free()


func show_tells(tells: Array, noted: Array) -> void:
	_clear_tells()
	for t in tells:
		var done: bool = t["id"] in noted
		var b := UI.button(("NOTED: " if done else "NOTE: ") + t["text"], 22)
		b.disabled = done
		b.autowrap_mode = TextServer.AUTOWRAP_WORD_SMART
		b.custom_minimum_size = Vector2(380, 0)
		b.alignment = HORIZONTAL_ALIGNMENT_LEFT
		b.pressed.connect(func(): tell_clicked.emit(t["id"]))
		_tells.add_child(b)


# --- modals ---

func _open_modal(title: String, body: String) -> VBoxContainer:
	_close_modal()
	_was_looking = cam != null and cam.look_enabled
	if cam:
		cam.set_look(false)
	_modal = Control.new()
	_modal.set_anchors_preset(Control.PRESET_FULL_RECT)
	var shade := ColorRect.new()
	shade.color = Color(0, 0, 0, 0.55)
	shade.set_anchors_preset(Control.PRESET_FULL_RECT)
	_modal.add_child(shade)
	var paper := PanelContainer.new()
	paper.add_theme_stylebox_override("panel", UI.box(UI.PAPER, 4, 40))
	paper.set_anchors_and_offsets_preset(Control.PRESET_CENTER)
	paper.grow_horizontal = Control.GROW_DIRECTION_BOTH
	paper.grow_vertical = Control.GROW_DIRECTION_BOTH
	paper.custom_minimum_size = Vector2(820, 0)
	var v := VBoxContainer.new()
	v.add_theme_constant_override("separation", 14)
	paper.add_child(v)
	v.add_child(UI.label(title, 38, UI.INK, UI.typed()))
	var rule := ColorRect.new()
	rule.color = UI.INK
	rule.custom_minimum_size = Vector2(0, 2)
	v.add_child(rule)
	var b := UI.label(body, 26, UI.INK, UI.serif())
	b.autowrap_mode = TextServer.AUTOWRAP_WORD_SMART
	b.custom_minimum_size = Vector2(740, 0)
	v.add_child(b)
	_modal.add_child(paper)
	add_child(_modal)
	return v


func _close_modal() -> void:
	if _modal:
		_modal.queue_free()
		_modal = null
		if cam and _was_looking:
			cam.set_look(true)


func _finish_modal(value) -> void:
	_modal_done.emit(value)


func info(title: String, body: String, button := "Continue") -> void:
	if Game.autoplay:
		return
	var box := _open_modal(title, body)
	var b := UI.button(button)
	b.pressed.connect(_finish_modal.bind(0))
	box.add_child(b)
	await _modal_done
	_close_modal()


func choose_one(title: String, body: String, buttons: Array) -> int:
	if Game.autoplay:
		return 0
	var box := _open_modal(title, body)
	for i in buttons.size():
		var b := UI.button(buttons[i])
		b.pressed.connect(_finish_modal.bind(i))
		box.add_child(b)
	var picked: int = await _modal_done
	_close_modal()
	return picked


func pick_many(title: String, body: String, labels: Array, button := "Done") -> Array:
	if Game.autoplay:
		return [0] if not labels.is_empty() else []
	var box := _open_modal(title, body)
	var checks: Array = []
	for l in labels:
		var c := CheckBox.new()
		c.text = l
		c.add_theme_font_override("font", UI.serif())
		c.add_theme_font_size_override("font_size", 26)
		for key in ["font_color", "font_hover_color", "font_pressed_color", "font_focus_color", "font_hover_pressed_color"]:
			c.add_theme_color_override(key, UI.INK)
		box.add_child(c)
		checks.append(c)
	var go := UI.button(button)
	go.pressed.connect(_finish_modal.bind(0))
	box.add_child(go)
	await _modal_done
	var picked: Array = []
	for i in checks.size():
		if checks[i].button_pressed:
			picked.append(i)
	_close_modal()
	return picked
```

- [ ] **Step 2: Write the caller card**

`scripts/caller_card.gd`:

```gdscript
extends SubViewport
## The scammer's face, rendered into the CRT screen. Its own little 3D world.

const Builder = preload("res://scripts/character_builder.gd")
## How each call mood shows on the face.
const MOOD_FACE := {"patient": "smug", "irritated": "angry", "threatening": "angry"}

var _who: Node3D


func _ready() -> void:
	size = Vector2i(320, 320)
	own_world_3d = true
	render_target_update_mode = SubViewport.UPDATE_ALWAYS
	var env := Environment.new()
	env.background_mode = Environment.BG_COLOR
	env.background_color = Color(0.08, 0.14, 0.12)
	env.ambient_light_source = Environment.AMBIENT_SOURCE_COLOR
	env.ambient_light_color = Color(0.8, 0.9, 0.85)
	env.ambient_light_energy = 0.9
	var we := WorldEnvironment.new()
	we.environment = env
	add_child(we)
	var cam := Camera3D.new()
	cam.fov = 28.0
	add_child(cam)
	cam.look_at_from_position(Vector3(0, 1.62, 1.7), Vector3(0, 1.58, 0))
	var light := DirectionalLight3D.new()
	light.rotation_degrees = Vector3(-30, 20, 0)
	add_child(light)


func show_scammer(s: Dictionary) -> void:
	clear()
	_who = Builder.build(s["face"])
	_who.pitch = float(s["pitch"])
	add_child(_who)
	_who.look_point = Vector3(0, 1.6, 3.0)   # they stare out of the screen at you
	set_mood("patient")


func set_mood(m: String) -> void:
	if _who:
		_who.set_mood(MOOD_FACE.get(m, "neutral"))
		_who.set_tell("sweat" if m == "threatening" else "none")


func talk(seconds: float) -> void:
	if _who:
		_who.talk(seconds)


func clear() -> void:
	if _who:
		_who.queue_free()
		_who = null
```

- [ ] **Step 3: Verify the scripts parse**

Run:
```bash
godot --headless --path . --quit 2>&1 | grep -E "ERROR|SCRIPT|Parse"; echo done
```
Expected: only `done`. (The HUD and card are exercised for real in Task 9; a parse error would show here.)

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "Add HUD and CRT caller card" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Lab controller: morning, calls, stamping, evening

**Files:**
- Modify (replace entirely): `scripts/lab.gd`

**Interfaces:**
- Consumes: everything above. `CallRunner` signals, `Hud` signals and awaitables, `LabRoom.build` refs, `Evidence.score`, `Channel.{accept_sponsor,post,settle}`, `Data.call_script/persona/note_text`, `Game.state/autosave/goto/autoplay`, `SeatedCamera.clicked(id, target)`.
- Produces: the playable day. User args understood by the scene: `--idle` (build the room and stop, for screenshots), `--demo` (skip the morning and the headset click so a call starts immediately), `--autoplay` (handled by `Game`; the lab plays the whole day with no input, rotating which option it picks, and prints `AUTOPLAY COMPLETE day=%d money=%d subs=%d rep=%d`).

- [ ] **Step 1: Write the controller**

`scripts/lab.gd`:

```gdscript
extends Node3D
## One evening in the basement: inbox, three calls, evidence stamping, the tally.

const GameState = preload("res://scripts/core/game_state.gd")
const Kit = preload("res://scripts/room_kit.gd")
const LabRoom = preload("res://scripts/lab_room.gd")
const SeatedCamera = preload("res://scripts/seated_camera.gd")
const Hud = preload("res://scripts/hud.gd")
const CallerCard = preload("res://scripts/caller_card.gd")
const CallRunner = preload("res://scripts/core/call_runner.gd")
const Evidence = preload("res://scripts/core/evidence.gd")
const Channel = preload("res://scripts/core/channel.gd")
const Data = preload("res://scripts/core/data.gd")

const OUTCOMES := {
	"hangup": "They hung up on you.",
	"complete": "They ran out of script and gave up.",
	"quit": "You hung up.",
}

var cam: Camera3D
var hud: CanvasLayer
var refs: Dictionary
var card: SubViewport
var runner
var persona: Dictionary
var _data: Dictionary
var _turn := 0                     # autoplay: rotates which option it picks
var _demo := "--demo" in OS.get_cmdline_user_args()


func _ready() -> void:
	refs = LabRoom.build(self)
	var env := WorldEnvironment.new()
	env.environment = LabRoom.environment()
	add_child(env)
	Kit.film_overlay(self)
	cam = SeatedCamera.new()
	cam.position = LabRoom.SEAT
	add_child(cam)
	cam.make_current()
	card = CallerCard.new()
	add_child(card)
	refs["screen"].material_override.albedo_texture = card.get_texture()
	hud = Hud.new()
	add_child(hud)
	hud.cam = cam
	cam.hovered.connect(hud.set_prompt)
	hud.choice_made.connect(_on_choice)
	hud.tell_clicked.connect(_on_tell)
	hud.hold_pressed.connect(_on_hold)
	hud.clip_pressed.connect(_on_clip)
	hud.hangup_pressed.connect(_on_hangup)
	persona = Data.persona()
	Sfx.ambience(true)
	_update_board()
	if "--idle" in OS.get_cmdline_user_args():
		return
	_run()


func _process(_delta: float) -> void:
	var t: String = Game.state.clock_text()
	refs["clock"].text = t
	if hud:
		hud.set_clock(t)


# --- the day ---

func _run() -> void:
	var st = Game.state
	if _demo:
		st.stage = "call"
	if st.stage == "morning":
		await _morning()
		st.stage = "call"
		Game.autosave()
	while st.stage == "call":
		await _call(st.current_call_id())
		st.advance_call()
	await _evening()
	if Game.autoplay:
		print("AUTOPLAY COMPLETE day=%d money=%d subs=%d rep=%d" % [st.day, st.money, st.subs, st.rep])
		get_tree().quit()
		return
	Game.goto(Game.MENU)


func _morning() -> void:
	var st = Game.state
	var body := "Rent is due tonight: $%d. You have $%d and %d subscribers." % [GameState.BILLS, st.money, st.subs]
	body += "\n\nA VPN company has emailed. They will pay $%d if you read their ad on a call. Your caller will not enjoy it, and the call starts 10 trust down." % Channel.SPONSOR_PAY
	if st.exposed:
		body += "\n\nA caller used your real street name last time. Be careful what you say."
	var pick: int = await hud.choose_one("INBOX", body, ["Take the sponsor deal", "Decline politely"])
	if pick == 0:
		Channel.accept_sponsor(st)


func _call(id: String) -> void:
	_data = Data.call_script(id)
	hud.set_hint("Click the headset to answer the phone")
	await _wait_click("headset")
	hud.set_hint("")
	Sfx.play("phone")
	runner = CallRunner.new()
	runner.scammer_changed.connect(_on_scammer)
	runner.line.connect(_on_line)
	runner.options_ready.connect(_on_options)
	runner.tells_changed.connect(func(t): hud.show_tells(t, runner.evidence))
	runner.mood_changed.connect(card.set_mood)
	hud.set_persona(persona)
	hud.set_notes([])
	hud.call_ui(true)
	cam.set_look(false)
	runner.start(_data, persona, Game.state)
	await runner.finished
	var res: Dictionary = runner.result
	hud.call_ui(false)
	card.clear()
	Sfx.hold_music(false)
	Sfx.play("click")
	refs["caller_id"].text = "--"
	_update_board()
	var summary := "%s\n\nTime on the line: %d min.\n%d of %d people on their list stayed safe tonight." % [OUTCOMES[res["outcome"]], res["minutes"], res["victims_safe"], res["victims_total"]]
	await hud.info("CALL ENDED", summary)
	hud.set_hint("Click the outbox to file your evidence")
	await _wait_click("outbox")
	hud.set_hint("")
	await _stamp(res)


func _stamp(res: Dictionary) -> void:
	var charges: Array = _data["charges"]
	var labels: Array = charges.map(func(c): return c["text"])
	var body := "You noted %d things on this call. Tick only what your notes can prove. Anything you can't back up costs you channel reputation." % res["evidence"].size()
	body += "\n\n" + "\n".join(_notes())
	var picked: Array = await hud.pick_many("EVIDENCE FILE", body, labels, "Stamp and file")
	var claims: Array = picked.map(func(i): return charges[i]["id"])
	Sfx.play("stamp")
	var sc: Dictionary = Evidence.score(_data, res["evidence"], claims)
	Game.state.change_rep(sc["rep_delta"])
	await hud.info("FILED", "%d claims backed. %d not backed.\nChannel reputation %+d (now %d)." % [sc["backed"], sc["unbacked"], sc["rep_delta"], Game.state.rep])


func _evening() -> void:
	var st = Game.state
	await hud.info("THE BOARD", _board_text(), "Next")
	if not st.clips.is_empty():
		var labels: Array = st.clips.map(func(c): return "%s (quality %d of 4)" % [str(c["call"]).replace("_", " ").capitalize(), int(c["quality"])])
		var picked: Array = await hud.pick_many("POST CLIPS", "Pick which clips go on the channel.", labels, "Post")
		var gain: Dictionary = Channel.post(st, picked)
		await hud.info("POSTED", "+%d subscribers and +$%d from ads." % [gain["subs"], gain["cash"]])
	var tally: Dictionary = Channel.settle(st)
	var text := "%d minutes on the line paid $%d.\n" % [tally["minutes"], tally["pay"]]
	if tally["sponsor"] > 0:
		text += "The sponsor paid $%d.\n" % tally["sponsor"]
	text += "Bills: -$%d.\n\nNet %s$%d. You have $%d and %d subscribers." % [tally["bills"], "+" if tally["net"] >= 0 else "-", absi(tally["net"]), st.money, st.subs]
	await hud.info("DAY'S TALLY", text)
	if st.exposed:
		await hud.info("HEADS UP", "A caller now knows your real details. First build: they just keep a file on you. Later days will make that matter.")
	st.start_next_day()
	Game.autosave()
	await hud.info("END OF DAY ONE", "That is all of the first build. Thanks for playing.", "Back to the menu")


# --- clicking things on the desk ---

## Wait until the player clicks `id` on the desk. Other desk objects can be read meanwhile.
func _wait_click(id: String) -> void:
	if Game.autoplay or _demo:
		return
	cam.set_look(true)
	while true:
		var args: Array = await cam.clicked
		var clicked: String = args[0]
		if clicked == id:
			break
		if clicked == "persona":
			await hud.info("PERSONA SHEET", Data.persona_lines(persona), "Put it down")
		elif clicked == "crt":
			await hud.info("THE CHANNEL", "%d subscribers\nReputation %d / 100\n$%d in the bank" % [Game.state.subs, Game.state.rep, Game.state.money], "Close")
		elif clicked == "corkboard":
			await hud.info("THE BOARD", _board_text(), "Close")
	cam.set_look(false)


func _board_text() -> String:
	var st = Game.state
	if st.results.is_empty():
		return "No calls yet tonight. Each row is a scam ring's call list. The longer you hold them, the more people on it stay safe."
	var lines: Array = []
	for r in st.results:
		var ring: Dictionary = Data.call_script(r["id"])["ring"]
		var names: Array = ring["victims"]
		var safe: int = r["victims_safe"]
		lines.append("%s: %d of %d stayed safe (%s)" % [ring["name"], safe, names.size(), ", ".join(names.slice(0, safe)) if safe > 0 else "nobody"])
	return "\n".join(lines)


func _update_board() -> void:
	var cards: Array = refs["cards"]
	for c in cards:
		c.text = ""
	var row := 0
	for r in Game.state.results:
		var names: Array = Data.call_script(r["id"])["ring"]["victims"]
		for i in names.size():
			var l: Label3D = cards[row * 3 + i]
			l.text = names[i]
			l.modulate = Color(0.1, 0.4, 0.15) if i < int(r["victims_safe"]) else Color(0.6, 0.1, 0.08)
		row += 1


# --- the call, wired to the interface ---

func _notes() -> Array:
	return runner.evidence.map(func(id): return "- " + Data.note_text(_data, id)) if runner else []


func _on_scammer(s: Dictionary) -> void:
	card.show_scammer(s)
	refs["caller_id"].text = s["name"].to_upper()


func _on_line(who: String, text: String, mood: String) -> void:
	hud.show_line(who, text, float(runner.scammer["pitch"]))
	card.talk(text.length() * 0.03)
	card.set_mood(mood)
	_refresh_meters()


func _on_options(opts: Array) -> void:
	hud.show_options(opts)
	hud.set_notes(_notes())
	_refresh_meters()
	if Game.autoplay:
		await get_tree().process_frame
		var pick := _turn % opts.size()
		_turn += 1
		runner.choose(pick)


func _refresh_meters() -> void:
	hud.set_trust(runner.trust, runner.mood(), runner.minutes)


func _on_choice(index: int) -> void:
	if runner and not runner.done:
		runner.choose(index)


func _on_tell(id: String) -> void:
	if runner and not runner.done and runner.log_tell(id):
		Sfx.play("paper")
		hud.show_tells(runner.revealed, runner.evidence)
		hud.set_notes(_notes())


func _on_hold() -> void:
	if runner == null or runner.done or runner.held:
		return
	runner.hold()
	if runner.done:
		return
	_refresh_meters()
	Sfx.hold_music(true)
	hud.set_hint("On hold. Listen to the room behind them...")
	await get_tree().create_timer(2.5).timeout
	Sfx.hold_music(false)
	hud.set_hint("")


func _on_clip() -> void:
	if runner == null or runner.done:
		return
	if not runner.clip().is_empty():
		Sfx.play("clip")
		hud.toast("CLIPPED")
		_refresh_meters()


func _on_hangup() -> void:
	if runner and not runner.done:
		runner.end_call()
```

- [ ] **Step 2: Screenshot the idle room, then a call in progress**

Run:
```bash
godot --path . res://tests/shot.tscn -- lab lab_idle 90 --idle
godot --path . res://tests/shot.tscn -- lab lab_call 240 --demo
```
Read both PNGs. Expected:
- `lab_idle.png`: the basement from Task 7 with the corkboard cards empty.
- `lab_call.png`: the call HUD over the lab: top-left TRUST bar with mood and minutes, a persona sheet and yellow NOTEPAD on the right, the dialogue box at the bottom with Agent Dave's first line typed out and three line choices above it, one note button on the left (the call-center chatter tell), HOLD / CLIP / HANG UP at bottom right, and the CRT in the 3D scene showing Dave's face.

If the CRT face is cropped or off-centre, change the camera in `caller_card.gd` (`look_at_from_position(Vector3(0, 1.62, 1.7), Vector3(0, 1.58, 0))`): raise or lower both Y values together to move the framing, or move Z in or out to zoom.

- [ ] **Step 3: Play one call by hand**

Run: `godot --path . scenes/lab.tscn -- --demo` and play call 1 with the mouse:
- Click a line; click the left-hand "note" button for the chatter tell and confirm it appears on the notepad.
- Press H to hold, press C to clip (a CLIPPED toast and a chime).
- Hang up or finish; confirm the CALL ENDED panel, then (in `--demo` mode the headset/outbox waits are skipped) the EVIDENCE FILE with checkboxes.
Expected: no errors in the Godot output, trust and mood update every turn.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "Add the lab controller: inbox, calls, evidence stamping, evening tally" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Main menu and settings

**Files:**
- Modify (replace entirely): `scenes/main_menu.tscn`
- Create: `scripts/main_menu.gd`

**Interfaces:**
- Consumes: `LabRoom`, `Kit`, `UI`, `Game` (`new_game`, `continue_game`, `has_save`, `goto`, `save_settings`, `volume`, `mouse_sensitivity`, `fullscreen`, `LAB`).
- Produces: title screen with Continue (disabled without a valid save), New Game, Settings (volume, mouse sensitivity, fullscreen), Quit. Under `--autoplay` it starts a new game and goes straight to the lab.

- [ ] **Step 1: Write the menu**

`scenes/main_menu.tscn`:

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/main_menu.gd" id="1"]

[node name="Root" type="Node3D"]
script = ExtResource("1")
```

`scripts/main_menu.gd`:

```gdscript
extends Node3D
## Title screen over a slow look around the basement.

const Kit = preload("res://scripts/room_kit.gd")
const LabRoom = preload("res://scripts/lab_room.gd")
const UI = preload("res://scripts/ui_kit.gd")

var cam: Camera3D
var _t := 0.0
var _menu: VBoxContainer
var _panel: VBoxContainer   # settings, shown in place of the menu


func _ready() -> void:
	LabRoom.build(self)
	var env := WorldEnvironment.new()
	env.environment = LabRoom.environment()
	add_child(env)
	Kit.film_overlay(self)
	cam = Camera3D.new()
	cam.position = LabRoom.SEAT
	cam.fov = 60.0
	add_child(cam)
	cam.make_current()
	Sfx.ambience(true)
	Input.mouse_mode = Input.MOUSE_MODE_VISIBLE

	var layer := CanvasLayer.new()
	add_child(layer)
	var shade := ColorRect.new()
	shade.color = Color(0.03, 0.02, 0.01, 0.45)
	shade.set_anchors_preset(Control.PRESET_FULL_RECT)
	shade.mouse_filter = Control.MOUSE_FILTER_IGNORE
	layer.add_child(shade)
	var root := VBoxContainer.new()
	root.set_anchors_preset(Control.PRESET_FULL_RECT)
	root.alignment = BoxContainer.ALIGNMENT_CENTER
	root.add_theme_constant_override("separation", 16)
	layer.add_child(root)
	var title := UI.label("PLEASE HOLD", 150, UI.PAPER, UI.serif())
	title.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	var sub := UI.label("A scam-baiting game  ·  your grandma is a bit of a character", 34, UI.AMBER, UI.typed())
	sub.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	root.add_child(title)
	root.add_child(sub)
	var gap := Control.new()
	gap.custom_minimum_size = Vector2(0, 50)
	root.add_child(gap)

	_menu = _column(root)
	var cont := _add(_menu, "Continue", _continue)
	cont.disabled = not Game.has_save()
	_add(_menu, "New Game", _new)
	_add(_menu, "Settings", _show_settings)
	_add(_menu, "Quit", func(): get_tree().quit())
	_panel = _column(root)
	_panel.visible = false

	if Game.autoplay:
		Game.new_game()
		Game.goto(Game.LAB)


func _process(delta: float) -> void:
	_t += delta
	cam.rotation = Vector3(-0.1 + sin(_t * 0.11) * 0.02, sin(_t * 0.07) * 0.3, 0)


func _continue() -> void:
	if Game.continue_game():
		Game.goto(Game.LAB)


func _new() -> void:
	Game.new_game()
	Game.goto(Game.LAB)


func _column(parent: Control) -> VBoxContainer:
	var v := VBoxContainer.new()
	v.size_flags_horizontal = Control.SIZE_SHRINK_CENTER
	v.add_theme_constant_override("separation", 14)
	parent.add_child(v)
	return v


func _add(into: VBoxContainer, text: String, action: Callable) -> Button:
	var b := UI.button(text, 32)
	b.custom_minimum_size = Vector2(420, 0)
	b.pressed.connect(action)
	into.add_child(b)
	return b


func _swap(show_panel: bool) -> void:
	for c in _panel.get_children():
		c.queue_free()
	_menu.visible = not show_panel
	_panel.visible = show_panel


func _show_settings() -> void:
	_swap(true)
	_slider("Volume", 0.0, 1.0, Game.volume, func(v): Game.volume = v; Game.apply_settings())
	_slider("Mouse sensitivity", 0.3, 2.5, Game.mouse_sensitivity, func(v): Game.mouse_sensitivity = v)
	var fs := CheckBox.new()
	fs.text = "Fullscreen"
	fs.button_pressed = Game.fullscreen
	fs.add_theme_font_override("font", UI.serif())
	fs.add_theme_font_size_override("font_size", 28)
	fs.toggled.connect(func(on): Game.fullscreen = on; Game.apply_settings())
	_panel.add_child(fs)
	_add(_panel, "Back", func():
		Game.save_settings()
		_swap(false))


func _slider(label: String, lo: float, hi: float, value: float, on_change: Callable) -> void:
	var row := HBoxContainer.new()
	row.add_theme_constant_override("separation", 20)
	var l := UI.label(label, 28, UI.PAPER, UI.serif())
	l.custom_minimum_size = Vector2(260, 0)
	row.add_child(l)
	var s := HSlider.new()
	s.min_value = lo
	s.max_value = hi
	s.step = 0.05
	s.value = value
	s.custom_minimum_size = Vector2(300, 30)
	s.value_changed.connect(on_change)
	row.add_child(s)
	_panel.add_child(row)
```

- [ ] **Step 2: Screenshot the menu and check it**

Run: `godot --path . res://tests/shot.tscn -- main_menu menu 90`
Read `shots/menu.png`. Expected: the basement behind a dark shade, the PLEASE HOLD title, the subtitle line, and four buttons (Continue greyed out if there is no save).

- [ ] **Step 3: Check settings and the save rules by hand**

Run `godot --path .` and: open Settings, move both sliders, tick Fullscreen, press Back (the settings should persist after restarting). With no save file, Continue must be disabled. To test the corrupt-save rule: quit the game, then run

```bash
echo "garbage" > ~/Library/Application\ Support/Godot/app_userdata/Please\ Hold/save.json
godot --path .
```

Expected: the menu appears with Continue disabled and no crash (this is `Game.has_save()` returning false for an unreadable file).

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "Add main menu with settings" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 11: End-to-end autoplay and the final manual playthrough

**Files:**
- Create: `tests/autoplay.sh`

**Interfaces:**
- Consumes: the whole game. `--autoplay` starts a new game from the menu, plays the whole day with no input, prints `AUTOPLAY COMPLETE ...`, and quits.

- [ ] **Step 1: Write the autoplay script**

`tests/autoplay.sh`:

```sh
#!/bin/sh
# Plays the whole first day with no input and prints the result. Fails if it does not finish.
cd "$(dirname "$0")/.."
LOG="${TMPDIR:-/tmp}/pleasehold_autoplay.log"
godot --headless --path . -- --autoplay "$@" > "$LOG" 2>&1 &
PID=$!
for i in $(seq 1 120); do
	kill -0 $PID 2>/dev/null || break
	sleep 1
done
kill $PID 2>/dev/null
grep -E "ERROR|AUTOPLAY" "$LOG" | head -30
grep -q "AUTOPLAY COMPLETE" "$LOG"
```

```bash
chmod +x tests/autoplay.sh
```

- [ ] **Step 2: Run it**

Run: `tests/autoplay.sh`
Expected: one line like `AUTOPLAY COMPLETE day=2 money=... subs=... rep=...` and exit status 0. `day=2` proves the evening ran and `start_next_day()` fired. If it hangs, find which `await` never resolves: the log (`$TMPDIR/pleasehold_autoplay.log`) shows the last output; the usual cause is a HUD awaitable that does not short-circuit under `Game.autoplay`.

- [ ] **Step 3: Run the full automated suite**

```bash
tests/run.sh && tests/autoplay.sh
```
Expected: `28 passed, 0 failed` then the `AUTOPLAY COMPLETE` line; both exit 0.

- [ ] **Step 4: Manual playthrough (the spec's own definition of done)**

Run `godot --path .` and play the whole day start to finish with the mouse:
1. New Game; read the inbox; try both sponsor choices on two separate runs (the sponsor run should start calls at trust 40).
2. Click the persona sheet and the CRT while waiting for the headset, then click the headset.
3. Call 1: find the `clock` tell by probing "Which office are you calling from" then pressing HOLD; note it; finish the call; file evidence at the outbox with one deliberate over-claim and confirm the reputation drops.
4. Call 2: reach the supervisor transfer ("Can I speak to your boss") and confirm the face changes and the call gets stricter. Say the toaster line in this call.
5. Call 3: confirm Kevin mentions Greg and the toaster (only if you said it in call 2).
6. Press CLIP a few times; at night post some clips and confirm subscribers and cash rise.
7. At the evening tally check the numbers against the rules: `$2` per minute on the line, `$60` bills, `$40` sponsor if taken, victims board shows who stayed safe.
8. Quit mid-day and press Continue: it must resume at the right stage with no crash.

Record any bugs found as follow-up fixes before calling the build done.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add autoplay end-to-end check" -m "Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage**

| Spec section | Where |
|---|---|
| §2 core loop: line types, trust, mood, tells, supervisor transfer, hold, call endings | Tasks 2-3 (rules), Task 5 (content), Task 8-9 (UI) |
| §3 day flow: inbox, three calls, evening | Task 9 (`_morning`, `_call`, `_evening`) |
| §4 desk and camera | Task 7 (room, clickables), Task 6 (seated camera copy) |
| §5 evidence and stamping | Task 4 (`Evidence`), Task 9 (`_stamp`) |
| §6 channel, sponsor, exposure, victims board, recurring rival | Task 4 (`Channel`), Task 5 (Kevin variants), Task 9 (board and exposure notice) |
| §7 look and sound | Tasks 6-8 (kit, film overlay, blips, hold music, chime) |
| §8 structure | File Structure above; deviations noted there |
| §9 saving | Task 1 (`GameState`), Task 6 (`Game`), Task 10 (corrupt save check) |
| §10 testing | Tasks 1-5 (unit), Task 5 (script validator and playability), Task 11 (autoplay, manual) |

Known deviations, kept honest: CLIP, HOLD and the notepad are HUD elements, not 3D desk objects (stated up front); `Persona`/`DialogueBox`/`Desk` are folded into data, HUD and `lab_room.gd`; the stamp is a checklist modal with the stamp sound, not a 3D stamp animation; the sponsor is one email offer with a trust penalty and a payout, not a scripted read.

**Placeholder scan:** every code step carries complete code. The one deliberate stub (`_refresh_tells` in Task 2) is replaced in Task 3 and is called out where it appears.

**Type consistency:** `CallRunner` public names (`start, options, choose, hold, log_tell, clip, end_call, mood, mood_index, trust, minutes, evidence, revealed, held, result, scammer, done`) match their use in `lab.gd`. `Hud` method names (`show_line, show_options, show_tells, set_trust, set_notes, set_persona, call_ui, info, choose_one, pick_many, toast, set_hint, set_clock, set_prompt`) match the controller. `LabRoom.build` refs keys (`screen, caller_id, clock, cards`) match `lab.gd`. `Game.LAB`/`Game.MENU` are defined in Task 6 and used in Tasks 9-10. `GameState` fields match across save, runner and channel.
