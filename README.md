# smartfuzzer

**smartfuzzer** is a fast web fuzzer written in Go. It replaces the `FUZZ` keyword in URLs, headers, or POST data with entries from a wordlist, sends HTTP requests, and reports responses that match your criteria.

Use it for:
- Directory and file discovery
- Virtual host enumeration
- GET/POST parameter fuzzing
- Content discovery
- API endpoint brute-forcing

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [How Fuzzing Works](#how-fuzzing-works)
- [Tutorial](#tutorial)
  - [Step 1: Directory Discovery](#step-1-directory-discovery)
  - [Step 2: Filtering Results](#step-2-filtering-results)
  - [Step 3: Matching Specific Responses](#step-3-matching-specific-responses)
  - [Step 4: Virtual Host Discovery](#step-4-virtual-host-discovery)
  - [Step 5: POST Data Fuzzing](#step-5-post-data-fuzzing)
  - [Step 6: Recursive Scanning](#step-6-recursive-scanning)
  - [Step 7: Output to File](#step-7-output-to-file)
  - [Step 8: Interactive Mode](#step-8-interactive-mode)
  - [Step 9: Rate Limiting and Delays](#step-9-rate-limiting-and-delays)
  - [Step 10: Configuration Files](#step-10-configuration-files)
- [Options Reference](#options-reference)
- [License](#license)

## Installation

```bash
git clone https://github.com/AR-92/smartfuzzer
cd smartfuzzer
go build -o smartfuzzer .
```

Requires Go 1.17+.

## Quick Start

```bash
# Basic directory scan
./smartfuzzer -w wordlist.txt -u https://example.com/FUZZ

# With colored output and verbose mode
./smartfuzzer -w wordlist.txt -u https://example.com/FUZZ -c -v

# Filter out 403 responses, only show 200
./smartfuzzer -w wordlist.txt -u https://example.com/FUZZ -mc 200
```

## How Fuzzing Works

smartfuzzer uses the `FUZZ` keyword as a placeholder. Each line from your wordlist replaces `FUZZ` in the request, and the tool reports responses that match your criteria.

```
Wordlist: admin, login, dashboard, config, backup

Request:  https://example.com/admin
Response: 200 OK (matched)

Request:  https://example.com/login
Response: 200 OK (matched)

Request:  https://example.com/dashboard
Response: 404 Not Found (filtered out)
```

You can place `FUZZ` in:
- The URL path: `-u https://target/FUZZ`
- A header value: `-H "Host: FUZZ"`
- POST data: `-d "username=admin&password=FUZZ"`
- Multiple locations: `-w wordlist1.txt:W1 -w wordlist2.txt:W2 -u https://target/W1?param=W2`

## Tutorial

### Step 1: Directory Discovery

The most common use case — find hidden files and directories on a web server.

```bash
# Get a wordlist first
curl -sL -o common.txt https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/common.txt

# Run the scan
./smartfuzzer -w common.txt -u https://example.com/FUZZ
```

Output shows the URL, status code, size, and word count for each match:
```
admin                   [Status: 200, Size: 5432, Words: 345]
login                   [Status: 200, Size: 2100, Words: 120]
backup                  [Status: 403, Size: 300, Words: 45]
.git                    [Status: 301, Size: 0, Words: 1]
```

### Step 2: Filtering Results

Filter out noise to focus on interesting responses.

```bash
# Filter by status code (hide 404s)
./smartfuzzer -w common.txt -u https://example.com/FUZZ -fc 404

# Filter by response size (hide default "not found" pages)
./smartfuzzer -w common.txt -u https://example.com/FUZZ -fs 1234

# Filter by word count
./smartfuzzer -w common.txt -u https://example.com/FUZZ -fw 57

# Filter by line count
./smartfuzzer -w common.txt -u https://example.com/FUZZ -fl 15

# Filter by response time
./smartfuzzer -w common.txt -u https://example.com/FUZZ -ft ">500"

# Combine multiple filters
./smartfuzzer -w common.txt -u https://example.com/FUZZ -fc 404,403 -fs 1234
```

### Step 3: Matching Specific Responses

Instead of filtering out, explicitly match only certain responses.

```bash
# Match only 200 OK
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mc 200

# Match multiple status codes
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mc 200,301,302

# Match all status codes
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mc all

# Match by size
./smartfuzzer -w common.txt -u https://example.com/FUZZ -ms 5432

# Match by regex in response body
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mr "admin|dashboard"

# Match by word count
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mw 345

# Combine matchers with AND mode
./smartfuzzer -w common.txt -u https://example.com/FUZZ -mc 200 -ms 5432 -mmode and
```

### Step 4: Virtual Host Discovery

Find subdomains/virtual hosts by fuzzing the `Host` header.

```bash
# Create a wordlist of subdomains
echo -e "admin\ndev\nstaging\napi\nmail\nwww\n" > vhosts.txt

# Scan with host header fuzzing
./smartfuzzer -w vhosts.txt -u https://example.com -H "Host: FUZZ.example.com" -fs 4242
```

Filter by the default virtual host's response size to reveal hidden vhosts.

### Step 5: POST Data Fuzzing

Fuzz login forms, API endpoints, or any POST data.

```bash
# Fuzz password field
./smartfuzzer -w passwords.txt -X POST -d "username=admin&password=FUZZ" -u https://example.com/login -fc 401

# Fuzz JSON body
./smartfuzzer -w payloads.txt -X POST -H "Content-Type: application/json" -d '{"user":"FUZZ","pass":"test"}' -u https://example.com/api/login -mc 200

# Fuzz multiple parameters at once
./smartfuzzer -w users.txt:USER -w passes.txt:PASS -X POST -d "username=USER&password=PASS" -u https://example.com/login -fc 401
```

### Step 6: Recursive Scanning

Automatically discover directories and then scan inside them.

```bash
# Recursive scan with depth limit
./smartfuzzer -w common.txt -u https://example.com/FUZZ -recursion -recursion-depth 2

# Recursive scan with max time per job
./smartfuzzer -w common.txt -u https://example.com/FUZZ -recursion -recursion-depth 3 -maxtime-job 120

# Greedy recursion (recurse on all matches, not just redirects)
./smartfuzzer -w common.txt -u https://example.com/FUZZ -recursion -recursion-strategy greedy
```

### Step 7: Output to File

Save results in various formats for analysis.

```bash
# JSON output
./smartfuzzer -w common.txt -u https://example.com/FUZZ -o results.json -of json

# HTML report
./smartfuzzer -w common.txt -u https://example.com/FUZZ -o report.html -of html

# Markdown
./smartfuzzer -w common.txt -u https://example.com/FUZZ -o results.md -of md

# CSV
./smartfuzzer -w common.txt -u https://example.com/FUZZ -o results.csv -of csv

# All formats at once
./smartfuzzer -w common.txt -u https://example.com/FUZZ -o results -of all

# Save matched results to a directory
./smartfuzzer -w common.txt -u https://example.com/FUZZ -od ./matches
```

### Step 8: Interactive Mode

Press `ENTER` during a running scan to pause and modify filters in real time.

```bash
# Start a scan
./smartfuzzer -w common.txt -u https://example.com/FUZZ -c

# Press ENTER when you see too much noise
# You'll see:
entering interactive mode
>

# Add a filter to hide 403s
> fc 403

# Check your current results
> show

# Resume scanning
> resume
```

Available interactive commands:

| Command | Description |
|---------|-------------|
| `fc [value]` | (re)configure status code filter |
| `afc [value]` | append to status code filter |
| `fl [value]` | (re)configure line count filter |
| `afl [value]` | append to line count filter |
| `fw [value]` | (re)configure word count filter |
| `afw [value]` | append to word count filter |
| `fs [value]` | (re)configure size filter |
| `afs [value]` | append to size filter |
| `ft [value]` | (re)configure time filter |
| `aft [value]` | append to time filter |
| `rate [value]` | adjust requests per second |
| `queueshow` | show job queue |
| `queuedel [n]` | delete a job from queue |
| `queueskip` | skip to next queued job |
| `restart` | restart current job from beginning |
| `resume` | resume current job (or ENTER) |
| `show` | display current matches |
| `savejson [file]` | save matches to JSON file |
| `help` | show help |

### Step 9: Rate Limiting and Delays

Avoid overwhelming the target or getting blocked.

```bash
# Limit to 100 requests per second
./smartfuzzer -w common.txt -u https://example.com/FUZZ -rate 100

# Add a fixed 0.5 second delay between requests
./smartfuzzer -w common.txt -u https://example.com/FUZZ -p 0.5

# Add a random delay between 0.1 and 1.0 seconds
./smartfuzzer -w common.txt -u https://example.com/FUZZ -p 0.1-1.0

# Limit total scan time to 5 minutes
./smartfuzzer -w common.txt -u https://example.com/FUZZ -maxtime 300
```

### Step 10: Configuration Files

Create a default config file to avoid typing the same flags every time.

```bash
# Create config directory
mkdir -p ~/.config/smartfuzzer

# Create smartfuzzerrc
cat > ~/.config/smartfuzzer/smartfuzzerrc << 'EOF'
# Default options applied to every run
-c                    # colored output
-t 80                 # 80 threads
-rate 200             # 200 req/s max
-timeout 10           # 10 second timeout
EOF

# Now you can run with fewer flags:
./smartfuzzer -w common.txt -u https://example.com/FUZZ

# Use a custom config file
./smartfuzzer -config ./myconfig.conf -w common.txt -u https://example.com/FUZZ
```

## Options Reference

### HTTP Options

| Flag | Description | Default |
|------|-------------|---------|
| `-H` | Header `"Name: Value"`, multiple accepted | |
| `-X` | HTTP method | GET |
| `-b` | Cookie data `"NAME1=VALUE1; NAME2=VALUE2"` | |
| `-cc` | Client certificate path | |
| `-ck` | Client key path | |
| `-d` | POST data | |
| `-http2` | Use HTTP2 | false |
| `-ignore-body` | Don't fetch response body | false |
| `-r` | Follow redirects | false |
| `-raw` | Don't encode URI | false |
| `-recursion` | Scan recursively | false |
| `-recursion-depth` | Max recursion depth | 0 |
| `-recursion-strategy` | `default` or `greedy` | default |
| `-replay-proxy` | Proxy for replayed requests | |
| `-sni` | TLS SNI value | |
| `-timeout` | Request timeout in seconds | 10 |
| `-u` | Target URL | |
| `-x` | Proxy URL (HTTP/SOCKS5) | |

### General Options

| Flag | Description | Default |
|------|-------------|---------|
| `-V` | Show version | false |
| `-ac` | Auto-calibrate filter | false |
| `-acc` | Custom auto-calibration string | |
| `-ach` | Per-host auto-calibration | false |
| `-ack` | Auto-calibration keyword | FUZZ |
| `-acs` | Custom auto-calibration strategy | |
| `-c` | Colorize output | false |
| `-config` | Config file path | |
| `-json` | JSON output (NDJSON) | false |
| `-maxtime` | Max total runtime in seconds | 0 |
| `-maxtime-job` | Max job runtime in seconds | 0 |
| `-noninteractive` | Disable interactive mode | false |
| `-p` | Delay (fixed or range) | |
| `-rate` | Requests per second | 0 |
| `-s` | Silent mode | false |
| `-sa` | Stop on all errors | false |
| `-scraperfile` | Custom scraper file | |
| `-scrapers` | Active scraper groups | all |
| `-se` | Stop on spurious errors | false |
| `-search` | Search for FFUFHASH in history | |
| `-sf` | Stop on 95%+ 403s | false |
| `-t` | Number of threads | 40 |
| `-v` | Verbose output | false |

### Matcher Options

| Flag | Description | Default |
|------|-------------|---------|
| `-mc` | Match status codes | 200-299,301,302,307,401,403,405,500 |
| `-ml` | Match line count | |
| `-mmode` | Matcher operator: `and`/`or` | or |
| `-mr` | Match regex | |
| `-ms` | Match response size | |
| `-mt` | Match response time (ms) | |
| `-mw` | Match word count | |

### Filter Options

| Flag | Description |
|------|-------------|
| `-fc` | Filter status codes |
| `-fl` | Filter line count |
| `-fmode` | Filter operator: `and`/`or` (default: or) |
| `-fr` | Filter regex |
| `-fs` | Filter response size |
| `-ft` | Filter response time (ms) |
| `-fw` | Filter word count |

### Input Options

| Flag | Description | Default |
|------|-------------|---------|
| `-D` | DirSearch compatibility | false |
| `-e` | File extensions (comma separated) | |
| `-enc` | Encoders for keywords | |
| `-ic` | Ignore wordlist comments | false |
| `-input-cmd` | Command to produce input | |
| `-input-num` | Number of inputs | 100 |
| `-input-shell` | Shell for input command | |
| `-mode` | Multi-wordlist mode: `clusterbomb`, `pitchfork`, `sniper` | clusterbomb |
| `-request` | Raw HTTP request file | |
| `-request-proto` | Protocol for raw request | https |
| `-w` | Wordlist file (path:KEYWORD) | |

### Output Options

| Flag | Description | Default |
|------|-------------|---------|
| `-audit-log` | Audit log file | |
| `-debug-log` | Debug log file | |
| `-o` | Output file | |
| `-od` | Output directory for matches | |
| `-of` | Output format: json, ejson, html, md, csv, ecsv | json |
| `-or` | Skip output file if no results | false |

## License

MIT
