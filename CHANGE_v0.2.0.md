# Changelog v0.2.0

All changes from the initial commit (`29b9f68`, 197 lines) to `v0.2.0` (~510 lines).

---

## 1. Removed `read_file` and `write_file` tools — shell-only design

The initial version exposed three tools to the LLM: `shell`, `read_file`, and `write_file`. v0.2.0 removes the two file tools entirely, leaving only `shell`.

**Initial (3 tools):**
```c
static const char TOOLS[]=
    "[{...\"name\":\"shell\",...},{...\"name\":\"read_file\",...},{...\"name\":\"write_file\",...}]";

char *tool_execute(const char *name, const char *aj) {
    cJSON *a = cJSON_Parse(aj);
    if (!strcmp(name,"shell")) { ... }
    else if (!strcmp(name,"read_file")) { ... }
    else if (!strcmp(name,"write_file")) { ... }
}
```

**v0.2.0 (1 tool):**
```c
static const char TOOLS_JSON[] =
    "[{\"type\":\"function\",\"function\":{\"name\":\"shell\","
    "\"description\":\"Run a shell command\","
    "\"parameters\":{...}}}]";

static char *tool_execute(const Config *cfg, const char *name, cJSON *arguments) {
    if (strcmp(name, "shell") != 0) return NULL;
    ...
}
```

The LLM now uses `cat`, `tee`, `sed`, etc. through the shell for file operations. This follows the anti-framework philosophy: the shell is the only integration layer needed.

---

## 2. Bubblewrap sandbox for tool execution

The initial version ran shell commands directly via `popen()` with no isolation. v0.2.0 adds optional sandboxing using [bubblewrap](https://github.com/containers/bubblewrap).

**Initial (no sandbox):**
```c
char *fc = malloc(cl+8);
memcpy(fc, f1->valuestring, cl);
memcpy(fc+cl, " 2>&1", 6);
FILE *fp = popen(fc, "r");
```

**v0.2.0 (bwrap sandbox):**
```c
if (cfg->sandbox_bwrap) {
    // Write command to temp file to avoid quoting issues
    snprintf(script_path, sizeof(script_path), "%s/.szc_cmd", cwd);
    FILE *sf = fopen(script_path, "w");
    fprintf(sf, "%s\n", command);

    snprintf(cmd_buf, sizeof(cmd_buf),
        "%s bwrap "
        "--unshare-user-try --unshare-pid --unshare-ipc "
        "--unshare-uts --unshare-cgroup-try "
        "--die-with-parent --new-session "
        "--tmpfs / "
        "--bind %s /work "         // current dir -> /work
        "--ro-bind /bin /bin "
        "--ro-bind /usr /usr "
        "--ro-bind /lib /lib --ro-bind /lib64 /lib64 "
        "--dev /dev --proc /proc --tmpfs /tmp "
        "%s "                      // optional: --unshare-net
        "bash /work/.szc_cmd 2>&1",
        timeout_str, cwd, skills_bind, net_str);
} else {
    snprintf(cmd_buf, sizeof(cmd_buf), "%s 2>&1", command);
}
```

Three new config options control sandbox behavior:

| Option | Default | Effect |
|--------|---------|--------|
| `sandbox_bwrap` | 1 | Enable bubblewrap isolation |
| `sandbox_net` | 1 | Block external network (localhost only) |
| `sandbox_timeout` | 60 | Kill commands after N seconds |

The sandbox mounts the current directory as `/work`, provides read-only access to system binaries, and optionally isolates networking with `--unshare-net`.

---

## 3. Config struct and parsing refactored for readability

The initial version used cryptic short field names. v0.2.0 uses full descriptive names and adds sandbox fields.

**Initial:**
```c
typedef struct { char key[MV],model[MV],ep[MV],skills[MP],logdir[MP]; int mt,mm,ck; } Cfg;
```

**v0.2.0:**
```c
typedef struct {
    char api_key[MAX_VALUE], model[MAX_VALUE], endpoint[MAX_VALUE];
    char skills_dir[MAX_PATH], log_dir[MAX_PATH];
    int  max_turns, max_messages;
    int sandbox_bwrap;
    int sandbox_net;
    int sandbox_timeout;
} Config;
```

The parser (`config_parse_line`) is now a separate function, and environment variable overrides are added for all sandbox options (`SUBZEROCLAW_SANDBOX_BWRAP`, `SUBZEROCLAW_SANDBOX_NET`, `SUBZEROCLAW_SANDBOX_TIMEOUT`).

---

## 4. Readable code style throughout

The initial version was aggressively compressed into 197 lines. v0.2.0 reformats everything for readability with proper indentation, spacing, and named constants.

**Initial (compressed):**
```c
#define MP 512
#define MV 1024
#define MR (128*1024)

static void logw(FILE *f,const char *r,const char *c){if(!f)return;time_t n=time(0);
    struct tm *t=localtime(&n);char ts[32];strftime(ts,32,"%Y-%m-%d %H:%M:%S",t);
    fprintf(f,"[%s] %s: %s\n",ts,r,c);fflush(f);}
```

**v0.2.0 (readable):**
```c
#define MAX_PATH   512
#define MAX_VALUE  1024
#define MAX_OUTPUT (128 * 1024)

static void log_write(FILE *log, const char *role, const char *content) {
    if (!log) return;
    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    char ts[32]; strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", t);
    fprintf(log, "[%s] %s: %s\n", ts, role, content); fflush(log);
}
```

All functions follow this pattern: `build_req` -> `build_request`, `parse_resp` -> `parse_response`, `compact` -> `compact_messages`, `logw` -> `log_write`, etc.

---

## 5. Secure HTTP posting (API key no longer in command line)

The initial version passed the API key directly in the curl command line, visible in `ps` output:

**Initial:**
```c
snprintf(cmd, 2048, "curl -s -m 120 -H 'Authorization: Bearer %s' "
    "-H 'Content-Type: application/json' -d @/tmp/.szc.json '%s' 2>&1", key, url);
```

**v0.2.0:**
```c
// Write header to temp file — API key never appears in process list
char hdr[MAX_VALUE + 64];
snprintf(hdr, sizeof(hdr), "-H \"Authorization: Bearer %s\"", api_key);
write_temp("hdr", hdr, hdr_path, sizeof(hdr_path));

snprintf(cmd, sizeof(cmd),
    "curl -s -m 120 -K '%s' -H 'Content-Type: application/json' -d @'%s' '%s' 2>&1",
    hdr_path, body_path, url);
```

The `-K` flag tells curl to read the authorization header from a file. Both temp files use `mkstemp()` for unique names (instead of the fixed `/tmp/.szc.json`), preventing race conditions when running multiple instances.

---

## 6. Project-local skills support

The initial version only loaded skills from one global directory (`~/.subzeroclaw/skills/`). v0.2.0 adds a second skill source: the `skills/` directory in the current working directory.

**Initial:**
```c
char *agent_build_system_prompt(const char *dir) {
    // loads skills from one directory only
}
```

**v0.2.0:**
```c
char *agent_build_system_prompt(const char *skills_dir, const char *project_skills_dir) {
    ...
    len = load_skills(&prompt, len, &cap, skills_dir, "SKILL");
    len = load_skills(&prompt, len, &cap, project_skills_dir, "PROJECT SKILL");
    return prompt;
}
```

The `load_skills` function is extracted as a reusable helper that scans a directory for `.md` files and appends them to the system prompt. Global skills are labeled `SKILL`, project-local ones `PROJECT SKILL`.

---

## 7. Tool argument parsing handles both string and object formats

Some LLM APIs return tool call arguments as a JSON string (double-encoded), others as a parsed JSON object. The initial version only handled string arguments. v0.2.0 handles both.

**v0.2.0:**
```c
static int process_tool_calls(const Config *cfg, cJSON *tool_calls, cJSON *msgs, FILE *log) {
    ...
    cJSON *args_parsed = NULL;
    if (cJSON_IsString(args)) {
        args_parsed = cJSON_Parse(args->valuestring);
        if (args_parsed) args = args_parsed;
    } else if (!cJSON_IsObject(args)) {
        continue;
    }
    char *result = tool_execute(cfg, name->valuestring, args);
    if (args_parsed) cJSON_Delete(args_parsed);
    ...
}
```

This makes SubZeroClaw compatible with more API providers (OpenRouter, OpenAI, xAI, etc.).

---

## 8. Improved context compaction

The initial compaction used a simple boundary-based approach that kept the last N messages and discarded the rest. v0.2.0 sends old messages to the LLM for summarization with trimming of long tool outputs.

**Initial:**
```c
static int compact(const Cfg *c, cJSON *msgs, FILE *log) {
    int bnd = tot - c->ck;
    // collect messages 1..bnd into text, delete them
    // insert summary at position 1
}
```

**v0.2.0:**
```c
static int compact_messages(const Config *cfg, cJSON *msgs, FILE *log) {
    // Trim long tool outputs in the summary input
    if (strcmp(r, "tool") == 0 && clen > 2000) {
        memcpy(convo + len, c, 1000);
        len += snprintf(convo + len, cap - len, "... [trimmed] ...\n");
    }
    // Keep last 10 messages, summarize the rest
    while (cJSON_GetArraySize(msgs) > 10) {
        cJSON_DeleteItemFromArray(msgs, 0);
    }
    cJSON_InsertItemInArray(msgs, 0, make_msg("assistant", resp.text));
    cJSON_InsertItemInArray(msgs, 0, make_msg("user", summary_prompt));
}
```

Tool outputs longer than 2000 characters are truncated to 1000 before being sent for summarization, reducing token usage.

---

## 9. Readline integration for interactive mode

The initial version used raw `fgets()` for interactive input. v0.2.0 uses GNU readline for line editing, history, and arrow key navigation.

**Initial:**
```c
printf("> "); fflush(stdout);
if (!fgets(in, 4096, stdin)) break;
```

**v0.2.0:**
```c
#include <readline/readline.h>
#include <readline/history.h>
...
char *line = readline("> ");
if (!line) break;
if (line[0]) add_history(line);
agent_run(&cfg, line, log);
free(line);
```

This adds proper terminal line editing (Ctrl-A, Ctrl-E, backspace, arrow keys) and command history (up/down arrows) to the interactive REPL.

---

## 10. Timestamped session IDs

The initial version generated random hex session IDs from `/dev/urandom`. v0.2.0 uses human-readable timestamps.

**Initial:**
```c
FILE *r = fopen("/dev/urandom","r");
unsigned char b[8];
fread(b, 1, 8, r);
snprintf(sid, 32, "%02x%02x%02x%02x%02x%02x%02x%02x", b[0],...);
```

**v0.2.0:**
```c
time_t now = time(NULL);
struct tm *tm = localtime(&now);
strftime(session_id, sizeof(session_id), "%Y%m%dT%H%M%S", tm);
```

Log files are now named like `20260222T153001.txt` instead of `f850c58ddd4ae72a.txt`, making them easy to find by date.

---

## 11. `--version` flag with license info

**v0.2.0:**
```c
if (argc == 2 && (!strcmp(argv[1], "--version") || !strcmp(argv[1], "-v"))) {
    printf("subzeroclaw %s\n"
           "Copyright (c) 2025-2026 SubZeroClaw contributors\n"
           "License MIT: https://opensource.org/licenses/MIT\n"
           "This is free software: you are free to change and redistribute it.\n"
           "There is NO WARRANTY, to the extent permitted by law.\n",
           SZC_VERSION);
    return 0;
}
```

---

## 12. Agent loop restructured

The initial version passed messages and tools arrays into `agent_run` from `main()`. v0.2.0 creates and manages them internally, keeping the public interface simpler.

**Initial:**
```c
static int agent_run(const Cfg *c, cJSON *msgs, cJSON *tools, const char *in, FILE *log) {
    compact(c, msgs, log);
    cJSON_AddItemToArray(msgs, um);
    for (int t = 0; ++t <= c->mt; ) { ... }
}

// in main():
cJSON *msgs = cJSON_CreateArray();
cJSON *tools = cJSON_Parse(TOOLS);
agent_run(&cfg, msgs, tools, in, log);
```

**v0.2.0:**
```c
static int agent_run(const Config *cfg, const char *task, FILE *log) {
    char *system_prompt = agent_build_system_prompt(cfg->skills_dir, project_skills);
    cJSON *msgs = cJSON_CreateArray();
    cJSON_AddItemToArray(msgs, make_msg("system", system_prompt));
    cJSON *tools = cJSON_Parse(TOOLS_JSON);
    for (int turn = 1; turn <= cfg->max_turns; turn++) { ... }
    cJSON_Delete(msgs); cJSON_Delete(tools);
    return 0;
}

// in main():
agent_run(&cfg, task, log);
```

Each call to `agent_run` now creates a fresh message array, which means context does not carry between separate invocations in interactive mode. This is a deliberate simplification.

---

## 13. Symlink replaced with real header wrapper

The vendored `src/cjson/cJSON.h` was a symlink to `../cJSON.h`. v0.2.0 replaces it with a real file containing just an include directive, which is more portable across filesystems and archives.

**Initial:** symlink `src/cjson/cJSON.h -> ../cJSON.h`

**v0.2.0:**
```c
/* Vendored cJSON wrapper */
#include "../cJSON.h"
```

---

## 14. Makefile links readline

```makefile
# Before:
LDFLAGS = -lm

# After:
LDFLAGS = -lm -lreadline
```

---

## 15. Test suite expanded (14 -> 23 tests)

New tests added for:
- Sandbox execution (echo, workdir, no /home access, /bin access, workdir files, user skill creation, timeout)
- Error handling (unknown tool returns NULL, missing command field, API error responses, garbage JSON input)
- Request building (`build_request` structure validation)
- Config validation (failure without API key)

All tests updated to use the new `Config`-based `tool_execute` signature and the extracted `parse_response`/`build_request` functions.

---

## 16. Supporting files added

- **`.env.example`** — template for environment variable setup
- **`CONTRIBUTING.md`** — contribution guidelines explaining the anti-framework philosophy
- **`.gitignore`** — expanded with editor files, build artifacts, macOS metadata
- **`config.example`** — updated with sandbox options and new default model
- **`szc-logo.png`** — project logo
- **`README.md`** — rewritten with sandbox docs, updated stats, philosophy section
