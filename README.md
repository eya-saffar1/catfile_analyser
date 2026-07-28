#!/usr/bin/env bash

set -euo pipefail

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <cat_file> <ticket_number>" >&2
    exit 1
fi

CAT_FILE="$1"
TICKET_NUMBER="$2"

if [ ! -f "$CAT_FILE" ]; then
    echo "Error: file not found: $CAT_FILE" >&2
    exit 1
fi

awk -v ticket="$TICKET_NUMBER" '
function has_ticket_number(line) {
    # Search the ticket number anywhere inside the line.
    return index(line, ticket) > 0
}

{
    lines[NR] = $0
}

END {
    header_start = 0
    header_end = 0
    footer_start = 0
    footer_end = 0
    ticket_line = 0
    tkt_start = 0
    tkp_end = 0

    # Find header: TTH -> TOH
    for (i = 1; i <= NR; i++) {
        if (lines[i] ~ /^[[:space:]]*TTH/) {
            header_start = i
        }

        if (header_start && lines[i] ~ /^[[:space:]]*TOH/) {
            header_end = i
            break
        }
    }

    # Find footer: from first TOT to end of file
    for (i = 1; i <= NR; i++) {
        if (lines[i] ~ /^[[:space:]]*TOT/) {
            footer_start = i
            footer_end = NR
            break
        }
    }

    if (!header_start || !header_end) {
        print "Error: header not found. Expected lines from TTH to TOH." > "/dev/stderr"
        exit 2
    }

    if (!footer_start) {
        print "Error: footer not found. Expected a line starting with TOT." > "/dev/stderr"
        exit 3
    }

    # Search for the ticket number after the header and before the footer
    for (i = header_end + 1; i < footer_start; i++) {
        if (has_ticket_number(lines[i])) {
            ticket_line = i
            break
        }
    }

    if (!ticket_line) {
        print "Error: ticket number " ticket " not found." > "/dev/stderr"
        exit 4
    }

    # Go up from the found ticket number until the previous TKT
    for (i = ticket_line; i >= 1; i--) {
        if (lines[i] ~ /^[[:space:]]*TKT/) {
            tkt_start = i
            break
        }
    }

    if (!tkt_start) {
        print "Error: found ticket number, but no TKT line before it." > "/dev/stderr"
        exit 5
    }

    # Go down from TKT until TKP
    for (i = tkt_start; i <= NR; i++) {
        if (lines[i] ~ /^[[:space:]]*TKP/) {
            tkp_end = i
            break
        }
    }

    if (!tkp_end) {
        print "Error: found TKT, but no TKP line after it." > "/dev/stderr"
        exit 6
    }

    # Print header
    for (i = header_start; i <= header_end; i++) {
        print lines[i]
    }

    # Print only the selected ticket
    for (i = tkt_start; i <= tkp_end; i++) {
        print lines[i]
    }

    # Print footer from TOT to end of file
    for (i = footer_start; i <= footer_end; i++) {
        print lines[i]
    }
}
' "$CAT_FILE"
