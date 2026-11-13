# AdPay – Phase 1: database + local infrastructure

## 1. Start Postgres and Redis
```bash
cp .env.example .env
docker compose up -d
docker compose ps          # both should show "healthy"
```

## 2. Install Python deps and run migrations
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
```

## 3. Check it worked
```bash
docker compose exec db psql -U adpay -d adpay -c "\dt"
python -m scripts.seed
```

## 4. Prove the ledger is append-only (this SHOULD fail)
```bash
docker compose exec db psql -U adpay -d adpay -c \
  "INSERT INTO wallet_ledger (user_id, amount, entry_type, ref_id)
   SELECT id, 5, 'adjustment', gen_random_uuid() FROM users LIMIT 1;"
docker compose exec db psql -U adpay -d adpay -c "UPDATE wallet_ledger SET amount = 999;"
# ERROR: wallet_ledger is append-only: UPDATE not allowed
docker compose exec db psql -U adpay -d adpay -c "SELECT * FROM wallet_balances;"
```

## Useful commands
```bash
alembic downgrade base                 # undo everything
alembic upgrade head --sql             # print the SQL without running it
docker compose down -v                 # stop AND wipe all data
```

## Design rules to keep in later phases
1. **Balance = SUM(wallet_ledger.amount).** Never add a mutable balance column.
2. **Budget deduction must be one atomic statement**, not read-then-write:
   ```sql
   UPDATE campaigns
      SET remaining_budget = remaining_budget - :cost
    WHERE id = :id AND status = 'active' AND remaining_budget >= :cost;
   -- 0 rows updated => budget exhausted, reject the reward
   ```
3. **Idempotency:** the unique constraint (entry_type, ref_id) means a view session can only
   ever be rewarded once, even if the client retries or two requests race.
4. **Withdrawals:** lock the user row (`SELECT ... FOR UPDATE`), check SUM(amount) >= requested,
   insert a negative `withdrawal` ledger entry, then call M-Pesa. On failure, insert a positive
   `withdrawal_reversal` entry. Never edit or delete ledger rows.
5. Change the schema only through new Alembic migrations (`alembic revision -m "..."`).
