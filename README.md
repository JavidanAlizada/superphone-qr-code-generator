# Superphone QR Code Generator: Customer Registration Backend

A Spring Boot backend that syncs customer registrations from **Google Sheets** into **MySQL**, generates a unique **QR code payload** and a one-time **password** for each customer, and exposes a REST API to look up customers and track their status.

## How it works

```
Google Form ──► Google Sheet ──(poll every 30s, Sheets API v4)──► CustomerService ──► MySQL
                                                                     │
                                                       QR content + generated password
                                                                     │
                                         REST API ◄── lookup by serial number / QR / password
```

1. A background loop polls the Google Sheet every 30 seconds via the **Google Sheets API (OAuth 2.0)**.
2. New rows are mapped to `Customer` records (full name, phone, gender, birth date, marital status, serial number).
3. For each new customer the service builds QR-code content (`serial_phone_name`) and generates a unique password.
4. Customers are saved to MySQL through plain JDBC queries.
5. The REST API serves lookups and status/QR updates (e.g. for a scanner app at an event or a point of sale).

## Tech stack
- **Java 8**, **Spring Boot 2.4** (Spring Web)
- **Google Sheets API v4** + Google OAuth client
- **MySQL 8** (JDBC)
- **Jackson**, **Lombok**, **Gradle**

## REST API
Base path: `/api/v1/customer`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/all` | List all customers |
| `GET` | `/bySerialNumber?serialNumber={sn}` | Find a customer by serial number |
| `GET` | `/byQrCodeContent?qrCodeContent={content}` | Find a customer by scanned QR content |
| `GET` | `/byPassword?password={pwd}` | Find a customer by generated password |
| `PATCH` | `/updateQrCode/{serialNumber}/` | Update QR content (request body: new content) |
| `PATCH` | `/status/{serialNumber}/` | Update customer status (request body: integer) |

Response format:
```json
{ "success": true, "data": { ... }, "error": null }
```

## Project structure
```
src/main/java/superfon
├── Main.java                  # Spring Boot entry point + sheet polling loop
├── auth/GoogleCredentials     # OAuth 2.0 flow for Google Sheets
├── controller/                # REST endpoints
├── service/                   # CustomerService, GoogleSheetService
├── repository/                # JDBC connection + SQL queries
├── model/Customer             # Domain model
└── util/                      # Password generator, property reader, response formatter
```

## Running locally
**Prerequisites:** JDK 8+, MySQL 8, a Google Cloud project with the Sheets API enabled.

1. Create a MySQL database and a `Customer` table.
2. Configure `src/main/resources/application.properties`:
   ```properties
   db_url=jdbc:mysql://localhost:3306/<database>
   db_user=<user>
   db_passwd=<password>
   sheet_id=<google-sheet-id>
   range=<sheet-range>
   appName=<application-name>
   ```
3. Put your own OAuth client file at `src/main/resources/credentials.json` (from Google Cloud Console → APIs & Services → Credentials).
4. Run:
   ```bash
   ./gradlew bootRun
   ```
   On the first run a browser window opens for Google authorization. The token is stored in `tokens/`.

> **Note:** never commit your real `credentials.json` or the `tokens/` folder. Keep them out of version control.
