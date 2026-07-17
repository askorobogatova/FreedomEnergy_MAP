# CRM Map — карта клієнтів KeyCRM + Google Maps

Це переписана версія вашого проєкту з Google Apps Script на звичайний
Node.js-застосунок, який можна тримати в GitHub і розгортати на
безкоштовному хостингу — без OAuth-запитів доступу до акаунту користувачів.

## Що змінилось порівняно з версією на Apps Script

| Було (Apps Script)                      | Стало (Node.js)                                   |
|------------------------------------------|----------------------------------------------------|
| `Code.js` виконується в Google-хмарі      | `server/` — звичайний Express-сервер               |
| Ключі в Script Properties                 | Ключі в змінних середовища (`.env`), сервер їх ніколи не віддає у фронтенд, крім Maps JS-ключа |
| Google Таблиця як міні-БД                 | SQLite-файл (`data/crm-map.db`), таблиці `built_objects` та `potential_clients` |
| `Session.getActiveUser()` + аркуш `AccessRights` (ролі admin/regional/manager) | Прибрано. Весь застосунок закритий одним спільним логіном/паролем (HTTP Basic Auth) — посилання доступне лише команді |
| `google.script.run.<fn>()` у фронтенді    | Звичайні `fetch('/api/...')`                       |
| `CacheService` (кеш геокодування на 24 год)| SQLite-кеш геокодування (постійний) + in-memory кеш карток/замовлень |

**Важливо:** рольовий доступ (admin/regional/manager), який фільтрував дані
по регіону чи менеджеру, я прибрав повністю — ви обрали варіант "без входу,
доступ лише по посиланню з паролем". Якщо він насправді був потрібен,
скажіть — можна повернути (проста форма логіну + таблиця користувачів
замість Google-акаунтів).

## Структура проєкту

```
crm-map/
├── server/
│   ├── index.js        — точка входу, Basic Auth, роздача статики
│   ├── config.js        — кастомні поля KeyCRM, кольори статусів (як CUSTOM_FIELDS у Code.js)
│   ├── db.js             — SQLite (заміна Google Таблиці)
│   ├── geocode.js         — геокодування адрес (як geocodeAddress_ у Code.js)
│   ├── keycrm.js           — запити до KeyCRM API + мапінг даних (як getMapData у Code.js)
│   └── routes/api.js        — REST-ендпоінти (заміна google.script.run)
├── public/
│   ├── index.html             — розмітка (як Index.html)
│   ├── style.css                — стилі (винесені з <style> в Index.html)
│   └── app.js                     — логіка карти/фільтрів/модалок (заміна google.script.run на fetch)
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

## Локальний запуск

```bash
npm install
cp .env.example .env
# відредагуйте .env — впишіть KEYCRM_API_KEY, GOOGLE_MAPS_KEY, BASIC_AUTH_USER/PASS
npm start
```

Відкрийте `http://localhost:3000` — браузер запитає логін/пароль
(значення з `BASIC_AUTH_USER` / `BASIC_AUTH_PASS`).

## Розгортання на Render (найпростіший безкоштовний варіант)

1. Створіть репозиторій на GitHub і запуште туди весь проєкт (окрім `.env` —
   він і так у `.gitignore`).
2. На [render.com](https://render.com) → **New → Web Service** → підключіть
   цей GitHub-репозиторій.
3. Налаштування:
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
   - **Instance Type:** Free вистачить для початку.
4. У розділі **Environment** додайте змінні (ті самі, що в `.env.example`):
   `KEYCRM_API_KEY`, `GOOGLE_MAPS_KEY`, `GOOGLE_GEOCODING_KEY`,
   `BASIC_AUTH_USER`, `BASIC_AUTH_PASS`, і за бажанням
   `KEYCRM_CARD_URL_TEMPLATE`.
5. **Важливо для SQLite на Render:** файлова система на безкоштовному тарифі
   не постійна між деплоями. Щоб дані (побудовані об'єкти, потенційні
   клієнти) не губились при кожному оновленні коду, додайте **Persistent
   Disk** (Render → Disks, ~1 GB безкоштовно на платних інстансах; на
   безкоштовному тарифі диска немає — тоді SQLite-файл буде скидатись при
   кожному релізі коду, хоч і зберігатиметься між звичайними
   рестартами/сном). Якщо це критично з першого дня — напишіть, замінимо
   SQLite на безкоштовний Postgres (Render/Supabase/Neon), там персистентність
   гарантована незалежно від тарифу.
6. Після деплою Render видасть URL на кшталт
   `https://crm-map.onrender.com` — саме ним ви ділитесь з командою.
   Безкоштовний інстанс "засинає" після ~15 хв бездіяльності і перше
   відкриття після сну займає кілька секунд.

## Обмеження Google Maps ключа

Обов'язково зайдіть у Google Cloud Console → Credentials → ваш ключ і
додайте **HTTP referrer restriction** з доменом Render (`*.onrender.com`
або ваш власний домен), щоб ключ не можна було використати з інших сайтів,
навіть якщо його побачать у мережевих запитах браузера (Maps JS API ключ
завжди видно клієнту — це нормально для Google Maps, головне обмежити
домени використання).

## Речі, які варто перевірити/доробити

- **`KEYCRM_CARD_URL_TEMPLATE`** — я не знаю точний шаблон посилання на
  картку угоди у вашому акаунті KeyCRM. Якщо залишити порожнім, кнопка
  "Відкрити картку в CRM" просто не з'явиться. Вкажіть реальний шаблон
  (наприклад `https://app.keycrm.app/pipeline/card/{id}`) — `{id}` буде
  замінено на ID картки.
- **`isOverdue` (прострочені задачі)** — у вашому оригінальному `Code.js`
  ця логіка так і не була реалізована (поле завжди `false`), я переніс це
  1:1. Якщо потрібно підсвічувати прострочені задачі — потрібна окрема
  логіка через `/pipelines/cards/{id}/tasks` чи подібний ендпоінт KeyCRM.
- **`amount` для замовлень** — я припустив поля `grand_total`/`total_amount`
  з відповіді KeyCRM `/order`; перевірте назву поля через
  `GET /api/discover-fields` і поправте в `server/keycrm.js`, якщо треба.
