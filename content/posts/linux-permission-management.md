+++
title = "Linux Permission Management"
date = 2019-08-06T08:59:37+02:00
tags = ["linux", "permissions", "cheatsheet", "devops" ]
+++

Cheatsheet for _Linux Permission Management_

### Standard Permissions
There are three basic permissions which are `r`ead, `w`rite and e`x`ecute. And they can be granted to 

1. a specific __user__ (so if you are not that user those permissions do not apply to you)
2. a __group__ of users (if you're not a member of that group those permissions do not apply to you)
3. to all __other__ users (so if the first two do not apply - these are then the permissions you'll have on the respective file/directory)

This is also the order in which linux checks the permissions.

{{< highlight shell >}}
-rwxrw-r-- 1 user1 shared       0 Aug  6 07:16 user1File
 └┬┘└┬┘└┬┘
  │  │  │
  │  │  └─> others permissions: all other users can only Read 
  │  │                          this file
  │  │
  │  └────> group-permissions: members of the 'shared' group can 
  │                            Read and Write to this file
  │
  └───────> user-permissions: 'user1' (the owner) can Read + 
                              Write + eXecute this file
{{< /highlight >}}

Commands to manage permissions:

{{< highlight shell >}}
# setting permissions in absolute mode where each digit 
# represents an octal number (binary representation) so: 
# 4 == 100 == r--
# 2 == 010 == -w-
# 1 == 001 == --x
# 6 == 110 == rw-
chmod 774 /data/shared # will reslut in rwxrwxr--

# in symbolic mode you reference who and what
chmod u+x,g-x /data/shared 

# sets the group of /data/shared to group 'shared'
chgrp shared /data/shared

# sets both the user and the group of /data/shared 
chown root:shared /data/shared

# creates a new group 'shared' and subsequently 
# creates a new user which is added to 'shared'
groupadd shared && useradd -G shared user1

# adds the 'shared' group to the already existing 
# user 'user1' (notice the '-a' to append to its 
# group list)
usermod -aG shared user1
{{< /highlight >}}

### Advanced Permissions

Besides the standard permissions `rwx` there are a couple of special permissions. I'd like to pick just two of 'em since they are of special use.

#### Set group-id (_on directory_)

`sgid` will inherit the directory group owner for files created in that directoy (instead of using the creators primary group).

{{< highlight shell >}}
# create a shared directory where members of 'shared' should
# place their common files in 
root:~# mkdir -p /data/shared && \
> chmod 770 /data/shared && \
> chgrp shared /data/shared && \
> ls -ld /data/shared
drwxrwx--- 2 root shared 4096 Aug  6 07:10 /data/shared

# introduce a user which is a member of 'shared' group
root:~# useradd -G shared user1 && id user1
uid=1000(user1) gid=1001(user1) groups=1001(user1),1000(shared)

# right now, 'shared'-members can create files but those files
# belong to the primary group of the owner
root:~# su - user1
user1:~$ touch /data/shared/user1File && exit
root:~# ls -l /data/shared
total 0
-rw-rw-r-- 1 user1 user1 0 Aug  6 07:16 user1File

# let's set the group-id of the directory and notice the change
# in the groups execution-right
root:~# chmod g+s /data/shared
root:~# ls -ld /data/shared/
drwxrws--- 2 root shared 4096 Aug  6 07:16 /data/shared/

# now, files created will belong to the shared group => every
# group-member can interact with those files
root:~# su - user1
user1:~$ touch /data/shared/user2File && exit
root:~# ls -l /data/shared
total 0
-rw-rw-r-- 1 user1 user1  0 Aug  6 07:16 user1File
-rw-rw-r-- 1 user1 shared 0 Aug  6 07:22 user2File
{{< /highlight >}}

#### Set sticky-bit (_on directory_)

`sticky` will allow deletion of files only for the owner of the file

{{< highlight bash >}}
# let's assume we've extended the example from above to look
# like this - write-access to both files for group-members
# In that situation user2 can also _delete_ 'user1File' (not
# just write to it)
root:~# ls -l /data/shared
total 0
-rwxrwx--- 1 user2 shared 0 Aug  6 07:23 user2File
-rwxrwx--- 1 user1 shared 0 Aug  6 07:22 user1File

# to prevent that, set the sticky-bit on the directory
root:~# chmod +t /data/shared
root:~# ls -ld /data/shared
drwxrws--T 2 root shared 4096 Aug  6 07:22 /data/shared/

# now, user2 cannot delete files from user1 (only write to 'em)
root:~# su - user2
user2:~$ rm /data/shared/user1File
rm: cannot remove 'user1File': Operation not permitted
{{< /highlight >}}

#### Set user-id

You can also set group-id on files or even set user-id on files and directories. For now I found not much use in may daily business for those commands and they can be even dangerous (especially `suid`) - hence, I skip 'em here.
