# Griffin Web UI Access Guide

How to access the Griffin, YARN, and Elasticsearch interfaces without going over the public internet, using AWS SSM port forwarding.

---

## Option 1: SSM Port Forwarding (Recommended — Zero Config)

The instance already has `AmazonSSMManagedInstanceCore` attached, so this works right now with no security group changes.

Traffic path: **Your browser → AWS SSM (private) → EC2 instance.** Never touches the public internet.

### Setup

Install the SSM plugin if you haven't:

```bash
# On Linux
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
```

### Get the Instance ID

```bash
aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=griffin-batch-test" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text --region ap-southeast-2
```

### Forward All 3 Ports (run each in a separate terminal)

```bash
# Terminal 1 — Griffin UI
aws ssm start-session \
  --target <INSTANCE_ID> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["38080"],"localPortNumber":["38080"]}' \
  --region ap-southeast-2

# Terminal 2 — YARN UI
aws ssm start-session \
  --target <INSTANCE_ID> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["38088"],"localPortNumber":["38088"]}' \
  --region ap-southeast-2

# Terminal 3 — Elasticsearch
aws ssm start-session \
  --target <INSTANCE_ID> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["39200"],"localPortNumber":["39200"]}' \
  --region ap-southeast-2
```

Then open in your browser:

- Griffin UI → `http://localhost:38080`
- YARN UI → `http://localhost:38088`
- Elasticsearch → `http://localhost:39200`

> Each terminal must stay open to keep the tunnel alive. Close it to end the session.

---

## Option 2: SSH Tunnel (Requires Port 22 Open)

Port 22 needs to be open in the security group, but only to your IP — not the world.

```bash
ssh -i ~/.ssh/griffin-test-key.pem \
  -L 38080:localhost:38080 \
  -L 38088:localhost:38088 \
  -L 39200:localhost:39200 \
  -N ec2-user@3.25.244.218
```

Same result — access via `localhost` in the browser. Less preferred because it still uses the public IP for the SSH connection itself.

---

## Option 3: SSM SSH Proxy (Best of Both Worlds)

SSH tunneling **through** SSM — no port 22 needed in the security group, no public IP used.

Add this to your `~/.ssh/config`:

```
Host i-* mi-*
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --region ap-southeast-2"
  User ec2-user
  IdentityFile ~/.ssh/griffin-test-key.pem
```

Then SSH tunnel exactly as in Option 2, but using the instance ID instead of the public IP:

```bash
ssh -L 38080:localhost:38080 \
    -L 38088:localhost:38088 \
    -L 39200:localhost:39200 \
    -N i-xxxxxxxxxxxxxxxxx
```

---

## Comparison

| Option | Security Group Change | Uses Public IP | Complexity |
|---|---|---|---|
| SSM Port Forwarding | None | No | Low |
| SSH Tunnel | Add port 22 | Yes (for SSH) | Low |
| SSM SSH Proxy | None | No | Medium |

**SSM Port Forwarding (Option 1) is the cleanest** — no security group changes, no public IP, works immediately with your current setup.
