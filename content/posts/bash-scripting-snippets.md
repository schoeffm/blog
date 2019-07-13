+++
title = "Bash scripting snippets"
date =  2019-07-12T14:58:39+02:00
draft = true
tags = ["bash", "scripting", "cli"]
+++

<!--more-->
### if statements
- there have to be spaces after/before the square brakets
- `[[` is bash sytax - `[` is actually a program in `/usr/bin` (will also work, but with some caveats) → always use `[[`

{{< highlight bash >}}
# one-line example
if [[ "bla" == "foo" ]];then echo "true"; else echo "false";fi

# use the RC of another program instead of [[
if grep foo text.txt;then echo "true"; else echo "false";fi
{{< /highlight >}}



### scripting flags
- `set -e` will stop the script after the first non-zero RC (and hopefully prevent any damage).
- `set -u`: In most programming languages, you get an error if you try to use an unset variable. Not in bash! By default, unset variables are just evaluated as if they were the empty strings. `set -u` fixes this. 
- `set -x`: will print out every command before running it (useful when debugging).

### return codes, `&&` and `||`
Every Unix program has a "return code" which is an integer from `0` to `127` (`0` means success - everthing else is an failure).

- `&&`: Combine several commands with `&&` => only if the first command returns `0` the second one will be executed. The cmd-chain will stop as soon as the first command returns `!= 0`
- `||`: When combined with `||` the second command will be executed only when the first fails (like a default to be used if the first one isn't possible).

{{< highlight bash >}}
create_user; make_home_dir      # will execute both, no matter 
                                # what the rc is
create_user && make_home_dir    # will execute the 2nd cmd only
                                # if the first one was 
                                # successful
create_user || make_home_dir    # will execute the 2nd cmd only
                                # if the first one fails


{{< /highlight >}}

### Looping
{{< highlight bash >}}
for i in {1..10};do                     # will print 1 till
    echo -n "$i";                       # 10 in one line
done

for i in {1..10};do echo -n "$i"; done  # or as one-liner

for i in 1 2 3;do echo -n "$i"; done    # with explicit values 
{{< /highlight >}}
