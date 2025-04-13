# Compass-Mining-Code-Screen
## TASK OVERVIEW


1. Install PostgreSQL
2. Create a book database and users
3. Set access privileges
4. Create a function and view
5. Write a cleanup script
6. Automate everything

---


### Option 1: Use AWS EC2 (Ubuntu 24.04)

1. Login to AWS Console → [https://console.aws.amazon.com](https://console.aws.amazon.com)
2. Launch EC2 Instance
   - AMsI: Ubuntu 24.04 LTS
   - Instance: `t2.micro` (Free tier)
   - Security Group: Allow port `22` (SSH)
   - Storage: Min 8GB
3. Connect to EC2
   ```bash
   chmod 400 your-key.pem             # Secure the key file
   ```
   ```bash
   ssh -i "your-key.pem" ubuntu@<IP>  # Connect to your instance
   ```

---

###  Option 2: Use WSL on Windows

```powershell
wsl --install -d Ubuntu    # Install Ubuntu via WSL
```

Then open Ubuntu from Start Menu and you're good to go!

---

##  **Validation Check**

```bash
lsb_release -a   # Check Linux version
```

Expected:
```
Distributor ID: Ubuntu
Description:    Ubuntu 24.04 LTS
```

---

## 🔧 Step 1: Install PostgreSQL

```bash
sudo apt update && sudo apt upgrade -y      #  Refresh & upgrade package list
```
```bash
sudo apt install postgresql postgresql-contrib -y   #  Install PostgreSQL
```

Check status:
```bash
sudo systemctl status postgresql   # Confirm it's running
```

If not:
```bash
sudo systemctl start postgresql    # Start PostgreSQL service
```

---

## Step 2: Create Project Directory & Files

```bash
mkdir book-db-project && cd book-db-project   #  Create and enter directory
```

```bash
touch .env setup.sh schema.sql cleanup.sh functions.sql views.sql README.md
```

Folder structure:
```
book-db-project/
├── .env
├── setup.sh
├── schema.sql
├── cleanup.sh
├── functions.sql
├── views.sql
└── README.md
```

---

##  .env File (Secrets & Configs)

```bash
# .env
DB_NAME=books_db
ADMIN_USER=admin_user
ADMIN_PASS=AdminPass123
VIEW_USER=view_user
VIEW_PASS=ViewPass123
```

These values will be used across setup/cleanup for flexibility.

---

## setup.sh – Automated Setup Script

```bash
#!/bin/bash
set -e                              # ❗ Exit on error
source .env                         # 📥 Load environment variables

echo "📦 Creating database and users..."

sudo -u postgres psql <<EOF
DROP DATABASE IF EXISTS $DB_NAME;              -- 🗑 Remove existing DB
DROP USER IF EXISTS $ADMIN_USER;              -- 🗑 Remove old users
DROP USER IF EXISTS $VIEW_USER;

CREATE DATABASE $DB_NAME;                     -- 🆕 Create new database
CREATE USER $ADMIN_USER WITH ENCRYPTED PASSWORD '$ADMIN_PASS'; -- 👤 Admin
CREATE USER $VIEW_USER WITH ENCRYPTED PASSWORD '$VIEW_PASS';   -- 👁️ View-only

GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO $ADMIN_USER;     -- ✅ Full access to admin
EOF

echo "📄 Running schema and object creation..."

sudo -u postgres psql -d $DB_NAME -f schema.sql       # 📚 Create tables
sudo -u postgres psql -d $DB_NAME -f functions.sql    # 🧠 Create function
sudo -u postgres psql -d $DB_NAME -f views.sql        # 👁️ Create view

echo "🔐 Granting view access to view_user..."

sudo -u postgres psql -d $DB_NAME <<EOF
GRANT CONNECT ON DATABASE $DB_NAME TO $VIEW_USER;
GRANT USAGE ON SCHEMA public TO $VIEW_USER;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO $VIEW_USER;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO $VIEW_USER;
EOF

echo "✅ Setup complete!"
```

✅ Don’t forget:
```bash
chmod +x setup.sh
```

---

## 🧹 cleanup.sh – Wipe Everything Clean

```bash
#!/bin/bash
set -e
source .env

echo "🧹 Dropping database and users..."

sudo -u postgres psql <<EOF
DROP DATABASE IF EXISTS $DB_NAME;
DROP USER IF EXISTS $ADMIN_USER;
DROP USER IF EXISTS $VIEW_USER;
EOF

echo "✅ Cleanup complete!"
```

✅ Don’t forget:
```bash
chmod +x cleanup.sh
```

---

## 📑 schema.sql – Table Definition

```sql
CREATE TABLE books (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  subtitle VARCHAR(255),
  author VARCHAR(255) NOT NULL,
  publisher VARCHAR(255)
);
```

📘 This stores book info like title, author, publisher, etc.

---

## 🧠 functions.sql – Create Function

```sql
CREATE OR REPLACE FUNCTION total_books()
RETURNS INTEGER AS $$
BEGIN
  RETURN (SELECT COUNT(*) FROM books);   -- 🔢 Return number of books
END;
$$ LANGUAGE plpgsql;
```

---

## 👁️ views.sql – Create View

```sql
CREATE OR REPLACE VIEW books_summary AS
SELECT title, author FROM books;   -- 👁️ View only title & author
```

---

## 🚀 Run the Setup

```bash
./setup.sh    # 🧠 Auto runs .env, creates DB, users, tables, views, etc.
```

---

## ✅ Verification Steps

### 👥 Check PostgreSQL Users

```bash
sudo -u postgres psql -c "\du"   # 🔍 List users
```

Expected:
```
 admin_user | 
 view_user  |
```

---

### 💽 Check Databases

```bash
sudo -u postgres psql -c "\l"   # 📂 List databases
```

---

### 📊 Check Tables

```bash
sudo -u postgres psql -d books_db -c "\dt"
```

Should show:
```
 public | books | table | postgres
```

---

### 🧠 Check Function

```bash
sudo -u postgres psql -d books_db -c "\df"
```

🔍 Run it:
```bash
sudo -u postgres psql -d books_db -c "SELECT total_books();"
```

---

### 👁️ Check Views

```bash
sudo -u postgres psql -d books_db -c "\dv"
```
```bash
sudo -u postgres psql -d books_db -c "SELECT * FROM books_summary;"
```

🧪 If empty, insert a book:

```bash
sudo -u postgres psql -d books_db -c \
```
```bash
"INSERT INTO books (title, subtitle, author, publisher) VALUES ('PostgreSQL Guide', 'For Beginners', 'Jane Doe', 'OpenSource Press');"
```

---

### 🔐 Test View-Only User Access

```bash
psql -U view_user -d books_db -h localhost
```
```bash
SELECT * FROM books_summary;
```

Or quick test:
```bash
sudo -u postgres psql -d books_db -U view_user -c "SELECT * FROM books_summary;"
```

> 🛠 If you face password errors, edit `pg_hba.conf` or reset password via SQL.

---

## 🧼 Run Cleanup

```bash
./cleanup.sh    # 🧹 Drops DB & users created during setup
```

---

