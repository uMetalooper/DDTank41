## Center Service (Center.Server host)

This app hosts the DDTank center server. The center server coordinates game shards, maintains the server list and player session state, and exposes a WCF service for admin/game web components.

### What it does
- Hosts a WCF service `Center.Server.CenterService` (net.tcp + basicHttp + mex) to:
  - Query server list, push notices, kick users, reload configs, update feature flags (AAS/DailyAward).
- Listens for TCP connections from game servers on `IP:Port` (default 127.0.0.1:9202) and:
  - Tracks online users (`LoginMgr`) and forwards cross‑server packets (`ServerClient`).
  - Periodically saves server state to DB and scans auctions/mail/consortia (`CenterServer` timers).
  - Maintains global/world events (`WorldMgr`).

### Requirements
- Windows 10/11.
- .NET Framework 4.8 Developer Pack (Visual Studio installs this).
- Visual Studio 2019/2022 with .NET desktop workload.
- SQL Server (Express is fine).

macOS/Linux: This project targets .NET Framework and uses WCF server features (net.tcp) that aren’t supported by .NET Core on non‑Windows. Use a Windows VM/host to run.

### Database
1. Restore databases from `Database/`:
   - `Player34.bak` to a database named `Project_Player34` (or update config accordingly)
   - `Game34.bak` to a database named `Project_Game34`
2. Ensure the login you use has rights to both DBs.

### Configure
Edit `Center.Service/App.config` appSettings and service endpoints:

```xml
<appSettings>
  <!-- Update to your SQL Server instance and DB names -->
  <add key="conString" value="Data Source=YOURMACHINE\SQLEXPRESS;Initial Catalog=Project_Player34;User ID=sa;Password=your_password" />
  <add key="crosszoneString" value="Data Source=YOURMACHINE\SQLEXPRESS;Initial Catalog=Project_Game34;User ID=sa;Password=your_password" />

  <!-- Center socket for game servers to connect -->
  <add key="IP" value="127.0.0.1" />
  <add key="Port" value="9202" />

  <!-- Timer intervals (minutes) -->
  <add key="LoginLapseInterval" value="1" />
  <add key="SaveInterval" value="1" />
  <add key="SaveRecordInterval" value="1" />
  <add key="ScanAuctionInterval" value="60" />
  <add key="ScanMailInterval" value="120" />
  <add key="ScanConsortiaInterval" value="1" />

  <!-- Feature flags -->
  <add key="AAS" value="false" />
  <add key="DailyAwardState" value="true" />

  <!-- Text resources -->
  <add key="LanguagePath" value="Languages\Language-vn.txt" />
  <add key="SystemNoticePath" value="Languages\SystemNotice.xml" />
</appSettings>

<system.serviceModel>
  <services>
    <service name="Center.Server.CenterService" behaviorConfiguration="Center.Server.CenterServiceBehavior">
      <host>
        <baseAddresses>
          <!-- HTTP metadata address; adjust host/IP if remote access needed -->
          <add baseAddress="http://127.0.0.1:2008/CenterService/" />
        </baseAddresses>
      </host>
      <!-- net.tcp endpoint for service calls -->
      <endpoint address="net.tcp://127.0.0.1:2009/" binding="netTcpBinding" bindingConfiguration="CenterService" contract="Center.Server.ICenterService" />
      <endpoint address="" contract="Center.Server.ICenterService" binding="basicHttpBinding" />
      <endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />
    </service>
  </services>
</system.serviceModel>
```

Notes
- Open firewall for TCP 9202 (center socket), 2008 (HTTP metadata), 2009 (net.tcp) if accessed remotely.
- Ensure `Languages/Language-vn.txt` and `Languages/SystemNotice.xml` exist under the working directory (copy from other project bins if needed).

### Build
1. Open `DDTank 3.0.sln` in Visual Studio.
2. Set startup project to `Center.Service`.
3. Build the solution (Debug or Release).

### Run
You can run from Visual Studio (F5) or from a terminal in the build output directory:

```bat
Center.Service.exe --start
```

When running, a console will accept simple admin commands:
- `exit` — stop and exit.
- `notice&Your message here` — broadcast a system notice.
- `reload&server` — hot‑reload with type `server` (reloads config/timers/server list and notifies shards).
- `shutdown` — broadcast shutdown to shards/clients.
- `help` — print help text.

The service also exposes programmatic operations via WCF (`ICenterService`). Clients (e.g., web admin, game services) should call the net.tcp endpoint at `net.tcp://<host>:2009/`.

### Troubleshooting
- Database connection errors: verify `conString` and `crosszoneString`, SQL Server is reachable, and DB names match your restores.
- Ports in use: change `IP`/`Port` in `App.config` or free the port.
- No `Languages` files: copy the language pack files into the working dir or adjust the paths in `App.config`.


