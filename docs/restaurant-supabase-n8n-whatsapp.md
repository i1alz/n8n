# Restaurant DB for Supabase + n8n + WhatsApp

## 1) SQL Schema (clean + practical)

```sql
-- Recommended for Supabase/Postgres
create extension if not exists pgcrypto;

-- 1) branches
create table if not exists public.branches (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  address text,
  phone text,
  is_active boolean not null default true,
  created_at timestamptz not null default now()
);

-- 2) categories
create table if not exists public.categories (
  id uuid primary key default gen_random_uuid(),
  name_ar text not null,
  name_en text,
  sort_order integer not null default 0,
  is_active boolean not null default true,
  created_at timestamptz not null default now()
);

-- 3) menu_items
create table if not exists public.menu_items (
  id uuid primary key default gen_random_uuid(),
  category_id uuid not null references public.categories(id) on update cascade on delete restrict,
  name_ar text not null,
  name_en text,
  description text,
  base_price numeric(10,2) not null check (base_price >= 0),
  is_available boolean not null default true,
  created_at timestamptz not null default now()
);

-- 4) branch_menu_prices
create table if not exists public.branch_menu_prices (
  id uuid primary key default gen_random_uuid(),
  branch_id uuid not null references public.branches(id) on update cascade on delete cascade,
  item_id uuid not null references public.menu_items(id) on update cascade on delete cascade,
  price numeric(10,2) not null check (price >= 0),
  is_available boolean not null default true,
  created_at timestamptz not null default now(),
  unique (branch_id, item_id)
);

-- Helpful indexes
create index if not exists idx_categories_sort_order on public.categories(sort_order);
create index if not exists idx_menu_items_category_id on public.menu_items(category_id);
create index if not exists idx_menu_items_name_ar on public.menu_items(name_ar);
create index if not exists idx_branch_menu_prices_branch_id on public.branch_menu_prices(branch_id);
create index if not exists idx_branch_menu_prices_item_id on public.branch_menu_prices(item_id);

```

## 2) Seed Data (مشاوي، مقبلات، مشروبات + 8 أصناف عربية)

```sql
-- Branches
insert into public.branches (name, address, phone, is_active)
values
  ('فرع الرياض - العليا', 'الرياض، حي العليا، شارع التحلية', '+966500000001', true),
  ('فرع جدة - الزهراء', 'جدة، حي الزهراء، شارع الأمير سلطان', '+966500000002', true)
on conflict do nothing;

-- Categories
insert into public.categories (name_ar, name_en, sort_order, is_active)
values
  ('مشاوي', 'Grills', 1, true),
  ('مقبلات', 'Appetizers', 2, true),
  ('مشروبات', 'Drinks', 3, true)
on conflict do nothing;

-- 8 Arabic menu items
with c as (
  select id, name_ar from public.categories
)
insert into public.menu_items (category_id, name_ar, name_en, description, base_price, is_available)
values
  ((select id from c where name_ar = 'مشاوي'), 'كباب لحم', 'Beef Kebab', 'سيخ كباب لحم مشوي على الفحم', 42.00, true),
  ((select id from c where name_ar = 'مشاوي'), 'شيش طاووق', 'Shish Tawook', 'قطع دجاج متبلة ومشوية', 36.00, true),
  ((select id from c where name_ar = 'مشاوي'), 'مطول', 'Lamb Skewer', 'لحم ضأن متبل ومشوي', 48.00, true),
  ((select id from c where name_ar = 'مقبلات'), 'حمص', 'Hummus', 'حمص بالطحينة وزيت الزيتون', 16.00, true),
  ((select id from c where name_ar = 'مقبلات'), 'متبل', 'Moutabal', 'باذنجان مشوي مع الطحينة', 17.00, true),
  ((select id from c where name_ar = 'مقبلات'), 'ورق عنب', 'Stuffed Grape Leaves', 'ورق عنب محشي بالأرز', 19.00, true),
  ((select id from c where name_ar = 'مشروبات'), 'عصير برتقال', 'Orange Juice', 'عصير برتقال طازج', 14.00, true),
  ((select id from c where name_ar = 'مشروبات'), 'مياه معدنية', 'Mineral Water', 'زجاجة مياه معدنية 330 مل', 5.00, true)
on conflict do nothing;

-- Branch-specific prices (example override)
insert into public.branch_menu_prices (branch_id, item_id, price, is_available)
select
  b.id,
  i.id,
  case
    when b.name like '%الرياض%' then i.base_price
    when b.name like '%جدة%' then round((i.base_price * 1.05)::numeric, 2)
    else i.base_price
  end as price,
  true
from public.branches b
cross join public.menu_items i
on conflict (branch_id, item_id) do nothing;
```

## 3) View: `menu_view`

```sql
create or replace view public.menu_view as
select
  b.name as branch_name,
  c.name_ar as category_name,
  i.name_ar as item_name,
  bmp.price,
  i.description,
  (i.is_available and bmp.is_available and b.is_active and c.is_active) as is_available
from public.branch_menu_prices bmp
join public.branches b on b.id = bmp.branch_id
join public.menu_items i on i.id = bmp.item_id
join public.categories c on c.id = i.category_id;
```

## 4) Ready Queries for n8n

> Use with Supabase Postgres node (Execute Query)

```sql
-- A) Get all categories
select id, name_ar, name_en, sort_order
from public.categories
where is_active = true
order by sort_order asc, name_ar asc;
```

```sql
-- B) Get items by category (Arabic category name)
-- n8n param example: {{ $json.category_name }}
select
  i.id,
  i.name_ar,
  i.name_en,
  i.description,
  i.base_price,
  i.is_available
from public.menu_items i
join public.categories c on c.id = i.category_id
where c.is_active = true
  and i.is_available = true
  and c.name_ar = $1
order by i.name_ar;
```

```sql
-- C) Get items by branch (from menu_view)
-- n8n param example: {{ $json.branch_name }}
select
  branch_name,
  category_name,
  item_name,
  price,
  description,
  is_available
from public.menu_view
where branch_name = $1
  and is_available = true
order by category_name, item_name;
```

```sql
-- D) Search item by Arabic name (partial search)
-- n8n param example: {{ $json.search_text }}
select
  branch_name,
  category_name,
  item_name,
  price,
  description,
  is_available
from public.menu_view
where item_name ilike '%' || $1 || '%'
order by item_name, branch_name;
```

## 5) WhatsApp Bot JSON Response Examples (Arabic)

```json
{
  "type": "text",
  "message": "هذه قائمة المشاوي:\n- كباب لحم: 42 ريال\n- شيش طاووق: 36 ريال\n- مطول: 48 ريال"
}
```

```json
{
  "type": "text",
  "message": "هذه قائمة المقبلات:\n- حمص: 16 ريال\n- متبل: 17 ريال\n- ورق عنب: 19 ريال"
}
```

```json
{
  "type": "text",
  "message": "سعر المطول هو 48 ريال في فرع الرياض - العليا."
}
```

## 6) Optional JSON shape for n8n standardized output

```json
{
  "ok": true,
  "intent": "get_category_items",
  "category": "مشاوي",
  "branch": "فرع الرياض - العليا",
  "items": [
    { "name": "كباب لحم", "price": 42 },
    { "name": "شيش طاووق", "price": 36 },
    { "name": "مطول", "price": 48 }
  ]
}
```
