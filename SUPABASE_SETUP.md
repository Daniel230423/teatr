# Подключение Supabase

## 1. Создайте проект

1. Создайте проект в [Supabase](https://supabase.com/dashboard).
2. В разделе **Project Settings → API** скопируйте Project URL и publishable key (или legacy `anon` key).
3. Впишите их в `supabase-config.js`:

```js
window.SUPABASE_CONFIG = {
  url: "https://YOUR_PROJECT_REF.supabase.co",
  anonKey: "YOUR_SUPABASE_PUBLISHABLE_OR_ANON_KEY"
};
```

Ключ `anon`/publishable предназначен для браузера. Никогда не добавляйте в сайт `service_role` key.

## 2. Создайте таблицу

Откройте **SQL Editor** в панели Supabase и выполните:

```sql
create table public.inventory_items (
  id text primary key,
  title text not null,
  article text not null default '',
  category text not null,
  status text not null default 'В наличии',
  booked_by text not null default '',
  tags jsonb not null default '[]'::jsonb,
  description text not null default '',
  images jsonb not null default '[]'::jsonb,
  variants jsonb not null default '[]'::jsonb,
  is_draft boolean not null default false,
  created_at timestamptz not null default now()
);

alter table public.inventory_items enable row level security;
grant select on public.inventory_items to anon, authenticated;
grant insert, update, delete on public.inventory_items to authenticated;

create policy "Anyone can view inventory"
on public.inventory_items for select
to anon, authenticated
using (true);

create policy "Employees can add inventory"
on public.inventory_items for insert
to authenticated
with check (auth.uid() is not null);

create policy "Employees can update inventory"
on public.inventory_items for update
to authenticated
using (auth.uid() is not null)
with check (auth.uid() is not null);

create policy "Employees can delete inventory"
on public.inventory_items for delete
to authenticated
using (auth.uid() is not null);
```

Карточки хранятся в `inventory_items`; варианты вложены в JSONB-поле `variants`. Черновики хранятся в той же таблице с `is_draft = true`: они видны сотрудникам в разделе МТО, но не отображаются в публичном каталоге.

Если таблица `inventory_items` уже создана, добавьте поле для черновиков:

```sql
alter table public.inventory_items
add column if not exists is_draft boolean not null default false;

notify pgrst, 'reload schema';
```

## 3. Настройте фотографии

Создайте в **Storage** публичный bucket с точным именем `inventory-photos`. Публичность нужна для отображения фотографий посетителям каталога. Затем в SQL Editor разрешите авторизованным сотрудникам загружать файлы:

```sql
drop policy if exists "Employees can upload inventory photos" on storage.objects;

create policy "Employees can upload inventory photos"
on storage.objects for insert
to authenticated
with check (bucket_id = 'inventory-photos');
```

Если при загрузке появляется `new row violates row-level security policy`, проверьте, что SQL выше выполнен именно в проекте, указанном в `supabase-config.js`, bucket называется ровно `inventory-photos`, а сотрудник вошёл в раздел МТО. Политика разрешает вставку только вошедшим пользователям и только в этот bucket; не заменяйте её политикой `to anon` или разрешением для всех.

Проверить наличие политики можно запросом:

```sql
select policyname, roles, cmd, with_check
from pg_policies
where schemaname = 'storage'
  and tablename = 'objects'
  and policyname = 'Employees can upload inventory photos';
```

Загрузка фотографий через сайт доступна только после входа. Настройте допустимый размер файла в настройках bucket согласно потребностям проекта.

## 4. Создайте учётные записи сотрудников

В **Authentication → Users** добавьте сотрудникам пользователей с email и паролем. В настройках **Authentication → Providers → Email** отключите публичную регистрацию, если создавать учётные записи должны только администраторы. Форма «Для МТО» входит через email и пароль Supabase Auth; прежние демо-данные `admin / teatr123` больше не используются.

Все созданные пользователи Supabase Auth получают права изменения каталога по политикам выше. Не создавайте пользователей, которым нельзя разрешать редактирование.

## 5. Запустите сайт

Сайт нужно открывать по HTTP(S), а не напрямую через `file://`. Из папки проекта запустите локальный сервер:

```bash
python3 -m http.server 8000
```

Откройте `http://localhost:8000`. Для публикации добавьте адрес сайта в **Authentication → URL Configuration → Site URL** и разрешённые redirect URLs. Каталог доступен для чтения всем посетителям, редактирование и загрузка фото требуют входа.

## Перенос данных из браузера

Старые записи `kulisy_inventory_v1` остаются в localStorage браузера, где они были созданы. Настройте Supabase и войдите на сайте в МТО именно из этого браузера. Если таблица пуста, появится кнопка **«Перенести данные из браузера»**. Она перенесёт карточки и варианты в таблицу, а изображения загрузит в Storage. Исходные данные в localStorage не удаляются. Импорт показывается только для пустой таблицы, чтобы не перезаписать общий каталог.