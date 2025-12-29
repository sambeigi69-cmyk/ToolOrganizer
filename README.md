# ToolOrganizer

ToolOrganizer is an industrial “tool cabinet” kiosk application designed to run 24/7 on a dedicated PC with an always-on display (touchscreen or standard monitor). It allows workers to take and return specific hand tools (e.g., 16 tools) stored inside a locked cabinet.

## How it works (high-level flow)

* A worker enters the process by scanning a barcode/code (usually an ID card/badge or an article/work order).
* The software determines which tool/lock is involved based on Excel/Mapping data.
* It then unlocks the correct compartment by sending a command to an Arduino (via Serial) connected to a relay module.
* Everything is logged (who, which tool, when), and sometimes a short video clip is recorded for audit purposes.
* The UI must be simple, full-screen, touch-friendly, and must not expose any admin menu to workers.

---

## What is this system and why does it exist?

This project is an industrial tool cabinet control kiosk that:

* Controls **16 tools**, each behind a lock/relay channel.
* Connects to an **Arduino Mega** and a **16-channel relay module**.
* Uses a **USB barcode scanner** that acts like a keyboard.
* Reads tool and article data from an Excel file: `tools.xlsx`
* Reads mapping data (**ToolCode → RelayNo** and Active/Inactive status) from: `tools_map.csv`
* When a worker scans an **Article**:

  * If a **Machine** is defined for that Article → the system asks: **“Why didn’t you use the machine?”**
  * Then it shows the list of allowed tools.
* When a tool is selected:

  * The software sends an **UNLOCK** command to the Arduino
  * The relay is activated for a few seconds
* The application is built with **PyQt5**, with a **dark theme** suitable for industrial 24/7 operation.
* All dispensing actions are recorded in `dispense_log.csv`, and additional logs are stored in the `logs/` directory.

---

## Changes

### V1.2.0

* Added **Article / Machine / Reason** fields and saved them in the log.

---

## Current limitation

Right now, the system only logs **User** and **Tool**.
In real production, there is a critical missing layer: **Article** (drawing number / part number / work number), and the related **Machine** for that job.

What we want is described below.

---

# Requested Improvements

## 1) Add Article + Machine + Reason (and store them in the log)

### Why this matters

These three fields (**Article + Machine + Reason**) are essential for traceability, audits, and process safety. Later we must be able to see, for each job:

* which machine was defined,
* whether the machine was used,
* if not, what reason was recorded for using hand tools instead.

### Desired behavior in the software

After the worker logs in (by scanning their ID):

1. Instead of going directly to tool selection, the worker must first **scan or enter the Article** (drawing/part/work number).

2. For each Article, we already have (on paper) this information:

* which **MachineCode** the job should be done on.

3. If the operator cannot or does not want to use the machine and chooses a manual tool, a **Reason** must be recorded.

If a Machine is defined for that Article, the system must show a panel/message like:

> “Why didn’t you use machine [MachineCode]?”

And show a **ComboBox** of predefined reasons, for example:

* Machine is broken
* Quantity is too low
* Special part / exception
* …

The operator selects one reason, and only then the system allows tool selection.

### Logging requirements

In the same log location where we currently store **User**, **Tool**, Persian text, and video path, also store:

* **Article** (drawing/part number)
* **Machine** (if defined for that Article)
* **Reason** (why the machine was not used, if applicable)

---

## 2) Article → Allowed Tools mapping (restrict tool selection)

### Current behavior

The system currently works like this:

* Worker logs in
* Chooses operation type (take / return)
* Enters tool code manually or clicks a tool card
* The system only checks:

  * whether the user has permission for that tool (roles),
  * tool status (FREE / INUSE / DEACTIVE)

### Required behavior

We must be able to define, for each Article, which tools are allowed.

Example:

* For Article `A123`, only tools `4`, `6`, and `8` are allowed (e.g., only these polishing stones are acceptable).

After login and Article selection:

1. Worker enters Article
2. The system looks up allowed tools for that Article (from Excel/CSV/DB)
3. On the tool selection page, **only those allowed tools are shown and selectable**
4. If the worker tries to enter another tool manually, the system must block it and show an error message:

   * “This tool is not allowed for this Article.”

### Implementation preference

It is strongly preferred that:

* The mapping **Article → Allowed Tools** is stored in an **Excel or CSV file**
* And can be uploaded/updated inside the software (e.g., via a settings page),
  so small mapping changes do not require code changes.

### Why this matters

This ensures that:

* incorrect tools cannot be used for sensitive jobs,
* process logic (which job can use which tool) is enforced by the system, not just operator memory.

---

## 3) Require clear Arduino acknowledgement for every unlock (OK / ERR / TIMEOUT)

### Current behavior (based on existing code)

* The software sends a command string to the Arduino to open a door/relay.
* There is also a periodic PING to check that the serial connection is alive.
* However:

  * the response to each unlock command is not checked,
  * the MainWindow does not analyze and log “success / error / no response”.

### Required behavior

For each unlock command (e.g., “activate relay 3 for 2 seconds”):

1. The software sends a clear unlock command to the Arduino
   Example formats (to be finalized together):

   * `UNLOCK 3 2000`
   * or the current custom format, but with a standardized response

2. After sending the command, the software waits for a defined period (e.g., up to 2 seconds) for a response.

3. It must detect three outcomes:

* **OK**: Arduino confirms the command was executed
* **ERR**: Arduino reports an error (invalid relay, hardware error, etc.)
* **TIMEOUT**: no response within the expected time (likely connection issue)

4. Based on the result:

* Log:

  * which tool / which relay was commanded,
  * the result: OK / ERR / TIMEOUT
* UI messaging:

  * If OK → normal success message (“Tool door opened”)
  * If ERR or TIMEOUT → clear error message to the operator
  * And in this case the system should **not assume** the tool was actually released; it must be tracked separately from successful unlocks.

---
