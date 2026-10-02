---
layout: page
title: SAM and Tape for SBND Users
---

# SAM and tape for SBND users: metadata, locations, tape copies

A step-by-step instruction for putting your own files into the SAM catalog and onto
Fermilab tape. Written 2026-09-23 (liangliu); every command below was run on an SBND
gpvm (AlmaLinux 9, samweb 3.6) unless marked otherwise. Companions: `sbnd-sam-cheat-sheet.md`
(allowed metadata values), `to_tape/tape-best-practices.md` (rules), and the bulk tools in
`to_tape/` (section 7). These companion files and tools are not part of this wiki: they live on
the SBND gpvms in `/exp/sbnd/app/users/liangliu/sbnd-data-management`, and the paths on this
page are relative to that directory.

Contents

1. Basics: what SAM is, what tape is, how they fit together
2. Setup for every shell (samweb, token)
3. Set up metadata and declare a file
4. Add a path (location) to SAM
5. Check the tape area
6. Move a file to tape and add the tape path to SAM
7. Bulk campaigns with the repo tools
8. Troubleshooting

---

## 1. Basics

### 1.1 SAM

SAM is a file catalog: one record per file, keyed by a **globally unique file name**
(the basename, no directory). A record holds

- **metadata**: size, checksum(s), standard dimensions (`file_type`, `file_format`,
  `data_tier`, `group`, `runs`, `event_count`, ...) and experiment parameters
  (`production.name`, `sbnd_project.stage`, `Dataset.Tag`, ...);
- **locations**: zero or more directories where a copy of the file lives, each written as
  `<prefix>:<directory>` (never the file path). A record with no location is a *virtual*
  file;
- **lineage**: `parents` (which SAM files it was made from); children follow automatically.

Two separate SAM instances matter for SBND. They share nothing: the same file can be
declared in one, both, or neither, and a location added in one is not visible in the other.

| instance | server | use |
|---|---|---|
| `sbnd` | `https://samsbnd.fnal.gov:8483` | SBND raw data, most SBND MC, user declarations |
| `sbn` | `https://samsbn.fnal.gov:8483` | shared SBND + ICARUS catalog (MCP2025B/C, SBND2026A, joint work) |

Other things to know:

- A record is **owned by the user who declared it**. Only the owner or an admin can add or
  remove its locations or change its metadata. Production records belong to `sbndpro`.
- A **definition** is a saved query (dimensions string) with a name. It is dynamic: files
  that later match the query appear in it. A **snapshot** freezes it.
- Dimensions queries look like `production.name=MCP2025A and sbnd_project.stage=caf`.
  Virtual files only show up with `... with availability anylocation`.
- Location prefixes in use at SBND:

| prefix | meaning | example |
|---|---|---|
| `dcache:` | disk dCache at FNAL (scratch, persistent, `/pnfs/sbn/...`) | `dcache:/pnfs/sbnd/persistent/users/alice/analysis` |
| `enstore:` | tape-backed dCache at FNAL (the name is historical, tape is now CTA) | `enstore:/pnfs/sbnd/archive/sam_managed_users/alice/tag1/00` |
| `imperial:` | the Imperial College dCache copy | `imperial:/pnfs/hep.ph.ic.ac.uk/data/sbnd/...` |

### 1.2 Tape

Fermilab tape is CTA behind dCache. You never talk to tape directly:

1. You write a file into a **tape-backed dCache directory** (NFS `cp` on a gpvm).
2. dCache migrates it to tape by itself, asynchronously (minutes to hours, sometimes
   longer). Until then the file exists only on disk.
3. Later the disk replica may be evicted. Reading a file that is tape-only triggers a
   **recall**; bulk recalls must be requested with a prestage, never by reading files one
   by one.

Whether a directory is tape-backed is decided by its dCache **tags**, which a new
subdirectory inherits from its parent. Users cannot set tags; dCache admins do. The tag
`file_family` names the tape pool the file goes to. State of the SBND namespace,
verified 2026-09-23:

| directory | file_family | goes to tape? |
|---|---|---|
| `/pnfs/sbnd/archive/...` (raw data, production tape copies) | `sbnd` (raw streams have their own families) | yes |
| `/pnfs/sbnd/archive/sam_managed_users/<user>/` | `sbnd` | yes, the user area |
| `/pnfs/sbnd/to_tape/<user>/` | `sbnd` | yes, the newer user area |
| `/pnfs/sbnd/archive/sbn/sbn_nd/mc/reco1/` | `reco1_mc` | yes, reserved for reco1 MC |
| `/pnfs/sbnd/archive/sbn/sbn_nd/data/reco1/` | `reco1_data` | yes, reserved for reco1 data |
| `/pnfs/sbnd/persistent`, `/pnfs/sbnd/scratch` | `persistent` / `scratch`, library `NONE` | no, files stay `ONLINE` |
| `/pnfs/sbn/...` (all of it) | `persdata*`, library `None` | no |

Per-file tape state is the dCache **locality**:

| NFS dot-command (`.(get)(FILE)(locality)`) | tape REST API | meaning |
|---|---|---|
| `ONLINE` | `DISK` | disk only, migration pending or never (disk-only pool) |
| `ONLINE_AND_NEARLINE` | `DISK_AND_TAPE` | on tape, a disk replica still present. Safe. |
| `NEARLINE` | `TAPE` | on tape only; reading needs a recall. Safe. |

Rules that follow from how tape works (details in `to_tape/tape-best-practices.md`):

- Tape is effectively **write-once**. Deleting frees nothing until the tape is repacked.
  Archive only finished, verified products.
- **File size matters**: aim for 1–10 GB per file, never below ~100 MB. Bundle small
  files into tars (section 7).
- "Copied" is not "on tape". Delete a source only after the tape copy reports
  `NEARLINE` / `DISK_AND_TAPE` and its checksum matches.
- A tape file that is not in SAM is effectively lost. Declare at archive time.

### 1.3 How the pieces fit

```
your file on disk dCache            (e.g. /pnfs/sbnd/persistent/users/<you>/...)
   |
   | 3. write metadata JSON  -> validate-metadata -> declare-file
   v
SAM record (metadata, no location yet)
   |
   | 4. add-file-location dcache:<disk dir>
   v
SAM record + disk location                 <- analysis jobs can now find it
   |
   | 6a. cp into a tape-backed directory, verify size + adler32 through the door
   | 6b. add-file-location enstore:<tape dir>
   v
SAM record + disk location + tape location
   |
   | 6c. wait until locality is NEARLINE / DISK_AND_TAPE
   | 6d. (optional) remove-file-location dcache:<disk dir>, delete the disk copy
   v
file safe on tape, findable by name, definition or query
```

---

## 2. Setup for every shell

Use a node with `/pnfs` mounted (sbndgpvm, sbndbuild). Tokens: the first
`htgettoken` of the week opens a browser login and stores a **vault token** (7 days) in
`/tmp/vt_u<uid>`; after that `--nooidc` refreshes the **bearer token** (about 1 h) from a
Kerberos ticket without a browser.

```bash
source /cvmfs/fermilab.opensciencegrid.org/packages/common/setup-env.sh
spack load /wqczc57                  # sam-web-client@3.6 on AL9; plain "sam-web-client@3.6" is ambiguous
export SAM_EXPERIMENT=sbnd           # or sbn; or pass -e <instance> per command

kinit                                                        # if klist -s fails
export BEARER_TOKEN_FILE=/run/user/$(id -u)/bt_u$(id -u)
htgettoken --nooidc -a htvaultprod.fnal.gov -i sbnd --minsecs 1200 -o $BEARER_TOKEN_FILE
#   first time / vault token expired: drop --nooidc and follow the browser link
#   sbndpro: add  -r production   (managed token /tmp/bt_u52084_production)

# minutes left on the token (renew when < 20)
httokendecode $BEARER_TOKEN_FILE | python3 -c 'import sys,json,time; print(int((json.load(sys.stdin)["exp"]-time.time())/60))'
```

Read-only samweb commands (`list-files`, `get-metadata`, `locate-file`) work without a
token. Every command that writes (`declare-file`, `add-file-location`,
`modify-metadata`, `create-definition`) needs a valid token whose identity matches the
`user` of the record. The dCache doors (section 5) need the token too.

---

## 3. Set up metadata and declare a file

### 3.1 What a record needs

SAM itself enforces very little: `validate-metadata` on a record with only `file_name`
and `file_size` answers "Missing file_type"; with `file_type` and `file_format` added
it says "Metadata is valid" (checked 2026-09-23). A record that is *useful* carries much
more. Use this as the minimum for an SBND file:

| key | value | where it comes from |
|---|---|---|
| `file_name` | basename, unique in the instance | your file; rename derived files that share a parent's basename |
| `file_size` | bytes | `stat -c %s FILE` |
| `checksum` | `["adler32:<hex>"]`, optionally also `"enstore:<decimal>"` | `samweb file-checksum --type=adler32,enstore FILE` (reads the file), or dCache's stored value `cat "DIR/.(get)(FILE)(checksum)"` (no read) |
| `user` | your username | must equal the token identity, otherwise "not an admin" |
| `file_type` | `data` / `mc` | |
| `file_format` | `artroot` / `caf` / `flat_caf` / `root` / `tar` / ... | `samweb list-values file_formats` |
| `data_tier` | `reconstructed` / `caf` / `sam-user` / ... | `samweb list-values data_tiers`; unknown values are rejected |
| `group` | `sbnd` | |
| `runs` | `[[run, subrun, "physics"]]` (or `"commissioning"`) | the file; SBND raw subrun is always 1 |
| `event_count` | events in the file | |
| `application` | `{"family": "art", "name": "sbndcode", "version": "v10_05_00"}` | your job |
| `Dataset.Tag` | your campaign tag, e.g. `alice_diffusion_v10_05_00` | your choice; makes definitions trivial |
| `production.name` / `production.type` | campaign name / `personal` | your choice |
| `sbnd_project.name/stage/version/software` | sample / stage / release / `sbndcode` | your job fcl |
| `parents` | `[{"file_name": "<parent>"}]`, parents must exist in the same instance | your job's input list |

Two ways to avoid inventing values:

```bash
samweb get-metadata <a similar existing file> --json      # copy its shape, replace the per-file fields
samweb list-parameters                                    # registered experiment parameters and their types
samweb list-parameters production.type                    # values already in use
```

`sbnd-sam-cheat-sheet.md` lists the vocabulary in use. New dimension values must be
registered by an admin before use (`samweb add-value data_tiers <value> "<description>"`).

### 3.2 Worked example: one file

```bash
F=myfile_run18618_reco1.root
D=/pnfs/sbnd/persistent/users/$USER/analysis            # where the file sits now

CK=$(samweb file-checksum --type=adler32,enstore $D/$F)   # -> ["adler32:3386a283", "enstore:281518722"]
SZ=$(stat -c %s $D/$F)

cat > $F.json <<EOT
{
 "file_name": "$F",
 "file_size": $SZ,
 "checksum": $CK,
 "user": "$USER",
 "file_type": "data",
 "file_format": "artroot",
 "data_tier": "reconstructed",
 "group": "sbnd",
 "runs": [[18618, 1, "commissioning"]],
 "event_count": 22,
 "application": {"family": "art", "name": "sbndcode", "version": "v10_05_00"},
 "art.process_name": "Reco1Comm",
 "Dataset.Tag": "${USER}_diffusion_v10_05_00",
 "production.name": "${USER}_diffusion_v10_05_00",
 "production.type": "personal",
 "sbnd_project.name": "diffusion",
 "sbnd_project.stage": "reco1",
 "sbnd_project.version": "v10_05_00",
 "sbnd_project.software": "sbndcode"
}
EOT

samweb validate-metadata $F.json      # one JSON object per call; prints the first problem
samweb declare-file $F.json           # creates the record (no location yet)
samweb get-metadata $F --json         # read it back
```

`declare-file` also accepts a JSON **list** of records; use lists of about 100 for bulk
declares. `validate-metadata` does not accept lists, so validate a sample.

### 3.3 Changing a record later

```bash
samweb modify-metadata $F changes.json          # changes.json = {"production.type": "official", ...}
samweb modify-metadata changes_list.json        # list of {"file_name": ..., <fields>} objects
samweb retire-file $F                           # the name stays reserved; bytes are untouched
```

- Descriptive fields (`Dataset.Tag`, `production.*`, `sbnd_project.*`, `runs`,
  `parents`, ...) are editable. Never put `file_size` or `checksum` in a modify payload:
  SAM accepts the change without checking it (tested 2026-09-22).
- `file_name` cannot be changed. A rename is `retire-file` + a new declare.
- Definitions are dynamic queries, so retagging a file moves it between definitions
  automatically.

### 3.4 Definitions

```bash
samweb create-definition ${USER}_diffusion_v10_05_00 "dataset.tag ${USER}_diffusion_v10_05_00"
samweb describe-definition ${USER}_diffusion_v10_05_00
samweb count-definition-files ${USER}_diffusion_v10_05_00
samweb list-definition-files ${USER}_diffusion_v10_05_00 | head
```

---

## 4. Add a path (location) to SAM

A location is the **directory** that holds the file under its SAM name. Add one per copy:

```bash
samweb add-file-location $F dcache:/pnfs/sbnd/persistent/users/$USER/analysis       # disk copy
samweb add-file-location $F enstore:/pnfs/sbnd/archive/sam_managed_users/$USER/tag1/00   # tape copy
samweb -e sbn add-file-location $F enstore:/pnfs/sbnd/archive/...                    # repeat per instance the file is declared in
```

Check and remove:

```bash
samweb locate-file $F
#   enstore:/pnfs/sbnd/archive/sam_managed_users/liangliu/bethanym/diffusion_v10_05_00/55kv_tpc0/03(nearline)
#   dcache:/pnfs/sbnd/scratch/users/bethanym/v10_05_00/diffusion/decodefilter/55kv_tpc0/93283523_905
samweb get-file-access-url --schema=root $F           # one root://fndcadoor.fnal.gov:1094/... URL per location
samweb list-file-locations --defname=<def> --checksums                  # bulk: location<TAB>name<TAB>size<TAB>checksums
samweb list-file-locations --defname=<def> --filter-path=/pnfs/sbnd/archive   # only the tape locations
samweb remove-file-location $F dcache:/pnfs/sbnd/persistent/users/$USER/analysis
```

Rules:

- Use `dcache:` for disk directories and `enstore:` for tape-backed directories
  (section 1.2 table). `locate-file` appends `(nearline)` to tape locations; strip it when
  comparing strings.
- Only the record owner can add a location. For a `sbndpro` file run the command as
  `sbndpro`; the error otherwise is
  `User <you> is not an admin and file <name> is owned by sbndpro`.
- Adding a location that already exists is harmless; check with `locate-file` first if
  you script it.
- The file must really be in that directory under exactly its SAM name: `ls -l DIR/$F`.
- A file may have several locations at once (disk + tape + Imperial). Jobs pick by
  availability; remove the disk location only after the tape copy is verified (section 6).

---

## 5. Check the tape area

### 5.1 Is this directory tape-backed, and which family?

```bash
D=/pnfs/sbnd/archive/sam_managed_users/$USER/tag1
cat "$D/.(tags)()"                        | tr -d '\r'   # which tags exist
cat "$D/.(tag)(file_family)"              | tr -d '\r'   # sbnd | reco1_mc | reco1_data | data_bnblight ...
cat "$D/.(tag)(library)"                  | tr -d '\r'   # CD-LTO8G2 = a tape library; None = disk only
cat "$D/.(tag)(storage_group)"            | tr -d '\r'   # sbnd
ls -ld $D                                                # write permission (parents are group sbnd, rwx)
```

Dot-command output ends in `\n\r`, hence the `tr`. A directory you `mkdir` under a tagged
parent inherits the parent's tags (verified for `sam_managed_users/<user>/`, `to_tape/<user>/`
and the two `reco1` directories). Trying to write a tag yourself gives "Permission denied".

There is no per-user tape quota to check. Disk-side pool usage for the source areas:
`python3 dcache_usage.py` (repo root). For multi-TB campaigns tell SBND data management first,
drives and staging pools are shared with production.

### 5.2 Is this file on tape? One file, via NFS

```bash
D=/pnfs/sbnd/archive/sam_managed_users/liangliu/bethanym/diffusion_v10_05_00/55kv_tpc0/03
F=data_EventBuilder1_art2_run18618_175_strmCrossingMuon_20250711T072955_decoded-filtered_Reco1Comm-20260817T174821.root
cat "$D/.(get)($F)(locality)" | tr -d '\r'      # ONLINE_AND_NEARLINE
cat "$D/.(get)($F)(checksum)" | tr -d '\r'      # ADLER32:08a89bef
```

**Caveat**: the NFS client caches dot-command answers per file name. After a file is
replaced or migrates, the same node can keep returning the old checksum or locality for a
long time (seen 2026-09-11). For anything you will act on, ask the server side (5.3, 5.4)
or use another node.

### 5.3 Server-side size and checksum of one file (WebDAV door)

Doors see `/pnfs/X` as `/pnfs/fnal.gov/usr/X`.

```bash
curl -s --capath /etc/grid-security/certificates -I \
     -H "Authorization: Bearer $(cat $BEARER_TOKEN_FILE)" -H "Want-Digest: adler32" \
     "https://fndcadoor.fnal.gov:2880/pnfs/fnal.gov/usr${D#/pnfs}/$F" | tr -d '\r' | grep -iE '^(HTTP|Content-Length|Digest)'
# HTTP/1.1 200 OK
# Content-Length: 600566017
# Digest: adler32=08a89bef
```

Compare with the `File Size` and `adler32:` lines of `samweb get-metadata $F`. An expired
token gives `HTTP/1.1 401` with an empty body, not a mismatch.

### 5.4 Tape locality in bulk (tape REST API)

```bash
python3 - <<'EOT'
import json, os, subprocess
paths = ["/pnfs/fnal.gov/usr" + p[len("/pnfs"):] for p in open("paths.txt").read().split()]   # NFS paths, one per line
tok = open(os.environ["BEARER_TOKEN_FILE"]).read().strip()
for i in range(0, len(paths), 500):                                                           # 500 paths per request
    body = json.dumps({"paths": paths[i:i+500]})
    out = subprocess.run(["curl", "-sS", "--capath", "/etc/grid-security/certificates", "-X", "POST",
                          "-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json",
                          "--data-binary", body, "https://fndcadoor.fnal.gov:3880/api/v1/tape/archiveinfo"],
                         capture_output=True, text=True).stdout
    for e in json.loads(out):
        print(e.get("locality", "ERROR " + e.get("error", "?")), e["path"])
EOT
```

Answers are `DISK`, `DISK_AND_TAPE`, `TAPE`, or an `error` entry (missing file). It takes
well under a second per 500 files. The repo's `copy_to_tape.sh <card> --status` does exactly
this for a whole campaign and writes `<TAG>.locality.tsv`.

### 5.5 What does SAM think is on tape?

```bash
samweb list-file-locations --defname=<def> --filter-path=/pnfs/sbnd/archive | wc -l   # files with a tape location
```

SAM only knows the *locations you told it*; it does not know the dCache locality. The
truth about tape comes from 5.2 to 5.4.

### 5.6 Reading files back from tape

```bash
samweb prestage-dataset --defname=<def> --parallel=4     # bulk recall, sorted by tape; do this before any batch read
gfal-bringonline --pin-lifetime 86400 davs://fndcadoor.fnal.gov:2880/pnfs/fnal.gov/usr/sbnd/archive/...   # one file
```

Never read `NEARLINE` files one by one through xrootd or NFS: each one becomes a separate
tape mount. Copy staged files out promptly; staged replicas are evicted from the shared pools.

---

## 6. Move a file to tape and add the tape path to SAM

The file must already be declared (section 3). The manual recipe for one or a few files:

```bash
# 0. environment + token (section 2); run long copies inside tmux

# 1. source and destination
SRC=/pnfs/sbnd/persistent/users/$USER/analysis
F=myfile_run18618_reco1.root
DST=/pnfs/sbnd/archive/sam_managed_users/$USER/tag1/00       # tape-backed; <= a few hundred files per dir
mkdir -p $DST
[[ $(cat "$DST/.(tag)(file_family)" | tr -d '\r\n') == sbnd ]] || echo "WRONG FAMILY, stop"

# 2. size must match SAM before you copy (catches truncated sources)
[[ $(stat -c %s $SRC/$F) == $(samweb get-metadata $F | awk '/File Size/{print $NF}') ]] || echo "SIZE MISMATCH, stop"

# 3. copy over NFS to a .part name, then rename (dCache NFS writes are sequential only)
cp $SRC/$F $DST/$F.part && mv $DST/$F.part $DST/$F

# 4. verify the copy on the server side (door), against SAM's adler32
SAM_ADLER=$(samweb get-metadata $F | grep -oE 'adler32:[0-9a-f]+' | cut -d: -f2)
curl -s --capath /etc/grid-security/certificates -I \
     -H "Authorization: Bearer $(cat $BEARER_TOKEN_FILE)" -H "Want-Digest: adler32" \
     "https://fndcadoor.fnal.gov:2880/pnfs/fnal.gov/usr${DST#/pnfs}/$F" | tr -d '\r' | grep -iE '^(HTTP|Content-Length|Digest)'
#   expect HTTP 200, Content-Length == file_size, Digest: adler32=$SAM_ADLER
#   (the Digest header may be empty for a few seconds after the copy while dCache computes it: retry)

# 5. record the tape location, in every instance the file is declared in
samweb -e sbnd add-file-location $F enstore:$DST
samweb -e sbnd locate-file $F

# 6. wait for migration (minutes to hours), then confirm
cat "$DST/.(get)($F)(locality)" | tr -d '\r'    # ONLINE -> ONLINE_AND_NEARLINE (or use 5.4 from another node)

# 7. only now, if you want to free the disk copy
samweb remove-file-location $F dcache:$SRC
rm $SRC/$F
```

Notes:

- Write through **NFS**, not through the doors: the standard Analysis token has no
  `storage.create` scope for `/sbnd/archive`, while NFS writes work through the `sbnd`
  group permission. Reads through the doors work.
- A wrong copy: `rm $DST/$F` before it migrates. Once on tape, deleting only removes the
  catalog entry from dCache; the tape bytes stay until a repack. Prefer
  `samweb retire-file` + leave the bytes for a file that was archived by mistake.
- A copy that landed in the wrong directory can be **moved inside dCache** (`mv`, same
  instance and same file family): bytes and tape copy stay put. Then add the new
  `enstore:` location and remove the old one. `to_tape/fix_tape_paths.sh` does this for a
  whole campaign.
- 4–8 parallel copies are the useful range; more streams only add contention. Measured
  2026-09-22: 120–150 MiB/s per copy, ~800 MiB/s aggregate with 8 workers.
- Keep the triangle simple: one directory subtree = one `Dataset.Tag` = one definition,
  and a few hundred files per directory at most.

---

## 7. Bulk campaigns with the repo tools

All of these are dry-run by default (`--commit` to act), resumable, and journaled. Run
them from `/exp/sbnd/app/users/liangliu/sbnd-data-management`.

| task | tool | runbook |
|---|---|---|
| Move a set of your own files to tape in five steps (declare, copy, add location, check, remove originals) | `to_tape/templates/step1_declare.py` ... `step5_remove_source.sh` | `to_tape/move-files-to-tape.md` |
| Inventory a directory tree before archiving (sizes, zero-byte files, path list) | `sbnd_scan.py DIR --paths paths.txt` | README |
| Declare a set of artroot job outputs that are not in SAM (metadata from sidecars, dCache checksums, fcl, raw inputs) | `to_tape/declare_artroot.py scan / gen / declare / definitions / verify` | `to_tape/bethanym-diffusion.md` |
| Copy every file of a SAM definition to a tape directory, keep the names, verify through the door, add `enstore:` locations | `to_tape/copy_to_tape.sh job.card`, then `--status` | `to_tape/copy-to-tape.md` |
| Archive many **small** files: pack into ~10 GiB tars with an embedded index, declare bundle (+ member) records | `to_tape/sbnd_tape.py template / tar / copy / declare / status` | `to_tape/sbnd-tape.md` |
| Relocate tape copies inside dCache and repoint SAM | `to_tape/fix_tape_paths.sh job.card OLD_DST_PATH` | `to_tape/copy-to-tape.md` |
| Declare SPINE h5 and merged CAFs | `sam_declare.py` | `sam-declare-spine.md` |

A `copy_to_tape.sh` job card (bash `key=value`) for a user campaign:

```bash
DEFINITION=alice_diffusion_v10_05_00        # SAM definition to copy
NFILES=5                                     # pilot first; then "all" (done files are skipped)
SRC_PATH=/pnfs/sbnd/scratch/users/alice/diffusion      # files' dcache: locations are below this
DST_PATH=/pnfs/sbnd/archive/sam_managed_users/alice/diffusion_v10_05_00   # parent must exist and be tagged
FILE_FAMILY=sbnd                             # refused if DST_PATH's tag differs
DST_LAYOUT=bucket:250                        # keep | strip | bucket:N (at most N files per directory)
EXPERIMENTS="sbnd"                           # instances that get the enstore: location
DECLARE=1
WORKERS=8
TAG=alice_diffusion
WORK_DIR=/exp/sbnd/data/users/alice/sbnd_tape/diffusion   # ledgers: <TAG>.done.tsv, failed.txt, locality.tsv
```

```bash
cd to_tape
./copy_to_tape.sh job_alice.card              # copy + verify + add-file-location, in tmux
./copy_to_tape.sh job_alice.card --status     # DISK / DISK_AND_TAPE per copied file
```

---

## 8. Troubleshooting

| symptom | cause | fix |
|---|---|---|
| `Attempting to set user to 'X' but authenticated as 'Y' and not an admin` | `user` in the JSON is not you | set `user` to your username, or run as that account |
| `User X is not an admin and file ... is owned by sbndpro` | adding a location to someone else's record | run the command as the owner |
| `File with name '...' already exists` | name taken in this instance | pick a new name (derived files: add a suffix), or `retire-file` the old record if it is yours and wrong |
| `Unknown data tier '...'` / unknown file format | value not registered | use a listed value (`samweb list-values data_tiers`) or ask an admin to `add-value` |
| `Parent file ... not found` | parents must be declared first, in the same instance | declare parents first, or drop `parents` |
| samweb write command loops on redirects and dies with `RecursionError` | no valid bearer token found | `export BEARER_TOKEN_FILE=...` and run `htgettoken` (section 2) |
| door answers `HTTP 401` with an empty body | expired token | renew the token; it is not a checksum mismatch |
| dot-command shows the old checksum/locality after a rewrite | NFS client cache per name | ask the door or the tape REST API, or another node |
| `ONLINE` for hours after the copy | migration not done yet | wait; check again with 5.4. Do not delete the source |
| `samweb verify-file-checksum` says file not found | it takes a **local path**, not a bare name | `samweb verify-file-checksum /pnfs/.../file` |
| a bash array named `LINES` loses its first element in tmux | bash rewrites `LINES`/`COLUMNS` when a tty is attached | never name variables `LINES` or `COLUMNS`; `shopt -u checkwinsize` |
| a script missed a samweb error | samweb writes its error text to stdout, not stderr | test the exit code and grep the captured output |
