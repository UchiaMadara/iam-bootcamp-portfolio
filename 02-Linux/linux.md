## Week 1: Linux user and group management

# Goal
To get an understanding of bacis commands for navigation and to understand
 file and user management.

## Commands Run & Output
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
