---
name: vault-audit
description: Audit a Bitwarden vault for hygiene problems and repair them in bulk — entries missing URIs (the usual reason autofill "randomly doesn't work"), missing usernames, stub entries with no password, duplicate items, reused or weak passwords, and URIs captured from a /login or /forgot-password page instead of the domain. Works with either the official `bw` CLI or `rbw`. Trigger on "audit my passwords", "why doesn't autofill work", "find reused passwords", "clean up my vault", "check my Bitwarden", or any vault-hygiene request. Repairs are written in place — no export/purge/reimport, so passkeys and attachments survive.
---

# Vault audit

Most "my password manager is unreliable" complaints are not the manager. They are
vault data problems — overwhelmingly **entries with no URI**, which can never be
auto-suggested because matching is done on the URI field, not the item name.

This skill finds those problems and fixes the fixable ones in bulk.

## Before anything

**Never enter the master password.** Unlocking is the user's job. Interactive
prompts also need a real TTY — they fail inside most agent shells with
`ERR_USE_AFTER_CLOSE` or a silent hang. Hand the unlock to the user in their
terminal.

**Never print a password.** Compare passwords by hashing them; report an 8-char
fingerprint or a boolean. A vault audit that dumps secrets into a transcript is
worse than no audit.

**Never paste a session key into a conversation.** `bw unlock` prints one to
stdout. Route it to a file instead.

## Detect the backend

```bash
command -v bw  && echo "official CLI"
command -v rbw && echo "rbw"
```

They behave very differently. `bw` can read and write the whole vault as JSON.
`rbw` is read-friendly but has no scriptable write path for URIs.

### bw — unlock

Have the user run, in their own terminal:

```bash
bw unlock --raw > ~/.bw-session && chmod 600 ~/.bw-session
```

Then every command reads it without the value ever entering the transcript:

```bash
export BW_SESSION=$(cat ~/.bw-session)
```

Lock up afterwards: `bw lock && rm ~/.bw-session`

### rbw — unlock

```bash
rbw unlocked || echo "user must run: rbw unlock"
```

`rbw` caches the unlock in `rbw-agent`, so this is usually already done.

## The checks

With `bw`, one JSON dump covers everything:

```bash
bw list items --session "$BW_SESSION" </dev/null > /tmp/vault.json
```

> **Gotcha:** `bw` reads stdin. Inside a loop it will eat the loop's input and
> silently skip entries. Always redirect `</dev/null` on `bw` calls in a loop.

```bash
jq -r '
  [ .[] | select(.type==1) ] as $l |
  "logins:            \($l|length)",
  "no URI:            \([$l[]|select((.login.uris//[])|length==0)]|length)",
  "no username:       \([$l[]|select((.login.username//"")=="")]|length)",
  "no password:       \([$l[]|select((.login.password//"")=="")]|length)",
  "has TOTP:          \([$l[]|select(.login.totp)]|length)"
' /tmp/vault.json
```

**Reused passwords** — hash locally, never display:

```bash
jq -r '.[]|select(.type==1)|select(.login.password)|[.login.password,.name]|@tsv' /tmp/vault.json \
| python3 -c '
import sys,hashlib,collections
g=collections.defaultdict(list)
for line in sys.stdin:
    pw,_,name=line.rstrip("\n").partition("\t")
    g[hashlib.sha256(pw.encode()).hexdigest()].append(name)
for h,names in g.items():
    if len(names)>1: print(f"  reused across {len(names)}: {\", \".join(names)}")
'
```

**Bad URIs** — captured from a reset/login page rather than the domain:

```bash
jq -r '.[]|select(.type==1)|.login.uris//[]|.[]|.uri' /tmp/vault.json \
  | grep -iE "forgot|reset|signin|login|auth/" | sort -u
```

With `rbw`, there is no whole-vault JSON export (as of 1.15) and
`rbw list --fields` exposes only `id,name,user,folder,type` — **no URI**. Try
`rbw list --raw` first; if it omits URIs, fall back to iterating
`rbw get <name> --full`. That is slow and prints passwords, so pipe it straight
into a hasher and never to the terminal.

## Bulk URI repair (bw only)

Two passes: generate a review file with guessed domains, let the user correct
the one column, then write back **in place**. Never export/purge/reimport —
that duplicates entries and drops passkeys and attachments.

**Pass 1 — build the review file:**

```bash
bw sync --session "$BW_SESSION" </dev/null
{
  printf 'id\tname\tusername\turi\n'
  bw list items --session "$BW_SESSION" </dev/null | jq -r '
    def guess(n):
      (n|ascii_downcase) as $l
      | if ($l|test("\\.")) then ($l|gsub("[^a-z0-9.-]";""))
        else ($l|gsub("[^a-z0-9]";"")) + ".com" end;
    .[] | select(.type==1) | select((.login.uris//[])|length==0)
    | [.id, .name, (.login.username//""), "https://" + guess(.name)] | @tsv'
} > ~/vault-uri-review.tsv
```

Show the user the `name -> guessed uri` pairs and fix the obvious misses
yourself before handing it over — the naive guess gets common cases right
(`arxiv` → wrongly `arxiv.com`, should be `.org`; `American Airlines` →
wrongly `americanairlines.com`, should be `aa.com`). Blank or `SKIP` in the
uri column means leave that entry alone; use it for entries that aren't
websites at all, like an SSH key or an email account.

**Pass 2 — apply.** Use Python, not a bash `while read` loop:

```python
#!/usr/bin/env python3
import csv, json, base64, subprocess, os, sys
sess = open(os.path.expanduser("~/.bw-session")).read().strip()
def bw(*a):
    return subprocess.run(["bw", *a, "--session", sess],
                          capture_output=True, text=True, stdin=subprocess.DEVNULL)
done = failed = 0
with open(sys.argv[1] if len(sys.argv) > 1 else os.path.expanduser("~/vault-uri-review.tsv")) as f:
    for row in csv.DictReader(f, delimiter="\t"):
        uri = (row.get("uri") or "").strip()
        if not uri or uri == "SKIP":
            continue
        r = bw("get", "item", row["id"])
        if r.returncode:
            print(f"  FAIL get {row['name']}"); failed += 1; continue
        item = json.loads(r.stdout)
        item.setdefault("login", {})["uris"] = [{"match": None, "uri": uri}]
        r = bw("edit", "item", row["id"],
               base64.b64encode(json.dumps(item).encode()).decode())
        if r.returncode:
            print(f"  FAIL edit {row['name']}"); failed += 1; continue
        done += 1
bw("sync")
print(f"wrote {done}, failed {failed}")
```

> **Why not bash?** `IFS=$'\t' read -r a b c d` silently collapses *consecutive*
> tabs, because tab is a whitespace IFS character. Any row with an empty field —
> an entry with no username, which is exactly the population being audited —
> shifts left and the URI lands in the wrong variable. The rows are skipped with
> no error. Use `csv.DictReader`.

`bw edit item` takes **base64-encoded JSON**, either as an argument or on stdin
via `bw encode`.

## Optional: breach check

Only with explicit opt-in, and state plainly what leaves the machine: the
**first 5 characters of the SHA-1** of each password, nothing else. The API
returns every suffix in that bucket and the match happens locally. This is the
standard k-anonymity model; the password itself is never transmitted.

```bash
# for one password on stdin
read -rs PW
H=$(printf '%s' "$PW" | shasum | cut -d' ' -f1 | tr 'a-f' 'A-F')
curl -s "https://api.pwnedpasswords.com/range/${H:0:5}" \
  | grep -i "^${H:5}:" | cut -d: -f2
```

Skip this entirely if the user hasn't asked for it.

## Fixing the rest

Missing usernames and stub entries can't be repaired automatically — only the
user knows the values. Don't guess. The efficient route is to tell them the
extension's **"Ask to update existing login"** notification will capture the
username the next time they log in, rather than sitting down to fill in dozens
by hand.

Duplicates: confirm they're genuinely identical (same username *and* same
password hash) before collapsing. Two entries for the same domain are often two
real accounts — a personal and a shared one, say.

## Deletion rules

Soft-delete only. `bw delete item <id>` moves to Trash, recoverable for 30 days.
`--permanent` is not recoverable; don't use it without an explicit request.
Always say which items went to Trash and that they can be restored.

## Prevention

The audit is a one-time cleanup. Stop the problem recurring:

- Extension → Settings → Notifications → **Ask to add login** and
  **Ask to update existing login** on. New signups then capture username,
  password, and the correct URI automatically.
- Extension → Settings → Auto-fill → **Default URI match detection → Base
  domain**. `Host` and `Exact` cause constant misses across subdomains.
- Leave **auto-fill on page load** off — it fills matching forms without being
  asked, including hidden ones on a compromised page.
- Generate passwords from inside the manager at signup, not by hand.
