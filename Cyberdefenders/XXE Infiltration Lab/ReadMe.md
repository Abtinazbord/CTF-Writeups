# XXE Infiltration Lab (Blue Team CTF)

**Category:** Network Forensics
**Tools:** Wireshark
**Artifact:** PCAP file

> ⚠️ Spoiler warning: this writeup contains the answers.

## Scenario

An automated alert flagged unusual XML data being processed by the server, pointing to a possible XML External Entity (XXE) injection attack. The goal is to analyze the provided PCAP, figure out how the attacker got in, and trace what they did after that.

## Answers at a Glance

| # | Question | Answer |
|---|----------|--------|
| 1 | Highest open TCP port found by the scan | `3306` |
| 2 | URI of the vulnerable PHP script | `/review/upload.php` |
| 3 | First malicious XML file uploaded | `TheGreatGatsby.xml` |
| 4 | Web app config file the attacker read | `config.php` |
| 5 | Password of the compromised DB user | `Winter2024` |
| 6 | Timestamp (UTC) of the first MySQL login attempt | `2024-05-31 12:08` |
| 7 | Name of the uploaded web shell | `booking.php` |

---

## Q1: Highest open port from the port scan

> During the attacker's port scan, what is the highest-numbered TCP port that responded as open on the victim host?

In a SYN scan, an open port answers the attacker's SYN with a SYN-ACK. So the open ports are the **source ports** of SYN-ACK packets sent by the victim.

To show only SYN-ACK packets, apply this display filter:

```
tcp.flags == 0x012
```

Next, go to **Statistics > Endpoints > TCP** (or sort the filtered packets by source port) to see which ports the victim answered from. Two ports show up as open: `80` (the web server) and `3306` (MySQL). The highest of the two is the answer.

**Answer:** `3306`

---

## Q2: The vulnerable PHP script

> What's the complete URI of the PHP script vulnerable to XXE Injection?

Since this is an XXE attack, the attacker has to send XML to the server, which means looking for uploads rather than downloads. Most of the HTTP traffic is GET requests, so filtering for POST requests narrows things down fast:

```
http.request.method == "POST"
```

Frames **88306 to 88410** contain a small group of POST requests to the server at `50.238.151.185`, all hitting the same PHP upload script. That makes them the obvious suspects, and the request line gives the URI.

**Answer:** `/review/upload.php`

---

## Q3: First malicious XML file

> What's the name of the first malicious XML file uploaded by the attacker?

Right-click frame **88306** and choose **Follow > HTTP Stream**. In the multipart upload body, the `Content-Disposition` header shows the name of the uploaded file.

**Answer:** `TheGreatGatsby.xml`

---

## Q4: Web app configuration file

> What's the name of the web app configuration file the attacker read?

Each malicious XML file defines an external entity in its `DOCTYPE` that points at a local file on the server. Walking through the upload streams in order:

1. First stream: `file:///etc/passwd` (system user list, not a web app config)
2. Next stream: `file:///var/www/html/index.php` (the site's main page, not a config)
3. Next stream: `file:///var/www/html/config.php`

The third file sits in the web root and is named `config.php`, which makes it the web app's configuration file.

**Answer:** `config.php`

---

## Q5: Compromised database password

> What is the password for the compromised database user?

In the same HTTP stream as Q4, the server's response contains the contents of `config.php`, including the database credentials in plain text.

**Answer:** `Winter2024`

---

## Q6: First MySQL login attempt after the theft

> Using the Wireshark filter `mysql.login_request`, what is the timestamp (UTC) of the attacker's first MySQL login attempt?

Apply the filter:

```
mysql.login_request
```

This lists every MySQL login attempt. Some attempts happen before the credentials were stolen, so they don't count. The question is about the login after the attacker read `config.php`, which happened around frame 88000+. The first login attempt after that point is **frame 88348**.

Expanding the frame details and checking the **Arrival Time** field gives the timestamp. (Set **View > Time Display Format > UTC Date and Time of Day** to make sure it shows in UTC.)

**Answer:** `2024-05-31 12:08` (UTC)

---

## Q7: The web shell

> What is the name of the web shell that the attacker uploaded for remote code execution and persistence?

Going back to the POST requests to `/review/upload.php`, most of them are XML payloads that ask the server to read a file and send it back (`/etc/passwd`, `index.php`, `config.php`). Frame **88410** is different. It uploads a PHP file instead of an XML payload, and nothing is being read back. A PHP file dropped onto a web server like this is there to be executed later, which is what a web shell is for.

**Answer:** `booking.php`

---

## Attack Timeline Summary

1. **Recon:** SYN scan of the victim finds ports `80` and `3306` open.
2. **Initial access:** XXE payloads uploaded through `/review/upload.php`, starting with `TheGreatGatsby.xml`.
3. **File disclosure:** Attacker reads `/etc/passwd`, `index.php`, and then `config.php`.
4. **Credential theft:** Database password `Winter2024` pulled from `config.php`.
5. **Lateral access:** Attacker logs into MySQL on port `3306` with the stolen credentials.
6. **Persistence:** Web shell `booking.php` uploaded for remote code execution.

## Takeaways

- Disable external entity and DTD processing in the XML parser.
- Validate uploaded files by content, not just by name or extension, and never let uploads land somewhere they can be executed.
- Keep database credentials out of files in the web root, and don't expose MySQL to the internet.
