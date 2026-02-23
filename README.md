Windows Server Setup Guide
IIS + ARR + Reverse Proxy + MongoDB + RabbitMQ
OS: Windows Server 2019/2022
Purpose: Complete setup guide for fresh system

Table of Contents
IIS Installation
ARR + URL Rewrite Module
Enable Reverse Proxy in ARR
Configure Reverse Proxy Rule
MongoDB Installation
RabbitMQ Installation
Verify Everything
Quick Reference
1. IIS Installation
Open Server Manager → Click "Add roles and features"
Click Next → Select "Role-based or feature-based installation" → Next
Select Local Server → Next
Check ✅ Web Server (IIS) → Click "Add Features" → Next
Next → Next → Click Install
Wait for installation to complete
Verify: Open browser → http://localhost → IIS default page should appear

2. ARR + URL Rewrite Module
These must be downloaded and installed manually.

URL Rewrite Module
Download: https://www.iis.net/downloads/microsoft/url-rewrite
Run .msi installer → default settings → Install
ARR (Application Request Routing)
Download: https://www.iis.net/downloads/microsoft/application-request-routing
Run .msi installer → default settings → Install
Note: After ARR install, you will see "Server Farms" in IIS Manager left panel — this confirms ARR is installed ✅

3. Enable Reverse Proxy in ARR
Open IIS Manager → Win + R → type inetmgr → Enter
In left panel, click on Server name (root level) e.g. WIN-SERVERNAME
In middle panel, double-click "Application Request Routing Cache"
In right panel (Actions), click "Server Proxy Settings..."
Check ✅ "Enable proxy"
Click Apply
4. Configure Reverse Proxy Rule
In IIS Manager left panel → click "Default Web Site" (under Sites)
Double-click "URL Rewrite" icon in middle panel
In right panel click "Add Rule(s)..."
Select "Reverse Proxy" → Click OK
In "Inbound Rules" field, enter your backend server address:
Backend	Value to Enter
Node.js / Express	localhost:3000
.NET / ASP.NET	localhost:5000
Python Flask	localhost:5000
Python Django	localhost:8000
Java Spring Boot	localhost:8080
Click OK
This will auto-generate a rule in web.config:

<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="ReverseProxy" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://localhost:3000/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
5. MongoDB Installation
Step 1: Download
Go to: https://www.mongodb.com/try/download/community
Select: Windows → MSI → Download
Step 2: Install
Run the .msi file
Choose "Complete" setup type
✅ Check "Install MongoDB as a Service" (auto-starts on boot)
Optionally install MongoDB Compass (GUI tool) ✅ Recommended
Click Install
Step 3: Add to PATH (if not auto-added)
Win + R → sysdm.cpl → Advanced → Environment Variables
Under System Variables → select Path → Edit
Click New → add: C:\Program Files\MongoDB\Server\7.0\bin
OK → OK → OK
Step 4: Verify
Open PowerShell:

mongod --version
mongosh
MongoDB runs on: mongodb://localhost:27017

6. RabbitMQ Installation
RabbitMQ requires Erlang to be installed first.

Step 1: Install Erlang
Download: https://www.erlang.org/downloads
Run .exe installer → default settings → Install
Step 2: Install RabbitMQ
Download: https://www.rabbitmq.com/install-windows.html
Run .exe installer → default settings → Install
Step 3: Enable Management Plugin
Open PowerShell as Administrator and run:

& "C:\Program Files\RabbitMQ Server\rabbitmq_server-4.2.4\sbin\rabbitmq-plugins.bat" enable rabbitmq_management
Note: Replace rabbitmq_server-4.2.4 with your installed version.
To check version: ls "C:\Program Files\RabbitMQ Server\"

Output should show:

The following plugins have been enabled:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Step 4: Restart RabbitMQ Service
# Option 1: Via PowerShell
& "C:\Program Files\RabbitMQ Server\rabbitmq_server-4.2.4\sbin\rabbitmq-service.bat" stop
& "C:\Program Files\RabbitMQ Server\rabbitmq_server-4.2.4\sbin\rabbitmq-service.bat" start

# Option 2: Via Services
# Win + R → services.msc → Find RabbitMQ → Right click → Restart
Step 5: Add RabbitMQ to PATH (optional but recommended)
[System.Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\RabbitMQ Server\rabbitmq_server-4.2.4\sbin", [System.EnvironmentVariableTarget]::Machine)
Restart PowerShell after this.

Step 6: Access Dashboard
Open browser:

http://localhost:15672
Username: guest
Password: guest
7. Verify Everything
IIS
Browser → http://localhost
✅ IIS default page should load

MongoDB
Get-Service -Name MongoDB
# Status should be: Running
RabbitMQ
Browser → http://localhost:15672
# Login with guest/guest
Firewall Rules (if accessing from other machines)
netsh advfirewall firewall add rule name="MongoDB" dir=in action=allow protocol=TCP localport=27017
netsh advfirewall firewall add rule name="RabbitMQ-AMQP" dir=in action=allow protocol=TCP localport=5672
netsh advfirewall firewall add rule name="RabbitMQ-UI" dir=in action=allow protocol=TCP localport=15672
8. Quick Reference
Service	Port	URL / Command
IIS Web Server	80 / 443	http://localhost
IIS Manager	—	Win+R → inetmgr
MongoDB	27017	mongodb://localhost:27017
MongoDB Shell	—	mongosh
RabbitMQ AMQP	5672	amqp://localhost
RabbitMQ Dashboard	15672	http://localhost:15672
Default Credentials
Service	Username	Password
RabbitMQ	guest	guest
MongoDB	(no auth by default)	—
Useful Commands
# Check all services status
Get-Service -Name MongoDB
Get-Service -Name RabbitMQ

# Restart IIS
iisreset

# MongoDB shell
mongosh

# RabbitMQ plugins list
& "C:\Program Files\RabbitMQ Server\rabbitmq_server-4.2.4\sbin\rabbitmq-plugins.bat" list
📝 Note: Version numbers may differ. Always check installed version folder names before running commands.
RabbitMQ path example: C:\Program Files\RabbitMQ Server\rabbitmq_server-X.X.X\sbin\
