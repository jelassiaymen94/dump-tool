# FiveM Server Dump Tool

Dumps FiveM server resources and decrypts FXAP resources that have been dumped.

## Scope

- Included: server dump
- Included: FXAP decryption
- Optional: generate a complete copy only for the FXAP resources decrypted in this run, and run model repair on the copy

## Requirements

- Windows 10 / Windows 11
- Python 3.7 or later
- Java 8 or later (Java 17 recommended, used to run `unluac54.jar`)
- .NET 8 Runtime for the optional model repair
- FiveM installed and already connected to the target server
- Running as administrator is recommended when the FiveM process token must be read

## Installation

1. Download or clone this repository.
2. Enter the project directory.
3. Run `install.bat` to install the dependencies.

You can also install the dependencies manually:

```bat
pip install -r requirements.txt
```

Lua 5.4 decompilation depends on Java. The program checks in order the location specified by `--java`, `JAVA_HOME`, and the system `PATH`; if no usable Java is available, it stops before the dump instead of mistakenly writing undecompiled bytecode as `.lua`.

Installing Eclipse Temurin Java 17 JRE is recommended:

https://adoptium.net/temurin/releases/?version=17&os=windows&arch=x64&package=jre

## Usage

Interactive mode:

```bat
python auto.py
```

Passing the target address directly:

```bat
python auto.py https://cfx.re/join/xxxx --token-choice 1
python auto.py 1.2.3.4:30120 --token-choice 1
python auto.py 1.2.3.4:30120 --token-choice 1 --java "C:\Program Files\Eclipse Adoptium\jre-17"
python auto.py 1.2.3.4:30120 --token-choice 1 --no-fxap-decrypt
python auto.py 1.2.3.4:30120 --token-choice 1 --no-download-stream
python auto.py 1.2.3.4:30120 --token-choice 1 --vertex-fix
python auto.py --retry-failed-report "C:\path\to\previous-report.json" --non-interactive
python auto.py --resume-session "C:\path\to\session.json" --non-interactive
python auto.py --auto-target --token-choice 1 --list-resources --non-interactive
```

Non-interactive mode, for use by the CK toolbox:

```bat
python auto.py 1.2.3.4:30120 --token-choice 1 --resources all --output Output --report Output\_server_dump_report.json --non-interactive
```

The default value of `token_choice` is `1`, which means automatically scanning the token from the FiveM process. To provide a manual token, use:

```bat
python auto.py 1.2.3.4:30120 --token-choice 2 --token YOUR_TOKEN
```

## Two-step resource selection

List the server resources only, without creating output or running a dump:

```bat
python auto.py 1.2.3.4:30120 --token-choice 1 --list-resources --non-interactive
```

A full dump accepts indices, exact resource names, and the `*` and `?` wildcards; separate multiple conditions with commas:

```bat
python auto.py 1.2.3.4:30120 --token-choice 1 --resources "esx_*,qb-*" --non-interactive
```

When `--resources` is not passed and interactive mode is used, the program displays a numbered menu after fetching the list, previews the matches, and asks for confirmation. The CK toolbox uses a separate "fetch resource list" step and, after menu confirmation, passes the exact resource names to the full dump through `--resources-file`.

## Output

- Decrypted files are written to `Output` by default
- The JSON report is written to `Output\_server_dump_report.json` by default
- The Markdown report is written to `Output\_server_dump_report.md` by default
- The report records the actual Java path and version, as well as details of Lua decompilation failures
- When `unluac54.jar` fails, the `.luac` bytecode and the error file are kept instead of disguising them as a successfully decrypted `.lua`
- `CK_PROGRESS` progress events are emitted during execution so the CK toolbox can display progress in real time
- When a resource file download encounters HTTP 403, 404, 408, 409, 425, 429, common 5xx errors, a dropped connection, or a timeout, up to 4 requests are made (initial attempt + 3 retries), with bounded backoff and a numeric `Retry-After` honored
- Every retry, successful retry, and final failure is written to the log; the JSON/Markdown reports record, item by item, the resource name, file name, HTTP status code, attempt count, and failure reason of every download that ultimately failed
- `--no-fxap-decrypt` skips FXAP decryption and does not require Java; after each resource is downloaded and unpacked, the `.fxap` files, encrypted files, scripts, and all subdirectories are copied to the output directory with their original structure before the temporary directory is cleaned up
- Files under each resource's `stream` directory are downloaded by default; `--no-download-stream` skips stream file downloads
- `--no-fxap-decrypt` cannot be used together with `--vertex-fix`; model repair only applies to successfully decrypted copies
- `--retry-failed-report` reads the previous JSON report, selects only the items in its `failed_downloads` and those waiting for decryption prerequisites for re-download, and merges them back into the original output directory recorded in the report; the new report can be used for another round of supplementary downloads
- A dump saves `session.json` in the task directory; after a connection interruption, the toolbox shows "Resume Dump", and clicking it after re-entering the game re-scans the token and skips resources already completed. Unfinished files use `.part` temporary files and are downloaded again when resumed
- `--wait-for-game --status-file <path>` waits when there is no FiveM token or when the connection is interrupted, and automatically resumes with `--resume-session`; the status file records the current stage, percentage, message, report path, and number of completed resources so the toolbox can display silent dump progress
- `--auto-target` scans the current FiveM process and memory candidates when the target address is empty, and verifies them by requesting `/client`'s `getConfiguration` with the current token; only addresses that return a valid `resources` configuration proceed to the dump
- A silent dump uses a resource matcher expression directly through `--resources` (for example `aaa*`) and does not open the resource selection window; when downloads finish but some files still failed, the report can be passed to `--retry-failed-report` for supplementary downloads
- Choosing "start server dump" creates a new task and does not take over a previously paused task; the old task's state and completed output are preserved
- In decryption mode, supplementary stream/normal file downloads also fetch the RPF/`.fxap` context that the resource needs; otherwise it cannot be decrypted safely. Files that already succeeded and are not decryption prerequisites are not downloaded again
- Supplementary downloads do not automatically run model repair; they only update the original decrypted output or the raw preserved output, so existing model-repaired copies are not overwritten
- If a resource in an old report has been removed from the server or renamed, it is marked as `unavailable` and kept for the next round; other resources that still exist continue to be supplemented, so a few missing resources no longer block the whole run
- `--vertex-fix` only processes resources that are confirmed to contain `.fxap` in this run and have already been decrypted; the original decrypted directory is not overwritten
- Model repair creates a sibling directory named `<original directory name>_模型修复` (the original directory name plus `_模型修复`, Chinese for "model repair"), copies the complete directory of each eligible resource, and then processes the copy; if a directory with that name already exists, a new one with a timestamp is used automatically

## Notes

- Use this only on servers or resources you have permission to analyze.
- If FiveM is closed during a dump, re-enter the target server and continue using the task state; do not delete `session.json` in the task directory.
- If automatic token scanning fails, confirm that FiveM has entered the server, or run as administrator.
- Antivirus software may flag process reading or the packer tool as risky, depending on local policy.

## Disclaimer

This project is intended for learning, research, and authorized scenarios only. Users are solely responsible for the consequences of misuse.
