================================================================
  n8n_CSV_Sheets_PDF_Gmail_Report
  Automated Sales Reporting Pipeline
================================================================

WHAT THIS WORKFLOW DOES
-----------------------
This n8n workflow reads a CSV file with sales and product data,
processes and sorts it, sends the data to Google Sheets, generates
a fully styled PDF report using Gotenberg, and emails the report
(with the PDF attached) via Gmail — all automatically.

The email body is a professional HTML report with KPI cards and
data tables. The attached PDF is a full-page A3 report with 8 KPI
cards, 7 analysis sections, and the top 10 orders by profit.


WORKFLOW FLOW (24 nodes)
------------------------

  [Manual Trigger]
        |
  [Read CSV File]          -- Reads the CSV from disk
        |
  [Parse CSV]              -- Parses raw bytes into structured rows
        |
  [Validate & Clean Data]  -- Drops rows with missing/invalid fields
        |
  [Sort by Gross Profit]   -- Sorts all rows highest profit first
        |
  [Calculate KPIs]         -- Computes: Revenue, COGS, Profit,
        |                     Margin %, Orders, Units, AOV, Best Order
        |
  ------+------+------+------
  |           |      |      |
[Agg.      [Agg.  [Agg.  [Top Products
 Category]  Region] Monthly] & Leaderboard]
  |           |      |      |
  ------+------+------+------
        |
  [Merge Aggregations]     -- Combines all 4 branch outputs
        |
  ------+------+------
  |           |      |
[Flatten   [Format  [Build HTML Report]
 Rows]     KPI Rows]      |
  |           |      [HTML → Binary Buffer]
[Sheets→  [Sheets→        |
 Sales     KPI        [Gotenberg → PDF]
 Data]     Summary]        |
                      [Rename PDF File]
                           |
                      [Set Email Subject & Meta]
                           |
                      [Build Email HTML Body]
                           |
                      [Gmail → Send Report]
                           |
                      [Success Log]


NODE-BY-NODE REFERENCE
----------------------

1. Manual Trigger
   - Starts the workflow on demand.
   - Can be replaced with a Schedule Trigger for daily/weekly runs.
   - Example: run every Monday at 8:00 AM.

2. Read CSV File
   - Reads the CSV file from disk.
   - CONFIGURE: Set "File(s) Selector" to your CSV path.
   - n8n only allows files from: C:\Users\[you]\.n8n-files\
   - Copy your CSV there and point this node to it.

3. Parse CSV
   - Converts the raw binary file into JSON rows.
   - Expects a header row. Delimiter: comma (,).
   - Works with any CSV that has the columns defined below.

4. Validate & Clean Data
   - Drops any row that is missing: order_id, date, revenue,
     gross_profit, quantity, unit_price, unit_cost.
   - Also drops rows where numeric fields contain non-numbers.
   - Logs skipped rows to the n8n execution log.
   - Throws an error if zero valid rows are found.

5. Sort by Gross Profit
   - Sorts all rows descending by gross_profit.
   - This ensures the top 10 orders table in the report
     always shows the most profitable orders first.

6. Calculate KPIs
   - Computes global metrics from all sorted rows:
     * Total Revenue
     * Total COGS (Cost of Goods Sold)
     * Gross Profit
     * Average Gross Margin %
     * Total Orders
     * Total Units Sold
     * Average Order Value
     * Best Single Order (highest profit)
   - Passes the full row array downstream for further use.

7. Split into Branches
   - Sends the same data to 4 parallel aggregation nodes.
   - All 4 run simultaneously to save time.

8. Aggregate by Category
   - Groups rows by product category.
   - Computes: Revenue, COGS, Gross Profit, Margin %, Orders, Units.
   - Sorted by Revenue descending.
   - Example output categories: Electronics, Furniture, Accessories.

9. Aggregate by Region
   - Groups rows by sales region (North, South, East, West).
   - Computes: Revenue, Gross Profit, Margin %, Orders, Units.
   - Sorted by Revenue descending.

10. Aggregate Monthly Trend
    - Groups rows by YYYY-MM (calendar month).
    - Computes: Revenue, Gross Profit, Margin %, Orders.
    - Sorted chronologically (oldest to newest).
    - Used to show the monthly trend table in the report.

11. Top Products & Leaderboard
    - Top 5 Products: by Revenue, with units and margin.
    - Salesperson Leaderboard: all reps ranked by Revenue.
    - Sales by Channel: Online, In-Store, Reseller breakdown.

12. Merge Aggregations
    - Combines the 4 parallel branch outputs into one item.
    - Mode: Append (no field matching needed).

13. Flatten Rows for Sheets
    - Takes the full sorted row list from Calculate KPIs.
    - Outputs one item per row so Google Sheets can receive them.

14. Sheets → Sales Data
    - Appends all 100 order rows to the "Sales Data" tab.
    - CONFIGURE: Replace YOUR_GOOGLE_SHEET_ID_HERE with your
      actual Google Sheet ID (from the URL).
    - Credential: Google Sheets OAuth2.
    - Operation: Append Row (no match column needed).

15. Format KPI Sheet Rows
    - Builds a structured list of metric/value pairs from all
      aggregations: KPIs, categories, regions, months, products,
      salespersons, channels.
    - Each row has: section, metric, value.

16. Sheets → KPI Summary
    - Appends all KPI rows to the "KPI Summary" tab.
    - Same Google Sheet ID as above.
    - CONFIGURE: Same as node 14.

17. Build HTML Report
    - Generates a full HTML page with:
      * Dark gradient header
      * 8 color-coded KPI cards
      * Monthly trend table
      * Top 5 products table
      * Category performance table
      * Region performance table
      * Salesperson leaderboard
      * Channel breakdown table
      * Top 10 orders table
      * Branded footer
    - This HTML is converted to PDF by Gotenberg.

18. HTML → Binary Buffer
    - Converts the HTML string to a base64 binary buffer.
    - Required because Gotenberg accepts multipart file uploads,
      not raw JSON strings.

19. Gotenberg → Generate PDF
    - Sends the HTML file to Gotenberg via HTTP POST.
    - Gotenberg uses headless Chromium to render it as PDF.
    - Paper: 11x17 inches (tabloid). Scale: 0.85. Margins: 0.5in.
    - CONFIGURE: URL must be http://localhost:3000/... if running
      Gotenberg locally, or http://gotenberg:3000/... in Docker.
    - Start Gotenberg with:
        docker run --rm -p 3000:3000 gotenberg/gotenberg:8

20. Rename PDF File
    - Renames the PDF binary to: Sales_Report_YYYY-MM-DD.pdf
    - Uses today's date so each report has a unique filename.

21. Set Email Subject & Meta
    - Builds the email subject line with today's date.
    - Builds the preview text shown in email clients.
    - Passes KPI data forward for use in the email body.

22. Build Email HTML Body
    - Generates a professional email with:
      * Gradient header
      * 4 KPI cards (Revenue, Profit, Margin, Avg Order)
      * Category performance table
      * Top 5 products table
      * Top 3 salesperson table
      * PDF attachment notice
      * Branded footer
    - Uses fully inline CSS for email client compatibility.

23. Gmail → Send Report
    - Sends the email to the configured recipient.
    - PDF is attached automatically from the binary field.
    - CONFIGURE: Set "Send To" to the recipient email address.
    - Credential: Gmail OAuth2.

24. Success Log
    - Logs a JSON summary of the completed execution:
      status, PDF filename, revenue, profit, margin, orders, units.
    - Visible in the n8n execution logs.


REQUIRED CSV COLUMNS
--------------------
Your CSV must have these exact column headers (case-sensitive):

  order_id, date, product_id, product_name, category, sku,
  quantity, unit_price, unit_cost, discount_pct, revenue, cogs,
  gross_profit, gross_margin_pct, region, salesperson, channel,
  customer_id, customer_name, payment_method, status

  date format: YYYY-MM-DD  (example: 2024-03-15)
  numeric fields: plain numbers, no $ or % symbols


EXAMPLE CSV ROW
---------------
order_id  : ORD-0001
date      : 2024-01-03
product_id: PRD-101
product_name: Wireless Headphones Pro
category  : Electronics
sku       : WHP-101-BLK
quantity  : 2
unit_price: 129.99
unit_cost : 62.00
discount_pct: 0
revenue   : 259.98
cogs      : 124.00
gross_profit: 135.98
gross_margin_pct: 52.30
region    : North
salesperson: Alice Johnson
channel   : Online
customer_id: CUST-441
customer_name: Acme Corp
payment_method: Credit Card
status    : Completed


GOOGLE SHEETS SETUP
-------------------
1. Create a new Google Sheet.
2. Add two tabs named exactly:
     "Sales Data"    (receives all order rows)
     "KPI Summary"   (receives aggregated metrics)
3. Copy the Sheet ID from the spreadsheet URL (the long string between /d/ and /edit).
4. In n8n go to Credentials → Add → Google Sheets OAuth2.
5. Follow the OAuth flow to authorize with your Google account.
6. In nodes 14 and 16: replace YOUR_GOOGLE_SHEET_ID_HERE
   with your actual Sheet ID.


GOTENBERG SETUP (PDF generation — free, self-hosted)
----------------------------------------------------
Gotenberg is a free open-source Docker service that converts
HTML to PDF using headless Chromium. No account needed.

Start it with one command:
  docker run --rm -p 3000:3000 gotenberg/gotenberg:8

Keep this terminal open while running the workflow.
The workflow calls: http://localhost:3000/forms/chromium/convert/html

If you run n8n inside Docker Compose, add Gotenberg to your
docker-compose.yml and use http://gotenberg:3000/... as the URL.


GMAIL SETUP
-----------
1. In n8n go to Credentials → Add → Gmail OAuth2.
2. You will need a Google Cloud project with Gmail API enabled.
3. Create OAuth 2.0 credentials (Desktop or Web app type).
4. Add this redirect URI in Google Cloud Console:
     http://localhost:5678/rest/oauth2-credential/callback
5. Copy Client ID and Client Secret into n8n.
6. Click "Sign in with Google" and authorize.
7. In node 23 (Gmail → Send Report):
   Set "Send To" to the recipient's email address.


SETUP CHECKLIST
---------------
[ ] CSV file copied to C:\Users\[you]\.n8n-files\
[ ] Node 2  (Read CSV File): file path updated
[ ] Node 14 (Sheets → Sales Data): Sheet ID updated
[ ] Node 16 (Sheets → KPI Summary): Sheet ID updated
[ ] Node 23 (Gmail → Send Report): recipient email set
[ ] Google Sheets credential connected (OAuth2)
[ ] Gmail credential connected (OAuth2)
[ ] Gotenberg running on localhost:3000 (Docker)


HOW TO IMPORT INTO n8n
-----------------------
1. Open n8n in your browser (default: http://localhost:5678)
2. Go to Workflows → New Workflow
3. Click the three-dot menu (top right) → Import from File
4. Select: n8n_CSV_Sheets_PDF_Gmail_Report.json
5. Complete the Setup Checklist above
6. Click "Execute Workflow" to run


CUSTOMIZATION TIPS
------------------
- Change trigger: swap Manual Trigger for Schedule Trigger
  to run automatically (e.g. every Monday at 8 AM).

- Change recipient: update "Send To" in the Gmail node.
  You can also add CC/BCC in the node options.

- Change report period label: in "Build HTML Report" node,
  find the badge text and update "Jan-Aug 2024" to your period.

- Add more KPI cards: edit the kpi-grid section in the
  "Build HTML Report" node HTML template.

- Change PDF paper size: in "Gotenberg → Generate PDF" node,
  adjust paperWidth and paperHeight (values are in inches).
  Common sizes: Letter = 8.5x11, Tabloid = 11x17, A4 = 8.27x11.69

- Use a different data source: replace "Read CSV File" and
  "Parse CSV" with a database query node (MySQL, PostgreSQL,
  Airtable, etc.) — the rest of the workflow stays the same
  as long as the field names match.


FILES INCLUDED
--------------
  n8n_CSV_Sheets_PDF_Gmail_Report.json  -- n8n workflow (import this)
  n8n_CSV_Sheets_PDF_Gmail_Report.txt   -- this documentation
  sales_products_sample.csv             -- 100-row sample CSV


================================================================
