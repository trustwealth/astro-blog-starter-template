-- ============================================================
-- AETHER EXCHANGE - FULL SCHEMA
-- Cryptocurrency Exchange Database for Supabase
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ====================== PROFILES ======================
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE,
  display_name TEXT,
  avatar_url TEXT,
  email TEXT,
  phone TEXT,
  kyc_status TEXT DEFAULT 'none' CHECK (kyc_status IN ('none', 'pending', 'verified', 'rejected')),
  kyc_level INTEGER DEFAULT 0,
  referral_code TEXT UNIQUE DEFAULT encode(gen_random_bytes(6), 'hex'),
  referred_by UUID REFERENCES public.profiles(id),
  is_admin BOOLEAN DEFAULT FALSE,
  is_active BOOLEAN DEFAULT TRUE,
  two_fa_enabled BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== ASSETS ======================
CREATE TABLE public.assets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  symbol TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('crypto', 'fiat')),
  decimals INTEGER DEFAULT 8,
  is_active BOOLEAN DEFAULT TRUE,
  min_deposit NUMERIC(30,18) DEFAULT 0,
  min_withdrawal NUMERIC(30,18) DEFAULT 0,
  withdrawal_fee NUMERIC(30,18) DEFAULT 0,
  deposit_enabled BOOLEAN DEFAULT TRUE,
  withdrawal_enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== MARKETS ======================
CREATE TABLE public.markets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  base_asset_id UUID NOT NULL REFERENCES public.assets(id),
  quote_asset_id UUID NOT NULL REFERENCES public.assets(id),
  symbol TEXT UNIQUE NOT NULL,
  maker_fee NUMERIC(8,6) DEFAULT 0.001,
  taker_fee NUMERIC(8,6) DEFAULT 0.001,
  min_order_size NUMERIC(30,18) DEFAULT 0,
  tick_size NUMERIC(30,18) DEFAULT 0.01,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(base_asset_id, quote_asset_id)
);

-- ====================== WALLETS ======================
CREATE TABLE public.wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  asset_id UUID NOT NULL REFERENCES public.assets(id),
  available_balance NUMERIC(30,18) DEFAULT 0 CHECK (available_balance >= 0),
  locked_balance NUMERIC(30,18) DEFAULT 0 CHECK (locked_balance >= 0),
  total_balance NUMERIC(30,18) GENERATED ALWAYS AS (available_balance + locked_balance) STORED,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, asset_id)
);

-- ====================== ORDERS ======================
CREATE TABLE public.orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  market_id UUID NOT NULL REFERENCES public.markets(id),
  side TEXT NOT NULL CHECK (side IN ('buy', 'sell')),
  type TEXT NOT NULL CHECK (type IN ('limit', 'market', 'stop_limit', 'stop_market')),
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'partially_filled', 'filled', 'cancelled', 'rejected', 'expired')),
  price NUMERIC(30,18),
  stop_price NUMERIC(30,18),
  quantity NUMERIC(30,18) NOT NULL,
  filled_quantity NUMERIC(30,18) DEFAULT 0,
  remaining_quantity NUMERIC(30,18) GENERATED ALWAYS AS (quantity - filled_quantity) STORED,
  fee NUMERIC(30,18) DEFAULT 0,
  time_in_force TEXT DEFAULT 'GTC' CHECK (time_in_force IN ('GTC', 'IOC', 'FOK')),
  client_order_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== TRADES ======================
CREATE TABLE public.trades (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  market_id UUID NOT NULL REFERENCES public.markets(id),
  buy_order_id UUID REFERENCES public.orders(id),
  sell_order_id UUID REFERENCES public.orders(id),
  buyer_id UUID NOT NULL REFERENCES auth.users(id),
  seller_id UUID NOT NULL REFERENCES auth.users(id),
  price NUMERIC(30,18) NOT NULL,
  quantity NUMERIC(30,18) NOT NULL,
  buyer_fee NUMERIC(30,18) DEFAULT 0,
  seller_fee NUMERIC(30,18) DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== TRANSACTIONS ======================
CREATE TABLE public.transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  asset_id UUID NOT NULL REFERENCES public.assets(id),
  type TEXT NOT NULL CHECK (type IN ('deposit', 'withdrawal', 'trade_buy', 'trade_sell', 'fee', 'referral_reward', 'adjustment', 'lock', 'unlock')),
  amount NUMERIC(30,18) NOT NULL,
  balance_after NUMERIC(30,18),
  reference_id UUID,
  reference_type TEXT,
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== DEPOSITS & WITHDRAWALS ======================
CREATE TABLE public.deposits (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  asset_id UUID NOT NULL REFERENCES public.assets(id),
  amount NUMERIC(30,18) NOT NULL,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'confirming', 'completed', 'failed', 'cancelled')),
  tx_hash TEXT,
  address TEXT,
  network TEXT,
  confirmations INTEGER DEFAULT 0,
  required_confirmations INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

CREATE TABLE public.withdrawals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  asset_id UUID NOT NULL REFERENCES public.assets(id),
  amount NUMERIC(30,18) NOT NULL,
  fee NUMERIC(30,18) DEFAULT 0,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled', 'rejected')),
  tx_hash TEXT,
  address TEXT NOT NULL,
  network TEXT,
  memo TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  processed_at TIMESTAMPTZ
);

-- ====================== KYC, NOTIFICATIONS, REFERRALS, AUDIT, API KEYS ======================
CREATE TABLE public.kyc_submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'under_review', 'approved', 'rejected')),
  document_type TEXT,
  document_url TEXT,
  rejection_reason TEXT,
  reviewed_by UUID REFERENCES auth.users(id),
  submitted_at TIMESTAMPTZ DEFAULT NOW(),
  reviewed_at TIMESTAMPTZ
);

CREATE TABLE public.notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  type TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT,
  data JSONB,
  is_read BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.referrals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  referrer_id UUID NOT NULL REFERENCES auth.users(id),
  referred_id UUID NOT NULL REFERENCES auth.users(id) UNIQUE,
  commission_rate NUMERIC(8,6) DEFAULT 0.002,
  total_earned NUMERIC(30,18) DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  action TEXT NOT NULL,
  entity_type TEXT,
  entity_id UUID,
  old_data JSONB,
  new_data JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name TEXT,
  key_hash TEXT NOT NULL,
  permissions TEXT[] DEFAULT ARRAY['read'],
  is_active BOOLEAN DEFAULT TRUE,
  last_used_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ
);

CREATE TABLE public.deposit_addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  asset_id UUID NOT NULL REFERENCES public.assets(id),
  address TEXT NOT NULL,
  network TEXT,
  memo TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, asset_id, network),
  UNIQUE(address)
);

CREATE TABLE public.price_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  market_id UUID NOT NULL REFERENCES public.markets(id),
  target_price NUMERIC(30,18) NOT NULL,
  condition TEXT NOT NULL CHECK (condition IN ('above', 'below')),
  is_active BOOLEAN DEFAULT TRUE,
  triggered_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ====================== INDEXES ======================
CREATE INDEX idx_wallets_user ON public.wallets(user_id);
CREATE INDEX idx_orders_user_status ON public.orders(user_id, status);
CREATE INDEX idx_orders_market_status ON public.orders(market_id, status, price);
CREATE INDEX idx_trades_market_time ON public.trades(market_id, created_at DESC);
CREATE INDEX idx_transactions_user ON public.transactions(user_id, created_at DESC);
CREATE INDEX idx_notifications_user_unread ON public.notifications(user_id, is_read) WHERE is_read = FALSE;
CREATE INDEX idx_price_alerts_active ON public.price_alerts(market_id, is_active) WHERE is_active = TRUE;

-- ====================== UPDATED_AT TRIGGER ======================
CREATE OR REPLACE FUNCTION public.handle_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.profiles FOR EACH ROW EXECUTE FUNCTION public.handle_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.wallets FOR EACH ROW EXECUTE FUNCTION public.handle_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.orders FOR EACH ROW EXECUTE FUNCTION public.handle_updated_at();

-- ====================== RLS ======================
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.wallets ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trades ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.deposits ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.withdrawals ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.kyc_submissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.deposit_addresses ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.price_alerts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile" ON public.profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON public.profiles FOR UPDATE USING (auth.uid() = id);
CREATE POLICY "Users can view own wallets" ON public.wallets FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can view own orders" ON public.orders FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own orders" ON public.orders FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own open orders" ON public.orders FOR UPDATE USING (auth.uid() = user_id AND status IN ('open', 'partially_filled'));
CREATE POLICY "Users can view own trades" ON public.trades FOR SELECT USING (auth.uid() = buyer_id OR auth.uid() = seller_id);
CREATE POLICY "Users can view own transactions" ON public.transactions FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can view own deposits" ON public.deposits FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can view own withdrawals" ON public.withdrawals FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can view own notifications" ON public.notifications FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can update own notifications" ON public.notifications FOR UPDATE USING (auth.uid() = user_id);
CREATE POLICY "Users can view own kyc" ON public.kyc_submissions FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own kyc" ON public.kyc_submissions FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can manage own api keys" ON public.api_keys FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "Users can view own deposit addresses" ON public.deposit_addresses FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can manage own price alerts" ON public.price_alerts FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "Anyone can view assets" ON public.assets FOR SELECT USING (true);
CREATE POLICY "Anyone can view markets" ON public.markets FOR SELECT USING (true);

-- ====================== SEED DATA ======================
INSERT INTO public.assets (symbol, name, type, decimals) VALUES
  ('BTC', 'Bitcoin', 'crypto', 8),
  ('ETH', 'Ethereum', 'crypto', 8),
  ('USDT', 'Tether', 'crypto', 6),
  ('USD', 'US Dollar', 'fiat', 2)
ON CONFLICT (symbol) DO NOTHING;