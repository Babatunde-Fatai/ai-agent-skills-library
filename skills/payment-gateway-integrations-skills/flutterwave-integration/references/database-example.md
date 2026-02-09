CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  tx_ref VARCHAR(255) UNIQUE NOT NULL,
  amount INTEGER NOT NULL, -- smallest unit
  currency VARCHAR(10) NOT NULL,
  status VARCHAR(20) NOT NULL,
  transaction_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);
