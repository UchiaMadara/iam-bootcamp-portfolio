##ACL's

#What is ACL's (Access Control List)
- ACLs are a flexible addition to *chmod* and *chown*. The default permissions
  give access to whole categories (owner, group, others). An ACL lets me adjust
  access for a **single account** — even if the default permissions would
  otherwise allow or block that account.
- In practice: if *chmod* says "group iam-hr can read this folder", an ACL
  can say "but alex (who is not in iam-hr) can also read it" — or
  "taylor (who IS in iam-hr) cannot."
- It is layered **on top of** standard permissions, not a replacement for them.
- The *+* at the end of an *ls -ld* line (e.g. *drwxr-x---+*) means an ACL
  is present on that file or directory.

#Commands
- *setfacl* sets the ACL entries on a file or directory.
- *getfacl* reports all ACL settings for the file or directory in question.

#Goal
- To understand how ACL permissions are assigned, how they are removed,
  and how to check them to verify what access actually exists.

# Lab A — The Drop Box (setgid + sticky)
`Goal`
- To build a shared folder that:
  1. anyone in iam-admins can enter and add files to
  2. new files inherit the folder's group (iam-admins), not the creator's primary group
  3. no one can delete anyone else's files

#Commands Run & Output

`Step 1 — create the folder`
- *sudo mkdir /srv/iamlab/drop*
  - creates a folder named *drop* inside */srv/iamlab*

`Step 2 — set ownership`
- *sudo chown root:iam-admins /srv/iamlab/drop*
  - owner = root (the user)
  - group = iam-admins (the group)
  - needs sudo because only root can reassign ownership

`Step 3 — set permissions`
- *sudo chmod 3770 /srv/iamlab/drop*
  - 3 = special bit (setgid 2 + sticky 1)
  - 770 = owner rwx, group rwx, others ---
  - combined: setgid + sticky + rwxrwx---

`Step 4 — verify`
- *ls -ld /srv/iamlab/drop*
  - output: drwxrws--T 2 root iam-admins 4096 ... /srv/iamlab/drop
  - the `s` in the group triad = setgid
  - the `T` in the others triad = sticky bit, uppercase because others has NO execute

`Step 5 — prove group inheritance`
- *sudo -u alex touch /srv/iamlab/drop/alex-file.txt*
  - creates a file as alex (run with sudo -u to act as that user)
- *sudo ls -la /srv/iamlab/drop*
  - output: -rw-r--r-- 1 alex iam-admins 0 ... alex-file.txt
  - group = iam-admins, NOT alex → setgid inherited the folder's group

#Permission String — Decoded

  drwxrws--T 2 root iam-admins 4096 ... /srv/iamlab/drop
  │└┬┘└┬┘└┬┘   └─┬┘ └────┬────┘
  │ │  │  │      │       └── group name = iam-admins
  │ │  │  │      └────────── owner name = root
  │ │  │  └───────────────── others: --T (sticky, no execute)
  │ │  └──────────────────── group:  rws (setgid + rwx)
  │ └─────────────────────── owner:  rwx
  └───────────────────────── d = directory

#Why the `ls` as itachi failed — Permission denied

When I ran *ls -la /srv/iamlab/drop* as itachi (not root), I got:
  ls: cannot open directory '/srv/iamlab/drop': Permission denied

Why:
- /srv/iamlab/drop is 3770
- itachi is NOT root, NOT in iam-admins → falls into "others"
- others = --T → NO read, NO write, NO execute
- so itachi cannot list the directory, correct behaviour

*sudo ls -la* works because root bypasses permission checks.

IAM triage pattern:
  1. which group is the user in?         id username
  2. what are the group/others perms?    ls -ld folder
  3. is the user in the owning group?    getent group iam-admins

#What I Learned
- setgid (s) makes new files inherit the FOLDER's group, not the creator's primary group
- sticky (T) stops users from deleting each other's files in a shared folder
- the `T` is uppercase because the others triad has no execute (x absent)
- `Permission denied` on `ls` is not a bug — it's the access control working
- `sudo -u username command` runs a command AS that user, useful for testing access

#Lab A result
- drop folder created with drwxrws--T (3770) 
- alex-file.txt created with group iam-admins (inherited via setgid) 


# Lab B — Prove the Sticky Bit

#Goal
- To prove that the sticky bit on /srv/iamlab/drop stops a non-owner
  from deleting another user's file.
- To prove it in BOTH directions:
  1. sticky ON  → alex CANNOT delete taylor's file
  2. sticky OFF → alex CAN delete taylor's file (proves sticky was the blocker)

#Baseline — verify BEFORE the test

`Check the drop folder`
- *ls -ld /srv/iamlab/drop*
  output: drwxrws--T 2 root iam-admins ... /srv/iamlab/drop
  the `s` = setgid (group inherits iam-admins)
  the `T` = sticky, uppercase because others has NO execute

`Check who owns what`
- *sudo ls -la /srv/iamlab/drop*
  alex-file.txt   owned by alex   group iam-admins
  (taylor-file.txt not yet created)

#Commands Run & Output

`B1 — create a file as taylor`
- *sudo -u taylor touch /srv/iamlab/drop/taylor-file.txt*
- *sudo ls -la /srv/iamlab/drop*
  output:
    -rw-r--r-- 1 alex   iam-admins ... alex-file.txt
    -rw-r--r-- 1 taylor iam-admins ... taylor-file.txt
  → owner differs (alex vs taylor), group is the SAME (iam-admins)
  → proves setgid inheritance across different creators ✅

`B2 — try to delete taylor's file as alex (should FAIL)`
- *sudo -u alex rm /srv/iamlab/drop/taylor-file.txt*
  output: rm: cannot remove '/srv/iamlab/drop/taylor-file.txt': Operation not permitted
  → sticky bit blocks non-owner delete ✅

`B3 — delete as taylor (should SUCCEED)`
- *sudo -u taylor rm /srv/iamlab/drop/taylor-file.txt*
- *sudo ls -la /srv/iamlab/drop*
  output: only alex-file.txt remains
  → owner CAN delete their own file ✅

`B4 — before/after comparison (portfolio gold)`

  # 1. recreate taylor's file
  sudo -u taylor touch /srv/iamlab/drop/taylor-file.txt

  # 2. remove sticky bit
  sudo chmod -t /srv/iamlab/drop
  ls -ld /srv/iamlab/drop
  output: drwxrws--- 2 root iam-admins ... /srv/iamlab/drop
  → `T` is GONE (uppercase T → no t at all)

  # 3. now alex CAN delete taylor's file
  sudo -u alex rm /srv/iamlab/drop/taylor-file.txt
  output: (no error — succeeds)
  → proves sticky bit was THE blocker ✅

  # 4. restore sticky bit
  sudo chmod +t /srv/iamlab/drop
  ls -ld /srv/iamlab/drop
  output: drwxrws--T 2 root iam-admins ... /srv/iamlab/drop
  → `T` is back ✅

#The Result — table of proof

  Step | Sticky | Can alex delete taylor's file? | Proves
  -----|--------|-------------------------------|--------
  B2   | ON     | NO — Operation not permitted  | sticky blocks
  B4.3 | OFF    | YES — succeeds                | sticky was the blocker
  B4.4 | ON     | (restored for safety)         | restored

#Why the sticky bit works this way

- Sticky bit on a folder = "only the file's OWNER (or root) can delete it"
- alex has WRITE access to the folder (group iam-admins, rwx)
- But write access to a folder does NOT grant delete rights to files he doesn't own
- The sticky bit overrides folder-write for delete purposes
- Root bypasses all permission checks → root can still delete anything

#What I Learned

- Sticky bit is the canonical way to make a shared "drop box" safe:
  everyone can add, but only the owner can delete their own file
- Without sticky, a shared writable folder = anyone can delete anyone's files
- `Operation not permitted` is the sticky-bit error; `Permission denied` would be
  a perms error — different messages, different causes
- `chmod -t` removes sticky; `chmod +t` restores it
- Before/after comparison (B4) is the strongest evidence — it isolates the variable
- /tmp is the real-world example: drwxrwxrwt (1777) — sticky + world-writable
- In AD (Week 5): this is "Creator Owner" permissions on a shared drive —
  each user gets full control only over files they create
- In ServiceNow (Week 8): "user cannot delete another user's file" tickets
  often trace back to missing sticky bit on a shared folder


# Lab C — Grant Alex Read-Only via ACL

#Goal
- To grant alex **read-only** access to /srv/iamlab/hr
  WITHOUT adding him to the iam-hr group.
- To prove least privilege: read works, write does not.
- To prove the ACL can be revoked cleanly.

#Baseline — verify BEFORE the change

`Check alex's groups`
- *id alex*
  output: uid=1001(alex) gid=1001(iam-admins) groups=1001(iam-admins)
  alex is NOT in iam-hr ✅

`Check the folder`
- *ls -ld /srv/iamlab/hr*
  output: drwxrwx---+ 2 root iam-hr ... /srv/iamlab/hr
  owner=root, group=iam-hr, perms=770 (+ existing ACL mask)

`Test baseline access`
- *sudo -u alex ls /srv/iamlab/hr*
  output: ls: cannot open directory '/srv/iamlab/hr': Permission denied
  ✅ correct — alex is in "others" = --- → no access

#Commands Run & Output

`C2 — apply the ACL`
- *sudo setfacl -m u:alex:r-x /srv/iamlab/hr*
  -m       = modify (add/change an entry)
  u:alex   = the USER alex
  :r-x     = read + execute, NO write
  target   = /srv/iamlab/hr
  (silence = success)

`C3 — verify with getfacl`
- *getfacl /srv/iamlab/hr*
  output:
    # file: srv/iamlab/hr
    # owner: root
    # group: iam-hr
    user::rwx
    user:alex:r-x     ← NEW ACL entry ✅
    group::rwx
    mask::rwx
    other::---

`C4 — verify the + marker`
- *ls -ld /srv/iamlab/hr*
  output: drwxrwx---+ 2 root iam-hr ... /srv/iamlab/hr
  the `+` = "ACL present" ✅

`C5 — prove read YES, write NO`
- *sudo -u alex ls /srv/iamlab/hr*
  output: (no error — read works) ✅
- *sudo -u alex touch /srv/iamlab/hr/alex-test.txt*
  output: touch: cannot touch '...': Permission denied ✅ (no `w` in r-x)

`C6 — revoke the ACL`
- *sudo setfacl -x u:alex /srv/iamlab/hr*
  -x       = remove the entry for this user
- *getfacl /srv/iamlab/hr*
  output: no `user:alex` line ✅ (but `mask::rwx` remains)
- *ls -ld /srv/iamlab/hr*
  output: drwxrwx---+ ... (`+` still present — because mask:: remains)
- *sudo -u alex ls /srv/iamlab/hr*
  output: ls: cannot open directory '...': Permission denied ✅
  → back to baseline

#What the + sign tells me

- `+` at the end of `ls -ld` perms = "this file/dir has an ACL attached"
- It does NOT tell me which users, what perms, or what type of entry
- It IS a flag: "there is more here than standard rwx shows — run getfacl"
- The `+` can remain after removing a user entry because the ACL **mask** persists
  → to remove ALL ACLs: *sudo setfacl -b /srv/iamlab/hr*

#What I Learned

- An ACL grants access to a SINGLE user without changing group membership
- `setfacl -m` = add/modify; `setfacl -x` = remove one entry; `setfacl -b` = remove all
- `getfacl` shows the full picture; `ls -ld` only shows the `+` flag
- Least privilege = grant exactly `r-x` (read+execute), never `rwx` by default
- "No error" at baseline can mean "already had access via group" — check `id user` first
- Always verify: baseline → apply → verify → restrict → revoke → verify

#Portfolio note

This lab is the direct parallel to:
- Week 5 (AD): "Advanced permissions" on a shared drive for one user
- Week 6 (JML): revoking a single user's access on offboarding
- Week 7 (Governance): auditing who has ACL access and why

# Checkpoint Answers

`Q1 — Why does setgid matter on the drop box?`

Without setgid, a file created in the drop folder inherits the CREATOR's
primary group, not the folder's group. If taylor (primary group iam-hr)
creates a file, its group becomes iam-hr — so iam-admins members lose
group access, and the folder stops being truly shared.

setgid fixes that: every new file inherits the folder's group iam-admins,
regardless of who created it.

IMPORTANT: setgid is ONLY about group inheritance. It says nothing about
deletion. Deletion is controlled by the sticky bit, not setgid.

`Q2 — Why does the sticky bit matter on the drop box?`

The sticky bit prevents a user from deleting or renaming a file they
don't own, even if they have write permission to the parent folder.

In Lab B, alex had write access to the drop folder (group iam-admins rwx),
but got `Operation not permitted` when trying to `rm` taylor's file —
because the sticky bit restricts deletion to the file's owner (and root).
Taylor could delete her own file.

Without sticky, a shared writable folder would let anyone delete anyone's
files. The sticky bit is what makes a "drop box" safe.

`Q3 — When would you use an ACL instead of changing group membership?`

Use an ACL when ONE specific user needs access that the group model
can't express cleanly.

Three real cases:
1. New starter needs read-only while the rest of the team has write.
2. A senior member needs extra access beyond the group default.
3. Inter-department sharing — one person from Finance needs to read
   an HR folder without joining the HR group.

Why not just create a new group? It's heavier: extra group in the directory,
extra membership to track, harder to audit and revoke. ACL = surgical,
group membership = broad. Use ACL when the need is one person, not the group.
