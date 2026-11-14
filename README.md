# AI Email Transaction Agent

An AI agent that reads bank alert emails from Gmail, keeps only genuine card transactions, extracts the details into structured data, and answers questions about your spending.

It is built to be careful with sensitive email: messages containing OTP codes are never sent to the LLM, and only the last four card digits are ever extracted.

## What it does

1. **Searches Gmail** (read-only) for emails from the banks you choose, e.g. `hdfc, rbl`
2. **Filters** every email before any AI sees it:
   - ❌ blocks emails that contain an actual OTP / verification code
   - ❌ skips non-transactions: loan offers, credit-limit updates, statements, bill reminders, promotions
   - ✅ keeps successful payments ("payment was made", "card was used", "swiped", "spent", "charged")
3. **Extracts** each transaction with an LLM into a validated Pydantic schema
4. **Answers questions** and writes summaries using only the extracted transactions
5. **Calculates totals in pandas**, not the LLM, so the numbers are exact

## How it works

```
Gmail (read-only)
      │  search: "hdfc OR rbl"
      ▼
Email filter (regex, runs before the LLM)
  ├── actual OTP code in body?            → IGNORE
  ├── sensitive subject (OTP, password)?  → IGNORE
  ├── loan / bill / statement / offer?    → IGNORE
  └── successful payment?                 → PROCESS
      │
      ▼
LLM extraction (gpt-4.1-mini, temperature 0)
  → CreditCardTransaction (Pydantic)
      │
      ▼
pandas DataFrame
  ├── ask_transactions("...")  → LLM answers from the table only
  └── groupby totals           → exact sums
```

### Transaction schema

```python
class CreditCardTransaction(BaseModel):
    bank_name: Optional[str]         # bank or card issuer
    card_last_four: Optional[str]    # last 4 digits only, never the full number
    amount: Optional[float]          # number, no currency symbol
    merchant: Optional[str]
    transaction_date: Optional[str]  # only if stated in the email
```

The extraction prompt tells the model to use only information stated in the email and return `None` for anything missing, instead of guessing.

## Real test run

Tracked banks: `hdfc`

```
Searching for: hdfc
Found 8 matching emails
✅ PROCESS | Fwd: A payment was made using your Credit Card
✅ PROCESS | Fwd: A payment was made using your Credit Card
✅ PROCESS | Fwd: A payment was made using your Credit Card
✅ PROCESS | Fwd: A payment was made using your Credit Card
✅ PROCESS | Fwd: A payment was made using your Credit Card
✅ PROCESS | Fwd: A payment was made using your Credit Card
⏭️ IGNORE  | Fwd: Credit Card xx3353: Higher Loan Amount Update
⏭️ IGNORE  | Fwd: your credit card bill payment was successful
```

6 payments processed; the loan offer and the bill-payment confirmation were correctly skipped.

### Extracted transactions

*Merchant names and amounts below are sample values for privacy; the structure is the real output.*

| bank_name | card_last_four | amount | merchant | transaction_date |
|---|---|---|---|---|
| HDFC Bank | 1234 | 3150.00 | SALON STUDIO | 28 Sep |
| HDFC Bank | 1234 | 2400.00 | CITY PHARMACY | 03 Oct |
| HDFC Bank | 1234 | 3319.09 | FIRSTCRY | 12 Oct |
| HDFC Bank | 1234 | 1500.00 | NOIDA POWER | 20 Oct |
| HDFC Bank | 1234 | 4841.60 | ASSPL | 21 Oct |
| HDFC Bank | 1234 | 2626.00 | ASSPL | 25 Oct |

### Asking questions

```python
print(ask_transactions("What did I spend at each merchant?"))
```
```
Here is the amount you spent at each merchant:
- SALON STUDIO: 3150.0
- CITY PHARMACY: 2400.0
- FIRSTCRY: 3319.09
- NOIDA POWER: 1500.0
- ASSPL: 4841.6 + 2626.0 = 7467.6
```

```python
print(ask_transactions("Summarise my transactions this month."))
```
```
For this month (October 2026), your transactions are:
- 03 Oct: 2400.0 at CITY PHARMACY
- 12 Oct: 3319.09 at FIRSTCRY
- 20 Oct: 1500.0 at NOIDA POWER
- 21 Oct: 4841.6 at ASSPL
- 25 Oct: 2626.0 at ASSPL
```

Exact totals from pandas:
```python
df.groupby("bank_name")["amount"].sum()
```
```
HDFC Bank    17836.69
```

## Design decisions

- **Filter before the LLM.** Security checks run on the raw email first, so OTP emails never reach the model.
- **Block real OTP codes, not the word "OTP".** Bank emails include footers like "never share your OTP or password", so a keyword filter rejected every genuine transaction. The filter now blocks only emails where an OTP phrase is followed by an actual 4–8 digit code, and classifies email type by subject.
- **Last four digits only.** The schema and prompt both restrict card numbers to the last four digits.
- **pandas for maths, LLM for language.** LLMs can make arithmetic mistakes, so totals come from pandas. The LLM explains and summarises.
- **Read-only Gmail access.** The OAuth scope is `gmail.readonly`; the agent cannot send, delete or modify email.

## Tech stack

| Layer | Tool |
|---|---|
| Email | Gmail API (`google-api-python-client`, OAuth, read-only) |
| LLM | OpenAI `gpt-4.1-mini` via LangChain (`with_structured_output`) |
| Schema | Pydantic |
| Data | pandas |
| Filtering | Python `re` |

## Setup

```bash
git clone https://github.com/nimishac28/AI-agent-Transaction.git
cd AI-agent-Transaction
python -m venv venv
venv\Scripts\activate          # Mac/Linux: source venv/bin/activate
pip install langchain langchain-openai pydantic pandas python-dotenv google-api-python-client google-auth-httplib2 google-auth-oauthlib ipykernel
```

1. Create a `.env` file:
   ```
   OPENAI_API_KEY=your_key
   ```
2. In Google Cloud Console, enable the **Gmail API**, create an **OAuth client ID (Desktop app)**, and save it as `credentials.json` in the project folder. Add your Gmail address under **OAuth consent screen → Test users**.
3. Open `AI_agent_gmail_tracker.ipynb`, select the `venv` kernel, and run all cells. On first run a browser window asks you to allow read-only Gmail access; this creates `token.json`.
4. When prompted, enter the banks to track, e.g. `hdfc, rbl`.

> `.env`, `credentials.json` and `token.json` are in `.gitignore`. Never commit them: `token.json` grants access to your Gmail.

## Limitations

- **Credit card payments only.** UPI, net banking, NEFT and IMPS alerts are not yet matched by the filters.
- **Tested on forwarded emails.** Forwarded alerts can lose the original date, so some transactions have no `transaction_date`, and "this month" means the most recent month in the data rather than the current calendar month.
- **Keyword-based filtering.** Banks word their alerts differently; a new bank may need extra patterns.
- **One-off run.** Transactions are held in memory; nothing is saved between runs.

## Next steps

- [ ] Add UPI / net banking / NEFT / IMPS patterns and test them on real alerts
- [ ] Pass today's date to the LLM so "this month" means the calendar month
- [ ] Save transactions to SQLite or CSV to track spending over time
- [ ] Wrap it in a FastAPI service (`/sync`, `/ask`, `/summary`)
- [ ] Monthly category summaries (food, travel, bills, shopping)

## Author

**Nimisha** · [GitHub](https://github.com/nimishac28)
