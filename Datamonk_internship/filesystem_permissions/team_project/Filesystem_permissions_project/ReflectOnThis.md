# Linux Permissions & `sudo` – Deep‑Dive Q\&A

> Answers prepared for internship submission. Concise, accurate, and focused on real‑world impact.

---

## 1) File vs. Directory `x` (execute) — why you need it to *enter* a directory

**Short answer:**

* On a **file**, `x` means “this file can be executed as a program.”
* On a **directory**, `x` means **search/traverse** permission: you’re allowed to *enter* the directory and follow path lookups inside it.

Even if you only want to **read** a file inside a directory, the kernel must walk each component of the path. That requires `x` on **every directory** in that path. Without `x`, you cannot `cd` into it, cannot `open()` a file by pathname inside it, and most tools will return *Permission denied*.

**Practical rule of thumb:**

* `r` on a directory lets you **list names** inside it.
* `x` on a directory lets you **access** those names (stat/open them) and **traverse** into subdirectories.
* To reliably work inside a directory: you usually need **`r+x`**.

**Demo you can try:**

```bash
mkdir -p demo/d && echo "hello" > demo/d/a.txt
chmod 444 demo/d             # remove x on the directory (r--r--r--)
cat demo/d/a.txt             # → Permission denied (cannot traverse without x)
chmod 555 demo/d             # add x back (r-xr-xr-x)
cat demo/d/a.txt             # → works, we can traverse + read the file
```

**Takeaway:** *Directory* `x` has nothing to do with executing code; it is about **traversal/search**. You need it to reach files by path.

---

## 2) The "777" risk — how one world‑writable file can sink a site

`777` (rwxrwxrwx) makes a file **writable by anyone** on the system, including other users, compromised processes, or untrusted services. On a web server, a **single** world‑writable file can lead to full compromise.

**Scenario:**

* A PHP app has `index.php` (or `config.php`) set to `777` to “fix” a write error.
* An attacker finds any minor foothold (e.g., a vulnerable plugin, an unrelated local user on shared hosting, or a service misconfiguration) that lets them **modify** that file.
* They append malicious code so that, on every request, the web server **executes attacker‑controlled code**. From there, the attacker can:

  * steal DB credentials from the config and dump data,
  * plant backdoors in other files,
  * serve malware to visitors and alter payment flows,
  * pivot to the OS user that runs the web server and escalate further.

**Why just one file is enough:** if that file is **executed or included** on requests, code execution becomes persistent and site‑wide.

**Safer alternatives:**

* Give the file the **least** required permissions (e.g., `640` or `644`) and ensure the **owner/group** are set so that only the web user or deployment user can write.
* For upload directories, use directory permissions like `rwx` for owner and **never** `777`. Prefer `750/755` directories with controlled ownership and server‑side validation.

---

## 3) Symbolic vs. octal `chmod` — why `chmod g+x a_file.txt` is safer

Given current mode: `-rwx-w-r--`

* Owner: `rwx` → `7`
* Group: `-w-` → `2`
* Others: `r--` → `4`
* Octal now is `724`.

You want to **add group execute** only. Two approaches:

**Octal approach (risky):**

* You must recompute the full number: group `2` → `2+1=3`, so you’d set `734`.
* Easy to miscalculate and accidentally flip unrelated bits.
* If **special bits** (setuid/setgid/sticky) are present, using a 3‑digit octal (e.g., `734`) can accidentally **clear** them unless you remember the leading digit (e.g., `2734`).

**Symbolic approach (safe & intention‑revealing):**

```bash
chmod g+x a_file.txt
```

* Changes **only** the requested bit (group execute), preserving all other existing bits and any special bits.
* Works regardless of the current mode, so it’s **idempotent** and safer in scripts.

**Bottom line:** Use **symbolic** mode when you want to tweak specific permissions without risking collateral changes.

---

## 4) `sudo` + a stray space: `sudo rm -rf / temp_files/` — why this is catastrophic

**What the command does:**

* It passes **two targets** to `rm -rf`: the root directory `/` and `temp_files/`.
* With `sudo`, `rm` runs as **root**, so it can delete system files that a normal user can’t.

**Outcome:**

* `rm -rf /` attempts to recursively delete the **entire filesystem**. Modern GNU `rm` typically has a safeguard (`--preserve-root`) that *refuses* to operate on `/`, but **you must not rely on this**: different environments (e.g., BusyBox, custom builds, containers, chroots) may not protect `/`, and flags/environment can disable the safeguard.
* Even if `/` is spared, `temp_files/` (in the current directory) will still be removed.

**Why `sudo` makes it worse:** it removes the natural protection of permission errors. If the safeguard isn’t active, this command can render the system **unbootable** within seconds.

**Safer patterns you should adopt:**

```bash
# 1) Always anchor to the current dir explicitly and stop option parsing
sudo rm -rf -- ./temp_files

# 2) Use an absolute path you verified with pwd/realpath
sudo rm -rf -- "/var/www/myapp/tmp"

# 3) Prefer safer tools during development
gio trash ./temp_files        # GNOME/GLib trash
trash-put ./temp_files        # from trash-cli

# 4) Guardrails
alias rm='rm -I'              # confirm when removing many files
# or install safe-rm to protect critical paths
```

