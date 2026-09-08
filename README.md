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

function starts_with(line, code) {
    return toupper(line) ~ ("^[[:space:]]*" code)
}

{
    lines[NR] = $0
}

END {
    header_start = 0
    header_end = 0

    ticket_line = 0
    tkt_start = 0
    tkp_end = 0

    ttt_line = 0
    last_tkp_before_ttt = 0
    footer_start = 0
    footer_end = 0

    # Find header: TTH -> TOH
    for (i = 1; i <= NR; i++) {
        if (starts_with(lines[i], "TTH")) {
            header_start = i
        }

        if (header_start && starts_with(lines[i], "TOH")) {
            header_end = i
            break
        }
    }

    if (!header_start || !header_end) {
        print "Error: header not found. Expected lines from TTH to TOH." > "/dev/stderr"
        exit 2
    }

    # Find TTT: end of footer
    for (i = 1; i <= NR; i++) {
        if (starts_with(lines[i], "TTT")) {
            ttt_line = i
            footer_end = i
            break
        }
    }

    if (!ttt_line) {
        print "Error: TTT not found." > "/dev/stderr"
        exit 3
    }

    # From TTT, go up to find the previous TKP
    for (i = ttt_line - 1; i >= 1; i--) {
        if (starts_with(lines[i], "TKP")) {
            last_tkp_before_ttt = i
            break
        }
    }

    if (!last_tkp_before_ttt) {
        print "Error: no TKP found before TTT." > "/dev/stderr"
        exit 4
    }

    # From that TKP, go down to find the first TOT
    # This allows multiple TOT blocks between TKP and TTT.
    for (i = last_tkp_before_ttt + 1; i <= ttt_line; i++) {
        if (starts_with(lines[i], "TOT")) {
            footer_start = i
            break
        }
    }

    if (!footer_start) {
        print "Error: no TOT found between last TKP and TTT." > "/dev/stderr"
        exit 5
    }

    # Search for the requested ticket number after header and before footer
    for (i = header_end + 1; i < footer_start; i++) {
        if (has_ticket_number(lines[i])) {
            ticket_line = i
            break
        }
    }

    if (!ticket_line) {
        print "Error: ticket number " ticket " not found." > "/dev/stderr"
        exit 6
    }

    # Go up from the found ticket number until the previous TKT
    for (i = ticket_line; i >= header_end + 1; i--) {
        if (starts_with(lines[i], "TKT")) {
            tkt_start = i
            break
        }
    }

    if (!tkt_start) {
        print "Error: found ticket number, but no TKT line before it." > "/dev/stderr"
        exit 7
    }

    # Go down from TKT until TKP
    for (i = tkt_start; i < footer_start; i++) {
        if (starts_with(lines[i], "TKP")) {
            tkp_end = i
            break
        }
    }

    if (!tkp_end) {
        print "Error: found TKT, but no TKP line after it." > "/dev/stderr"
        exit 8
    }

    # Print header
    for (i = header_start; i <= header_end; i++) {
        print lines[i]
    }

    # Print only the selected ticket
    for (i = tkt_start; i <= tkp_end; i++) {
        print lines[i]
    }

    # Print footer: from first TOT after last TKP before TTT, until TTT
    for (i = footer_start; i <= footer_end; i++) {
        print lines[i]
    }
}
' "$CAT_FILE"
