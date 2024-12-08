-- Enable necessary extensions
create extension if not exists "uuid-ossp";

-- Profiles table (extends auth.users)
create table profiles (
  id uuid references auth.users on delete cascade not null primary key,
  email text not null,
  full_name text,
  avatar_url text,
  company_id uuid references companies(id),
  role text not null check (role in ('admin', 'client', 'team_member')),
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  constraint unique_email unique(email)
);

-- Companies table
create table companies (
  id uuid default uuid_generate_v4() primary key,
  name text not null,
  address jsonb,
  phone text,
  website text,
  industry text,
  integration_settings jsonb default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Proposals table
create table proposals (
  id uuid default uuid_generate_v4() primary key,
  company_id uuid references companies(id) not null,
  status text check (status in ('draft', 'review', 'approved', 'rejected')) default 'draft',
  version integer default 1,
  content jsonb not null,
  created_by uuid references auth.users not null,
  updated_by uuid references auth.users not null,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Contracts table
create table contracts (
  id uuid default uuid_generate_v4() primary key,
  proposal_id uuid references proposals(id) not null,
  company_id uuid references companies(id) not null,
  status text check (status in ('draft', 'sent', 'signed', 'active', 'completed')) default 'draft',
  content jsonb not null,
  signed_date timestamptz,
  signatures jsonb default '[]',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Payments table
create table payments (
  id uuid default uuid_generate_v4() primary key,
  contract_id uuid references contracts(id) not null,
  amount decimal not null,
  status text check (status in ('pending', 'processing', 'completed', 'failed')) default 'pending',
  payment_date timestamptz,
  payment_method jsonb,
  stripe_payment_id text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Project Assets table
create table project_assets (
  id uuid default uuid_generate_v4() primary key,
  contract_id uuid references contracts(id) not null,
  type text not null,
  status text check (status in ('pending', 'uploaded', 'approved', 'rejected')) default 'pending',
  storage_path text,
  metadata jsonb default '{}',
  uploaded_by uuid references auth.users not null,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Integration Logs table
create table integration_logs (
  id uuid default uuid_generate_v4() primary key,
  company_id uuid references companies(id) not null,
  integration_type text not null,
  event_type text not null,
  status text check (status in ('success', 'failed')) not null,
  details jsonb default '{}',
  created_at timestamptz default now()
);

-- Enable Row Level Security
alter table profiles enable row level security;
alter table companies enable row level security;
alter table proposals enable row level security;
alter table contracts enable row level security;
alter table payments enable row level security;
alter table project_assets enable row level security;
alter table integration_logs enable row level security;

-- RLS Policies

-- Profiles policies
create policy "Users can view own profile"
  on profiles for select
  using ( auth.uid() = id );

create policy "Admins can view all profiles"
  on profiles for select
  using ( auth.role() = 'admin' );

-- Companies policies
create policy "Company members can view their company"
  on companies for select
  using (
    exists (
      select 1 from profiles
      where profiles.company_id = companies.id
      and profiles.id = auth.uid()
    )
  );

-- Enable real-time subscriptions
alter publication supabase_realtime add table proposals;
alter publication supabase_realtime add table contracts;
alter publication supabase_realtime add table payments;

erDiagram
    profiles ||--|| auth.users : extends
    profiles ||--o{ companies : belongs_to
    companies ||--o{ proposals : has
    proposals ||--o{ contracts : generates
    contracts ||--o{ payments : receives
    contracts ||--o{ project_assets : contains
    companies ||--o{ integration_logs : generates
flowchart TD
    A[Client Signs Up] --> B[Create Client Profile]
    B --> C{Integration Setup}
    C --> |Google| D[Setup Google Workspace]
    C --> |Basecamp| E[Setup Basecamp]
    C --> |Other| F[Setup Additional Tools]
    
    D & E & F --> G[Initial Questionnaire]
    G --> H[Generate Proposal]
    H --> I[Client Review]
    I --> |Approved| J[Generate Contract]
    I --> |Revision Needed| H
    J --> K[E-Signature]
    K --> L[Invoice Generation]
    L --> M[Payment Processing]
    M --> N[Project Setup]
    N --> O[Client Onboarding]
