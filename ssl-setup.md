Yes. If you have a **BigRock Wildcard SSL** and want to expose an application running on a **Windows Server through IIS + ARR**, the recommended architecture is:

```text
                    Internet
                       |
                       | HTTPS :443
                       |
              yourdomain.com
                       |
                ┌──────▼──────┐
                │ Windows     │
                │ Server      │
                │ IIS + ARR   │
                │ Wildcard SSL│
                └──────┬──────┘
                       |
                       | HTTP
                       | or HTTPS
                       |
                ┌──────▼──────┐
                │ Application │
                │ localhost   │
                │ :3000       │
                └─────────────┘
```

For example:

```text
https://app.example.com
        |
        | IIS :443
        |
        | ARR Reverse Proxy
        ↓
http://127.0.0.1:3000
```

IIS ARR uses URL Rewrite to forward incoming requests to the backend application. Microsoft documents this exact reverse-proxy pattern. ([Microsoft Learn][1])

---

# 1. What you need before starting

You should have:

| Requirement         | Example                              |
| ------------------- | ------------------------------------ |
| Windows Server      | Windows Server 2019/2022/2025        |
| IIS                 | Installed                            |
| IIS URL Rewrite     | Installed                            |
| IIS ARR             | Installed                            |
| Domain              | `example.com`                        |
| Wildcard SSL        | `*.example.com`                      |
| Backend application | `localhost:3000`                     |
| Public DNS          | `app.example.com` → server public IP |
| Firewall            | TCP 80 and 443 allowed               |

For example, suppose your application is running on:

```text
http://127.0.0.1:3000
```

and you want users to access it through:

```text
https://app.example.com
```

---

# 2. Understand the BigRock wildcard certificate

A wildcard certificate such as:

```text
*.example.com
```

normally covers:

```text
app.example.com
api.example.com
admin.example.com
uat.example.com
```

It does **not** normally cover:

```text
example.com
```

unless the certificate also contains the root domain as a SAN.

So first check what BigRock actually issued.

You may have received files such as:

```text
yourdomain.crt
yourdomain.ca-bundle
private.key
```

or potentially a `.pfx`.

### Important

For IIS, a `.pfx` containing the:

* certificate
* private key
* certificate chain

is the easiest option.

If BigRock provided only `.crt` + private key, you may need to create a PFX before importing it into IIS.

---

# 3. Install IIS

On Windows Server:

### Server Manager

Go to:

```text
Server Manager
   ↓
Add Roles and Features
   ↓
Role-based or feature-based installation
   ↓
Select your server
   ↓
Web Server (IIS)
```

Install IIS.

You can also use PowerShell as Administrator:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

Then verify:

```powershell
Get-WindowsFeature Web-Server
```

You should see:

```text
[X] Web Server (IIS)
```

Open:

```text
http://localhost
```

You should get the IIS welcome page.

---

# 4. Install URL Rewrite

You need **IIS URL Rewrite** for ARR reverse proxy functionality. Microsoft explicitly notes that ARR relies on URL Rewrite. ([Microsoft Learn][2])

After installation, open:

```text
IIS Manager
```

Select the server.

You should see:

```text
URL Rewrite
```

If you don't see it, URL Rewrite isn't installed correctly.

---

# 5. Install Application Request Routing

Install **Microsoft Application Request Routing (ARR)**.

Microsoft's ARR installation documentation describes ARR as the IIS component used for proxy/routing functionality. ([Microsoft Learn][3])

After installation:

```text
IIS Manager
   ↓
Server
   ↓
Application Request Routing Cache
```

Open it.

Then on the right:

```text
Server Proxy Settings
```

Enable:

```text
[x] Enable proxy
```

Click:

```text
Apply
```

This is a very important step.

Without it, your URL Rewrite rule will not function as an ARR reverse proxy.

---

# 6. Check your backend application

Before configuring IIS, make sure the application itself works.

For example, if your application runs on port `3000`:

```powershell
netstat -ano | findstr :3000
```

You should see something like:

```text
TCP    127.0.0.1:3000    0.0.0.0:0    LISTENING    1234
```

Test:

```powershell
curl http://127.0.0.1:3000
```

or open:

```text
http://localhost:3000
```

in the browser.

**Do not continue until the backend works directly.**

---

# 7. Decide how your application will run

For example:

### Node.js

```text
localhost:3000
```

### React/Vite production

Usually:

```text
localhost:3000
```

or

```text
localhost:5173
```

during development.

### Python/FastAPI

For example:

```text
127.0.0.1:8000
```

### .NET

For example:

```text
localhost:5000
```

### Java

For example:

```text
localhost:8080
```

The IIS/ARR configuration is basically the same.

---

# 8. Import the BigRock wildcard SSL

This is one of the most important parts.

If BigRock provided a `.pfx`, use that directly.

Open:

```text
IIS Manager
```

Select:

```text
Server
```

Then:

```text
Server Certificates
```

On the right:

```text
Import...
```

Select your `.pfx`.

For example:

```text
bigrock-wildcard.pfx
```

Enter the PFX password.

For the certificate store, use:

```text
Personal
```

Then click:

```text
OK
```

You should now see your wildcard certificate.

For example:

```text
*.example.com
```

Check:

```text
Issued To: *.example.com
Issued By: BigRock/CA
Expiration: ...
```

---

# 9. If BigRock gave you CRT + private key instead

If you have:

```text
wildcard_example_com.crt
private.key
ca-bundle.crt
```

don't simply import the `.crt` into IIS expecting HTTPS to work.

The certificate needs to be associated with the **private key**.

The easiest approach is to create a `.pfx`.

If OpenSSL is available:

```powershell
openssl pkcs12 -export `
  -out wildcard-example-com.pfx `
  -inkey private.key `
  -in wildcard_example_com.crt `
  -certfile ca-bundle.crt
```

You'll be asked for a PFX password.

Then import:

```text
wildcard-example-com.pfx
```

into:

```text
IIS
→ Server Certificates
→ Import
```

### Important

Do not lose the PFX password.

Also protect the private key carefully.

---

# 10. Configure DNS in BigRock

Now you need the hostname to point to your Windows server.

Suppose your Windows server public IP is:

```text
203.0.113.50
```

and your application should be:

```text
app.example.com
```

Create an A record:

```text
Type: A
Host: app
Value: 203.0.113.50
TTL: 300
```

Result:

```text
app.example.com
        ↓
203.0.113.50
```

You can verify from Windows:

```powershell
nslookup app.example.com
```

Expected:

```text
Name:    app.example.com
Address: 203.0.113.50
```

---

# 11. Open Windows Firewall

You need TCP:

```text
80
443
```

Open PowerShell as Administrator:

```powershell
New-NetFirewallRule `
  -DisplayName "HTTP 80" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 80 `
  -Action Allow
```

And:

```powershell
New-NetFirewallRule `
  -DisplayName "HTTPS 443" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 443 `
  -Action Allow
```

If the server is in a cloud environment, you also need to allow 80/443 in the cloud security group/security list/firewall.

---

# 12. Create the IIS website

Open:

```text
IIS Manager
```

Right-click:

```text
Sites
```

Select:

```text
Add Website...
```

For example:

```text
Site name:
MyApplication

Physical path:
C:\inetpub\myapplication

Binding:
Type: http
IP address: All Unassigned
Port: 80
Host name: app.example.com
```

Click:

```text
OK
```

However, because we're using ARR, this website is primarily acting as the **reverse proxy**.

You don't need your actual application files here.

You can create:

```text
C:\inetpub\myapplication
```

and place a simple `web.config` there.

---

# 13. Add HTTPS binding

Select:

```text
MyApplication
```

Then:

```text
Bindings...
```

Click:

```text
Add
```

Configure:

```text
Type: https
IP address: All Unassigned
Port: 443
Host name: app.example.com
SSL certificate: *.example.com
```

If your server has multiple HTTPS sites, enable:

```text
Require Server Name Indication
```

So the configuration becomes:

```text
Type:       https
IP:         All Unassigned
Port:       443
Host name:  app.example.com
Certificate: *.example.com
SNI:        enabled
```

Click:

```text
OK
```

---

# 14. Keep HTTP binding as well

Keep:

```text
http :80
```

because we'll use it to redirect:

```text
http://app.example.com
```

to:

```text
https://app.example.com
```

So you should have:

```text
HTTP
*:80
app.example.com

HTTPS
*:443
app.example.com
*.example.com
```

---

# 15. Configure ARR Reverse Proxy

Now select:

```text
MyApplication
```

Then:

```text
URL Rewrite
```

Click:

```text
Add Rule(s)...
```

Select:

```text
Reverse Proxy
```

Microsoft provides a built-in reverse-proxy rule template for this scenario. ([Microsoft Learn][4])

Enter your backend:

```text
127.0.0.1:3000
```

For example:

```text
http://127.0.0.1:3000
```

Then create the rule.

---

# 16. Recommended web.config

For a simple application running on port `3000`, you can use:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<configuration>

    <system.webServer>

        <rewrite>

            <rules>

                <!-- HTTP to HTTPS -->
                <rule name="HTTP to HTTPS" stopProcessing="true">

                    <match url="(.*)" />

                    <conditions>
                        <add input="{HTTPS}" pattern="^OFF$" />
                    </conditions>

                    <action
                        type="Redirect"
                        url="https://app.example.com/{R:1}"
                        redirectType="Permanent" />

                </rule>


                <!-- Reverse Proxy -->
                <rule name="Reverse Proxy to Application"
                      stopProcessing="true">

                    <match url="(.*)" />

                    <action
                        type="Rewrite"
                        url="http://127.0.0.1:3000/{R:1}" />

                </rule>

            </rules>

        </rewrite>

    </system.webServer>

</configuration>
```

Put this in:

```text
C:\inetpub\myapplication\web.config
```

Microsoft's documented ARR pattern similarly uses URL Rewrite with an `http://backend/{R:1}` destination, which ARR then proxies. ([Microsoft Learn][1])

---

# 17. Important: HTTP → HTTPS rule ordering

The rules should be ordered:

```text
1. HTTP → HTTPS
2. Reverse Proxy
```

Like:

```text
Incoming request
       |
       ↓
Is HTTPS OFF?
       |
      YES
       |
       ↓
Redirect to HTTPS
       |
       ↓
HTTPS request
       |
       ↓
ARR
       |
       ↓
Backend application
```

This prevents the backend from being exposed through plain HTTP.

---

# 18. Test the backend first

On Windows Server:

```powershell
curl http://127.0.0.1:3000
```

Then test IIS:

```powershell
curl http://app.example.com
```

It should redirect to:

```text
https://app.example.com
```

Then:

```powershell
curl https://app.example.com
```

You should receive the application response.

---

# 19. Browser test

Open:

```text
https://app.example.com
```

You should see:

```text
🔒 Secure
```

Click the certificate information.

It should show something similar to:

```text
Subject:
*.example.com

Issued by:
BigRock / Certificate Authority

Valid:
2026-... → 2027-...
```

---

# 20. Check the SSL certificate from PowerShell

You can test:

```powershell
curl.exe -v https://app.example.com
```

Look for:

```text
SSL connection
```

and certificate information.

You can also use:

```powershell
openssl s_client -connect app.example.com:443 -servername app.example.com
```

if OpenSSL is installed.

---

# 21. If your application is Node.js + PM2

Since you are using Windows + PM2 in some of your environments, a common architecture would be:

```text
Internet
   |
   | HTTPS 443
   ↓
IIS
   |
   | ARR
   ↓
127.0.0.1:3000
   |
   ↓
Node.js / PM2
```

PM2 keeps the application alive:

```powershell
pm2 list
```

Example:

```text
App name       status
-----------------------
frontend       online
backend        online
```

IIS should **not** directly expose port `3000` to the Internet.

Ideally:

```text
Internet → 443 → IIS → localhost:3000
```

rather than:

```text
Internet → 3000
```

So you don't need to open port 3000 in the firewall.

---

# 22. If your backend is FastAPI

Suppose your FastAPI application is:

```text
127.0.0.1:8000
```

Your ARR rule becomes:

```xml
<rule name="Reverse Proxy to FastAPI"
      stopProcessing="true">

    <match url="(.*)" />

    <action
        type="Rewrite"
        url="http://127.0.0.1:8000/{R:1}" />

</rule>
```

Architecture:

```text
https://api.example.com
          |
          ↓
        IIS
       :443
          |
          ↓
        ARR
          |
          ↓
127.0.0.1:8000
          |
          ↓
       FastAPI
```

---

# 23. If your backend is HTTPS

Suppose your backend itself runs:

```text
https://127.0.0.1:8000
```

Then use:

```xml
<action
    type="Rewrite"
    url="https://127.0.0.1:8000/{R:1}" />
```

But if the backend doesn't need HTTPS because it is on the same server, I would normally use:

```text
IIS HTTPS
      ↓
ARR
      ↓
HTTP localhost
```

This is commonly called **SSL termination/offloading**.

```text
Internet
   |
 HTTPS
   |
   ↓
 IIS
 SSL certificate
   |
 HTTP
   |
   ↓
 localhost:3000
```

The public connection remains encrypted.

---

# 24. Handling WebSockets

If your application uses:

* Socket.IO
* WebSocket
* SignalR
* live notifications
* real-time communication

you need to make sure IIS/ARR supports WebSockets.

Install the Windows feature:

```powershell
Install-WindowsFeature Web-WebSockets
```

Then restart IIS:

```powershell
iisreset
```

**Be careful with `iisreset` on a production server** because it restarts IIS applications. If this is your production server, schedule it appropriately.

---

# 25. Large file uploads

If your application uploads large files, IIS may reject them before they reach your backend.

For example, if you want 100 MB:

```xml
<system.webServer>

    <security>
        <requestFiltering>
            <requestLimits maxAllowedContentLength="104857600" />
        </requestFiltering>
    </security>

</system.webServer>
```

`104857600` bytes = 100 MB.

This is separate from your application's own upload limit.

---

# 26. Timeout configuration

If your application takes a long time, for example:

```text
5 minutes
10 minutes
30 minutes
```

you may need to adjust ARR proxy timeout settings as well as the backend application's timeout.

This is especially relevant for:

```text
AI requests
large uploads
long API calls
report generation
file processing
```

Don't blindly increase every timeout to extremely high values. First identify whether the timeout is coming from:

```text
Client
 ↓
IIS
 ↓
ARR
 ↓
Backend
 ↓
Database/API
```

---

# 27. Multiple applications with the same wildcard certificate

This is where a wildcard certificate becomes very useful.

You can have:

```text
app.example.com
api.example.com
admin.example.com
uat.example.com
```

all using:

```text
*.example.com
```

Example:

```text
                    IIS
                     |
        ┌────────────┼────────────┐
        ↓            ↓            ↓
 app.example.com api.example.com admin.example.com
        |            |            |
        ↓            ↓            ↓
 localhost:3000 localhost:8000 localhost:5000
```

Each hostname can have its own IIS site.

---

# 28. Example IIS configuration

### Application 1

```text
app.example.com
        ↓
127.0.0.1:3000
```

### API

```text
api.example.com
        ↓
127.0.0.1:8000
```

### Admin

```text
admin.example.com
        ↓
127.0.0.1:5000
```

All three can use:

```text
*.example.com
```

certificate.

---

# 29. DNS configuration

You could configure:

```text
app.example.com       A       SERVER_PUBLIC_IP
api.example.com       A       SERVER_PUBLIC_IP
admin.example.com     A       SERVER_PUBLIC_IP
```

Or, if appropriate:

```text
*.example.com         A       SERVER_PUBLIC_IP
```

Then:

```text
app.example.com
api.example.com
admin.example.com
```

all resolve to the IIS server.

---

# 30. IIS bindings

For `app.example.com`:

```text
HTTP
*:80
app.example.com

HTTPS
*:443
app.example.com
*.example.com
```

For `api.example.com`:

```text
HTTP
*:80
api.example.com

HTTPS
*:443
api.example.com
*.example.com
```

For `admin.example.com`:

```text
HTTP
*:80
admin.example.com

HTTPS
*:443
admin.example.com
*.example.com
```

If these are on the same IIS server, use **SNI** for the HTTPS bindings.

---

# 31. Recommended final architecture

For your use case, I recommend:

```text
                         INTERNET
                            |
                            |
                      HTTPS :443
                            |
                            ↓
                  ┌─────────────────┐
                  │   Windows       │
                  │   Server        │
                  │                 │
                  │      IIS        │
                  │       +         │
                  │      ARR        │
                  │       +         │
                  │ URL Rewrite     │
                  │       +         │
                  │ Wildcard SSL    │
                  └────────┬────────┘
                           |
            ┌──────────────┼──────────────┐
            |              |              |
            ↓              ↓              ↓
       :3000          :8000          :5000
            |              |              |
            ↓              ↓              ↓
       Frontend        FastAPI        Backend
```

Public users only need access to:

```text
80
443
```

The application ports:

```text
3000
8000
5000
```

should ideally remain accessible only locally or through the internal network.

---

# 32. Troubleshooting checklist

If you get **502 Bad Gateway**:

Check:

```powershell
curl http://127.0.0.1:3000
```

If this fails, the application is the problem.

If it works, check:

```text
IIS
→ URL Rewrite
→ ARR
→ Enable Proxy
```

---

### If you get 404

Check the URL Rewrite rule:

```xml
<match url="(.*)" />
```

and:

```xml
<action type="Rewrite"
        url="http://127.0.0.1:3000/{R:1}" />
```

---

### If you get 500

Check:

```text
Event Viewer
→ Windows Logs
→ Application
```

and:

```text
IIS logs
```

Usually:

```text
C:\inetpub\logs\LogFiles
```

---

### If you get 503

Check whether your backend is running:

```powershell
netstat -ano | findstr :3000
```

and:

```powershell
pm2 list
```

if you're using PM2.

---

### If HTTPS doesn't work

Check:

```text
IIS
→ Site
→ Bindings
→ HTTPS :443
```

Verify:

```text
Host name = app.example.com
Certificate = *.example.com
SNI = enabled
```

Also check:

```powershell
nslookup app.example.com
```

The result must point to your Windows server's public IP.

---

# 33. The complete flow

In practical terms, your deployment sequence should be:

```text
1. Prepare Windows Server
        ↓
2. Install IIS
        ↓
3. Install URL Rewrite
        ↓
4. Install ARR
        ↓
5. Enable ARR Proxy
        ↓
6. Deploy application
        ↓
7. Verify localhost:PORT
        ↓
8. Get BigRock Wildcard SSL
        ↓
9. Import PFX into IIS
        ↓
10. Configure DNS
        ↓
11. Open TCP 80/443
        ↓
12. Create IIS website
        ↓
13. Add HTTP binding
        ↓
14. Add HTTPS :443 binding
        ↓
15. Select *.example.com certificate
        ↓
16. Configure URL Rewrite
        ↓
17. Configure ARR → localhost:PORT
        ↓
18. Redirect HTTP → HTTPS
        ↓
19. Test DNS
        ↓
20. Test HTTPS
        ↓
21. Test application
        ↓
22. Test WebSocket/uploads if applicable
```

Microsoft's current documentation confirms the key ARR setup: install ARR/URL Rewrite, enable the proxy, and use URL Rewrite rules to forward requests to the backend. ([Microsoft Learn][1])

### One important point about your BigRock SSL

If you tell me **exactly what files BigRock gave you** (for example `*.crt`, `*.key`, `ca-bundle`, `.pfx`) and your application's **port** (e.g. `3000`, `8000`, `5000`), I can give you the **exact IIS + ARR + BigRock configuration**, including the final `web.config`, DNS records, SSL import steps, and commands for your server.

[1]: https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing?utm_source=chatgpt.com "Reverse Proxy with URL Rewrite v2 and Application Request Routing | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/iis/extensions/planning-for-arr/using-the-application-request-routing-module?utm_source=chatgpt.com "Using the Application Request Routing Module | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/iis/extensions/installing-application-request-routing-arr/install-application-request-routing?utm_source=chatgpt.com "Install Application Request Routing | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/reverse-proxy-rule-template?utm_source=chatgpt.com "Reverse Proxy - Rule Template | Microsoft Learn"
