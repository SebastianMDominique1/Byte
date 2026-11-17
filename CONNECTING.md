# Connecting ByteBites, step by step

The code is complete enough to test the shared database workflow. Your provider accounts and secret connection settings are the remaining external connections. Work with fictional information until your team and studio approve the launch.

## 1. Open the project

1. Extract `ByteBites_Connected.zip`.
2. Open VS Code and choose **File → Open Folder**.
3. Select the `bytebites-connected` folder that contains `package.json`.
4. Choose **Terminal → New Terminal**. On Windows the commands below work in PowerShell.
5. Install Node.js 22.13 or later if `node --version` does not work. Restart VS Code after installation.

```powershell
node --version
npm --version
npm ci
npm run keys:generate
```

`keys:generate` creates a private `.env` file with fresh local database credentials and a 32-byte encryption key. It refuses to overwrite an existing `.env`. Back up this file securely. Losing or changing `DATA_ENCRYPTION_KEY` makes the existing encrypted records unreadable. Never commit `.env` to GitHub.

## 2. Choose your database connection

### Local PostgreSQL for the first test

Install Docker Desktop, start it, and run this from the same folder.

```powershell
docker compose up -d
docker compose ps
npm run db:migrate
npm run db:seed
npm run check:connections
```

Wait until `docker compose ps` reports PostgreSQL healthy before running migrations. The generated `.env` already matches this Docker configuration. The database listens only on your own computer. The named Docker volume preserves records across restarts. `docker compose down` stops the database while preserving that volume. Do not use `docker compose down -v` on records you need to keep.

### Hosted PostgreSQL for sharing between team computers

For the planned Vercel deployment, provision a PostgreSQL database from a supported provider such as Neon through your account or the Vercel storage marketplace. This action belongs to your provider account and may involve its terms and costs.

Copy the provider's **pooled PostgreSQL connection string** and its **direct/unpooled connection string** into your local `.env`.

```dotenv
DATABASE_URL="postgresql://USER:PASSWORD@POOLED_HOST/DATABASE?sslmode=require"
DIRECT_URL="postgresql://USER:PASSWORD@DIRECT_HOST/DATABASE?sslmode=require"
```

These are placeholders. Use the exact provider-supplied URLs. The running website uses `DATABASE_URL`. Prisma migrations use `DIRECT_URL`. A provider may name the direct variable `DATABASE_URL_UNPOOLED`; copy that value into this app's `DIRECT_URL`. URL-encode special characters if you construct a password-containing URL yourself. Keep TLS enabled. Never place these values in variables starting with `NEXT_PUBLIC_`.

Then run the same database commands.

```powershell
npm run db:migrate
npm run db:seed
npm run check:connections
```

Migrations create the tables in `prisma/schema.prisma` and track the migration. Seed creates three sample artist profiles without creating shared login credentials. If you already have tables from another project, create a new empty database for this version instead of applying this initial migration over them.

## 3. Create your employee account

Open `.env` in VS Code and temporarily fill these three entries with the employee's real account details and a new password of at least 12 characters.

```dotenv
STAFF_NAME="Your Name"
STAFF_EMAIL="your-staff-email@example.com"
STAFF_PASSWORD="a-new-unique-password-of-at-least-12-characters"
```

Run the bootstrap command.

```powershell
npm run staff:create
```

Remove the `STAFF_PASSWORD` value after success. The database keeps a salted scrypt hash. The command refuses to overwrite an existing account or silently turn a customer into an employee. Repeat with different details for another approved staff member. Customer registration always creates a customer role. Use your own unique accounts; the old MorganLee/AlexBrooks demo passwords are not included in this application.

## 4. Start the website

```powershell
npm run dev
```

Open `http://localhost:3000` in your browser. The default `.env` contains `APP_URL=http://localhost:3000`. The origin in the browser must match `APP_URL` exactly. If you choose `http://127.0.0.1:3000` instead, set that exact origin in `.env` and restart the server.

Choose **Create customer account** to register a customer using an individual email and password. Choose **Employee login** to sign in with the staff account from step 3.

To run a production build locally after the first test, stop the dev server using Ctrl+C and run:

```powershell
npm run typecheck
npm run build
npm start
```

## 5. Test the whole connection

Use a regular browser window for the customer and an Incognito/InPrivate window for the employee. They have separate session cookies and use the same database.

1. In the customer window, register and select **Verify my ID**. Enter fictional adult identity details and a future document expiry. Submit.
2. In the employee window, sign in and open **ID reviews**. Use the three checks only after a real in-person original-ID inspection when running the actual studio workflow. For this test, inspect fictional sample details and approve.
3. Customer selects **My paperwork**, reads the sample waiver, fills the form, signs and submits.
4. Customer selects **Body canvas**, rotates the 3D model, picks a location and changes the size. Select **Use in appointment request**.
5. Customer requests a future appointment with an artist.
6. Employee opens **Appointments**, selects **Refresh records**, and sees that request. Confirm it, optionally with a quote.
7. Customer refreshes and sees **CONFIRMED**. Open its message thread and send a question. Employee opens the same appointment thread and replies.
8. Create a second customer and confirm it cannot see the first customer's records. Attempt an overlapping booking to confirm the employee sees a conflict.
9. After the appointment time, employee records completion with the actual visit checks and client signature. **Service records** shows the retention review date and legal-hold control.
10. Sign out, refresh, and confirm the workspace requires sign-in.

Business records are stored in PostgreSQL. Browser refresh and separate computers do not create separate copies. The UI refreshes every 20 seconds while visible, or immediately with **Refresh records**. All staff accounts can access studio records. Customers can access their own records only.

## 6. Deploy the connected version to Vercel

This is a standard Next.js/Node project for the stack your team selected. The earlier ChatGPT Sites demo used a different hosting runtime. The code in this folder should be deployed as its own Next.js project.

1. Create a private GitHub repository. Add the contents of `bytebites-connected` with `package.json` at the repository root. Check `.gitignore` before committing. Keep `.env`, `.next*`, database dumps, uploads and test data out of the repository.
2. In Vercel, import that repository as a Next.js project. Set a supported Node version of at least 22.13.
3. In the project's environment settings, add `DATABASE_URL`, `DIRECT_URL`, `DATA_ENCRYPTION_KEY` and `APP_URL`. Use the same encryption key used for the records already in the database. Set `APP_URL` to the exact HTTPS origin assigned to the deployment or your final custom domain.
4. Configure both preview and production environments separately as needed. Each origin must match its own `APP_URL`. Prefer a separate test database for previews.
5. Run `npm run db:migrate` against the hosted database from your local machine or a trusted CI job before directing visitors to the new code. Run seed and staff creation once against that same database. Back up first for subsequent schema changes. Do not run `migrate dev` against production.
6. Set the build command to `npm run build`. Deploy. Environment changes require a redeployment.
7. Keep access restricted while you test. Check which deployment-protection options your Vercel account supports. Review both role flows using fictional records before approving access for actual customers.
8. Point your final domain at the deployment, set `APP_URL` to that domain's HTTPS origin, redeploy and repeat the checks.

The download does not provision a Vercel project, a PostgreSQL account, DNS, a bucket or a mail sender. Do not upload `.env` or paste credentials into a public issue or chat. Add secrets directly through your provider's settings.

## 7. Connect password-reset emails

1. Create a Resend account and verify a sending domain you control.
2. Create a server-side API key.
3. Set these variables locally and in Vercel.

```dotenv
RESEND_API_KEY="your-provider-api-key"
MAIL_FROM="ByteBites <accounts@your-verified-domain.example>"
```

4. Restart locally or redeploy remotely.
5. Choose **Forgot password?**, request a reset for your customer account, and check delivery.
6. Open the email link, choose a new 12+ character password, and verify that the old password and prior sessions stop working.

The reset token is random, hashed in the database, single-use and expires after 30 minutes. The app never returns the token in an API response. Actual email delivery must be tested with your verified sender. Without these settings the UI reports that email is not configured.

## 8. Optional private tattoo-reference storage

ID review works through original-ID inspection and does not need image storage. To enable private tattoo reference images:

1. Create a private Amazon S3 bucket or compatible private object store. Keep all S3 public-access blocks enabled. Do not add public-read policies or website hosting.
2. Create limited server credentials granting PutObject, GetObject and DeleteObject only on the bucket's `references/*` prefix. The application does not need full account administration.
3. Enable provider encryption and backups/retention suitable for the studio.
4. Set the following server variables.

```dotenv
S3_BUCKET="your-private-bucket"
S3_REGION="your-bucket-region"
S3_ACCESS_KEY_ID="your-limited-access-key"
S3_SECRET_ACCESS_KEY="your-limited-secret"
S3_SSE="AES256"
```

For another S3-compatible provider, also set its `S3_ENDPOINT`. Set `S3_FORCE_PATH_STYLE=true` only if required by that provider. Leave `S3_SSE` empty if that provider uses its own compatible server-encryption settings instead of the AWS SSE-S3 header.

5. Restart/redeploy. Appointment cards show the upload control when the storage variables exist.
6. Upload a fictional JPEG, PNG or WEBP below 3 MB. Confirm the customer owner and staff can view it through the authorized API, and another customer cannot access it.

Bytes are validated by detected file type, encrypted using AES-256-GCM before storage and served only after ownership checks. Provider URLs and object keys are kept on the server. IDs, PDFs and executable files are not supported by this upload endpoint. A security review should add image re-encoding/malware scanning appropriate to the studio before broad public uploads.

## 9. Retention and backups

```powershell
npm run retention:report
```

The report lists only service-record IDs, actual visit dates and review dates when at least two years have elapsed and no legal hold exists. It never deletes records. Store reports privately. Your team must agree on disposal approval, legal holds and how backups/objects are handled before adding any deletion operation.

Enable database backups with your hosted provider and test a restore into a separate database. Store an encrypted backup of `DATA_ENCRYPTION_KEY` separately from the data backup. Confirm the restored app can decrypt an intake and service record. Keep studio/inspector access available as required by the approved workflow.

## Common connection errors

| Message | Check |
| --- | --- |
| Cannot reach database | PostgreSQL service is running, host/port/password are correct, provider network access is allowed and TLS settings match |
| Table does not exist | Run `npm run db:migrate` against the intended database |
| Request origin is not allowed | Browser origin exactly matches `APP_URL`; restart or redeploy after editing it |
| Encryption failure | Correct original `DATA_ENCRYPTION_KEY` is present; do not generate a replacement for existing records |
| Email/password incorrect for portal | Use the employee portal for STAFF accounts and customer portal for CLIENT accounts |
| Customer cannot book | Latest ID review must be approved and unexpired, and an intake must be saved |
| ID details changed | Employee reviews the new ID submission; customer requests a new appointment bound to that review |
| Reference-image storage unconfigured | Set the private bucket credentials, region and bucket, then redeploy |
| Reset email unconfigured | Verify the sender and set Resend/Mail From variables |

The sample waiver and tattoo service workflow must be approved by the studio and confirmed with its health department. The Department's available linked statute PDF is dated 2014. Check current law and record/signature acceptance before real client use. This implementation is not a legal compliance determination.
