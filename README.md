# RHCSA-Practice-Configure-Privileged-Access-with-sudo
RHCSA Practice — Configure Privileged Access with sudo
Today’s objective: Safely grant a non-root user permission to run specific administrative commands through a validated /etc/sudoers.d/ rule.
Estimated time: 45 minutes
Lab system: RHEL 9 or RHEL 10
Privileges required: Root or an existing working sudo account
This builds directly on your local users and groups lesson. Configuring privileged access is an explicit RHCSA objective.
Critical authentication warning
This lab changes local authentication and sudo authorization. A malformed sudoers rule can prevent administrative access.
Before beginning:
- Use only a disposable RHEL lab VM.
- Take a VM snapshot if practical.
- Keep your current root shell open throughout the lab.
- Never edit /etc/sudoers with an ordinary text editor.
- Validate every rule with visudo before installing it.
- Do not modify your current administrator’s access.
1. The concept in plain language — 7 minutes
Linux administrators should not perform every task while logged in as root. The sudo command allows an authorized user to run a command with elevated privileges.
For example:
sudo systemctl status sshd

The authorization policy is stored in:
/etc/sudoers
/etc/sudoers.d/*

I prefer a separate file under /etc/sudoers.d/ because it is easier to validate, audit, and remove than changing the main file.
A sudoers rule follows this general structure:
WHO  WHERE=(RUN-AS)  COMMANDS

Example:
labsudo ALL=(root) /usr/bin/id

This means:
- labsudo — the authorized user
- ALL — valid on all hosts where this file is installed
- (root) — command may run as root
- /usr/bin/id — only this command is authorized
A group begins with %:
%labadmins ALL=(root) /usr/bin/id

About NOPASSWD
This lab uses NOPASSWD only for harmless, tightly restricted commands. That lets us test a temporary account without placing a training password in terminal history.
A broad rule like this is dangerous:
labsudo ALL=(ALL) NOPASSWD: ALL

Do not use that rule in this lab.
2. Preflight checks — 4 minutes
Enter a root shell:
sudo -i

Confirm:
whoami

Expected:
root

Confirm that the required commands exist:
command -v sudo
command -v visudo
command -v id
command -v whoami

Expected paths normally include:
/usr/bin/sudo
/usr/sbin/visudo
/usr/bin/id
/usr/bin/whoami

Check whether the temporary identity or rule already exists:
getent passwd labsudo
test -e /etc/sudoers.d/labsudo && echo "STOP: sudoers rule already exists"

Expected result:
No output

If either item exists, confirm that it came from an earlier run of this lab before removing or replacing it.
3. Guided practice: create the test user — 4 minutes
Authentication-change warning
The next command creates a temporary local account and home directory. It does not modify your normal administrator account.
useradd -m -c "RHCSA Sudo Lab User" labsudo

Verify:
getent passwd labsudo
id labsudo
ls -ld /home/labsudo

Expected account pattern:
labsudo:x:UID:GID:RHCSA Sudo Lab User:/home/labsudo:/bin/bash

No password is assigned. We will use runuser from the open root shell to test it.
4. Establish the starting state — 4 minutes
Display the user’s current sudo permissions:
sudo -l -U labsudo

The output should indicate that labsudo is not currently allowed to run administrative commands.
Test from the temporary account:
runuser -u labsudo -- sudo -n /usr/bin/id
echo "Exit status: $?"

The -n option tells sudo not to prompt for a password. The command should fail because no rule exists.
A typical result is:
labsudo is not in the sudoers file.
Exit status: 1

This establishes a clear before-and-after comparison.
5. Guided practice: build and validate a restricted rule — 8 minutes
Privileged-access warning
The following rule will permit labsudo to run only /usr/bin/id and /usr/bin/whoami as root without a password. Keep the current root shell open.
Create a draft outside /etc/sudoers.d:
printf '%s\n' \
'labsudo ALL=(root) NOPASSWD: /usr/bin/id, /usr/bin/whoami' \
> /tmp/labsudo.rules

Protect the draft:
chmod 440 /tmp/labsudo.rules
chown root:root /tmp/labsudo.rules

Inspect it:
ls -l /tmp/labsudo.rules
cat /tmp/labsudo.rules

Expected mode and ownership:
-r--r----- root root

Validate the syntax before installing it:
visudo -cf /tmp/labsudo.rules

Expected result:
/tmp/labsudo.rules: parsed OK

Do not proceed unless you receive parsed OK.
Why visudo matters
visudo understands sudoers grammar. A normal editor can save invalid syntax without warning, but visudo -c detects many errors before they damage administrative access.
6. Install and verify the rule — 7 minutes
Privileged-access warning
The next command activates the validated authorization rule.
Install it with the correct ownership and permissions:
install -o root -g root -m 0440 \
  /tmp/labsudo.rules /etc/sudoers.d/labsudo

install copies the file and applies its owner, group, and mode in one operation.
Validate the complete sudo configuration:
visudo -c

Expected output should end with sudoers files parsing successfully.
Confirm the installed file:
stat -c '%A %a %U %G %n' /etc/sudoers.d/labsudo

Expected:
-r--r----- 440 root root /etc/sudoers.d/labsudo

List the user’s authorized commands:
sudo -l -U labsudo

Expected permissions should include:
(root) NOPASSWD: /usr/bin/id, /usr/bin/whoami

7. Test allowed and denied commands — 6 minutes
Run an authorized command:
runuser -u labsudo -- sudo -n /usr/bin/id

Expected result begins with:
uid=0(root) gid=0(root)

Run the other authorized command:
runuser -u labsudo -- sudo -n /usr/bin/whoami

Expected:
root

Confirm the account itself is still unprivileged without sudo:
runuser -u labsudo -- /usr/bin/id
runuser -u labsudo -- /usr/bin/whoami

Expected identity:
uid=... (labsudo)
labsudo

Now test an unauthorized command.
Sensitive-file warning
The next test attempts to use sudo to read /etc/shadow, but the restricted rule should deny it. The command uses -n, so it cannot pause for a password.
runuser -u labsudo -- \
  sudo -n /usr/bin/cat /etc/shadow

echo "Exit status: $?"

Expected behavior:
- No shadow contents are displayed.
- sudo reports that authorization or a password is required.
- Exit status is nonzero.
This proves that sudo access is command-specific, not automatically unrestricted.
8. Diagnose a deliberately malformed draft — 4 minutes
Create an invalid draft:
printf '%s\n' \
'labsudo ALL=(root NOPASSWD /usr/bin/id' \
> /tmp/broken-sudo-rule

Validate it:
visudo -cf /tmp/broken-sudo-rule
echo "Exit status: $?"

Expected:
- A syntax-error message
- A nonzero exit status
Do not copy this broken file into /etc/sudoers.d.
Remove only the broken draft:
rm /tmp/broken-sudo-rule

This exercise demonstrates why validation must occur before installation.
9. Independent challenge — 7 minutes
Complete this section without copying the guided commands.
Assignment
Create:
User:  labaudit
Group: labauditors
Rule:  /etc/sudoers.d/labauditors

Requirements:
1. Create labauditors.
2. Create labaudit with a home directory and Bash shell.
3. Add labaudit to labauditors as a supplementary group.
4. Create a group-based sudoers rule using %labauditors.
5. Permit group members to run only:
/usr/bin/uptime
/usr/bin/journalctl

6. Allow those two commands without a password.
7. Create the rule as a draft under /tmp.
8. Set the draft to owner root, group root, and mode 0440.
9. Validate the draft with visudo.
10. Install it as /etc/sudoers.d/labauditors.
11. Validate the complete sudo configuration.
12. As labaudit, successfully run:
sudo -n /usr/bin/uptime
sudo -n /usr/bin/journalctl -n 5 --no-pager

13. Prove that this command is not authorized:
/usr/bin/id

Do not grant ALL command access.
10. Independent verification
Verify the identity:
id labaudit
getent group labauditors

Verify the rule file:
stat -c '%A %a %U %G %n' /etc/sudoers.d/labauditors
visudo -cf /etc/sudoers.d/labauditors

Expected:
440 root root
parsed OK

List effective sudo permissions:
sudo -l -U labaudit

Test the required commands:
runuser -u labaudit -- \
  sudo -n /usr/bin/uptime

runuser -u labaudit -- \
  sudo -n /usr/bin/journalctl -n 5 --no-pager

Test the unauthorized command:
runuser -u labaudit -- \
  sudo -n /usr/bin/id

test $? -ne 0 \
  && echo "PASS: unauthorized id command was denied" \
  || echo "FAIL: labaudit received excessive privilege"

Final configuration validation:
visudo -c

Do not consider the challenge complete unless the full configuration parses successfully.
11. Knowledge check
Answer these without reviewing the lesson:
1. What problem does sudo solve?
2. Why is a separate /etc/sudoers.d/ file useful?
3. What does %groupname mean in a sudoers rule?
4. What does (root) mean?
5. What does NOPASSWD change?
6. Why is NOPASSWD: ALL usually too broad?
7. Why should sudoers commands use absolute paths?
8. What does sudo -l display?
9. What does sudo -n do?
10. Why should a rule be validated before installation?
11. What permissions should a sudoers drop-in normally have?
12. Why should the current root session remain open?
13. How do you prove that an account has only the intended privileges?
14. Why should the denied-command test be part of verification?
I would consider this objective mastered when you can grant narrowly scoped sudo access, validate the configuration, test allowed commands, and prove that unrelated commands remain denied.
Safe rollback and cleanup
Privileged-access warning
Remove the sudoers rules before deleting their users and groups. Verify each filename carefully.
rm -f /etc/sudoers.d/labsudo
rm -f /etc/sudoers.d/labauditors

Immediately validate that the remaining configuration is healthy:
visudo -c

Remove the temporary users and their home directories:
userdel -r labsudo
userdel -r labaudit

Remove the temporary group:
groupdel labauditors

Remove the draft file:
rm -f /tmp/labsudo.rules

Verify cleanup:
getent passwd labsudo
getent passwd labaudit
getent group labauditors
test ! -e /etc/sudoers.d/labsudo \
  && echo "PASS: labsudo rule removed"
test ! -e /etc/sudoers.d/labauditors \
  && echo "PASS: labauditors rule removed"
visudo -c

Expected result:
- No temporary account or group records
- Both rule-removal checks pass
- The complete sudo configuration parses successfully
If you entered the root shell with sudo -i, leave it:
exit




Open chat


RHCSA Practice — Manage Local Users and Groups Today’s objective: Create and modify local users and groups, assign supplementary group membership, configure password aging, and verify access. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or sudo This builds on your work with permissions and shell commands. Local account and group administration is an explicit RHCSA objective. Important safety warning The commands in this lessonSep 24RHCSA Practice — Process Multiple Files with a Bash for Loop Today’s focused objective: Use a Bash for loop to process multiple command-line arguments and generate a consolidated log report. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: All work stays under /tmp/rhcsa-loop-lab. This practice does not modify networking, storage, boot configuration, authentication, SELinux, firewall rules, or system services. This buSep 22RHCSA Practice — Build a Log-Checking Bash Script Today’s focused objective: Create a simple Bash script that accepts a command-line argument, validates input with an if statement, processes command output, and returns a meaningful exit status. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: This lesson works only under /tmp/rhcsa-script-lab. It does not modify services, networking, storage, boot configuration, authentiSep 20RHCSA Practice — Search Logs with grep and Regular Expressions Today’s focused objective: Use grep, pipes, and regular expressions to locate useful information in configuration files and application logs. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: Everything stays under /tmp/rhcsa-grep-lab. This lesson does not change networking, storage, boot, authentication, SELinux, firewall rules, or system services. This isSep 17RHCSA Practice — Archive and Compress Files with tar Today’s focused objective: Create, inspect, extract, compress, and verify archives using tar, gzip, and bzip2. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneExam relevance: Red Hat’s current EX200 objectives require candidates to archive, compress, unpack, and uncompress files using tar, gzip, and bzip2. [Official RHCSA EX200 objectives](https://www.redhat.com/en/Sep 15RHCSA Practice — Hard and Symbolic Links Today’s focused objective: Create, inspect, troubleshoot, and remove hard links and symbolic links. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10 VMExam relevance: Creating hard and soft links is listed under the current RHCSA EX200 essential-tools objectives. Official Red Hat EX200 objectives This lesson builds on yoSep 13RHCSA Practice — Standard Linux Permissions Today’s objective: Read, set, and verify standard user/group/other permissions using symbolic and numeric notation. Estimated time: 45 minutesLab system: Your RHEL 9 VM is suitable for this practice.Safety: This lab works entirely inside /tmp/rhcsa-permissions-lab. It does not change networking, storage, boot settings, authentication, SELinux, the firewall, or system availability. Red Hat’s current EX200 objectives includSep 10












    To pick up a draggable item, press the space bar.
    While dragging, use the arrow keys to move the item.
    Press space again to drop the item in its new position, or press escape to cancel.
