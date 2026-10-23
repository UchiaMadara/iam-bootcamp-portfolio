## Week 1: Linux 

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

- *sudo usermod -aG iam-admins taylor*
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
