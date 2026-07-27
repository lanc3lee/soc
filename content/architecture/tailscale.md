# CyberStorm ELK — Tailscale Access Setup

This gets you connected to the CyberStorm production ELK (Elasticsearch/Kibana) stack over Tailscale, and walks through confirming your access works.

## Prerequisites

- A device (laptop is easiest) you'll use to access Kibana
- An invite to the CyberStorm tailnet (ask Lance if you haven't received one)

## 1. Install Tailscale

Go to [tailscale.com/download](https://tailscale.com/download) and install the client for your OS (Mac, Windows, and Linux are all supported).

## 2. Accept the tailnet invite

You should have received an invite link or email from Lance to join the CyberStorm tailnet. Open it and follow the prompts to accept — this may ask you to sign in or create a free Tailscale account first.

## 3. Connect

Open the Tailscale app and make sure it shows **Connected**. On Mac/Windows this is a menu bar/system tray icon; on Linux it's usually `sudo tailscale up` from a terminal if you installed the CLI version.

## 4. Test your connection to the ELK box

Once connected, open a terminal and ping the ELK server's Tailscale address:

```bash
ping 100.107.55.21
```

You should see replies. If it times out, your device likely hasn't finished joining the tailnet yet — check the Tailscale app shows "Connected" and try again in a minute.

## 5. Access Kibana

Open this in your browser:

```
http://100.107.55.21:5601
```

You should land on the Kibana login screen.

## 6. Log in

Ask Lance for your individual Kibana username and password — everyone gets their own account rather than a shared login. If you don't have one yet, let him know and he'll set one up for you.

## Troubleshooting

| Problem | Likely cause |
|---|---|
| `ping` times out | Tailscale not connected yet, or you haven't been added to the tailnet — check the app status |
| Browser can't reach the page at all | Double-check you typed `http://` (not `https://`) and included `:5601` |
| Page loads but login fails | You may not have an account yet, or a typo in username/password — ping Lance |
| Everything was working, now it isn't | Tailscale may have disconnected — reopen the app and check status |

## Questions

Reach out to Lance if anything here doesn't work as expected — better to ask than to guess at a fix on production infrastructure.
