# Supabase Swift Example

A SwiftUI app, showcasing usage of Supabase with examples of using auth, database, edge-functions, storage and realtime.

## Introduction

This example app demonstrates how to integrate Supabase with a SwiftUI application. It covers several key Supabase features:

- **Authentication**, to sign up and sign in users with email and password.
- **Database** to handle showing list of accounts and showing and operating on the list of transactions associated to an account.
- **Edge Functions** to make a transfer, it actually create 2 transactions on sender and recipient accounts.
- **Storage** to store user's profile photo.
- **Realtime** to track changes in the list of user's transactions.

## Prerequisites

Before running this example, you'll need:

1. A Supabase account (sign up at [supabase.com](https://supabase.com) if you don't have one)
2. A Supabase project created in your account
3. Configuration of your Supabase project details in the app

## Setup

1. Clone this repository
2. Open the project in Xcode
3. Locate the `Supabase.xcconfig` file and update it with your Supabase project credentials from [Data API Configurations](https://supabase.com/dashboard/project/_/settings/api):
   ```
   SUPABASE_PROJECT_ID=<your-project-id>
   SUPABASE_ANON_KEY=<your-anon-key>
   ```
4. Make sure your Supabase project has the necessary tables and configurations described in the [Models.swift](/SupabaseIntro/Models.swift) file. \
There are also example data provided. You can run all SQLs in the Supabase SQL Editor.
5. Use Supabase [Local Development & CLI](https://supabase.com/docs/guides/local-development) document to install Supabase CLI on your machine.
6. Use [Deploy to Production](https://supabase.com/docs/guides/functions/deploy) guide from edge functions to deploy transfer function to your Supabase project.
7. You will need to define a Postgres function as described in [transfer function](/supabase/functions/transfer/index.ts#l62) in order to find transfer's recipient user by their email address. (This is required because Supabase does not allow you to run queries on protected schemas, including auth schema.)
8. Create a public bucket named "avatars".
9. As per Supabase's requirements, you need to setup RLS policy for the said bucket to control access to the files in the bucket. Create a policy for the avatars bucket, select all operations to allow all. Use the following as the policy definition.
    ```sql
    (bucket_id = 'avatars') AND (auth.role() = 'authenticated')
    ```
10. Lastly, make sure to use [RLS policies](https://supabase.com/docs/guides/database/postgres/row-level-security) and other security measurements to keep access to your supabase project safe. Always check [security advisor](https://supabase.com/dashboard/project/_/advisors/security) for warnings and errors.

You are now ready to use the App with the Supabase.

## Supabase Codes

For the examples on how to use supabase features, see the following files.

### Database

The following files, and the specified functions, contain simple examples on how to use databases in Supabase.

#### [TransactionList.swift](/SupabaseIntro/TransactionList.swift)

```swift
class TransactionList: TransactionListProtocol {
    var supabase: SupabaseClient

    // ...

    func fetchTransactions() async throws

    func insertTransaction(_ transaction: Transaction) async throws 

    func updateTransaction(_ transaction: Transaction, id: UUID) async throws

    func deleteTransaction(_ transaction: Transaction) async throws 

    // ...
}
```

#### [AccountOverview.swift](/SupabaseIntro/AccountOverview.swift)

```swift
class AccountOverview: AccountOverviewProtocol {
    var supabase: SupabaseClient

    // ...

    func fetchAccounts() async throws

    // ...
}
```

### Edge Function

#### [TransactionList.swift](/SupabaseIntro/TransactionList.swift)

```swift
class TransactionList: TransactionListProtocol {
    var supabase: SupabaseClient

    // ...

    func makeTransfer(_ transfer: TransferRequest) async throws

    // ...
}
```

### Storage

#### [AccountOverview.swift](/SupabaseIntro/AccountOverview.swift)

```swift
class AccountOverview: AccountOverviewProtocol {
    var supabase: SupabaseClient

    // ...

    func fetchAvatar() async throws

    func saveAvatar(_ image: AvatarImage) async throws
}
```

### Realtime

#### [TransactionList.swift](/SupabaseIntro/TransactionList.swift)

```swift
class TransactionList: TransactionListProtocol {
    var supabase: SupabaseClient

    // ...

    func observeChanges() async throws
}
```

### Auth

#### [LoginView.swift](/SupabaseIntro/LoginView.swift)

```swift
struct LoginView: View {
    // ...

    func signIn() async
    
    func signUp() async
}
```

### FQA
Supabase manages auth.users internally through its authentication system.

So you can’t insert directly into auth.users, but you can fetch a user that signed up via Supabase Auth (email, magic link, etc.).

If you want to bypass auth temporarily for testing, create a dummy users table instead of using auth.users like this:

```sql
-- 1. Create ENUM types
CREATE TYPE transaction_type AS ENUM ('credit', 'debit');
CREATE TYPE account_status AS ENUM ('open', 'restricted', 'closed');

-- 2. Create 'accounts' table
CREATE TABLE public.accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status account_status NOT NULL,
    CONSTRAINT accounts_user_id_fkey FOREIGN KEY (user_id) REFERENCES auth.users (id) ON DELETE CASCADE
);

-- 3. Create 'transactions' table
CREATE TABLE public.transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    account_id UUID NOT NULL,
    type transaction_type NOT NULL,
    amount NUMERIC(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    category TEXT NOT NULL,
    description TEXT,
    date TIMESTAMPTZ NOT NULL,
    CONSTRAINT transactions_account_id_fkey FOREIGN KEY (account_id) REFERENCES public.accounts(id) ON DELETE CASCADE
);

-- 4. Disable RLS (Row-Level Security)
ALTER TABLE public.accounts DISABLE ROW LEVEL SECURITY;
ALTER TABLE public.transactions DISABLE ROW LEVEL SECURITY;

-- 5. Insert sample account (replace user_id with a real user from auth.users)
-- Run this query to get a user_id from your auth.users table:
-- SELECT id FROM auth.users LIMIT 1;
-- Then replace the value below:

INSERT INTO public.accounts (id, user_id, created_at, status)
VALUES (
    '11111111-1111-1111-1111-111111111111',
    '7479a9f3-8e53-4054-a956-ce9440d03af1',
    now(),
    'open'
);

-- 6. Insert sample transactions (associated with the account above)
INSERT INTO public.transactions (id, account_id, type, amount, currency, category, description, date)
VALUES 
(
    'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa',
    '11111111-1111-1111-1111-111111111111',
    'credit',
    150.00,
    'USD',
    'Salary',
    'Monthly salary deposit',
    now()
),
(
    'bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb',
    '11111111-1111-1111-1111-111111111111',
    'debit',
    50.00,
    'USD',
    'Groceries',
    'Weekly grocery shopping',
    now()
),
(
    'cccccccc-cccc-cccc-cccc-cccccccccccc',
    '11111111-1111-1111-1111-111111111111',
    'debit',
    30.00,
    'USD',
    'Transport',
    'Monthly bus pass',
    now()
);
```
