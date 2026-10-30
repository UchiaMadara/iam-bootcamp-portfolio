### Terminal notes
- , is NOT a separator in bash. mkdir a, chown b makes mkdir try to create dirs literally
 named a,, chown, and b. Classic footgun.
- Use ; (run anyway), && (run only if success), || (run only if fail), or just newlines.
   && is the safe chain: stops on failure, prevents building on broken state.
- Before any rm -rf, run ls on the target first. rm -rf is the most dangerous command in Linux
- setgid *chmod g+s* — shared folders

Problem: without setgid
-  new files inherit the CREATOR's primary group,
   not the folder's group. So a shared folder stops being shared.

Solution: setgid on the folder makes new files inherit the FOLDER's group.
-  Set:      chmod g+s /srv/iamlab/hr
   Numeric:  chmod 2770 /srv/iamlab/hr   (2 = setgid)
   Recognise: drwxrws---  ← the 's' in group triad

AD parallel: "inherited permissions" on shared drives. Same principle.




## Week 1: Linux user and group management

# Goal
To get an understanding of bacis commands for navigation and to understand
 file and user management.

# Commands Run & Output
`Groups`
- *sudo groupadd* iam_admins These commands create the group
- *sudo groupadd* iam_hr

`User`
- *sudo useradd -m -s /bin/bash -g iam_admins alex* creates the user 
- *sudo useradd -m -s /bin/bash -g iam_hr taylor*
	-m creates a home directory
	-s creates the login shell (defines which command-line interpreter
	   launches when that user logs into the system.)
	-g Specify primary group (the group must already exist)
	-G Specify additional groups (comma-separated, no spaces).
	   Users needing multiple role permissions at once
	-d Custom home directory path (Specialized directories)
	-u Specify UID manually	(Migration or consistency with existing systems)
	-e Account expiry date (YYYY-MM-DD). (Temporary/contractor accounts)
	-r Create service accounts
	-c Comment field (usually full name) User identification

- *sudo passwd alex* create a password for the user (this is the best practice)
- *sudo passwd taylor*

- *id alex* find out user information userid, groupdid
- *id taylor*

- *sudo usermod -aG iam-admins taylor* adds to new group and stays in old
- *sudo usermod -G iam-admins taylor*     # OVERWRITES, removes taylor from iam-hr

       -aG  Append additional groups (must be used together), add to more groups
	    without remove for the groups they already in.
	-G  Replace additional groups (old groups all removed)
	    Use with caution, can easily cause permission loss
	-g  Change primary group	Department transfers (primary group changes)
	-L  Lock account	JML offboarding first step
	-U  Unlock account. Rehire or temporary unlock
	-e  Modify account expiry date. Extend/shorten temporary access
	-s  Change shell	Disable login or switch environments
	-l  Change username. Rename after marriage or naming convention changes
    -d + -m	Modify home directory + move contents. Directory adjustments during
	        department transfers
	-u  Change UID	Rare, but needed during migrations

- *sudo gpasswd -a username groupname *  adds user 
- *sudo userdel username* removes user 
- *sudo groupdel (groupname)* deletes a group
- *getent passwd* Used to verify changes made or user created


# Different Shells

- */bin/bash*	Bourne Again Shell, Ubuntu's default login shell.Normal interactive
   users use this; when you run useradd -s /bin/bash in your lab, you're specifying it
- */bin/sh*     System's default shell (usually points to dash in Ubuntu).Script execution
   uses this, not for manual login
- */sbin/nologin* "Reject login" shell	Key point: After a user is set to this, they cannot
   log in interactively; commonly used for service accounts or accounts pending disabling
- */usr/sbin/nologin*	Same as above, slightly different path depending on distribution. 
   Treat it as equivalent to /sbin/nologin
- */bin/false*	Returns failure directly and exits. Older practice for disabling login;
   rarely used now, nologin is more common.

#Troubleshhoting possible issues
I ran some command to tcheck the output to varify user account and groups
	User / Group / Password state
- *groups taylor* what groups are they in
- *sudo passwd -S taylor* shows that they have a password and when it expires
- *sudo chage -l taylor* confirms password length and if the account is live or not
- *getent group iam-hr*  shows every account ecept if it is the users primary account
   in which case it there account wont show up.
- *sudo grep -E '^(alex|taylor):' /etc/shadow | awk -F: '{print $1, $3, $5, $8}'*

	/home navigation + permissions evidence
This was to check were i currently am, what users are on the system and
 permissions to access accounts.
 
- *cd /home/ ; pwd ; ls -la*
- *cd /home/taylor ; pwd*          
- *sudo ls -la /home/taylor*

	~ vs /home proof
This shows multiple ways to find home
- *echo "HOME=$HOME"*
- *ls -d /home ; ls -d ~*
- *ls ~/..*    

	the -aG two-direction drill (the actual warm-up)
- *sudo usermod -aG iam-admins taylor* adds to the group while remaining in the
   home directory
- *id taylor*   check if they appear in both groups
- *sudo gpasswd -d taylor iam-admins* remove from group
*id taylor*   check removal from group

#Notes
1- *getent group iam-hr* shows empty members because taylor's primary group
  is iam-hr (set via *useradd -g*), stored in `/etc/passwd` field 4, NOT in
  `/etc/group`. groups and id show it correctly. This is the #1 cause of
  "user not in group" false alarms in L1 tickets.
2- *sudo cd /home/taylor* → sudo: 'cd': command not found. Cause: cd is a
 shell builtin, not an external binary; sudo wraps binaries only. Workaround:
 sudo -i then cd, or *sudo ls -al /home/taylor*, or su taylor sudo cd /home/taylor
 → sudo: 'cd': command not found. Cause: cd is a shell builtin, not an external
 binary; sudo wraps binaries only. Workaround: sudo -i then cd, or
 *sudo ls -al /home/taylor*, or *su taylor*




##Permission — File, Group and User Permissions

#Goal
- Be fluent in understaning file, group and user permissions.
   understand both numerical notation and alphabet notation. How to take and give
   permissions.

#Theory — The Permission Model

`Three sets` — every file/dir has permissions for three audiences:
- *Owner (u)* — the user who owns it
- *Group (g)* — the group that owns it
- *Others (o)* — everyone else

`Three bits` — each set can have:
- *r* = read = *4*
- *w* = write = *2*
- *x* = execute = *1*
- *-* = none = *0*

`Numeric notation` — add bits per set:

  chmod 755 file
    owner = 4+2+1 = 7 → rwx
    group = 4+0+1 = 5 → r-x
    other = 4+0+1 = 5 → r-x
    result: rwxr-xr-x

  chmod 750 file → rwxr-x---   (owner rwx, group r-x, other ---)
  chmod 770 file → rwxrwx---   (owner rwx, group rwx, other ---)
  chmod 640 file → rw-r-----   (owner rw-, group r--, other ---)
  chmod 700 file → rwx------   (owner rwx, group ---, other ---)  
  
- `Symbolic notation` — change one bit at a time with + - =
- *+* add
- *-* remove
- *=* set exactly

  chmod u-r file.doc     removes read from owner
  chmod o-rwx file       removes all from others
  chmod g+s folder       adds setgid to group
  chmod +t folder        adds sticky bit 

`Reading ls -ld output`

  drwxr-x--- 2 root iam-admins 4096 ... /srv/iamlab/finance
  │└┬┘└┬┘└┬┘   └─┬┘ └────┬────┘
  │ │  │  │      │       └── group name
  │ │  │  │      └────────── owner name
  │ │  │  └───────────────── others perms
  │ │  └──────────────────── group perms
  │ └─────────────────────── owner perms
  └───────────────────────── type: d=dir, -=file, l=symlink


#Lab 
 ## Week 1: Linux user and group management

# Goal
To get an understanding of basic commands for navigation and to understand
file and user management.

# Commands Run & Output
`Groups`
- *sudo groupadd iam-admins*  creates the group
- *sudo groupadd iam-hr*

`User`
- *sudo useradd -m -s /bin/bash -g iam-admins alex*  creates the user
- *sudo useradd -m -s /bin/bash -g iam-hr taylor*
        -m creates a home directory
        -s creates the login shell (defines which command-line interpreter
           launches when that user logs into the system.)
        -g Specify primary group (the group must already exist)
        -G Specify additional groups (comma-separated, no spaces).
           Users needing multiple role permissions at once
        -d Custom home directory path (Specialized directories)
        -u Specify UID manually (Migration or consistency with existing systems)
        -e Account expiry date (YYYY-MM-DD). (Temporary/contractor accounts)
        -r Create service accounts
        -c Comment field (usually full name) User identification

- *sudo passwd alex*  create a password for the user (this is the best practice)
- *sudo passwd taylor*

- *id alex*  find out user information userid, groupid
- *id taylor*

- *sudo usermod -aG iam-admins taylor*  adds to new group and stays in old
- *sudo usermod -G iam-admins taylor*   # OVERWRITES, removes taylor from iam-hr

---

##Permission — File, Group and User Permissions (Complete)

#Goal
- Be fluent in understanding file, group and user permissions.
  Understand both numeric and symbolic notation.
  Know how to give and take permissions.
  Know how to read `ls -ld` output.
  Understand why shared folders need setgid.

#Theory — The Permission Model

`Three sets` — every file/dir has permissions for three audiences:
- **Owner (u)** — the user who owns it
- **Group (g)** — the group that owns it
- **Others (o)** — everyone else

`Three bits` — each set can have:
- **r** = read = **4**
- **w** = write = **2**
- **x** = execute = **1**
- **-** = none = **0**

`Numeric notation` — add bits per set:

  chmod 755 file
    owner = 4+2+1 = 7 → rwx
    group = 4+0+1 = 5 → r-x
    other = 4+0+1 = 5 → r-x
    result: rwxr-xr-x

  chmod 750 file → rwxr-x---   (owner rwx, group r-x, other ---)
  chmod 770 file → rwxrwx---   (owner rwx, group rwx, other ---)
  chmod 640 file → rw-r-----   (owner rw-, group r--, other ---)
  chmod 700 file → rwx------   (owner rwx, group ---, other ---)

`Symbolic notation` — change one bit at a time with + - =
- **+** add
- **-** remove
- **=** set exactly

  chmod u-r file.doc     removes read from owner
  chmod o-rwx file       removes all from others
  chmod g+s folder       adds setgid to group
  chmod +t folder        adds sticky bit

`Key difference — numeric vs symbolic`
- **Numeric** = absolute — sets all three triads at once
- **Symbolic** = surgical — changes only the bits you name

Starting from 775 (rwxrwxr-x):
  chmod o-rwx → 770 (rwxrwx---)   ← only others touched
  chmod 700   → 700 (rwx------)   ← group ALSO wiped

`Reading ls -ld output`

  drwxr-x--- 2 root iam-admins 4096 ... /srv/iamlab/finance
  │└┬┘└┬┘└┬┘   └─┬┘ └────┬────┘
  │ │  │  │      │       └── group name
  │ │  │  │      └────────── owner name
  │ │  │  └───────────────── others perms
  │ │  └──────────────────── group perms
  │ └─────────────────────── owner perms
  └───────────────────────── type: d=dir, -=file, l=symlink

#LAB - Commands Run & Output

`mkdir` — create directories
- *sudo mkdir -p /srv/iamlab/{finance,hr,public}*
  - `-p` = create parents if missing AND ignore "already exists" (idempotent)
  - `{a,b,c}` = brace expansion, shell rewrites to 3 separate paths
  - `/srv` is owned by root → needs sudo

`chown` — change owner and group
- *sudo chown root:iam-admins /srv/iamlab/finance*
- *sudo chown root:iam-admins /srv/iamlab/hr*
- *sudo chown root:iam-admins /srv/iamlab/public*
  - `OWNER:GROUP` — colon separates them
  - `root` = owner (first rwx triad)
  - `iam-admins` = group (second rwx triad)
  - Being in the group does NOT grant root — only grants what the group bits allow
  - Needs sudo because only root can reassign ownership

`chmod` — change permissions
- *sudo chmod 750 /srv/iamlab/finance*  → rwxr-x---
- *sudo chmod 770 /srv/iamlab/hr*       → rwxrwx---
- *sudo chmod 755 /srv/iamlab/public*   → rwxr-xr-x

`ls -ld` — verify (the -d lists the dir itself, not its contents)
- *ls -ld /srv/iamlab /srv/iamlab/**

Final output:
  drwxr-xr-x 5 root root       4096 ... /srv/iamlab
  drwxr-x--- 2 root iam-admins 4096 ... /srv/iamlab/finance   ← 750 ✅
  drwxrwx--- 2 root iam-admins 4096 ... /srv/iamlab/hr        ← 770 ✅
  drwxr-xr-x 2 root iam-admins 4096 ... /srv/iamlab/public    ← 755 ✅

# The Four Bugs I Hit (and What They Taught Me)

# Bug 1 — Comma is a character, not a separator
 `Comma Bug — Visual Evidence (2026-10-27)`
- What I ran (WRONG)
  *sudo mkdir -p /srv/iamlab/{finance,hr,public}, chown root:iam-admins*

- What I expected
  3 dirs (finance, hr, public) + chown applied

- What actually happened
   ls -la /srv/iamlab/ showed:
   drwxr-xr-x 2 root root 4096 ... finance,   ← comma in NAME
   drwxr-xr-x 2 root root 4096 ... hr,        ← comma in NAME
   drwxr-xr-x 2 root root 4096 ... public,    ← comma in NAME

- Why
   `,` is NOT a command separator in bash. It's a literal character
   glued to the argument. mkdir created dirs literally named "finance," etc.
   The subsequent "chown root:iam-admins" became EXTRA mkdir arguments,
   creating stray dirs in ~ (no / prefix).

- Lesson
   Use newline, ;, &&, or || to separate commands. Never a comma.
   && is the safe chain: runs next only if previous succeeded.
   Before rm -rf, run ls on the target first.

- Fix
   sudo rm -rf /srv/iamlab
   rm -rf ~/chown ~/'root:iam-admins'

# Bug 2 — Trailing tilde after brace expansion
 `Tilde Bug (2026-10-27)`
- What I ran (WRONG)
  *sudo mkdir -p /srv/iamlab/{finance,hr,public}~*

- What actually happened
  brace expansion produced: finance, hr, public~
  only the LAST element got the ~

- Why
  `~` is appended to the LAST expanded element only. It becomes part of the filename.

- Lesson
  Never put characters after a closing `}` in a path.

- Fix
  rebuild with explicit paths (no trailing characters)

# Bug 3 — Guessed group name and path
 `Wrong Names Bug (2026-10-27)`
- What I ran (WRONG)
  *sudo chown root:iamlab /srv/iam-iamlab*

- Why it failed
  group is `iam-admins` (not `iamlab`)
  path is `/srv/iamlab` (not `/srv/iam-iamlab`)

- Lesson
  Verify inputs BEFORE running mutating commands:
    getent group iam-admins
    ls -ld /srv/iamlab/finance

# Bug 4 — Two commands glued on one line
 `Glued Commands Bug (2026-10-27)`
- What I ran (WRONG)
  *sudo chown root:iam-admins /srv/iamlab/{finance,hr,public}getent group iam-admins*

- What actually happened
  chown tried to change ownership of:
    /srv/iamlab/financegetent
    /srv/iamlab/hrgetent
    /srv/iamlab/publicgetent
    group
    iam-admins
  ALL failed with "cannot dereference"

- Why
  no separator between the two commands. The second became
  extra arguments to the first.

- Lesson
  Press Enter between commands. One command per line. Always.

- Fix
  run each command separately, wait for the prompt, read the output

# The Rules I Learned

`Command separators`
- newline = run next command (safest)
- *;* = run next command regardless of success
- *&&* = run next command only if previous succeeded (safest chain)
- *||* = run next command only if previous failed
- `,` = NOT a separator — just a literal character

`Verify before mutate`
1. Verify the SOURCE exists:    *getent group iam-admins*
2. Verify the TARGET exists:    *ls -ld /srv/iamlab/finance*
3. Run the mutating command:    *sudo chown root:iam-admins /srv/iamlab/finance*
4. Verify the RESULT:           *ls -ld /srv/iamlab/finance*

`Boring is correct`
- Use longhand (one command per line) until it is muscle memory
- Add shortcuts (braces, loops) only AFTER the longhand is proven
- No one gets fired for typing chown three times
- People DO get fired for rm -rf on the wrong path

`rm -rf safety`
- Before running, always *ls* the target first
- *rm -rf /srv/iamlab* is safe (lab)
- *rm -rf /* or *rm -rf ~* is catastrophic

# Checkpoint Answers

`Q1 — What does chmod 640 grant?`

  6 4 0
  │ │ └── others: 0 = ---
  │ └──── group:  4 = r--
  └────── owner:  6 = rw-

Result: rw-r-----  (owner read+write, group read, others nothing)

`Q2 — Why does setgid (chmod g+s) matter on a shared HR folder?`

Without setgid, new files inherit the CREATOR's primary group,
not the folder's group. So a shared folder stops being shared —
the group has no access to files created inside it.

setgid on the folder makes new files inherit the FOLDER's group.
Now everyone in iam-hr can access everything created there.

  Without setgid:
    Alice creates file.txt → group = alice   ❌ (not iam-hr)
    Bob creates file.txt   → group = bob     ❌ (not iam-hr)

  With setgid on folder:
    Alice creates file.txt → group = iam-hr  ✅ (inherited)
    Bob creates file.txt   → group = iam-hr  ✅ (inherited)

Set:        *sudo chmod g+s /srv/iamlab/hr*
Numeric:    *sudo chmod 2770 /srv/iamlab/hr*   (2 = setgid)
Recognise:  drwxrws---   ← the `s` in the group triad

AD parallel: "inherited permissions" on shared drives (Week 5).

`Q3 — chmod o-rwx vs chmod 700?`

Starting from 775 (rwxrwxr-x):

  chmod o-rwx → 770 (rwxrwx---)   ← only others touched (surgical)
  chmod 700   → 700 (rwx------)   ← group ALSO wiped (absolute)

- Symbolic (o-rwx) = surgical — changes only the bits you name
- Numeric (700) = absolute — sets all three triads

Use symbolic to tweak one thing. Use numeric to set a known state.

# Special Bits (intro)

`setgid (g+s / 2xxx)`
- On a folder: new files inherit the folder's group (shared folders)
- Recognise: drwxrws---  (s in group triad)

`sticky bit (+t / 1xxx)`
- On a folder: only the file's owner (or root) can delete it
- Used on /tmp so users can't delete each other's files
- Recognise: drwxrwxrwt  (t in others triad)

# Lab Recap — The Full Command Sequence

  # 1. Create dirs
  sudo mkdir -p /srv/iamlab/finance
  sudo mkdir -p /srv/iamlab/hr
  sudo mkdir -p /srv/iamlab/public

  # 2. Set ownership
  sudo chown root:iam-admins /srv/iamlab/finance
  sudo chown root:iam-admins /srv/iamlab/hr
  sudo chown root:iam-admins /srv/iamlab/public

  # 3. Set permissions
  sudo chmod 750 /srv/iamlab/finance
  sudo chmod 770 /srv/iamlab/hr
  sudo chmod 755 /srv/iamlab/public

  # 4. Verify
  ls -ld /srv/iamlab /srv/iamlab/*

# umask

`Definition` — a subtraction mask applied at file/dir creation.
Not a permission itself — a "remove these bits" instruction.

`Formula`
  Directory: base 777  - umask = result
  File:      base 666  - umask = result

`Examples`
  umask 022 → dirs 755, files 644  (textbook default)
  umask 002 → dirs 775, files 664  (Ubuntu user default — group-writable)
  umask 027 → dirs 750, files 640  (server default)
  umask 077 → dirs 700, files 600  (private)

`Check / set`
  umask          # show current (e.g. 0022)
  umask -S       # symbolic (u=rwx,g=rx,o=rx)
  umask 077      # set for this session
  echo "umask 027" >> ~/.bashrc   # permanent

`Always run `umask` FIRST` before predicting new-file perms.
Never assume the default — it varies by distro and user type.

# /tmp — temporary storage

`Purpose` — world-writable scratch space for temp files.
Not for user data. Not for anything you need to survive a reboot.

`Cleanup mechanisms`
  - tmpfs (RAM): /tmp wiped on EVERY reboot
  - systemd-tmpfiles: deletes files older than 10 days (default)
  - custom rules in /etc/tmpfiles.d/

`Check which one`
  mount | grep ' /tmp '

`Safety` — /tmp is `drwxrwxrwt` (1777) — the sticky bit
means you can only delete YOUR OWN files. Canonical sticky-bit example.

`/var/tmp` vs `/tmp`
  /tmp      → ephemeral, wiped on reboot (or 10-day age)
  /var/tmp  → on disk, survives reboot, 30-day age

`For the bootcamp` — NEVER store portfolio evidence in /tmp.
Use ~/Desktop/iam-bootcamp-portfolio/ or the PSSD.

# Combined special bits (g+s + t)

  chmod 3770 folder   →  drwxrws--T
    3 = setgid(2) + sticky(1)
    s in group triad  = setgid (inherits group)
    T in others triad = sticky, uppercase because others has NO execute

  t (lowercase) = sticky + execute on others (others has x)
  T (uppercase) = sticky WITHOUT execute on others (others has no x)
  Same rule for s/S in owner/group triads.

Real use case: /srv/iamlab/drop — team can add files,
new files inherit iam-admins group, no one can delete each other's files.
