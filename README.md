-- ══════════════════════════════════════════════════════════
-- TransferLog · SQL de configuración para el panel admin
-- Ejecuta esto UNA VEZ en el SQL Editor de Supabase
-- ══════════════════════════════════════════════════════════

-- 1. Tabla de perfiles (rol por usuario)
create table if not exists profiles (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid references auth.users(id) on delete cascade,
  email       text,
  rol         text default 'operador',  -- 'operador' | 'supervisor' | 'admin'
  created_at  timestamptz default now(),
  unique(user_id)
);

-- 2. Habilitar RLS en profiles
alter table profiles enable row level security;

-- 3. Políticas: el usuario ve su propio perfil; admin ve todos
drop policy if exists "own profile"   on profiles;
drop policy if exists "admin profiles" on profiles;
create policy "own profile"    on profiles for select  using (auth.uid() = user_id);
create policy "admin profiles" on profiles for all     using (true);  -- reemplazar con check de rol si quieres más restricción

-- 4. Trigger: crear perfil automáticamente al registrar usuario
create or replace function handle_new_user()
returns trigger language plpgsql security definer as $$
begin
  insert into profiles (user_id, email, rol)
  values (new.id, new.email, 'operador')
  on conflict (user_id) do nothing;
  return new;
end;
$$;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();

-- 5. Asegura que receipts y routes tengan user_id con RLS correcta
alter table routes   add column if not exists user_id uuid references auth.users(id);
alter table receipts add column if not exists user_id uuid references auth.users(id);

alter table routes   enable row level security;
alter table receipts enable row level security;

drop policy if exists "user_routes"   on routes;
drop policy if exists "user_receipts" on receipts;
create policy "user_routes"   on routes   for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "user_receipts" on receipts for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- 6. Marcar manualmente a tu primer admin (cambia el email)
-- update profiles set rol = 'admin' where email = 'tuadmin@ejemplo.com';
