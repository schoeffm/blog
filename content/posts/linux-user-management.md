+++
title = "Linux User Management"
date = 2019-08-10T10:40:33+02:00
publishDate = 2019-08-15
tags = ["cheatsheet", "users", "linux", "devops" ]
+++

Cheatsheet for _Linux User Management_

### Handling users

{{< highlight shell >}}
# adds a new user called 'schoeffm'
useradd -m -s /bin/zsh -G docker schoeffm
#       └┬┘└────┬─────┘└─┬─────┘
#        │      │        │
#        │      │        └─> List of groups the user 
#        │      │            should be a member of 
#        │      └─> login shell of the new account
#        └────────> also create a home directory for this 
#                   account
{{< /highlight >}}

{{< highlight shell >}}
#           Assigns a new UID to <─────┐
#           this account               │
#                                   ┌──┴──┐ 
usermod -aG docker -s /bin/zsh -L|U -u 4711 schoeffm
#       └───┬────┘ └───┬─────┘ └─┬─┘
#           │          │         └─> Lock or Unlock user
#           │          └─> set new login shell for account
#           └────────────> append to the list of groups 
#                          without 'a' this will replace
#                          the group list
{{< /highlight >}}

`userdel` not explicitly shown here since it's comprehensible.

### Dealing with groups

{{< highlight shell >}}
# Introduces a new user-group called 'shared'
groupadd -f -g 4711 shared
#         │ └──┬──┘
#         │    └─> assigns the given GID to the new group 
#         └─> force: will exit with rc zero even if the 
#             group already exists (won't change the 
#             existing group - so GID of existing group
#             isn't updated)
{{< /highlight >}}

`groupmod` and `groupdel` not explicitly shown here since they're comprehensible.

### Information gathering

{{< highlight shell >}}
# list all available groups and their GID
awk -F':' '{print $1 " " $3}' /etc/group
#   └─┬─┘         └───┬────┘  └───┬────┘
#     │               │           └─> input file
#     │               └─> print field 1 and 3 and
#     │                   separate 'em with a single
#     │                   space
#     └─> change default field-separator to use :
{{< /highlight >}}

{{< highlight shell >}}
# prints one entry of /etc/passwd (whole line)
awk -F':' '/proxy/ {print}' /etc/passwd
#   └─┬─┘ └──┬──┘
#     │      └─> print only the line which matches this
#     │           regex pattern (here username 'proxy')
#     └─> actually not needed here since we print the
#         whole line anyways
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
#     │ └─┬─┘ └─┬─┘ └─┬┘└──────┬─────────┘
#     │   │     │     │        └─> login-shell
#     │   │     │     └─> user home directory
#     │   │     └─> primary group of this account
#     │   └─> UID:GID of this user and his primary 
#     │       group
#     └─> stands for the password which isn't shown
{{< /highlight >}}

{{< highlight shell >}}
# lists all users currently logged in and what they're executing 
# right now
w
{{< /highlight >}}

{{< highlight shell >}}
# although offers a bunch of options the simplest form of the
# command already prints all necessary information
id [username]

uid=0(root) gid=0(root) groups=0(root)
#   └──┬──┘ └────┬────┘ └─┬──────────┘
#      │         │        └─> all groups the user is a member of
#      │         └─> gid and name of the primary group of this 
#      │             account
#      └───────────> uid and name of this account
{{< /highlight >}}
