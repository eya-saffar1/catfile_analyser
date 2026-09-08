#!/usr/bin/env bash

set -euo pipefail

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <hot_file> <ticket_number>" >&2
    exit 1
fi

HOT_FILE="$1"
TICKET_NUMBER="$2"

if [ ! -f "$HOT_FILE" ]; then
    echo "Error: file not found: $HOT_FILE" >&2
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
    bkt_start = 0
    bkp_end = 0

    bft_line = 0
    last_bkp_before_bft = 0
    footer_start = 0
    footer_end = 0

    # Find header: BFH -> BOH
    for (i = 1; i <= NR; i++) {
        if (starts_with(lines[i], "BFH")) {
            header_start = i
        }

        if (header_start && starts_with(lines[i], "BOH")) {
            header_end = i
            break
        }
    }

    if (!header_start || !header_end) {
        print "Error: header not found. Expected lines from BFH to BOH." > "/dev/stderr"
        exit 2
    }

    # Find BFT: end of footer
    for (i = 1; i <= NR; i++) {
        if (starts_with(lines[i], "BFT")) {
            bft_line = i
            footer_end = i
            break
        }
    }

    if (!bft_line) {
        print "Error: BFT not found." > "/dev/stderr"
        exit 3
    }

    # From BFT, go up to find the previous BKP
    for (i = bft_line - 1; i >= 1; i--) {
        if (starts_with(lines[i], "BKP")) {
            last_bkp_before_bft = i
            break
        }
    }

    if (!last_bkp_before_bft) {
        print "Error: no BKP found before BFT." > "/dev/stderr"
        exit 4
    }

    # From that BKP, go down to find the first BOT
    # This allows multiple BOT blocks between BKP and BFT.
    for (i = last_bkp_before_bft + 1; i <= bft_line; i++) {
        if (starts_with(lines[i], "BOT")) {
            footer_start = i
            break
        }
    }

    if (!footer_start) {
        print "Error: no BOT found between last BKP and BFT." > "/dev/stderr"
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

    # Go up from the found ticket number until the previous BKT
    for (i = ticket_line; i >= header_end + 1; i--) {
        if (starts_with(lines[i], "BKT")) {
            bkt_start = i
            break
        }
    }

    if (!bkt_start) {
        print "Error: found ticket number, but no BKT line before it." > "/dev/stderr"
        exit 7
    }

    # Go down from BKT until BKP
    for (i = bkt_start; i < footer_start; i++) {
        if (starts_with(lines[i], "BKP")) {
            bkp_end = i
            break
        }
    }

    if (!bkp_end) {
        print "Error: found BKT, but no BKP line after it." > "/dev/stderr"
        exit 8
    }

    # Print header
    for (i = header_start; i <= header_end; i++) {
        print lines[i]
    }

    # Print only the selected ticket
    for (i = bkt_start; i <= bkp_end; i++) {
        print lines[i]
    }

    # Print footer: from first BOT after last BKP before BFT, until BFT
    for (i = footer_start; i <= footer_end; i++) {
        print lines[i]
    }
}
' "$HOT_FILE"
