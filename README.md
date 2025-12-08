# Bank Client Analytics Database

This project builds a relational database and analytics layer on top of the **PKDD’99 financial dataset**, a public banking dataset used for loan risk modelling. It was originally done as a university coursework project.

The goal is to help a bank understand its clients, accounts, loans, credit cards and transactions

---

## Dataset

The project uses the PKDD’99 financial dataset, which:

- Contains eight related tables and around one million rows of data.
- Covers clients, accounts, dispositions, permanent orders, transactions, loans, credit cards and district-level demographic data.
- Is provided as a collection of CSV files.
---

## Data model

The schema is based on an ER/EER diagram that captures the relationships between the main banking entities: accounts, clients, loans, credit cards, transactions, permanent orders, dispositions and districts. 

### Main entities

- **Accounts** – static characteristics of an account:
  - `account_id` (PK), `district_id`, `frequency`, `date` (creation date)
- **Clients** – bank customers:
  - `client_id` (PK), `birth_number`, `district_id 
- **Dispositions** – links clients to accounts and define rights:
  - `disp_id` (PK), `client_id`, `account_id`, `disp_type` (`owner` / `user`)
- **Permanent orders** – standing payment instructions:
  - `order_id`, `account_id`, `bank_to`, `account_to`, `amount`, `k_symbol` 
- **Transactions** – individual account movements:
  - `trans_id` (PK), `account_id`, `date`, `type`, `operation`, `amount`, `balance`, plus optional bank/account fields
- **Loans** – loans granted to accounts:
  - `loan_id` (PK), `account_id`, `date`, `amount`, `duration`, `payments`, `status` (`A`, `B`, `C`, `D`)
- **Credit cards** – cards issued to dispositions:
  - `card_id` (PK), `disp_id`, `type` (`junior`, `classic`, `gold`), `issued` date
- **Districts** – demographic data for each district:
  - Attributes include district name, region, population breakdowns, urbanisation ratio, average salary, unemployment rates, entrepreneurs per 1000 inhabitants, and crime statistics.

### Key relationships

- An account has at least one disposition, each tying it to a client; a client can have multiple dispositions (e.g. owner of one account, user of another). 
- Each disposition links exactly one client and one card/account combination, but clients and cards can appear in multiple dispositions.
- An account can issue many orders and have at most one loan, and can be associated with many transactions over time.
- Both accounts and clients belong to a single district, while each district can host many accounts and clients.

---

## Technology stack

The project runs entirely in a Jupyter Notebook, combining Python and SQL:

- **Python** for data loading, cleaning, and orchestration
- **pandas** to read CSV files and inspect/transform the data
- **SQLite** as the relational database
- **peewee** and/or `sqlite3` for table creation, inserts, queries, and view/trigger definitions 
- **DB Browser for SQLite** to visually inspect tables and views
- Additional Python libraries: `os` for file management and `datetime` for handling date objects

---

## Features

### 1. Data preparation & schema creation

- Loads the eight CSV files into pandas, cleans and transforms columns, especially date fields.
- Creates empty SQL tables with:
  - Explicit data types and `NOT NULL` constraints
  - Primary keys and foreign keys between all entities
  - Additional CHECK constraints for categorical data (e.g. valid loan status codes)
- Inserts the cleaned data into the database and validates the construction by inspecting the schema and key relationships.

### 2. Analytical queries

The notebook implements a set of queries that demonstrate how the bank can use the data to understand its clients and risk:

1. **Client distribution by district**  
   Lists all districts ordered by how many clients they have.  
   Helps identify where the client base is concentrated and where extra branch staff might be needed. 

2. **Credit card portfolio mix**  
   Counts how many `classic`, `junior`, and `gold` cards are in use.  
   Useful for understanding product popularity and guiding marketing or product development.

3. **Average salary vs average loan per district**  
   Compares each district’s average salary with the average yearly loan amount, giving the bank a way to assess affordability and lending risk by region.

4. **Demographics of high-risk loans**  
   Selects districts where clients are in debt or haven’t paid their loans (`status` B or D) and returns the full demographic profile of those districts.  
   Helps identify regions with higher risk of default and inform approval policies.

5. **Transaction volumes over time**  
   Counts how many transactions were made each year from 1993 to 1998, giving a view of growth trends in customer activity.

6. **Dynamic transaction management query**  
   Demonstrates multi-statement transaction control: inserting a new loan application, updating its status, and deleting loans with very short duration using `BEGIN`, `SAVEPOINT`, `ROLLBACK`, and `COMMIT` to keep the database consistent.

### 3. Views

Several views are defined to support reusable business analytics:

1. **`district_loan_debt`** - *Total loan amount per district*  
   - Purpose: show how much debt is outstanding in each district, supporting risk segmentation and portfolio monitoring.

2. **`card_type_counts`** – *Number of cards by type*  
   - Purpose: quickly see how many `classic`, `gold`, and `junior` cards are active.

3. **`dis_trans`** – *Total transaction value per district*  
   - Purpose: identify profitable districts that generate high transaction value and combine this with loan data to build risk/return profiles.

4. **`dis_accounts`** – *Number of accounts per district and region*  
   - Purpose: highlight regions with many active accounts, useful for detecting anomalies or targeting expansion opportunities.

### 4. Trigger

A logging trigger is implemented on the `Loans` table:

- **Log trigger on `Loans`**  
  - Fires whenever a loan record is updated.  
  - Inserts the old value, the new value, and a timestamp into a separate log table.  
