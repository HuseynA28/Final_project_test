# Connect Power BI and DBeaver to the MySQL database

Every project runs MySQL 8 in Docker with three schemas: **bronze**, **silver** and **gold**.

| Tool | What we use it for | Which schema |
|---|---|---|
| **DBeaver Community** (free) | write and test SQL, look at the tables, draw ER diagrams | bronze, silver, gold |
| **Power BI Desktop** (free) | build the dashboard / report | **gold only** |

Connection details (the same for all 23 projects):

| Setting | Value |
|---|---|
| Host | `localhost` |
| Port | `3306` |
| User | `de_user` |
| Password | `de_pass` |
| Database | `gold` (Power BI) — leave empty in DBeaver to see all three |

---

## Downloads (install these on your own Windows PC)

| What | Link | Note |
|---|---|---|
| **Power BI Desktop** | https://www.microsoft.com/en-us/download/details.aspx?id=58494 | file `PBIDesktopSetup_x64.exe` (or install it from the Microsoft Store) |
| **MySQL Connector/NET 8.0.28** ⭐ | https://downloads.mysql.com/archives/get/p/6/file/mysql-connector-net-8.0.28.msi | direct download; the version most people report works with Power BI |
| MySQL Connector/NET 8.0.33 (alternative) | https://downloads.mysql.com/archives/get/p/6/file/mysql-connector-net-8.0.33.msi | last 8.0 version; use it if 8.0.28 does not work |
| All Connector/NET versions | https://downloads.mysql.com/archives/c-net/ | choose **Product Version 8.0.x**, Operating System **Microsoft Windows** |
| **DBeaver Community** | https://dbeaver.io/files/dbeaver-ce-latest-windows-x86_64.exe | direct download (all versions: https://dbeaver.io/download/) |
| Docker Desktop (only if Docker runs on your PC) | https://www.docker.com/products/docker-desktop/ | not needed for GitHub Codespaces |
| GitHub CLI (only for Codespaces) | https://cli.github.com/ | or in PowerShell: `winget install --id GitHub.cli` |
| VS Code (alternative for Codespaces) | https://code.visualstudio.com/download | + extension **GitHub Codespaces**: https://marketplace.visualstudio.com/items?itemName=GitHub.codespaces |

> ⚠️ **Connector/NET: use any 8.0.x version, NOT 9.x** (and not 8.1–8.4). Power BI does not find the 9.x versions and says
> *"This connector requires one or more additional components"*.
> The MySQL page calls the 8.0 file *"Windows (x86, 32-bit), MSI Installer"*. That is normal: it is the only Windows
> installer for Connector/NET 8.0, and it also works with the 64-bit Power BI Desktop.

**Install order:** Power BI Desktop → Connector/NET 8.0.28 → **restart the PC** (or at least Power BI) → DBeaver.

---

## Step 0 — Make sure the database is running and reachable

```bash
docker compose up -d      # in the project folder
docker ps                 # you should see <project>_mysql with 0.0.0.0:3306->3306/tcp
```

**A) Docker runs on your own PC** → nothing else to do, use `localhost:3306`.

**B) The project runs in GitHub Codespaces** → MySQL is in the cloud, not on your PC.
Power BI and DBeaver run on your PC, so you must open a "tunnel" first. Choose one:

- **GitHub CLI** (on your own PC, in PowerShell or a terminal):
  ```bash
  gh auth login                              # once
  gh codespace ports forward 3306:3306       # pick your codespace, keep this window open
  ```
- **VS Code desktop app:** open the codespace in VS Code on your PC (not in the browser) →
  **Ports** tab → **Forward a Port** → `3306`.

Now `localhost:3306` on your PC goes to the MySQL in the codespace. Keep the tunnel open while you work.

> **Port 3306 already in use?** (for example, MySQL is also installed on your PC)
> Change `"3306:3306"` to `"3307:3306"` in `docker-compose.yml`, run `docker compose up -d` again, and use port `3307` everywhere.

---

## Part 1 — DBeaver Community: write the SQL

1. Download and install **DBeaver Community**: https://dbeaver.io/files/dbeaver-ce-latest-windows-x86_64.exe
2. **Database → New Database Connection → MySQL → Next**
3. Fill in:
   - Server Host: `localhost` · Port: `3306`
   - Database: *(leave empty → you see bronze, silver and gold)*
   - Username: `de_user` · Password: `de_pass` · tick **Save password**
4. Open the **Driver properties** tab and set:
   - `allowPublicKeyRetrieval` = `true`
   - `useSSL` = `false`

   (Without this, MySQL 8 in Docker gives the error *"Public Key Retrieval is not allowed"*.)
5. Click **Test Connection**. The first time DBeaver asks to **download the MySQL driver** → click **Download**.
6. Click **Finish**.

**Working in DBeaver**

| What | How |
|---|---|
| New SQL editor | select the connection → **SQL Editor → New SQL script** (or `Ctrl + ]`) |
| Choose the schema | the dropdown at the top of the editor (e.g. `gold`), or write `silver.orders` |
| Run one statement | put the cursor in it → `Ctrl + Enter` |
| Run the whole script | `Alt + X` |
| See a table's data | double-click the table → **Data** tab |
| ER diagram of the star schema | right-click `gold` → **View Diagram** |
| Export a result | right-click the result → **Export data** (CSV, Excel …) |

**Good routine:** test a new query in DBeaver first. When it works, copy it into `sql/gold_tables.sql`
(as a `CREATE OR REPLACE VIEW vw_...`), run notebook `03_gold_sql` again, and refresh Power BI.

> Data in DBeaver does not change by itself: after you rerun a notebook, press **F5** on the schema to refresh.

---

## Part 2 — Power BI Desktop: build the dashboard

### 2.1 Install (Windows only)

1. **Power BI Desktop:** https://www.microsoft.com/en-us/download/details.aspx?id=58494
2. **MySQL Connector/NET 8.0.28:** https://downloads.mysql.com/archives/get/p/6/file/mysql-connector-net-8.0.28.msi
   → double-click the `.msi` → **Typical** install.

   ⚠️ **Not version 9.x.** If you already installed 9.x: Windows **Settings → Apps** → uninstall every
   *MySQL Connector/NET*, then install 8.0.28.
3. Restart Power BI Desktop (or the PC).
4. Check: **Get data → MySQL database** must open the server box **without** the message
   *"requires one or more additional components"*.

### 2.2 Connect

1. **Home → Get data → More… → Database → MySQL database → Connect**
2. Server: `localhost:3306` · Database: `gold` → **OK**
3. Choose the **Database** tab on the left (not Windows) → User `de_user` · Password `de_pass` → **Connect**
4. If Power BI asks about an encrypted connection, click **OK** (no SSL is fine on your own PC).
5. In the **Navigator** tick the `dim_…`, `fact_…`, `vw_…`, `analysis_…` and `ml_…` tables you need
   (each project README lists them) → **Load**. Use **Import** mode.

### 2.3 Build the model

1. **Model view:** drag the keys to create the relationships (dimension `1` → fact `*`, single direction).
   The exact relationships for every project are in its README, e.g. `dim_customer[customer_id] → fact_orders[customer_id]`.
2. Select the date table (`dim_date`) → **Mark as date table**.
3. **Modeling → New measure** for the DAX measures in the README, for example:
   ```
   Total Revenue = SUM(fact_orders[order_revenue])
   Revenue LY    = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(dim_date[date]))
   ```
4. Build the report pages (also listed in each README): KPI cards, line chart over time, map, tables with slicers.

### 2.4 After new data

Rerun the notebooks → in Power BI click **Home → Refresh**. The connection stays the same.

---

## Problems and fixes

| Problem | Fix |
|---|---|
| *Can't connect* / *connection refused* | Is the container running (`docker ps`)? In Codespaces: is the tunnel open? |
| *Access denied for user* | Use `de_user` / `de_pass`. In Power BI choose the **Database** tab, not Windows. |
| *Public Key Retrieval is not allowed* (DBeaver) | Driver properties: `allowPublicKeyRetrieval=true`, `useSSL=false` |
| Power BI: *requires additional components* | Uninstall all Connector/NET versions → install **8.0.28** (link above) → restart. Still not found? Try 8.0.33. |
| Port 3306 is used by another MySQL | Map `"3307:3306"` in `docker-compose.yml` and use port 3307 |
| Tables are empty | Run the notebooks 01 → 05 first; in DBeaver press F5 |
| Numbers are text in Power BI | Power Query → change the column type to Decimal / Whole number |
| Wrong project's data | All projects use port 3306. Run `docker compose down` in one project before `up -d` in the next. |