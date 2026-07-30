# Casting Jellyfin from Android to TV from any network

## Introduction

Let's say you have a Jellyfin server running on your home network, and you want to cast it to your TV from any network.

You have a tailscale network setup, your android and your jellyfin server are on the same network.

The issue is the following:

> When you cast from your phone to your TV, the TV tries to connect directly to the jellyfin server. But it is actually not reachable from this remote wifi network.
>
> This leads to the streamcast not working. If you are connected via android to "my-computer", your phone has the DNS record because it is part of the same tailscale network. But when you are connected via a remote wifi network, the TV itself can't resolve the DNS record, and are simply not in the same network, because not connected itself to the tailscale network.

The solution is to:

* Use a port forwarding on your Android
* Connect to your Android local IP address instead of the jellyfin server address
* Cast

This setup will work, because the TV will connect to your Android local IP address, and your Android will forward the request to the jellyfin server.

Downside: You have to check and connect to the local IP address of your Android, which will change if you change your network.

## Setup

One time setup:

* Install the jellyfin app from the play store (using a browser won't lead to discover the cast devices)
* Install "Fwd: port forwarder" from the play store
* Copy the IP address of your Jellyfin server in the tailscale network from the tailscale app
* Inside of the port forwarder app, create a new rule to forward the port 8096 TCP from WLAN to your the IP adress of the jellyfin server on the tailscale subnet.

Every time you want to cast:

* In the port forwarder app, start the forwarding service
* Check your local IP address from the wifi settings of your Android (long press the wifi widget, then tap on the information icon next to the name of the actual wifi)
* On the jellyfin app, connect to your Android local IP address on port 8096
* Browse and cast your media

## Additional notes

This setup allows also to watch media from an other person computer for example.

This can be very useful to watch TV from an other country if Jellyfin has access to TV streams from your local network. For this, you will have to setup a tuner device and a xmltv provider (<https://xmltvfr.fr/xmltv/xmltv_fr.xml> works well for french TV channels)

### Casting from google chrome

In chrome, you can't cast if the jellyfin is on http and not https.

#### 1. Using a reverse proxy

A small trick is to use a reverse proxy to encapsulate into https. See `caddy` section in the linux tools section in this knowledge base.

```bash
caddy reverse-proxy --to <ip of your jellyfin server>:8096 --from <your local computer ip address>:8096
```

Then go on jellyfin from your localhost. The HTTPS certificate will not be trusted, but you will be able to cast.

#### 2. By setting a flag in the chrome browser

In chrome, you can set a flag to allow casting from http.

```bash
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

Set the flag to `Enabled`. Then you will be able to cast from the http version of jellyfin.
