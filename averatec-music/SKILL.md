---
name: averatec-music
description: Use when asked to read, analyze, or summarize music project files — MIDI (.mid) or FL Studio (.flp). Extracts BPM, key, time signature, plugins, samples, and style info using inline Python (no pip installs needed).
---

# Music File Analysis (averatec)

## CRITICAL: Act immediately. Never ask first.

When a music file is present, run the parser script right away. Output Snapshot first, then Analysis. Do not list options. Do not ask what kind of analysis is wanted. Just do it.

**File path resolution — in order:**
1. Explicit path in the message (e.g., `/path/to/file.mid`, `~/Music/track.flp`)
2. `MediaPaths` / `MediaPath` from message context (attachment via Discord, Telegram, etc.)
3. Current working directory — `ls *.mid *.flp 2>/dev/null`
4. Common music directories — `find ~/Music ~/Desktop ~/Downloads -name "*.mid" -o -name "*.flp" 2>/dev/null | head -10`
5. Whole home directory as last resort — `find ~ -maxdepth 4 -name "*.mid" -o -name "*.flp" 2>/dev/null | head -10`

If multiple files found in steps 3–5, list them and pick the most recently modified one, or ask which to analyze.

**Filename hint:** Extract BPM and key from filename first (e.g., `[140Fm]` → BPM=140, key=F minor). Use as ground truth when binary data is ambiguous.

---

## Part 1: MIDI (.mid) — run this script

Replace `FILE.mid` with the actual file path.

```bash
python3 - <<'EOF'
import struct, os, sys

def read_vlq(data, offset):
    value = 0
    while True:
        byte = data[offset]; offset += 1
        value = (value << 7) | (byte & 0x7F)
        if not (byte & 0x80): break
    return value, offset

path = 'FILE.mid'
with open(path, 'rb') as f:
    data = f.read()

size_kb = len(data) / 1024
fmt        = struct.unpack('>H', data[8:10])[0]
num_tracks = struct.unpack('>H', data[10:12])[0]
ppq        = struct.unpack('>H', data[12:14])[0]

NOTE_NAMES = ['C','C#','D','D#','E','F','F#','G','G#','A','A#','B']
KEY_SIGS = {
    (0,0):'C maj',(1,0):'G maj',(2,0):'D maj',(3,0):'A maj',
    (4,0):'E maj',(5,0):'B maj',(6,0):'F# maj',(7,0):'C# maj',
    (255,0):'F maj',(254,0):'Bb maj',(253,0):'Eb maj',(252,0):'Ab maj',
    (251,0):'Db maj',(250,0):'Gb maj',(249,0):'Cb maj',
    (0,1):'A min',(1,1):'E min',(2,1):'B min',(3,1):'F# min',
    (4,1):'C# min',(5,1):'G# min',(6,1):'D# min',(7,1):'A# min',
    (255,1):'D min',(254,1):'G min',(253,1):'C min',(252,1):'F min',
    (251,1):'Bb min',(250,1):'Eb min',(249,1):'Ab min'
}

offset = 14
note_freq = {}; note_on_times = {}; note_durations = []
tempo_events = []; timesig = None; key = None; tracks = []
total_ticks = 0; channels_used = set(); program_changes = {}

for trk_idx in range(num_tracks):
    tag     = data[offset:offset+4]
    trk_len = struct.unpack('>I', data[offset+4:offset+8])[0]
    trk_data = data[offset+8:offset+8+trk_len]
    offset += 8 + trk_len
    pos = 0; last_status = 0; trk_name = ''; abs_tick = 0

    while pos < len(trk_data):
        delta, pos = read_vlq(trk_data, pos)
        abs_tick += delta
        b = trk_data[pos]; pos += 1

        if b == 0xFF:
            mtype = trk_data[pos]; pos += 1
            mlen, pos = read_vlq(trk_data, pos)
            mdata = trk_data[pos:pos+mlen]; pos += mlen
            if mtype == 0x51 and mlen == 3:
                us = (mdata[0]<<16)|(mdata[1]<<8)|mdata[2]
                tempo_events.append((abs_tick, us))
            elif mtype == 0x58 and mlen == 4:
                timesig = f"{mdata[0]}/{1<<mdata[1]}"
            elif mtype == 0x59 and mlen == 2:
                key = KEY_SIGS.get((mdata[0], mdata[1]), f"raw({mdata[0]},{mdata[1]})")
            elif mtype == 0x03:
                trk_name = mdata.decode('utf-8', errors='ignore').rstrip('\x00')
            elif mtype == 0x2F:
                total_ticks = max(total_ticks, abs_tick)
        elif b in (0xF0, 0xF7):
            slen, pos = read_vlq(trk_data, pos); pos += slen
        else:
            status = b if b & 0x80 else last_status
            if b & 0x80: last_status = status
            else: pos -= 1
            ch = status & 0x0F; etype = status >> 4
            channels_used.add(ch)
            if etype in (0x8, 0x9, 0xA):
                note = trk_data[pos] & 0x7F
                vel  = trk_data[pos+1] & 0x7F; pos += 2
                if etype == 0x9 and vel > 0:
                    note_freq[note] = note_freq.get(note, 0) + 1
                    note_on_times[(ch, note)] = (abs_tick, vel)
                elif etype in (0x8,) or (etype == 0x9 and vel == 0):
                    key_on = note_on_times.pop((ch, note), None)
                    if key_on:
                        note_durations.append(abs_tick - key_on[0])
            elif etype == 0xC:
                prog = trk_data[pos] & 0x7F; pos += 1
                program_changes[ch] = prog
            elif etype in (0xB, 0xE): pos += 2
            elif etype == 0xD: pos += 1

    if trk_name: tracks.append(trk_name)
    total_ticks = max(total_ticks, abs_tick)

# BPM
bpm = round(60_000_000 / tempo_events[0][1]) if tempo_events else 'unknown'

# Duration estimate
if tempo_events and total_ticks:
    us_per_tick = tempo_events[0][1] / ppq
    duration_sec = total_ticks * us_per_tick / 1_000_000
    mins = int(duration_sec // 60); secs = int(duration_sec % 60)
    duration_str = f"{mins}m{secs:02d}s"
else:
    duration_str = 'unknown'

# Note range
if note_freq:
    lo = min(note_freq); hi = max(note_freq)
    lo_name = f"{NOTE_NAMES[lo%12]}{lo//12-1}"
    hi_name = f"{NOTE_NAMES[hi%12]}{hi//12-1}"
    range_str = f"{lo_name} – {hi_name}"
else:
    range_str = 'none'

# Average note duration (in ticks → beats)
if note_durations:
    avg_dur_beats = round(sum(note_durations) / len(note_durations) / ppq, 2)
else:
    avg_dur_beats = 'n/a'

# Velocity stats
all_vels = []
for trk_idx2 in range(num_tracks):
    pass  # already collected above — approximate from note_freq
vel_note = 'n/a'

# Top notes bar chart
top_notes = sorted(note_freq, key=note_freq.get, reverse=True)[:12]
max_count = max(note_freq.values()) if note_freq else 1
bar_width = 20

print("━━━ SNAPSHOT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━")
print(f"File     : {os.path.basename(path)}  ({size_kb:.1f} KB)")
print(f"Format   : MIDI Type {fmt}  |  Tracks: {num_tracks}  |  PPQ: {ppq}")
print(f"Track names : {tracks if tracks else '(unnamed)'}")
print(f"BPM      : {bpm}  |  Time sig: {timesig or '?'}  |  Key: {key or '?'}")
print(f"Duration : ~{duration_str}  |  Total ticks: {total_ticks}")
print(f"Note range : {range_str}  |  Avg note dur: {avg_dur_beats} beats")
print(f"Channels used : {sorted(channels_used)}")
print(f"Program changes : {program_changes}")
print(f"\nNote density (top 12):")
for n in top_notes:
    name  = f"{NOTE_NAMES[n%12]}{n//12-1:<3}"
    count = note_freq[n]
    bar   = '█' * int(count / max_count * bar_width)
    print(f"  {name} {bar:<{bar_width}} {count}")
print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━")

# Interval pattern (for melody character)
sorted_notes = sorted(note_freq.keys())
if len(sorted_notes) >= 3:
    intervals = [sorted_notes[i+1]-sorted_notes[i] for i in range(len(sorted_notes)-1)]
    print(f"\nSemitone intervals between used notes: {intervals}")
EOF
```

---

## Part 2: FL Studio (.flp) — run this script

Replace `FILE.flp` with the actual file path.

```bash
python3 - <<'EOF'
import struct, re, os

def read_vlq(data, pos):
    length = 0; shift = 0
    while True:
        b = data[pos]; pos += 1
        length |= (b & 0x7F) << shift; shift += 7
        if not (b & 0x80): break
    return length, pos

path = 'FILE.flp'
with open(path, 'rb') as f:
    data = f.read()

assert data[:4] == b'FLhd', "Not an FLP file"
size_kb = len(data) / 1024
header_len   = struct.unpack('<I', data[4:8])[0]
ppq          = struct.unpack('<H', data[12:14])[0]
pos = 8 + header_len + 8

version=''; bpm=None; timesig=''; title=''
channels=[]; patterns=[]; samples=[]; plugins=[]

while pos < len(data) - 2:
    eid = data[pos]; pos += 1
    if eid < 64:
        val = data[pos]; pos += 1
    elif eid < 128:
        val = struct.unpack('<H', data[pos:pos+2])[0]; pos += 2
    elif eid < 192:
        val = struct.unpack('<I', data[pos:pos+4])[0]; pos += 4
    else:
        length, pos = read_vlq(data, pos)
        val = data[pos:pos+length]; pos += length

    if   eid == 135:
        raw = val if isinstance(val,int) else struct.unpack('<I',val[:4])[0]
        bpm = raw/1000 if raw > 1000 else raw
    elif eid == 199: version = val.decode('utf-8',errors='ignore').rstrip('\x00')
    elif eid == 194: title   = val.decode('utf-16-le',errors='ignore').rstrip('\x00')
    elif eid == 192: channels.append(val.decode('utf-16-le',errors='ignore').rstrip('\x00'))
    elif eid == 193: patterns.append(val.decode('utf-16-le',errors='ignore').rstrip('\x00'))
    elif eid == 196: samples.append(val.decode('utf-16-le',errors='ignore').rstrip('\x00'))
    elif eid == 228: plugins.append(val.decode('utf-16-le',errors='ignore').rstrip('\x00'))
    elif eid == 242 and not isinstance(val,int):
        timesig = f"{val[0]}/{val[1]}" if len(val)>=2 else ''

fname = os.path.basename(path)
m = re.search(r'\[(\d+)', fname)
if not bpm and m: bpm = int(m.group(1))

# Key hint from filename
km = re.search(r'\[[\d]+([A-Ga-g][#b]?[mM]?)\]', fname)
key_hint = km.group(1) if km else None

KNOWN = {
    'Keyscape':b'K\x00e\x00y\x00','Omnisphere':b'O\x00m\x00n\x00i\x00',
    'Gross Beat':b'G\x00r\x00o\x00s\x00','Slicex':b'S\x00l\x00i\x00c\x00e\x00x\x00',
    'FLEX':b'F\x00L\x00E\x00X\x00','Maximus':b'M\x00a\x00x\x00i\x00m\x00',
    'Parametric EQ':b'P\x00a\x00r\x00a\x00m\x00','Fruity Wrapper':b'F\x00r\x00u\x00i\x00t\x00y\x00 \x00W\x00',
    'Serum':b'S\x00e\x00r\x00u\x00m\x00','Vital':b'V\x00i\x00t\x00a\x00l\x00',
    'Kontakt':b'K\x00o\x00n\x00t\x00a\x00k\x00t\x00',
}
for name, pat in KNOWN.items():
    if pat in data and name not in plugins:
        plugins.append(name+' (scan)')

deduped = list(dict.fromkeys(plugins))
print("━━━ SNAPSHOT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━")
print(f"File       : {fname}  ({size_kb:.1f} KB)")
print(f"Title      : {title or '(none)'}")
print(f"FL Version : {version or 'unknown'}  |  PPQ: {ppq}")
print(f"BPM        : {bpm or '?'}  |  Time sig: {timesig or '?'}")
print(f"Key hint   : {key_hint or '(from filename)'}")
print(f"Channels   : {len(channels)}  →  {channels[:12]}")
print(f"Patterns   : {len(patterns)}  →  {patterns[:12]}")
print(f"Plugins    : {deduped}")
print(f"Samples    : {samples[:10]}")
print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━")
EOF
```

---

## Output Format

### Phase 1 — Snapshot

Print the script output verbatim. This is the raw data block — no paraphrasing, no omissions.

### Phase 2 — Musical Analysis

After the snapshot, provide a full musical analysis covering **all of the following**:

**Tempo & Feel**
- What does this BPM feel like physically? (e.g., 140 = energetic, nodding-head pace; 70 = slow, heavy)
- Half-time or double-time feel possible?

**Key & Mood**
- Major vs minor — brightness/darkness
- Specific key character (e.g., F minor = brooding, D major = triumphant)
- Does note frequency distribution confirm or contradict the key signature?

**Melody Character** (MIDI)
- Pitch range: how wide is the register? (narrow = intimate, wide = dramatic)
- Interval character: stepwise (smooth, lyrical) vs leaps (dramatic, angular)
- Note density and rhythm feel from avg note duration (long = sustained/ambient, short = busy/percussive)
- Dominant notes: do they form a recognizable chord shape or scale fragment?

**Harmonic Content** (infer from top notes)
- What chord or scale do the most-used notes suggest?
- Any characteristic intervals (minor 3rd, tritone, perfect 5th)?

**Instrumentation & Role** (FLP / program changes)
- Map each plugin/channel/program to a likely role: lead, chords, bass, drums, FX, texture
- Identify any signature sounds (e.g., Keyscape = piano/keys, 808 = trap bass)

**Rhythm & Structure** (FLP patterns / MIDI duration)
- Pattern names or structure hints
- Is this a loop, a full arrangement, or a sketch?

**Genre Tags**
- List 2–4 specific genre tags with brief reasoning
- Note if the file is ambiguous or spans multiple styles

**One-paragraph Summary**
- Synthesize everything into a single paragraph a music producer would find useful

---

## Pitfalls

- Attachments from any platform (Discord, Telegram, iMessage…) arrive via MediaPaths — use that path directly before falling back to filesystem search
- Always rstrip('\x00') after UTF-16-LE decode
- FL 20+ changed Tempo encoding; filename [BPM] annotation is most reliable
- For MIDI key inference, cross-check note frequency distribution with Meta 0x59
- Plugin param blocks (Event 206, 209, etc.) are plugin-private binary — skip
- If file is only a short loop (< 4 bars), say so in the analysis — do not over-extrapolate style
