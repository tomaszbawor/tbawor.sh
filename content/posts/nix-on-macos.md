+++
title = "Nix-Darwin on MacOs with zscaler"
date = "2025-03-25"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Tomasz Bawor"
authorTwitter = "" #do not include @
tags = ["nix", "macbook", "zscaler", "nix-darwin", "ssl", "certificates"]
description = "How to install nix package manager on macbook with zscaler"
showFullContent = false
+++
Once upon a time I have tried to install nix package manager into my work macbook. I have been looking for the same terminal setup for all of my machines that can be easly stored in one place. 

This however was not so easy since my work macbook has preinstaled zscaler wchich rips open the TLS connections and replaces the certificates with company accepted one. 

I have spend to many hours that I would like to admit on searching why my nix commands are not working and I have errors with SSL certificates. Since I have set up my computer many years ago the zscaler did not gave issues and I forgot it exists. 

Here is the instruction for people struggling with the same problem, or my colleagues if I ever convince them to use Nix. 

## Step 1: Export Trusted Certificates from macOS Keychain

Use the `security` tool to export certificates from the system keychains:

```bash
security export -t certs -f pemseq -k /Library/Keychains/System.keychain -o /tmp/certs-system.pem
security export -t certs -f pemseq -k /System/Library/Keychains/SystemRootCertificates.keychain -o /tmp/certs-root.pem
cat /tmp/certs-root.pem /tmp/certs-system.pem > /tmp/ca_cert.pem
```

This creates a single `ca_cert.pem` bundle from both system and root certificates.

## Step 2: Move Certificate Bundle to Nix Directory

Copy the combined certificate bundle to a location where the Nix daemon can access it:

```bash
sudo mv /tmp/ca_cert.pem /etc/nix/
```

## Step 3: Configure Nix Daemon to Use the Certificate

Edit the `nix-daemon` launch daemon configuration:

```bash
sudo vim /Library/LaunchDaemons/org.nixos.nix-daemon.plist
```

Add or ensure the following entries exist under `EnvironmentVariables`, it is just an example, there may be other entries that I would not change. This is just an example. 

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>NIX_SSL_CERT_FILE</key>
  <string>/etc/nix/ca_cert.pem</string>
  <key>SSL_CERT_FILE</key>
  <string>/etc/nix/ca_cert.pem</string>
  <key>REQUEST_CA_BUNDLE</key>
  <string>/etc/nix/ca_cert.pem</string>
</dict>
```

## Step 4: Restart the Nix Daemon

After saving the file, you may restart the Nix daemon. This is reported to work for some people, not for me though... I needed to reboot my computer. 

```bash
sudo launchctl unload /Library/LaunchDaemons/org.nixos.nix-daemon.plist
sudo launchctl load /Library/LaunchDaemons/org.nixos.nix-daemon.plist
```

## Step 5: Export Environment Variables (Optional)

If the above steps don't fully solve the issue, you may also want to export the certificate path manually in your shell:

```bash
export NIX_SSL_CERT_FILE=/etc/nix/ca_cert.pem
export SSL_CERT_FILE=/etc/nix/ca_cert.pem
```

Just to be sure I have included setting up this environment variables in my nix config by adding following lines to my darwin config. 

```nix
environment.variables = {
    NIX_SSL_CERT_FILE = "/etc/nix/ca_cert.pem";
    SSL_CERT_FILE = "/etc/nix/ca_cert.pem";
    REQUEST_CA_BUNDLE = "/etc/nix/ca_cert.pem";
};
```

## References

- [Discourse Post on SSL Errors with Nix on macOS](https://discourse.nixos.org/t/ssl-ca-cert-error-on-macos/31171/6)
- [My Nix Setup](https://github.com/tomaszbawor/nix-configurations)

---

This should be it. If you encountered other issues leave a comment.
