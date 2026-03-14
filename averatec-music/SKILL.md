---
name: averatec-music
description: Use when asked to read, analyze, or summarize local music project files — MIDI (.mid) or FL Studio (.flp). Extracts BPM, key, time signature, plugins, samples, and style info using inline Python (no pip installs needed).
---

# Music File Analysis (averatec)

Parse local music project files using `Bash` with inline Python scripts. No third-party libraries required — stdlib only.

**Priority rule:** The filename often encodes BPM and key (e.g., `[147Dm]`, `[140Fm]`) — extract these first as ground truth before parsing binary data.

---

## Step 0: Find files

```bash
# Glob for .mid / .flp files in the target directory
find ~/Music -name "*.mid" -o -name "*.flp" | head -30
```

---

## Part 1: MIDI (.mid)

```bash
python3 - <<'EOF'
import struct, sys

def read_vlq(data, offset):
    value = 0
    while True:
        byte = data[offset]; offset += 1
        value = (value << 7) | (byte & 0x7F)
        if not (byte & 0x80): break
    return value, offset

with open('FILE.mid', 'rb') as f:
    data = f.read()

fmt        = struct.unpack('>H', data[8:10])[0]
num_tracks = struct.unpack('>H', data[10:12])[0]
ppq        = struct.unpack('>H', data[12:14])[0]
print(f"Format={fmt}  Tracks={num_tracks}  PPQ={ppq}")

NOTE_NAMES = ['C','C#','D','D#','E','F','F#','G','G#','A','A#','B']
KEY_SIGS   = {(0,0):'C maj',(1,0):'G maj',(2,0):'D maj',(3,0):'A maj',
              (4,0):'E maj',(5,0):'B maj',(6,0):'F# maj',(7,0):'C# maj',
              (255,0):'F maj',(254,0):'Bb maj',(253,0):'Eb maj',(252,0):'Ab maj',
              (251,0):'Db maj',(250,0):'Gb maj',(249,0):'Cb maj',
              (0,1):'A min',(1,1):'E min',(2,1):'B min',(3,1):'F# min',
              (4,1):'C# min',(5,1):'G# min',(6,1):'D# min',(7,1):'A# min',
              (255,1):'D min',(254,1):'G min',(253,1):'C min',(252,1):'F min',
              (251,1):'Bb min',(250,1):'Eb min',(249,1):'Ab min'}

offset = 14; note_freq = {}; tempo = None; timesig = None; key = None; tracks = []
for _ in range(num_tracks):
    tag      = data[offset:offset+4]
    trk_len  = struct.unpack('>I', data[offset+4:offset+8])[0]
    trk_data = data[offset+8:offset+8+trk_len]
    offset  += 8 + trk_len
    pos = 0; last_status = 0; trk_name = ''
    while pos < len(trk_data):
        _, pos = read_vlq(trk_data, pos)
        b = trk_data[pos]; pos += 1
        if b == 0xFF:  # Meta
            mtype = trk_data[pos]; pos += 1
            mlen, pos = read_vlq(trk_data, pos)
            mdata = trk_data[pos:pos+mlen]; pos += mlen
            if mtype == 0x51 and mlen == 3:
                tempo = (mdata[0]<<16)|(mdata[1]<<8)|mdata[2]
            elif mtype == 0x58 and mlen == 4:
                timesig = f"{mdata[0]}/{1<<mdata[1]}"
            elif mtype == 0x59 and mlen == 2:
                key = KEY_SIGS.get((mdata[0], mdata[1]), f"raw({mdata[0]},{mdata[1]})")
            elif mtype == 0x03:
                trk_name = mdata.decode('utf-8', errors='ignore').rstrip('\x00')
        elif b in (0xF0, 0xF7):
            slen, pos = read_vlq(trk_data, pos); pos += slen
        else:
            status = b if b & 0x80 else last_status
            if b & 0x80: last_status = status
            else: pos -= 1
            ch = status & 0x0F; etype = status >> 4
            if etype in (0x8, 0x9, 0xA):
                note = trk_data[pos] & 0x7F; pos += 2
                if etype == 0x9: note_freq[note] = note_freq.get(note, 0) + 1
            elif etype in (0xB, 0xE): pos += 2
            elif etype in (0xC, 0xD): pos += 1
    if trk_name: tracks.append(trk_name)

bpm = round(60_000_000 / tempo) if tempo else 'unknown'
bpm = round(60_000_000 / tempo) if tempo else 'unknown'
top_notes = sorted(note_freq, key=note_freq.get, reverse=True)[:10]
top_names = [f"{NOTE_NAMES[n%12]}{n//12-1}" for n in top_notes]

# ASCII note bar chart
max_count = max(note_freq.values()) if note_freq else 1
bar_width = 24
print("\n── SNAPSHOT ──────────────────────────────")
print(f"File format : MIDI type {fmt}")
print(f"Tracks      : {num_tracks}  PPQ: {ppq}")
print(f"Track names : {tracks if tracks else '(none)'}")
print(f"BPM         : {bpm}  |  Time sig: {timesig}  |  Key: {key}")
print(f"\nNote frequency (top 10):")
for n in top_notes:
    name  = f"{NOTE_NAMES[n%12]}{n//12-1:<2}"
    count = note_freq[n]
    bar   = '█' * int(count / max_count * bar_width)
    print(f"  {name}  {bar:<{bar_width}} {count}")
print("──────────────────────────────────────────")
EOF
```

**Fields extracted:**

| Field | Source |
|---|---|
| BPM | Meta 0x51 → `60_000_000 / µs` |
| Time signature | Meta 0x58 |
| Key | Meta 0x59 |
| Track names | Meta 0x03 |
| Top 10 notes | Note On frequency table (with ASCII bar chart) |

---

## Part 2: FL Studio (.flp)

```bash
python3 - <<'EOF'
import struct, re, sys

def read_vlq(data, pos):
    length = 0; shift = 0
    while True:
        b = data[pos]; pos += 1
        length |= (b & 0x7F) << shift; shift += 7
        if not (b & 0x80): break
    return length, pos

with open('FILE.flp', 'rb') as f:
    data = f.read()

assert data[:4] == b'FLhd', "Not an FLP file"
header_len   = struct.unpack('<I', data[4:8])[0]
num_channels = struct.unpack('<H', data[10:12])[0]
ppq          = struct.unpack('<H', data[12:14])[0]
pos = 8 + header_len + 8  # skip to FLdt data

version = ''; bpm = None; timesig = ''; title = ''
channels = []; patterns = []; samples = []; plugins = []

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

    if eid == 135:   # Tempo
        raw = val if isinstance(val, int) else struct.unpack('<I', val[:4])[0]
        bpm = raw / 1000 if raw > 1000 else raw
    elif eid == 199: version = val.decode('utf-8', errors='ignore').rstrip('\x00')
    elif eid == 194: title   = val.decode('utf-16-le', errors='ignore').rstrip('\x00')
    elif eid == 192: channels.append(val.decode('utf-16-le', errors='ignore').rstrip('\x00'))
    elif eid == 193: patterns.append(val.decode('utf-16-le', errors='ignore').rstrip('\x00'))
    elif eid == 196: samples.append(val.decode('utf-16-le', errors='ignore').rstrip('\x00'))
    elif eid == 228: plugins.append(val.decode('utf-16-le', errors='ignore').rstrip('\x00'))
    elif eid == 242 and not isinstance(val, int):
        timesig = f"{val[0]}/{val[1]}" if len(val) >= 2 else ''

# BPM fallback: filename annotation
import os
fname = os.path.basename('FILE.flp')
m = re.search(r'\[(\d+)', fname)
if not bpm and m: bpm = int(m.group(1))

# Plugin fallback: UTF-16 scan for known names
KNOWN = {'Keyscape':b'K\x00e\x00y\x00','Omnisphere':b'O\x00m\x00n\x00i\x00',
         'Gross Beat':b'G\x00r\x00o\x00s\x00','Slicex':b'S\x00l\x00i\x00c\x00e\x00x\x00',
         'FLEX':b'F\x00L\x00E\x00X\x00','Maximus':b'M\x00a\x00x\x00i\x00m\x00',
         'Parametric EQ':b'P\x00a\x00r\x00a\x00m\x00e\x00t\x00r\x00i\x00c\x00',
         'Fruity Wrapper':b'F\x00r\x00u\x00i\x00t\x00y\x00 \x00W\x00'}
for name, pat in KNOWN.items():
    if pat in data and name not in plugins:
        plugins.append(name + ' (scan)')

deduped_plugins = list(dict.fromkeys(plugins))
print("\n── SNAPSHOT ──────────────────────────────")
print(f"Title       : {title or fname}")
print(f"FL Version  : {version or 'unknown'}  |  BPM: {bpm}  |  Time sig: {timesig or 'unknown'}")
print(f"Channels    : {len(channels)}  →  {channels[:10]}")
print(f"Patterns    : {len(patterns)}  →  {patterns[:10]}")
print(f"Plugins     : {deduped_plugins}")
print(f"Samples     : {samples[:10]}")
print("──────────────────────────────────────────")
EOF
```

**Key event IDs:**

| ID | Name | Notes |
|---|---|---|
| 135 | Tempo | raw / 1000 in modern FL |
| 192 | Channel.Name | UTF-16-LE |
| 193 | Pattern.Name | UTF-16-LE |
| 194 | Title | Project title |
| 196 | SmpFileName | Full sample path |
| 199 | Version | FL Studio version |
| 228 | Plugin.Name | UTF-16-LE (may be truncated in FL 25+) |
| 242 | TimeSignature | byte[0]=num, byte[1]=denom |

---

## Output Format

Always two phases — Snapshot first, Analysis second.

### Phase 1 — Snapshot (raw data, no interpretation)

Present exactly what was extracted from the file. Do not editorialize here.

```
📄 [filename] · [size] · [format/version]

BPM: [value]   Key: [value]   Time sig: [value]

Tracks / Channels:
  [list]

Note frequency (MIDI only):
  C4  ████████████████ 45
  A3  ████████████     33
  ...

Plugins:
  [list]

Samples (first 10):
  [list]
```

### Phase 2 — Analysis (interpretation)

After the snapshot, interpret the data:

- **Tempo context** — how does the BPM feel (e.g., 140 BPM = energetic, house/trap range)
- **Key + top notes** — cross-check Meta key sig with note frequency; infer mood (minor = darker, major = brighter)
- **Instrument roles** — map each plugin/channel to a likely role (kick, bass, melody, chords, FX)
- **Sample style** — infer genre from sample filenames if present (e.g., "808", "hihat", "phonk")
- **Style summary** — one short paragraph: BPM + key + instrumentation → genre tags

```
🎵 Analysis

Tempo: 140 BPM — mid-tempo, typical for trap/melodic rap
Key: F minor — darker, melancholic feel
Structure: 3 tracks — likely intro pattern (single piano phrase)
Top notes: F4, Ab4, C5 confirm F minor triad is the core motif
Style: Minimalist piano loop, trap-adjacent — consistent with filename [140Fm]
```

---

## Pitfalls

- Skip unknown FLP event IDs — many are undocumented; do not abort
- Always `.rstrip('\x00')` after UTF-16-LE decode
- FL 20+ changed Tempo encoding; filename `[BPM]` annotation is most reliable
- For MIDI key inference, cross-check note frequency with Meta 0x59
- Plugin param blocks (Event 206, 209, etc.) are plugin-private binary — skip
