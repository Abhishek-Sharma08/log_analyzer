# Go Log Analyzer

![Go
Version](https://img.shields.io/badge/Go-1.21%2B-00ADD8?style=flat&logo=go)
![Dependencies](https://img.shields.io/badge/Dependencies-Standard%20Library-brightgreen?style=flat)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat)

A lightweight command-line log analyzer written in Go. It reads log
files incrementally, filters entries by severity or keyword, and reports
useful summary statistics without loading the entire file into memory.

> **Project status:** This README describes the intended MVP interface
> and behaviour. Keep the feature list, examples, and roadmap in sync
> with the implementation as features are completed.

## Features

The MVP is designed to provide:

-   **Streaming file processing:** Read a file incrementally instead of
    loading the complete file into memory.
-   **Severity filtering:** Select entries with levels such as `INFO`,
    `WARN`, or `ERROR`.
-   **Keyword search:** Find entries containing a specified text string.
-   **Combined filters:** Apply severity and keyword filters together.
-   **Summary statistics:** Count valid entries by severity and report
    malformed lines.
-   **Readable terminal output:** Use ANSI colours to make warnings and
    errors easier to spot.
-   **Standard-library implementation:** No third-party Go modules are
    required.

## Project structure

The initial version keeps the code in one Go package. This is
intentionally simple for a small CLI application.

``` text
log-analyzer/
├── go.mod
├── main.go          # CLI flags, file handling, and application flow
├── parser.go        # Converts raw lines into LogEntry values
├── analyzer.go      # Filtering and statistics
├── sample.log       # Small example log file
├── README.md
└── LICENSE
```

As the application grows, terminal formatting can move into an
`output.go` file and tests can be added under `*_test.go` files.

## How it works

The application follows a streaming pipeline:

``` text
Command-line flags
       |
       v
Open log file
       |
       v
Read one line at a time
       |
       v
Parse line into LogEntry
       |
       +---- malformed/blank ----> increment skipped-line count
       |
       v
Update overall statistics
       |
       v
Apply active filters
       |
       +---- no match -----------> continue
       |
       v
Print matching entry
       |
       v
Print summary at end of file
```

### Responsibilities

1.  **`main.go` --- CLI and orchestration:** Reads command-line flags,
    opens the input file, streams lines, and coordinates the other
    components.
2.  **`parser.go` --- parsing:** Converts a raw line into a structured
    `LogEntry`. Invalid or unsupported lines are reported to the caller
    as unparsed.
3.  **`analyzer.go` --- filtering and statistics:** Counts all valid
    parsed entries and determines whether each entry matches the
    requested filters.
4.  **Terminal output:** Displays matching entries as they are processed
    and prints the final summary.

## Requirements

-   Go **1.21 or later**
-   A terminal that supports standard command-line applications

Check your Go installation:

``` bash
go version
```

## Installation

Clone the repository:

``` bash
git clone https://github.com/yourusername/log-analyzer.git
cd log-analyzer
```

Replace `yourusername` with your GitHub username and adjust the
repository URL if you choose a different name.

Build the executable:

``` bash
go build -o log-analyzer .
```

Run it:

``` bash
./log-analyzer --file sample.log
```

You can also run the project without building a separate executable:

``` bash
go run . --file sample.log
```

## Usage

### Command-line flags

  -----------------------------------------------------------------------
  Flag              Type              Default           Description
  ----------------- ----------------- ----------------- -----------------
  `--file`          string            empty             Path to the log
                                                        file to analyze.
                                                        Required.

  `--level`         string            empty             Show matching
                                                        entries for a
                                                        severity such as
                                                        `INFO`, `WARN`,
                                                        or `ERROR`.

  `--search`        string            empty             Show entries
                                                        containing the
                                                        specified text.
  -----------------------------------------------------------------------

The Go standard-library `flag` package accepts either one or two leading
hyphens, so `-file` and `--file` are equivalent.

### Examples

Analyze the complete file and print a summary:

``` bash
./log-analyzer --file sample.log
```

Show only error entries:

``` bash
./log-analyzer --file sample.log --level ERROR
```

Search for a keyword:

``` bash
./log-analyzer --file sample.log --search "ConnectionTimeout"
```

Combine severity and keyword filters:

``` bash
./log-analyzer --file sample.log --level WARN --search "latency"
```

When both `--level` and `--search` are supplied, an entry should match
**both** conditions to be displayed (AND behaviour).

## Expected log format

The initial parser expects one log entry per line in this format:

``` text
YYYY-MM-DD HH:MM:SS [LEVEL] Message
```

Example:

``` text
2026-11-15 14:20:10 [INFO] Server started on port 8080
2026-11-15 14:21:05 [WARN] High memory usage detected: 82%
2026-11-15 14:22:01 [ERROR] Database connection failed: ConnectionTimeout
2026-11-15 14:23:12 [INFO] Health check completed
```

The parser should recognise the timestamp, severity, and remaining
message text. Blank lines and lines that do not follow the supported
format should be counted as malformed or skipped rather than causing the
program to panic.

## Statistics and filtering behaviour

To keep the summary meaningful, overall statistics and filtered output
are separate:

-   **Total lines:** Every line read from the input file.
-   **Valid entries:** Lines successfully parsed.
-   **Malformed/blank lines:** Lines that could not be parsed.
-   **Level counts:** Counts for all valid entries, regardless of active
    display filters.
-   **Matching entries:** Valid entries that satisfy the active filters.

For example, using `--level ERROR` should limit the displayed entries,
but the overall `INFO`, `WARN`, and `ERROR` counts should still describe
the entire file.

## Memory and large-file handling

The analyzer processes the file incrementally, so it does not need to
keep the entire log file or every parsed entry in memory.

If the implementation uses `bufio.Scanner`, remember that its default
maximum token size is limited. Configure an appropriate buffer and
maximum line size, or handle scanner errors clearly. A line that exceeds
the configured limit must not be silently treated as successfully
processed.

Memory usage is therefore tied mainly to the current line, the
configured line-size limit, and the statistics being collected---not the
total size of the input file. Actual performance depends on the input,
hardware, and implementation; benchmark results will be added only after
they have been measured.

## Example output

The following is an illustrative example. Exact wording and formatting
may change as the implementation develops.

``` text
2026-11-15 14:22:01 [ERROR] Database connection failed: ConnectionTimeout
2026-11-15 14:25:43 [ERROR] Unhandled exception in worker goroutine #4

--- Log Analysis Summary ---
Total lines read : 1,250
Valid entries    : 1,250
Malformed lines  : 0
Matching entries : 30

Log levels:
[INFO]           : 1,102
[WARN]           : 118
[ERROR]          : 30
```

## Development

Format the Go source files:

``` bash
gofmt -w .
```

Run the tests:

``` bash
go test ./...
```

Run the Go static analysis tool:

``` bash
go vet ./...
```

Build the application:

``` bash
go build -o log-analyzer .
```

## Roadmap

Planned improvements:

-   [ ] Add unit tests for valid, malformed, and empty log lines.
-   [ ] Add integration tests for CLI flags and file errors.
-   [ ] Add regular-expression filtering with a `--regex` flag.
-   [ ] Add JSON summary export with a `--json` flag.
-   [ ] Add date/time range filtering.
-   [ ] Add top IP addresses and HTTP status-code summaries for
    supported access-log formats.
-   [ ] Add benchmarks for different input sizes and report measured
    throughput and memory usage.
-   [ ] Evaluate concurrent processing for workloads where it provides a
    measurable benefit.

## Contributing

Contributions, bug reports, and suggestions are welcome.

1.  Fork the repository.

2.  Create a branch: `git checkout -b feature/your-change`.

3.  Make your changes and add or update tests.

4.  Format and test the code:

    ``` bash
    gofmt -w .
    go test ./...
    go vet ./...
    ```

5.  Commit your changes: `git commit -m "Describe your change"`.

6.  Push your branch and open a Pull Request.

## License

This project is intended to be released under the MIT License. See the
[`LICENSE`](LICENSE) file for the full license text.
