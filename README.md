# LLT Multi-Terminal Loader v4.0

Desktop app for parallel loading of packages and parameters to Ingenico POS terminals.

```
python llt_loader.py
```

Python 3.10+, no pip installs (only built-in modules).

---

## How parallel loading works

The LLT User Guide says: **"only one terminal connection is possible at one time per LLT instance."**

So to load 3 terminals at once, the app does this:

```
Terminal 1 (COM3)          Terminal 2 (COM5)          Terminal 3 (COM8)
     │                          │                          │
     ▼                          ▼                          ▼
Write _loader_COM3.xml    Write _loader_COM5.xml    Write _loader_COM8.xml
  (has <connect            (has <connect              (has <connect
   device="COM3"/>)          device="COM5"/>)           device="COM8"/>)
     │                          │                          │
     ▼                          ▼                          ▼
cmdLine.bat /q /p COM3    cmdLine.bat /q /p COM5    cmdLine.bat /q /p COM8
  _loader_COM3.xml          _loader_COM5.xml          _loader_COM8.xml
     │                          │                          │
     └──────────── all 3 run simultaneously ───────────────┘
                    (ThreadPoolExecutor)
                           │
                           ▼
                   Results per terminal
                   shown in status cards
```

Each terminal gets:
- **Its own XML file** with `<connect device="COMx"/>` hardcoded to that port
- **Its own cmdLine.bat process** (subprocess.run in a worker thread)
- **Its own result** shown in a color-coded status card (✓ green / ✗ red)

The XML files are written to the **XML Folder** (defaults to system temp) as temp files (`_loader_COMx.xml`) and deleted after each process finishes.

---

## What the program does step by step

### 1. Set paths (saved forever)

| Field | Purpose |
|-------|---------|
| **LLT Install** | Where `cmdLine.bat` lives. The subprocess `cd`s here before running. |
| **Package Folder** | Where your .P3A, .INI etc. files are. All file paths = `folder\filename`. |
| **XML Folder** | Where temp XML files are written during loading. Defaults to system temp if blank. Must NOT be inside the LLT install folder. |
| **Output Folder** | Where output files are saved: query results (`MfgCode_*.txt`, `Serial_*.txt`), terminal list (`Terminals_*.txt`), result file (`result_*.log`), upload config (`HTERMINAL_*.INI` / `APPRESET_*.DIA`). Defaults to system temp if blank. |

Change the folder → all file paths update instantly. No re-adding files.

### 2. Add files

Three ways:
- **Type filename** in the Add field (e.g. `MyApp.P3A`) and press Enter
- **Browse…** — file picker dialog
- **Scan** — auto-finds all loadable files in the package folder

Files are saved by name only. The full path is built as `[folder]\[filename]` automatically.

### 3. Pick terminals

- **Detect** — scans COM ports via WMI (`Win32_PnPEntity`) for full device names; falls back to registry (`HARDWARE\DEVICEMAP\SERIALCOMM`) if WMI is unavailable
- **+ Add** — add more terminals for parallel loading
- Each terminal has its own checkbox (enable/disable) and COM port dropdown
- First detected port is auto-assigned to Terminal 1

### 4. Set options

| Option | LLT XML action | What it does |
|--------|----------------|-------------|
| Force uppercase | `disable` option `force uppercase` | T1/T2: forces full filename uppercase. Tetra: extension only. Default ON. |
| Ignore errors | `enable` option `ignore errors` | Non-blocking — continues past download failures |
| Auto-disconnect | `disconnect` | Commits the transaction. Terminal restarts and validates packages. |
| Verbose /v | cmdLine flag `/v` | Shows extra debug logs on errors |
| Auto-destination | omits `target` in XML | LLT auto-detects `/package/` vs `/import/` by file extension |
| Clean before load | `clean` | Wipes terminal: T1/T2 = `/HOST/`+`/SWAP/`, Tetra = `/export/`+`/import/`+`/package/` |
| Query info | `query` | Saves Manufacturing Code + Islero Serial Number to text files |
| List terminals | `terminals` | Writes all plugged terminals (Port/Range/Description/State) to a file |
| Result file | `enable` option `result file` | Logs first error (or success) per terminal to a `.log` file |
| Upload config | `upload` | Pulls HTERMINAL.INI (Tetra) or APPRESET.DIA (T1/T2) from terminal to PC |
| Activity | `activity` mode=... | Changes terminal to download/diagnostic/maintenance after loading |

### 5. Generate & Load

Clicking the green button:
1. Validates files and terminals
2. Checks `cmdLine.bat` exists in the LLT path
3. Launches all terminals in parallel via `ThreadPoolExecutor`
4. Each thread independently: builds its own XML (with its own `<connect device="COMx"/>`), writes it to `{xml_dir}/_loader_COMx.xml` (defaults to system temp), `cd`s to LLT path, runs `cmdLine.bat`, then deletes the temp XML
5. Status cards update in real-time: ○ idle → ◌ running → ✓ success / ✗ failed
6. Log shows per-terminal output with timestamps

### 6. Preview / Export

- **Preview XML** — popup showing the generated XML + batch for the first terminal
- **Export Scripts…** — saves XML + `.bat` per terminal to a folder you choose

---

## File routing

Based on LLT User Guide Section 1.2:

| Extension | Telium Tetra | Telium 1 & 2 |
|-----------|-------------|--------------|
| .P3A .P3L .P3O .P3P .P3S .P3T .AGN .LGN .PGN .PKG | `/package/` | `/SWAP/` |
| .INI .CFG .TXT .DAT .XML .JSON .CSV .BIN .PAR .BMP | `/import/` | `/HOST/` |
| .Mnn .MXX (catalogues) | `/package/` | `/SWAP/` |

Packages in `/SWAP/` or `/package/` are validated on disconnect.
**INVALID_SIGNATURE** = package not properly signed → delete from `/package/` manually.

---

## Generated XML format

```xml
<?xml version = "1.0" encoding = "ISO-8859-1"?>
<project
    name = "AutoLoader"
    basedir = "."
    default = "load">

    <target name = "load">
        <jllt.commandLineTask
                action = "connect"
                device = "COM5"/>
        <jllt.commandLineTask
                action = "download"
                source = "C:\\packages\\MyApp.P3A"
                target = "/package/"/>
        <jllt.commandLineTask
                action = "download"
                source = "C:\\packages\\config.INI"
                target = "/import/"/>
        <jllt.commandLineTask
                action = "disconnect"/>
    </target>

</project>
```

Note: all backslashes are doubled per the LLT User Guide CAUTION box.

---

## Terminal prep

| Terminal | How to enter LLT mode |
|----------|----------------------|
| **Telium Tetra** | Restart → hold **"."** (dot) until folder icon |
| **Telium 1 & 2** | Restart → hold **"F3"** until "LLT" on screen |

---

## Error codes

| Code | Meaning | Likely fix |
|------|---------|-----------|
| 0 | OK | — |
| -4 | Cannot access files | Check folder path, do files exist? |
| -15 | Activity badly finished | Activity change failed |
| -19 | Unknown port | Terminal not in LLT mode, or wrong COM port |
| -21 | File not for this range | Tetra file on T1/T2 or vice versa |
| -22 | Download error | File corrupted or path wrong |
| -24 | Destination unreachable | Terminal disconnected mid-transfer? |

---

## Config file

Saved at `~/.llt_loader_config.json`. Contains all paths, file names, terminal ports, options, and window geometry. Delete this file to reset everything.

## Build standalone .exe

```
pip install pyinstaller
pyinstaller --onefile --windowed --name LLT_Loader llt_loader.py
```

Output: `dist/LLT_Loader.exe`
