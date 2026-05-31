# Migration-Hell-Causes-Investigation-and-Recovery
This report documents a real-world Django/PostgreSQL debugging incident that initially appeared to be a catastrophic migration failure. 

Environment
Application Stack
    • Django 5.1.15
    • PostgreSQL 16
    • Ubuntu 24.04 LTS
    • Python 3.12
    • Virtual Environment (venv)
Project Components
    • users app
    • ledger app
    • api app
    • security app

Initial Symptoms
The application generated database connection errors and appeared unable to access existing data.
Observed errors included:
django.db.utils.OperationalError:
connection to server at "127.0.0.1", port 5432 failed:
fe_sendauth: no password supplied
and later:
django.db.utils.OperationalError:
password authentication failed for user "bankuser"
The project also appeared to contain multiple databases:
bank_db
bankdb
creating uncertainty regarding which database actually contained production data.

Investigation Timeline
Step 1: Verify PostgreSQL Service
Check PostgreSQL status:
sudo systemctl status postgresql
Result:
active (running)
Check clusters:
sudo pg_lsclusters
Result:
16 main 5432 online
Conclusion:
PostgreSQL infrastructure was healthy.

Step 2: Discover Existing Databases
List databases:
sudo -u postgres psql -l
Results:
bank_db
bankdb
postgres
template0
template1
This immediately raised suspicion because Django referenced one database while application data might reside in another.

Step 3: Identify Real Database
Inspect tables:
sudo -u postgres psql -d bank_db -c "\dt"
Result:
Did not find any relations.
Inspect second database:
sudo -u postgres psql -d bankdb -c "\dt"
Result:
ledger_account
ledger_transaction
users_bankuser
django_migrations
...
Conclusion:
The real application database was:
bankdb
while:
bank_db
was essentially empty.

Step 4: Verify Django Configuration
Current Django configuration:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'bank_db',
        'USER': 'postgres',
        'PASSWORD': '',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}
Problem:
Django was configured for the wrong database.

Step 5: Create Emergency Backup
Before making changes:
pg_dump -U bankuser -h 127.0.0.1 -Fc bankdb > backups/bankdb_before_fix.dump
Backup successfully created.
This became the recovery checkpoint.

Step 6: Inspect Environment Variables
Discovered:
DB_NAME=bankdb
DB_USER=bankuser
DB_PASSWORD=********
DB_HOST=localhost
DB_PORT=5432
The .env file contained the correct information.
However, Django was not using it.

Step 7: Detect Configuration Bug
An attempted fix introduced:
os.getenv("bankdb")
os.getenv("bankuser")
instead of:
os.getenv("DB_NAME")
os.getenv("DB_USER")
Result:
{
    'NAME': None,
    'USER': None,
    'PASSWORD': None
}
Root Cause #1:
Environment variable values were used instead of environment variable names.

Step 8: Verify Environment Loading
Check:
python manage.py shell -c "import os; print(os.getenv('DB_NAME'))"
Output:
bankdb
Conclusion:
load_dotenv() was functioning correctly.
The issue existed inside DATABASES configuration.

Step 9: Fix DATABASES Configuration
Correct configuration:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT'),
    }
}

Step 10: Discover Credential Drift
Django now attempted authentication but failed:
password authentication failed for user bankuser
Investigation showed:
Database password:
Do*******
Environment password:
su*******
Root Cause #2:
Credential mismatch after password reset.

Step 11: Synchronize Credentials
Update:
DB_PASSWORD=
to match PostgreSQL role password.
Verification:
python manage.py shell -c \
"from django.db import connection;
connection.ensure_connection();
print('Connected OK')"
Output:
Connected OK

Migration Analysis
Review pending migrations:
python manage.py migrate --plan
Result:
api.0001_initial
security.0001_initial
security.0002_initial
Review generated SQL:
python manage.py sqlmigrate api 0001
Tables created:
api_account
api_transaction
No collisions existed with:
ledger_account
ledger_transaction
Migration risk determined to be low.

Migration Execution
Apply migrations:
python manage.py migrate
Result:
Applying api.0001_initial... OK
Applying security.0001_initial... OK
Applying security.0002_initial... OK

Final Validation
Migration status:
python manage.py showmigrations
Result:
All migrations applied.
Database inspection:
python manage.py shell -c \
"from django.db import connection;
print(sorted(connection.introspection.table_names()))"
Result:
api_account
api_transaction
ledger_account
ledger_transaction
security_auditlog
security_mfadevice
users_bankuser
...
System validation:
python manage.py check
Output:
System check identified no issues.

Root Causes
Root Cause 1
Wrong database configured.
Configured:
bank_db
Actual:
bankdb

Root Cause 2
Wrong database user.
Configured:
postgres
Actual:
bankuser

Root Cause 3
Missing password.
Configured:
PASSWORD=''

Root Cause 4
Incorrect use of environment variables.
Wrong:
os.getenv("bankdb")
Correct:
os.getenv("DB_NAME")

Root Cause 5
Credential drift after password reset.
Environment variables no longer matched PostgreSQL credentials.

Lessons Learned
    1. Always create a backup before migration work.
    2. Verify which database actually contains data.
    3. Trust evidence, not assumptions.
    4. Inspect migration plans before execution.
    5. Use environment variables consistently.
    6. Keep database credentials synchronized.
    7. Verify connectivity before troubleshooting migrations.
    8. Most migration disasters are configuration disasters in disguise.

Outcome
The database was never corrupted.
The migrations were never broken.
The application failed because Django, PostgreSQL, and environment configuration had drifted apart.
Once configuration alignment was restored, connectivity, migrations, and application state recovered successfully without data loss.






Response 2
Migration Hell: Causes, Diagnosis, and Recovery
A real-world Django + PostgreSQL debugging report
Executive Summary
This incident began as what appeared to be a PostgreSQL authentication failure and evolved into a multi-layer configuration investigation involving:
    • Django settings 
    • PostgreSQL users and databases 
    • Environment variables (.env) 
    • Migration state 
    • Database ownership 
    • Credential mismatches 
The root cause was not database corruption, failed migrations, or PostgreSQL service failure. The actual problem was a chain of configuration inconsistencies that accumulated over time.
The project was successfully recovered without data loss.

Initial Symptoms
Django failed to connect to PostgreSQL with:
django.db.utils.OperationalError:
connection to server at "127.0.0.1", port 5432 failed:
fe_sendauth: no password supplied
At first glance, this suggested:
    • PostgreSQL down 
    • Network issue 
    • Corrupted database 
    • Incorrect authentication configuration 
None of those turned out to be the primary cause.

Phase 1: PostgreSQL Health Verification
Service Status
sudo systemctl status postgresql
Result:
Active: active (exited)
This can be misleading.
On Ubuntu, PostgreSQL uses clusters.
The real verification was:
sudo pg_lsclusters
Output:
16 main 5432 online
Meaning:
✅ PostgreSQL cluster running

Phase 2: Database Discovery
Listing databases:
\l
Revealed:
bank_db
bankdb
postgres
This immediately raised suspicion.
Two similarly named databases existed:
bank_db
bankdb

Phase 3: Inspecting Both Databases
bank_db
sudo -u postgres psql -d bank_db -c "\dt"
Output:
Did not find any relations.
Database was empty.

bankdb
sudo -u postgres psql -d bankdb -c "\dt"
Output:
ledger_account
ledger_transaction
users_bankuser
django_migrations
...
This database contained the entire application.
Conclusion:
bankdb = real application database
bank_db = empty database

Phase 4: Django Configuration Audit
Current settings:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'bank_db',
        'USER': 'postgres',
        'PASSWORD': '',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}
Problem:
Django pointed to:
bank_db
postgres
(no password)
while application data existed in:
bankdb
bankuser

Phase 5: User Authentication Investigation
Listing PostgreSQL roles:
\du
Output:
bankuser
postgres
Attempting login:
psql -U bankuser -d bankdb -h 127.0.0.1
Initially failed.
Password was forgotten.
Password reset:
ALTER ROLE bankuser
WITH PASSWORD 'new_password';
Verification:
psql -U bankuser -d bankdb -h 127.0.0.1
Success.

Phase 6: Backup Before Surgery
Before changing configuration:
pg_dump -U bankuser -h 127.0.0.1 -Fc bankdb \
> backups/bankdb_before_fix.dump
Result:
90 KB backup
Recovery point established.

Phase 7: Environment Variable Investigation
.env contained:
DB_NAME=bankdb
DB_USER=bankuser
DB_PASSWORD=*****
DB_HOST=localhost
DB_PORT=5432
Settings loaded dotenv:
load_dotenv(...)
However Django still failed.

Phase 8: The getenv() Trap
Initial attempt:
NAME = os.getenv('bankdb')
USER = os.getenv('bankuser')
PASSWORD = os.getenv('mypassword')
Critical mistake.
os.getenv() expects:
os.getenv('VARIABLE_NAME')
not:
os.getenv('variable_value')
Result:
None
None
None
Django configuration became:
NAME=None
USER=None
PASSWORD=None

Phase 9: Correct Environment Configuration
Corrected:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT'),
    }
}
Verification:
print(settings.DATABASES)
Output:
NAME=bankdb
USER=bankuser
PASSWORD=*******

Phase 10: Password Mismatch Discovery
New error:
password authentication failed for user "bankuser"
Meaning:
Before:
No password supplied
After:
Wrong password supplied
Progress.
Investigation revealed:
PostgreSQL password:
Do******
.env password:
su******
Passwords differed.
Updating .env resolved authentication.

Phase 11: Successful Connection
Verification:
python manage.py shell -c "
from django.db import connection
connection.ensure_connection()
print('Connected OK')
"
Output:
Connected OK

Phase 12: Migration State Verification
python manage.py showmigrations
Found:
api
 [ ] 0001_initial
security
 [ ] 0001_initial
 [ ] 0002_initial
Pending migrations existed.

Phase 13: Migration Risk Assessment
Review:
python manage.py migrate --plan
Planned operations:
Create model Account
Create model Transaction
Create model AuditLog
Create model MFADevice
No destructive actions.
No:
DROP TABLE
DELETE MODEL
REMOVE FIELD

Phase 14: SQL Inspection
python manage.py sqlmigrate api 0001
Confirmed creation of:
api_account
api_transaction
No collision with:
ledger_account
ledger_transaction

Phase 15: Migration Execution
python manage.py migrate
Result:
Applying api.0001_initial... OK
Applying security.0001_initial... OK
Applying security.0002_initial... OK

Final State
Database
bankdb
User
bankuser
Authentication
Working
Migrations
Fully applied
Tables
api_account
api_transaction
ledger_account
ledger_transaction
security_auditlog
security_mfadevice
users_bankuser
...

Root Causes
1. Configuration Drift
Django settings no longer matched the database actually in use.

2. Duplicate Databases
Both existed:
bank_db
bankdb
leading to confusion.

3. Forgotten Credentials
bankuser password was unknown.

4. Environment Variable Misuse
Incorrect:
os.getenv("bankdb")
Correct:
os.getenv("DB_NAME")

5. Credential Mismatch
.env password differed from PostgreSQL password.

Lessons Learned
Always Back Up First
pg_dump ...
before changing authentication or migrations.

Verify Actual Data Location
Never assume:
settings.py
matches:
production database
Verify.

Inspect Before Migrating
Use:
python manage.py migrate --plan
and
python manage.py sqlmigrate
before applying changes.

Environment Variables Require Names
Wrong:
os.getenv("bankdb")
Right:
os.getenv("DB_NAME")

Authentication Errors Tell a Story
fe_sendauth: no password supplied
means:
No password sent.
while:
password authentication failed
means:
Password sent, but incorrect.
Those are very different debugging signals.

Outcome
Data Loss: None
Downtime: Local development only
Database Recovery Required: No
Root Cause: Configuration drift + credential mismatch + environment variable misuse
Resolution: Successful
Status: Fully operational. ✅
I prefer this response







