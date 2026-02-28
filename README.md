# Linux-monitoring-scropt
This script monitors key metrics in Linux
#!/usr/bin/env bash
# ══════════════════════════════════════════════════════════════════════════════
#  sysmon.sh — Linux System Monitor
#  Senior Linux Admin | Checks: Disk I/O · CPU Utilization · Load Average
#                               · Bandwidth Usage
#
#  Usage:
#    chmod +x sysmon.sh
#    ./sysmon.sh              # Single snapshot, pretty-printed
#    ./sysmon.sh -w           # Watch mode: refreshes every 5s (live dashboard)
#    ./sysmon.sh -i 10        # Custom interval (seconds) in watch mode
#    ./sysmon.sh -l           # Log mode: append each snapshot to sysmon.log
#    ./sysmon.sh -t 80        # Alert if CPU% exceeds threshold (default: 90)
#    ./sysmon.sh -n eth0      # Specify network interface (default: auto-detect)
#    ./sysmon.sh -h           # Help
#
#  Dependencies (auto-detected, graceful fallback if missing):
#    iostat  (sysstat package)  — disk I/O
#    mpstat  (sysstat package)  — CPU per-core
#    ifstat  (ifstat package)   — bandwidth
#    vmstat                     — load/memory (usually built-in)
#    awk, grep, sed, top, df, free, uptime, ip/ifconfig — standard utils
# ══════════════════════════════════════════════════════════════════════════════

set -euo pipefail

# ─────────────────────────────────────────────────────────────────────────────
# CONFIGURATION — override via CLI flags
# ─────────────────────────────────────────────────────────────────────────────
WATCH_MODE=false
INTERVAL=5              # seconds between refreshes in watch mode
LOG_MODE=false
LOG_FILE="sysmon.log"
CPU_ALERT_THRESHOLD=90  # % CPU — will print alert if exceeded
NET_IFACE=""            # empty = auto-detect first non-loopback iface
SAMPLE_DURATION=1       # seconds to sample I/O & bandwidth (per snapshot)

# ─────────────────────────────────────────────────────────────────────────────
# COLORS & SYMBOLS
# ─────────────────────────────────────────────────────────────────────────────
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
CYAN='\033[0;36m'
BLUE='\033[0;34m'
MAGENTA='\033[0;35m'
WHITE='\033[1;37m'
BOLD='\033[1m'
DIM='\033[2m'
RESET='\033[0m'

OK="${GREEN}✔${RESET}"
WARN="${YELLOW}⚠${RESET}"
CRIT="${RED}✘${RESET}"
ARROW="${CYAN}▶${RESET}"

# ─────────────────────────────────────────────────────────────────────────────
# HELPERS
# ─────────────────────────────────────────────────────────────────────────────
has_cmd() { command -v "$1" &>/dev/null; }

print_header() {
    local title="$1"
    local width=70
    local line
    line=$(printf '─%.0s' $(seq 1 $width))
    echo -e "${BOLD}${BLUE}┌${line}┐${RESET}"
    printf "${BOLD}${BLUE}│${RESET}  ${WHITE}%-66s${RESET}${BOLD}${BLUE}│${RESET}\n" "$title"
    echo -e "${BOLD}${BLUE}└${line}┘${RESET}"
}

print_section() {
    local title="$1"
    echo ""
    echo -e "${BOLD}${CYAN}╔══ ${title} ${RESET}${DIM}$(printf '═%.0s' $(seq 1 $((58 - ${#title}))))${RESET}"
}

print_row() {
    local label="$1"
    local value="$2"
    local color="${3:-$WHITE}"
    printf "  ${DIM}%-28s${RESET} ${color}%s${RESET}\n" "$label" "$value"
}

progress_bar() {
    local pct="${1%.*}"   # strip decimals
    pct="${pct:-0}"
    [[ "$pct" -gt 100 ]] && pct=100
    local filled=$(( pct * 30 / 100 ))
    local empty=$(( 30 - filled ))
    local bar=""
    local color

    if   [[ "$pct" -ge 90 ]]; then color="$RED"
    elif [[ "$pct" -ge 70 ]]; then color="$YELLOW"
    else                            color="$GREEN"
    fi

    bar+="${color}"
    for (( i=0; i<filled; i++ )); do bar+="█"; done
    bar+="${DIM}"
    for (( i=0; i<empty;  i++ )); do bar+="░"; done
    bar+="${RESET}"

    printf "[%s] %s%3d%%%s" "$bar" "$color" "$pct" "$RESET"
}

alert_icon() {
    local val="${1%.*}"
    local warn="$2"
    local crit="$3"
    if   [[ "$val" -ge "$crit" ]]; then echo -e "$CRIT"
    elif [[ "$val" -ge "$warn" ]]; then echo -e "$WARN"
    else                                echo -e "$OK"
    fi
}

timestamp() { date '+%Y-%m-%d %H:%M:%S'; }

log_entry() {
    if [[ "$LOG_MODE" == true ]]; then
        echo "$1" >> "$LOG_FILE"
    fi
}

# ─────────────────────────────────────────────────────────────────────────────
# AUTO-DETECT NETWORK INTERFACE
# ─────────────────────────────────────────────────────────────────────────────
detect_iface() {
    if [[ -n "$NET_IFACE" ]]; then return; fi

    if has_cmd ip; then
        NET_IFACE=$(ip route get 8.8.8.8 2>/dev/null \
            | awk '/dev/ {for(i=1;i<=NF;i++) if($i=="dev") {print $(i+1); exit}}')
    fi

    # Fallback: first non-loopback interface
    if [[ -z "$NET_IFACE" ]]; then
        if has_cmd ip; then
            NET_IFACE=$(ip -o link show | awk -F': ' '$3 !~ /LOOPBACK/ {print $2; exit}')
        elif has_cmd ifconfig; then
            NET_IFACE=$(ifconfig -a | awk '/^[a-z]/ && !/lo/ {print $1; exit}' | tr -d ':')
        fi
    fi

    NET_IFACE="${NET_IFACE:-eth0}"
}

# ─────────────────────────────────────────────────────────────────────────────
# SECTION 1 — CPU UTILIZATION
# ─────────────────────────────────────────────────────────────────────────────
check_cpu() {
    print_section "CPU UTILIZATION"

    # ── Overall CPU % via /proc/stat (no external deps needed)
    local cpu_idle cpu_total cpu_used_pct

    read -r cpu_line1 < /proc/stat
    sleep "$SAMPLE_DURATION"
    read -r cpu_line2 < /proc/stat

    # Parse user nice system idle iowait irq softirq
    read -r _ u1 n1 s1 id1 iw1 irq1 sirq1 _ <<< "$cpu_line1"
    read -r _ u2 n2 s2 id2 iw2 irq2 sirq2 _ <<< "$cpu_line2"

    local total1=$(( u1 + n1 + s1 + id1 + iw1 + irq1 + sirq1 ))
    local total2=$(( u2 + n2 + s2 + id2 + iw2 + irq2 + sirq2 ))
    local delta_total=$(( total2 - total1 ))
    local delta_idle=$(( id2 - id1 ))
    local delta_iowait=$(( iw2 - iw1 ))
    local delta_user=$(( (u2+n2) - (u1+n1) ))
    local delta_sys=$(( s2 - s1 ))

    if [[ "$delta_total" -eq 0 ]]; then delta_total=1; fi  # avoid div/0

    cpu_used_pct=$(( (delta_total - delta_idle) * 100 / delta_total ))
    local user_pct=$(( delta_user   * 100 / delta_total ))
    local sys_pct=$(( delta_sys     * 100 / delta_total ))
    local iowait_pct=$(( delta_iowait * 100 / delta_total ))
    local idle_pct=$(( delta_idle   * 100 / delta_total ))

    local icon; icon=$(alert_icon "$cpu_used_pct" 70 "$CPU_ALERT_THRESHOLD")

    printf "  %b Overall CPU Used  " "$icon"
    progress_bar "$cpu_used_pct"
    echo ""

    print_row "  User / Nice"    "${user_pct}%"    "$CYAN"
    print_row "  System"         "${sys_pct}%"     "$MAGENTA"
    print_row "  I/O Wait"       "${iowait_pct}%"  "$YELLOW"
    print_row "  Idle"           "${idle_pct}%"    "$GREEN"

    # ── CPU core count & model
    local cores freq model
    cores=$(nproc 2>/dev/null || grep -c "^processor" /proc/cpuinfo)
    model=$(grep "model name" /proc/cpuinfo | head -1 | cut -d: -f2 | xargs)
    freq=$(grep "cpu MHz" /proc/cpuinfo | head -1 | awk '{printf "%.0f MHz", $4}' 2>/dev/null || echo "N/A")
    print_row "  CPU Model"      "${model:-Unknown}"  "$DIM"
    print_row "  Cores / Freq"   "${cores} cores @ ${freq}"  "$DIM"

    # ── Per-core breakdown (via mpstat if available)
    if has_cmd mpstat; then
        echo ""
        echo -e "  ${DIM}Per-Core Utilization (mpstat):${RESET}"
        mpstat -P ALL 1 1 2>/dev/null \
            | awk '/^[0-9]/ && $3 ~ /^[0-9]/ {
                idle=$NF; used=100-idle;
                printf "    Core %-4s %5.1f%%\n", $3, used
              }' | head -16
    fi

    # ── Alert
    if [[ "$cpu_used_pct" -ge "$CPU_ALERT_THRESHOLD" ]]; then
        echo -e "  ${RED}${BOLD}⚠ ALERT: CPU utilization (${cpu_used_pct}%) exceeds threshold (${CPU_ALERT_THRESHOLD}%)!${RESET}"
    fi

    log_entry "$(timestamp) CPU_USED=${cpu_used_pct}% USER=${user_pct}% SYS=${sys_pct}% IOWAIT=${iowait_pct}%"
}

# ─────────────────────────────────────────────────────────────────────────────
# SECTION 2 — LOAD AVERAGE
# ─────────────────────────────────────────────────────────────────────────────
check_load() {
    print_section "LOAD AVERAGE"

    local cores; cores=$(nproc 2>/dev/null || echo 1)

    # /proc/loadavg: 1min 5min 15min running/total lastpid
    read -r la1 la5 la15 procs _ < /proc/loadavg

    # Color-code based on load vs core count
    load_color() {
        local la="$1"
        local ratio; ratio=$(awk "BEGIN{printf \"%.0f\", ($la/$cores)*100}")
        if   [[ "$ratio" -ge 100 ]]; then echo "$RED"
        elif [[ "$ratio" -ge  75 ]]; then echo "$YELLOW"
        else                               echo "$GREEN"
        fi
    }

    local c1; c1=$(load_color "$la1")
    local c5; c5=$(load_color "$la5")
    local c15; c15=$(load_color "$la15")

    printf "  %b 1-min  Load Avg   " "$(alert_icon "$(awk "BEGIN{printf \"%.0f\", ($la1/$cores)*100}")" 75 100)"
    echo -e " ${c1}${BOLD}${la1}${RESET}  ${DIM}(${cores} cores)${RESET}"

    printf "  %b 5-min  Load Avg   " "$(alert_icon "$(awk "BEGIN{printf \"%.0f\", ($la5/$cores)*100}")" 75 100)"
    echo -e " ${c5}${BOLD}${la5}${RESET}"

    printf "  %b 15-min Load Avg   " "$(alert_icon "$(awk "BEGIN{printf \"%.0f\", ($la15/$cores)*100}")" 75 100)"
    echo -e " ${c15}${BOLD}${la15}${RESET}"

    # Running processes
    local running total
    running=$(echo "$procs" | cut -d/ -f1)
    total=$(echo "$procs"   | cut -d/ -f2)
    print_row "  Processes"         "${running} running / ${total} total"  "$WHITE"
    print_row "  Uptime"            "$(uptime -p 2>/dev/null || uptime | awk -F'up ' '{print $2}' | cut -d, -f1-2 | xargs)"  "$DIM"
    print_row "  Logged-in Users"   "$(who | wc -l)"  "$DIM"

    # Rule of thumb guidance
    echo ""
    local la1_pct; la1_pct=$(awk "BEGIN{printf \"%.1f\", ($la1/$cores)*100}")
    if awk "BEGIN{exit !($la1 >= $cores)}"; then
        echo -e "  ${RED}${BOLD}⚠ OVERLOADED: 1-min load (${la1}) ≥ CPU cores (${cores}). System is saturated.${RESET}"
    elif awk "BEGIN{exit !($la1 >= $cores * 0.75)}"; then
        echo -e "  ${YELLOW}⚠ HIGH LOAD: 1-min load at ${la1_pct}% of capacity. Monitor closely.${RESET}"
    else
        echo -e "  ${GREEN}✔ Load is healthy (${la1_pct}% of ${cores}-core capacity).${RESET}"
    fi

    log_entry "$(timestamp) LOAD_1=${la1} LOAD_5=${la5} LOAD_15=${la15} PROCS=${procs}"
}

# ─────────────────────────────────────────────────────────────────────────────
# SECTION 3 — DISK I/O
# ─────────────────────────────────────────────────────────────────────────────
check_disk_io() {
    print_section "DISK I/O"

    # ── iostat (best method)
    if has_cmd iostat; then
        echo -e "  ${DIM}Device           r/s        w/s       rkB/s      wkB/s    %util${RESET}"
        echo -e "  ${DIM}$(printf '─%.0s' $(seq 1 65))${RESET}"

        iostat -dx "$SAMPLE_DURATION" 2 2>/dev/null \
            | awk 'NR>6 && /^[a-z]/ && !/loop/ {
                printf "  %-16s %8.1f   %8.1f   %9.1f  %9.1f   %6.1f%%\n",
                       $1, $4, $5, $6, $7, $NF
              }' | while IFS= read -r line; do
                    util=$(echo "$line" | awk '{gsub(/%/,""); print $NF}')
                    if awk "BEGIN{exit !($util >= 90)}"; then
                        echo -e "${RED}${line}${RESET}"
                    elif awk "BEGIN{exit !($util >= 70)}"; then
                        echo -e "${YELLOW}${line}${RESET}"
                    else
                        echo -e "${GREEN}${line}${RESET}"
                    fi
                  done

    # ── Fallback: /proc/diskstats (no sysstat needed)
    else
        echo -e "  ${YELLOW}iostat not found — reading /proc/diskstats${RESET}"
        echo -e "  ${DIM}Device            reads      writes     read_KB    write_KB${RESET}"
        echo -e "  ${DIM}$(printf '─%.0s' $(seq 1 60))${RESET}"

        # Sample twice with sleep to compute delta
        local snap1 snap2
        snap1=$(cat /proc/diskstats)
        sleep "$SAMPLE_DURATION"
        snap2=$(cat /proc/diskstats)

        # Parse and diff (fields: 4=reads 6=read_sectors 8=writes 10=write_sectors)
        while IFS= read -r line2; do
            local dev; dev=$(echo "$line2" | awk '{print $3}')
            [[ "$dev" =~ loop|ram ]] && continue

            local r2 w2 rs2 ws2
            r2=$(echo  "$line2" | awk '{print $4}')
            rs2=$(echo "$line2" | awk '{print $6}')
            w2=$(echo  "$line2" | awk '{print $8}')
            ws2=$(echo "$line2" | awk '{print $10}')

            local line1; line1=$(echo "$snap1" | awk -v d="$dev" '$3==d {print}')
            [[ -z "$line1" ]] && continue

            local r1 w1 rs1 ws1
            r1=$(echo  "$line1" | awk '{print $4}')
            rs1=$(echo "$line1" | awk '{print $6}')
            w1=$(echo  "$line1" | awk '{print $8}')
            ws1=$(echo "$line1" | awk '{print $10}')

            local dr dw drs dws
            dr=$(( r2  - r1  ))
            dw=$(( w2  - w1  ))
            drs=$(( (rs2 - rs1) / 2 ))   # sectors→KB (512B sectors)
            dws=$(( (ws2 - ws1) / 2 ))

            if [[ $((dr + dw)) -gt 0 ]]; then
                printf "  ${GREEN}%-16s %8d   %8d   %8d   %8d${RESET}\n" \
                       "$dev" "$dr" "$dw" "$drs" "$dws"
            fi
        done <<< "$snap2"
    fi

    # ── Disk Space (df)
    echo ""
    echo -e "  ${DIM}Disk Space Usage:${RESET}"
    echo -e "  ${DIM}Filesystem              Size    Used    Avail   Use%  Mounted${RESET}"
    echo -e "  ${DIM}$(printf '─%.0s' $(seq 1 65))${RESET}"

    df -h --output=source,size,used,avail,pcent,target 2>/dev/null \
        | grep -v "^Filesystem\|tmpfs\|devtmpfs\|udev" \
        | while IFS= read -r line; do
            local pct; pct=$(echo "$line" | awk '{gsub(/%/,"",$5); print $5}')
            if [[ -n "$pct" ]] && [[ "$pct" =~ ^[0-9]+$ ]]; then
                if   [[ "$pct" -ge 90 ]]; then echo -e "  ${RED}${line}${RESET}"
                elif [[ "$pct" -ge 75 ]]; then echo -e "  ${YELLOW}${line}${RESET}"
                else                           echo -e "  ${GREEN}${line}${RESET}"
                fi
            fi
          done

    # ── Top disk I/O processes (iotop if available)
    if has_cmd iotop; then
        echo ""
        echo -e "  ${DIM}Top I/O Processes (iotop):${RESET}"
        iotop -b -n 1 -P 2>/dev/null \
            | grep -v "^Total\|^Actual\|^TID" \
            | head -8 \
            | while IFS= read -r line; do
                echo "  $line"
              done
    fi

    log_entry "$(timestamp) DISK_IO=sampled"
}

# ─────────────────────────────────────────────────────────────────────────────
# SECTION 4 — BANDWIDTH USAGE
# ─────────────────────────────────────────────────────────────────────────────
check_bandwidth() {
    print_section "BANDWIDTH USAGE  (interface: ${NET_IFACE})"

    # ── Method 1: /proc/net/dev delta (always available)
    local rx1 tx1 rx2 tx2

    rx1=$(grep "$NET_IFACE:" /proc/net/dev 2>/dev/null | awk '{print $2}' || echo 0)
    tx1=$(grep "$NET_IFACE:" /proc/net/dev 2>/dev/null | awk '{print $10}' || echo 0)
    sleep "$SAMPLE_DURATION"
    rx2=$(grep "$NET_IFACE:" /proc/net/dev 2>/dev/null | awk '{print $2}' || echo 0)
    tx2=$(grep "$NET_IFACE:" /proc/net/dev 2>/dev/null | awk '{print $10}' || echo 0)

    local rx_bytes=$(( rx2 - rx1 ))
    local tx_bytes=$(( tx2 - tx1 ))

    # Convert to human-readable per second
    human_bps() {
        local bytes="$1"
        if   [[ "$bytes" -ge 1073741824 ]]; then awk "BEGIN{printf \"%.2f GB/s\", $bytes/1073741824}"
        elif [[ "$bytes" -ge 1048576    ]]; then awk "BEGIN{printf \"%.2f MB/s\", $bytes/1048576}"
        elif [[ "$bytes" -ge 1024       ]]; then awk "BEGIN{printf \"%.2f KB/s\", $bytes/1024}"
        else echo "${bytes} B/s"
        fi
    }

    local rx_hr; rx_hr=$(human_bps "$rx_bytes")
    local tx_hr; tx_hr=$(human_bps "$tx_bytes")

    # Color-code: >100MB/s=red, >10MB/s=yellow
    bw_color() {
        local bytes="$1"
        if   [[ "$bytes" -ge 104857600 ]]; then echo "$RED"
        elif [[ "$bytes" -ge 10485760  ]]; then echo "$YELLOW"
        else                                    echo "$GREEN"
        fi
    }

    local rxc; rxc=$(bw_color "$rx_bytes")
    local txc; txc=$(bw_color "$tx_bytes")

    echo ""
    printf "  ${ARROW} %-26s ${rxc}${BOLD}%s${RESET}\n" "↓ Download (RX rate)"  "$rx_hr"
    printf "  ${ARROW} %-26s ${txc}${BOLD}%s${RESET}\n" "↑ Upload   (TX rate)"  "$tx_hr"

    # ── Cumulative totals since boot
    echo ""
    echo -e "  ${DIM}Cumulative totals since boot:${RESET}"

    human_bytes() {
        local b="$1"
        if   [[ "$b" -ge 1073741824 ]]; then awk "BEGIN{printf \"%.2f GB\", $b/1073741824}"
        elif [[ "$b" -ge 1048576    ]]; then awk "BEGIN{printf \"%.2f MB\", $b/1048576}"
        elif [[ "$b" -ge 1024       ]]; then awk "BEGIN{printf \"%.2f KB\", $b/1024}"
        else echo "${b} B"
        fi
    }

    print_row "  Total RX (received)" "$(human_bytes "$rx2")"  "$CYAN"
    print_row "  Total TX (sent)"     "$(human_bytes "$tx2")"  "$MAGENTA"

    # ── All interfaces summary
    echo ""
    echo -e "  ${DIM}All interfaces:${RESET}"
    echo -e "  ${DIM}Interface        RX bytes (total)    TX bytes (total)${RESET}"
    echo -e "  ${DIM}$(printf '─%.0s' $(seq 1 55))${RESET}"

    awk 'NR>2 && /[a-z]/ {
        gsub(/:/,"",$1);
        if ($1=="lo") next;
        rx=$2; tx=$10;
        # Convert bytes to MB
        rxm=rx/1048576; txm=tx/1048576;
        printf "  %-16s  %12.2f MB        %12.2f MB\n", $1, rxm, txm
    }' /proc/net/dev | while IFS= read -r line; do
        echo -e "  ${GREEN}${line}${RESET}"
    done

    # ── Socket connections summary
    if has_cmd ss; then
        echo ""
        echo -e "  ${DIM}Active socket connections:${RESET}"
        local estab timewait closewait
        estab=$(ss -s 2>/dev/null | awk '/^TCP/ {print $4}' | tr -d ,)
        print_row "  TCP Established" "${estab:-N/A}"  "$WHITE"
        ss -s 2>/dev/null | awk 'NR<=5' | while IFS= read -r line; do
            echo -e "  ${DIM}  ${line}${RESET}"
        done
    fi

    # ── ifstat for confirmation (if available)
    if has_cmd ifstat; then
        echo ""
        echo -e "  ${DIM}ifstat live sample (KB/s):${RESET}"
        ifstat -i "$NET_IFACE" "$SAMPLE_DURATION" 1 2>/dev/null | tail -1 | \
            awk "{printf \"  ↓ RX: %s KB/s   ↑ TX: %s KB/s\n\", \$1, \$2}"
    fi

    log_entry "$(timestamp) IFACE=${NET_IFACE} RX=${rx_hr} TX=${tx_hr}"
}

# ─────────────────────────────────────────────────────────────────────────────
# SECTION 5 — MEMORY (bonus, always useful alongside the above)
# ─────────────────────────────────────────────────────────────────────────────
check_memory() {
    print_section "MEMORY USAGE"

    local total used free available buffers cached
    total=$(awk '/MemTotal/     {print $2}' /proc/meminfo)
    free=$(awk  '/MemFree/      {print $2}' /proc/meminfo)
    available=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    buffers=$(awk '/^Buffers/   {print $2}' /proc/meminfo)
    cached=$(awk  '/^Cached/    {print $2}' /proc/meminfo)

    used=$(( total - available ))
    local used_pct=$(( used * 100 / total ))

    kb_to_human() {
        local kb="$1"
        if   [[ "$kb" -ge 1048576 ]]; then awk "BEGIN{printf \"%.1f GB\", $kb/1048576}"
        else awk "BEGIN{printf \"%.1f MB\", $kb/1024}"
        fi
    }

    printf "  %b RAM Used         " "$(alert_icon "$used_pct" 75 90)"
    progress_bar "$used_pct"
    echo ""

    print_row "  Total RAM"     "$(kb_to_human "$total")"   "$WHITE"
    print_row "  Used"          "$(kb_to_human "$used")"    "$YELLOW"
    print_row "  Available"     "$(kb_to_human "$available")" "$GREEN"
    print_row "  Buffers/Cache" "$(kb_to_human $(( buffers + cached )))" "$DIM"

    # Swap
    local swap_total swap_free swap_used swap_pct
    swap_total=$(awk '/SwapTotal/ {print $2}' /proc/meminfo)
    swap_free=$(awk  '/SwapFree/  {print $2}' /proc/meminfo)
    swap_used=$(( swap_total - swap_free ))
    if [[ "$swap_total" -gt 0 ]]; then
        swap_pct=$(( swap_used * 100 / swap_total ))
        printf "  %b Swap Used        " "$(alert_icon "$swap_pct" 50 80)"
        progress_bar "$swap_pct"
        echo ""
        print_row "  Swap Total" "$(kb_to_human "$swap_total")"  "$DIM"
        print_row "  Swap Used"  "$(kb_to_human "$swap_used")"   "$YELLOW"
    else
        print_row "  Swap" "Not configured" "$DIM"
    fi

    log_entry "$(timestamp) MEM_USED=${used_pct}% MEM_AVAIL=$(kb_to_human "$available")"
}

# ─────────────────────────────────────────────────────────────────────────────
# MAIN REPORT
# ─────────────────────────────────────────────────────────────────────────────
run_report() {
    detect_iface

    clear

    print_header "  LINUX SYSTEM MONITOR  |  $(hostname)  |  $(timestamp)"

    echo -e "  ${DIM}Kernel: $(uname -r)   OS: $(grep PRETTY_NAME /etc/os-release 2>/dev/null | cut -d'"' -f2)${RESET}"
    echo -e "  ${DIM}Sampling interval: ${SAMPLE_DURATION}s   Network interface: ${NET_IFACE}   CPU alert: ${CPU_ALERT_THRESHOLD}%${RESET}"

    # Run all checks
    check_cpu
    check_load
    check_disk_io
    check_bandwidth
    check_memory

    echo ""
    echo -e "${BOLD}${BLUE}$(printf '─%.0s' $(seq 1 70))${RESET}"
    echo -e "  ${DIM}Report generated: $(timestamp)   Log: $( [[ $LOG_MODE == true ]] && echo "$LOG_FILE" || echo "disabled")${RESET}"
    echo ""
}

# ─────────────────────────────────────────────────────────────────────────────
# ARGUMENT PARSING
# ─────────────────────────────────────────────────────────────────────────────
usage() {
    cat <<EOF
${BOLD}sysmon.sh${RESET} — Linux System Monitor (Disk I/O · CPU · Load · Bandwidth)

${BOLD}Usage:${RESET}
  ./sysmon.sh              Single snapshot
  ./sysmon.sh -w           Watch mode (live dashboard, refreshes every 5s)
  ./sysmon.sh -w -i 10     Watch mode with 10s refresh interval
  ./sysmon.sh -l           Enable logging to sysmon.log
  ./sysmon.sh -t 80        Set CPU alert threshold to 80%
  ./sysmon.sh -n eth0      Specify network interface
  ./sysmon.sh -h           Show this help

${BOLD}Options:${RESET}
  -w          Watch / live mode
  -i SECS     Refresh interval in watch mode (default: 5)
  -l          Log each snapshot to sysmon.log
  -t PCT      CPU alert threshold percentage (default: 90)
  -n IFACE    Network interface (default: auto-detect)
  -h          Help

${BOLD}Dependencies:${RESET}
  Required : awk, grep, /proc/stat, /proc/diskstats, /proc/net/dev (always available)
  Optional : iostat, mpstat (sysstat pkg), iotop, ifstat, ss

${BOLD}Install optional tools:${RESET}
  Ubuntu/Debian : sudo apt install sysstat iotop ifstat
  RHEL/CentOS   : sudo yum install sysstat iotop
  Arch Linux    : sudo pacman -S sysstat iotop
EOF
}

while getopts "wi:lt:n:h" opt; do
    case "$opt" in
        w) WATCH_MODE=true ;;
        i) INTERVAL="$OPTARG" ;;
        l) LOG_MODE=true ;;
        t) CPU_ALERT_THRESHOLD="$OPTARG" ;;
        n) NET_IFACE="$OPTARG" ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done

# ─────────────────────────────────────────────────────────────────────────────
# ENTRY POINT
# ─────────────────────────────────────────────────────────────────────────────
if [[ "$WATCH_MODE" == true ]]; then
    echo -e "${BOLD}${GREEN}Starting live dashboard — press Ctrl+C to exit${RESET}"
    while true; do
        run_report
        echo -e "  ${DIM}Next refresh in ${INTERVAL}s ... (Ctrl+C to quit)${RESET}"
        sleep "$INTERVAL"
    done
else
    run_report
fi
