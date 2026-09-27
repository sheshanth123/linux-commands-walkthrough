# Linux CLI + AI Automation

Notes for the 120-min walkthrough. Everything below is meant to be typed straight into a terminal — I've built a small set of fake logs/CSVs (see the bottom of this doc) so none of this needs a real AWS box or real customer data to try out.

---

## 1. Before touching anything

First thing on any new terminal, just glance at the prompt:

```bash
whoami
hostname
pwd
```

Sounds obvious but it's saved me more than once — running something destructive on the wrong box because I forgot which SSH session I was in. `user@host:directory$` is the pattern to read.

Also worth a quick look before pasting in any script, especially one an AI wrote for you:

```bash
cat /etc/os-release
bash --version
uname -a
```

Ubuntu's bash is usually pretty current (5.x), but scripts written on a Mac assume an ancient bash 3.2 and things quietly break. Good to know before you're debugging the wrong problem.

---

## 2. Moving around and managing files

Basics:

```bash
pwd
ls -la
cd /home/ubuntu/data_pipeline     # full path
cd ../scripts                       # relative
cd -                                 # back to wherever you just were
```

`ls -la` over plain `ls` — the `-a` catches hidden stuff like `.env` files, which you'll want to know are sitting there.

Setting up a working folder:

```bash
mkdir -p data_pipeline/{raw,staged,scripts,logs}
cd data_pipeline
touch config.yaml
```

`-p` just means "make the parents too, and don't complain if it's already there" — always safe to add.

Moving/copying/deleting — the main thing is knowing `mv` deletes the original and `cp` doesn't, which matters a lot once you're several commands deep and not thinking about it anymore:

```bash
cp scripts/extract.py scripts/extract.py.bak
mv extract.py ./scripts/
rm -f old_test_file.txt
rm -rf ./staging/obsolete_test/       # no undo on this, be sure first
rmdir ./empty_folder/                  # only works if it's actually empty, which is a nice safety net
```

Honestly before running `rm -rf` on anything, just `ls` the path first. Takes two seconds, saves a bad afternoon.

Batch moves with wildcards — handy when you've got a folder full of the day's data drops:

```bash
mv sample-data/data/incoming/*.csv ./staged/
cp sample-data/logs/*.log ./logs_backup/
ls sample-data/data/incoming/*.csv | wc -l   # sanity-check the count before you move them
```

And `find` for anything time- or pattern-based, since `ls` can't really do this on its own:

```bash
find ./logs -name "*.json" -mtime -1     # anything touched in the last day
find sample-data/data/incoming -name "*.csv" -mtime -1
find /var/log -type f -size +10M
find ./logs -name "*.tmp" -mtime +7 -exec rm {} \;
```

That last one — always run it without `-exec rm` first, just let it print the matches, and eyeball the list before you let it start deleting.

---

## 3. Reading, searching, and stitching commands together

`cat` for short stuff, `less` for anything long enough that you want to scroll or search (`/` to search, `q` to quit), `head`/`tail` for a quick peek:

```bash
head -5 sample-data/data/incoming/orders_part1.csv
tail -20 sample-data/logs/syslog
less sample-data/logs/access.log
tail -f sample-data/logs/agent_memory.log     # live-follows the file, ctrl+c to stop
```

`head` on a CSV before loading it anywhere is a habit worth having — takes one second to check the delimiter's actually a comma, there's a header row, and nothing looks garbled.

**grep** — pulling specific lines out of a big log:

```bash
grep "Out of memory" sample-data/logs/syslog
grep -i "error" sample-data/logs/agent_memory.log
grep -c "Action:" sample-data/logs/agent_memory.log      # just the count
grep -A 5 "Traceback" sample-data/logs/agent_memory.log    # 5 lines after the match — gets you the whole stack
grep -B 2 -A 10 "ERROR" sample-data/logs/agent_memory.log
grep -E "50[0-9]" sample-data/logs/access.log                # any 5xx
```

**awk** — once grep found the lines, awk pulls specific columns out of them:

```bash
awk '{print $9, $10}' sample-data/logs/access.log
awk '{print $9}' sample-data/logs/access.log | sort | uniq -c | sort -nr
awk -F',' '{print $1, $5}' sample-data/data/incoming/orders_part1.csv
awk '$9 ~ /^5/ {print $7}' sample-data/logs/access.log
```

**sed** — find/replace across a file, useful for swapping config values or stripping placeholder junk:

```bash
sed 's/dev_db/prod_db/g' sample-data/config.yaml        # prints the result, doesn't touch the file
sed -i.bak 's/dev_db/prod_db/g' sample-data/config.yaml   # actually edits it, but keeps a .bak copy
sed -i 's/NULL//g' sample-data/data/incoming/orders_part1.csv
```

I'd get in the habit of running the non `-i` version first to see what it'll do, then adding `-i.bak` rather than a bare `-i` — gives you an instant way back if the regex was wrong.

Chaining it all together is really the point of the shell — each tool does one small thing, pipes connect them:

```bash
cat sample-data/logs/syslog | grep "Out of memory" > oom_summary.txt
grep "python3" oom_summary.txt | wc -l
cat sample-data/logs/access.log | awk '{print $7}' | sort | uniq -c | sort -nr | head -n 5
```

Redirection quick reference:
- `>` overwrite
- `>>` append
- `<` feed a file in
- `2>` stderr only
- `&>` both stdout and stderr

`wc -l` before and after a filtering step is a cheap gut-check that you didn't just filter everything out by accident.

Archiving:

```bash
tar -czvf artifact.tar.gz ./dist/
tar -tzvf artifact.tar.gz              # list contents without extracting — always do this first
tar -xzvf artifact.tar.gz -C ./restored/
```

---

## 4. Permissions, processes, and checking what's actually running

Permissions come up constantly with SSH keys and `.env` files:

```bash
ls -l sample-data/dummy-key.pem
chmod 400 sample-data/dummy-key.pem     # owner read-only, nobody else — ssh will actually refuse the key otherwise
chmod +x scripts/stage_data.sh
chmod 750 scripts/stage_data.sh
chown ubuntu:ubuntu config.yaml
```

Reading `-rwxr-xr--`: first character is file type, then three groups of three — owner, group, everyone else — each rwx.

Environment variables instead of hardcoding secrets:

```bash
export OPENAI_API_KEY="sk-..."
env | grep API_KEY
python3 my_langchain_script.py
unset OPENAI_API_KEY
```

Process stuff — this is what you reach for when something's hung or eating all the RAM:

```bash
ps aux | grep python
top          # q to quit
htop          # nicer version, sudo apt install htop if it's missing
kill <PID>      # ask nicely first
kill -9 <PID>    # only if it's ignoring you
```

Disk and memory, worth checking before pulling down anything large:

```bash
df -h
du -sh ./staging/
du -sh ./* | sort -rh | head
free -h
```

Networking/services:

```bash
ssh -i sample-data/dummy-key.pem ubuntu@<ec2-public-ip>
curl -I https://api.example.com/health
curl -s https://api.example.com/v1/ping | jq .
systemctl status docker
sudo systemctl restart nginx
journalctl -u docker.service --since "1 hour ago"
journalctl -u docker.service -f
```

If something can't connect, check `systemctl status` before assuming it's a network problem — half the time the service itself just isn't up.

---

## 5. Getting good scripts out of an LLM

The pattern that actually works: tell it who it is, tell it exactly what environment it's writing for, and tell it exactly what you want back. Skip any of the three and you get generic, half-portable output.

**Debugging a broken Jenkins build**

Input file: `sample-data/logs/jenkins_console_snippet.log`

```bash
cat sample-data/logs/jenkins_console_snippet.log
```

Prompt:

```
Act as a Senior DevOps Engineer. I am encountering a failure in a Jenkins
pipeline executing a Bash script on an Ubuntu 24.04 runner. Below is the
extracted block from the Jenkins console output. The script attempts to
compress a directory and push it to AWS S3, but it fails with "tar: exit
delayed from previous errors".

1. Identify the root cause from the logs.
2. Provide the corrected CLI command.
3. Explain how to use grep and awk to proactively catch this specific error
   string in future builds.

[paste the log here]
```

The thing to check for in the answer: does it actually catch that `tar` is failing because a file (`model_weights.bin`) is being written to while it's being archived — a race condition, not an S3 permissions problem. If the answer just says "retry the upload," it missed it.

**Turning a manual CSV cleanup into a script**

Input files: `sample-data/data/incoming/*.csv` — these have `NULL` strings scattered through them, same as the sed example above.

Prompt:

```
Write a Bash script for a data engineering pipeline with the following
strict requirements:
- It must search the directory /data/incoming/ for all .csv files modified
  in the last 24 hours.
- It must use sed to replace all occurrences of the string 'NULL' with an
  empty string "" in these files.
- It must move the processed files to /data/staged_for_snowflake/ and
  append a timestamp to each filename.
- It must log all actions (successes and failures) to
  /var/log/etl_staging.log with timestamps.
- Include error handling if the source directory is empty.

Output only the raw bash script, no markdown code fences, no commentary.
```

Worth checking: does it use `find ... -mtime -1` rather than `ls` (which can't reliably filter by time), does it quote its variables so filenames with spaces don't break it, and does the empty-directory check actually happen before the loop starts, not as an afterthought.

Try whatever it gives you against the sample data:

```bash
mkdir -p /tmp/staged_for_snowflake
bash your_generated_script.sh sample-data/data/incoming /tmp/staged_for_snowflake
```

**Making sense of a messy agent log**

Input file: `sample-data/logs/agent_memory.log` — a few thousand lines mixing LangChain reasoning steps with stack traces.

Prompt:

```
I have a text file named agent_memory.log containing thousands of lines of
mixed LangChain reasoning logs, standard outputs, and Python tracebacks.

Provide a single pipeline of Linux commands (using cat, grep, awk, etc.)
that extracts only the lines containing the word "Action:", counts how
many times each unique tool was called, and sorts them from most used to
least used. Do not write a Python script; provide only the native Linux
CLI commands.
```

Something close to this is what you're hoping to get back — worth running it yourself to see the actual breakdown:

```bash
grep "Action:" sample-data/logs/agent_memory.log | awk -F"Action: " '{print $2}' | sort | uniq -c | sort -nr
```

---

## 6. Wrapping up

When you half-remember a flag, `man` and `--help` beat guessing or hunting for a five-year-old blog post, since they reflect whatever's actually installed on the machine in front of you:

```bash
man tar
man grep | less -p "context"
tar --help
```

Quick reference, the stuff that'll come up again and again:

| Need | Command |
|---|---|
| Where/who am I | `pwd`, `whoami` |
| Files touched recently | `find <dir> -name "*.ext" -mtime -1` |
| Search inside a file | `grep -i "pattern" file` |
| Pull a column | `awk '{print $N}' file` |
| Replace text in place | `sed -i.bak 's/old/new/g' file` |
| Rank by frequency | `sort \| uniq -c \| sort -nr` |
| Lock down a key file | `chmod 400 key.pem` |
| Find and kill a process | `ps aux \| grep name` then `kill -9 <PID>` |
| Check disk space | `df -h` |
| Check a service | `systemctl status <svc>` |
| Tail a service's logs | `journalctl -u <svc> --since "1 hour ago"` |

---

## Sample data

Everything above runs against a small set of made-up files (zipped alongside this doc) — no real hosts, keys, or customer data involved:

- `logs/syslog` — ~400 lines with some fake "Out of memory" kill events sprinkled in
- `logs/access.log` — 1,000 lines styled like an Nginx access log, mixed status codes
- `logs/agent_memory.log` — ~1,400 lines mixing fake LangChain `Action:`/`Observation:` steps with a repeating stack trace
- `logs/jenkins_console_snippet.log` — a short fake Jenkins failure showing the `tar: exit delayed` error
- `data/incoming/orders_part1.csv`, `part2.csv`, `part3.csv` — sample order data with `NULL` placeholders to strip out
- `config.yaml` — a small config with a `dev_db` URI to swap for `prod_db`
- `dummy-key.pem` — just a placeholder text file shaped like a key, safe to practice `chmod` on, not a real credential
