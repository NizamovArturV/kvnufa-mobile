# Настройка мобильного приложения — серверная часть

Операционная инструкция: **что и где настроить**, чтобы работали пуши, deep links и
принудительное обновление. Продуктовый/архитектурный план — в [mobile-app.md](mobile-app.md).

Где что лежит:

| Что | Где |
|---|---|
| Настройки (все секреты и версии) | Bitrix admin → **Настройки сайта** → страница «Мобильное приложение» (sprint.options) |
| Модуль приложения | `local/modules/avn.kvn.mobile/` (namespace `Avn\Kvn\Mobile`) |
| Таблица устройств | `avn_kvn_mobile_device` |
| Таблица регистраций Apple pass | `avn_kvn_pass_device` |
| Миграция (регистрация модуля + таблица) | `local/php_interface/migrations/InstallMobilePushDeviceTable20261013130000.php` |
| Миграция (Wallet: таблица регистраций pass) | `local/php_interface/migrations/InstallWalletPassDeviceTable20261013140000.php` |

---

## 1. Установка на сайте

Модуль регистрируется и таблица создаётся миграцией. Выполнить в корне проекта:

```powershell
php local/modules/sprint.migration/tools/migrate.php up InstallMobilePushDeviceTable20261013130000
php local/modules/sprint.migration/tools/migrate.php up InstallWalletPassDeviceTable20261013140000
```

Проверить, что миграция применилась:

```powershell
php local/modules/sprint.migration/tools/migrate.php ls
```

Откат (удаляет таблицу и снимает модуль):

```powershell
php local/modules/sprint.migration/tools/migrate.php down InstallMobilePushDeviceTable20261013130000
```

> Если модуль уже установлен штатно через админку «Marketplace/Модули», миграция просто
> пропустит регистрацию и создаст таблицу.

## 2. Где настраивать

Bitrix admin → **Настройки → Настройки сайта → «Мобильное приложение»**.
Пока не заполнены креды хотя бы одного провайдера, канал `push` считается ненастроенным
и уведомления по нему не уходят (работает fallback `push → email`).

---

## 3. iOS — APNs (push)

Вкладка **«iOS (APNs)»**.

Что сделать в Apple Developer (один раз):

1. **Keys → +** → включить «Apple Push Notifications service (APNs)» → скачать ключ `.p8`
   (скачивается **один раз**). Записать **Key ID**.
2. Из **Membership details** взять **Team ID**.
3. **Bundle ID** (Identifier) приложения — он же topic.

Заполнить опции:

| Опция | Значение |
|---|---|
| `MOBILE_APNS_KEY` | Содержимое `.p8` (вставить текстом) **или** абсолютный путь к файлу |
| `MOBILE_APNS_KEY_ID` | Key ID ключа `.p8` |
| `MOBILE_APNS_TEAM_ID` | Team ID |
| `MOBILE_APNS_TOPIC` | Bundle Identifier приложения (напр. `ru.kvnufa.app`) |
| `MOBILE_APNS_ENV` | `production` — сборки App Store/TestFlight; `sandbox` — сборки из Xcode |

> **Важно.** `production` не доставляет на debug-сборку и наоборот. Для TestFlight — `production`.
> Team ID и Bundle ID отсюда же используются для Universal Links (файл AASA, см. §6).

## 4. Android — RuStore (push)

Вкладка **«Android (RuStore / FCM)»**.

Что сделать в консоли RuStore (один раз):

1. Учётная запись разработчика RuStore → страница приложения → **Push-уведомления → Проекты**.
2. Создать проект, взять **Project ID** и **Service token**.

Заполнить опции:

| Опция | Значение |
|---|---|
| `MOBILE_RUSTORE_PROJECT_ID` | Project ID проекта пушей |
| `MOBILE_RUSTORE_SERVICE_TOKEN` | Сервисный токен (уходит в `Authorization: Bearer`) |

Доставка идёт на `POST https://vkpns.rustore.ru/v1/projects/{projectId}/messages:send`
(контракт drop-in FCM; сверено с официальной документацией RuStore).

## 5. Android — FCM / Google Play (push)

Та же вкладка. Нужен, если приложение публикуется в Google Play.

Что сделать во Firebase (один раз):

1. Firebase Console → проект → **Project settings → Service accounts** → **Generate new private key**.
2. Скачанный JSON — это сервис-аккаунт.

Заполнить опции:

| Опция | Значение |
|---|---|
| `MOBILE_FCM_SERVICE_ACCOUNT` | Содержимое JSON сервис-аккаунта **или** абсолютный путь к файлу |
| `MOBILE_FCM_PROJECT_ID` | Оставить пустым, если `project_id` есть в JSON |

> На одной платформе `android` сосуществуют RuStore- и FCM-токены. Провайдер выбирается по коду,
> который устройство сообщило при регистрации (`provider=rustore` / `provider=fcm`), — токены
> не перепутываются.

## 6. Deep links (универсальные ссылки)

Вкладка **«Deep links / домены»**. Файлы отдаются динамически из настроек (в репозиторий
идентификаторы не коммитятся):

| URL | Назначение | Источник данных |
|---|---|---|
| `/.well-known/apple-app-site-association` | Universal Links (iOS) | `MOBILE_APNS_TEAM_ID` + `MOBILE_APNS_TOPIC` |
| `/.well-known/assetlinks.json` | App Links (Android) | опции ниже |

Заполнить:

| Опция | Значение |
|---|---|
| `MOBILE_ANDROID_PACKAGE_ID` | `applicationId` приложения (напр. `ru.kvnufa.app`) |
| `MOBILE_ANDROID_SHA256_FINGERPRINTS` | SHA-256 отпечатки ключа подписи через запятую (debug / release / Play App Signing) |

> iOS: чтобы ссылки открывались в приложении, в Xcode включить **Associated Domains**
> (`applinks:<домен>`). Android: в манифесте — intent-filter с `autoVerify="true"`.

## 7. Обновление приложения (forced update)

Вкладка **«Обновление приложения»**. Приложение при старте дёргает `GET /api/mobile/config/`
и само сравнивает версии:

| Опция | Значение |
|---|---|
| `MOBILE_APP_MIN_VERSION` | Ниже этой версии — **блокирующее** обновление. Пусто — блокировки нет |
| `MOBILE_APP_LATEST_VERSION` | Ниже этой — **необязательное** предложение обновиться |
| `MOBILE_APP_IOS_STORE_URL` | Ссылка на App Store |
| `MOBILE_APP_ANDROID_STORE_URL` | Ссылка на RuStore / Google Play |
| `MOBILE_APP_UPDATE_MESSAGE` | Необязательный текст в окне обновления |

Порядок выката новой версии: опубликовать сборку → поднять `MOBILE_APP_LATEST_VERSION` (мягко) →
через некоторое время поднять `MOBILE_APP_MIN_VERSION` (жёстко).

## 8. Wallet-карта КВНщика (Apple / Google)

Карта обновляется автоматически при смене уровня участника
(`AwardAchievementToMember` → `MemberPassService::sync`). Отдаётся публично:

| URL | Назначение |
|---|---|
| `GET /wallet/apple/pass/{serial}/` | Скачать Apple `.pkpass` (ссылка «Добавить в Apple Wallet») |
| `GET /wallet/google/{serial}/` | Редирект в Google Wallet («Добавить в Google Wallet») |
| `POST/DELETE /wallet/apple/v1/devices/{device}/registrations/{passType}/{serial}/` | Web-service Apple (регистрация/снятие устройства) |
| `GET /wallet/apple/v1/devices/{device}/registrations/{passType}/` | Web-service Apple (список регистраций) |
| `GET /wallet/apple/v1/passes/{passType}/{serial}/` | Web-service Apple (перегенерация pass) |

`serial` = `Member.guid` — неугадываемое значение. Apple-эндпоинты защищены токеном pass'а
(заголовок `Authorization: ApplePass <token>`).

> **Кнопки «Добавить в Wallet»** выводятся в профиле, в блоке **KVN.ID** (вкладка «Достижения»).
> Показываются автоматически и по отдельности — только когда соответствующий Wallet реально
> настроен. Мёртвых ссылок не будет: настроили Apple → появилась Apple-кнопка, настроили Google →
> появилась Google-кнопка. Отдельного переключателя в интерфейсе нет.

### 8.1. Общие настройки (вкладка «Wallet — общее»)

| Опция | Значение |
|---|---|
| `MOBILE_WALLET_WEB_SERVICE_URL` | Публичный https-адрес сайта без слэша (напр. `https://kvnrb.ru`). Пусто — берётся домен сайта |

### 8.2. Apple (вкладка «Wallet — Apple»)

> Требуется **платный аккаунт Apple Developer Program** (~$99/год) — без него Pass Type ID и
> сертификат не выдать. Google Wallet (см. §8.3) бесплатен.

1. Apple Developer → **Identifiers → + → Pass Type IDs** → создать (напр. `pass.ru.kvnufa.member`).
   Это `MOBILE_WALLET_APPLE_PASS_TYPE_ID`.
2. На странице Pass Type ID → **Create Certificate** → по инструкции выпустить сертификат,
   затем в Keychain экспортировать его вместе с приватным ключом в **`.p12`**.
3. Скачать промежуточный **Apple WWDR** (G4) в PEM — `MOBILE_WALLET_APPLE_WWDR`.
4. Задать случайный `MOBILE_WALLET_APPLE_AUTH_SECRET`.

| Опция | Значение |
|---|---|
| `MOBILE_WALLET_APPLE_ENABLED` | Вкл. |
| `MOBILE_WALLET_APPLE_PASS_TYPE_ID` | Pass Type ID |
| `MOBILE_WALLET_APPLE_TEAM_ID` | Team ID (можно тот же, что в APNs) |
| `MOBILE_WALLET_APPLE_ORG_NAME` | Название организации |
| `MOBILE_WALLET_APPLE_LOGO_TEXT` | Текст у логотипа (пусто — «КВН») |
| `MOBILE_WALLET_APPLE_CERT` | Содержимое `.p12` **или** абсолютный путь к файлу |
| `MOBILE_WALLET_APPLE_CERT_PASSWORD` | Пароль `.p12` (пусто — без пароля) |
| `MOBILE_WALLET_APPLE_WWDR` | PEM WWDR (содержимое или путь) |
| `MOBILE_WALLET_APPLE_ICON` | Необязательно: `icon.png` 29×29 (путь). Пусто — генерируется по цвету уровня |
| `MOBILE_WALLET_APPLE_LOGO` | Необязательно: `logo.png` до 160×50 (путь) |
| `MOBILE_WALLET_APPLE_AUTH_SECRET` | Случайная строка (authenticationToken) |

> Push-to-update использует те же креды APNs, что и push во вкладке «iOS (APNs)»:
> ключ `.p8`, Key ID, Team ID. Topic для pass-пуша — сам Pass Type ID (берётся автоматически).

### 8.3. Google (вкладка «Wallet — Google»)

1. Google Wallet Business Console: получить **Issuer ID**.
2. В Google Cloud создать сервис-аккаунт, включить **Google Wallet API**, скачать JSON-ключ.
3. Добавить e-mail сервис-аккаунта в Google Wallet Business Console (Issuer → Users).

| Опция | Значение |
|---|---|
| `MOBILE_WALLET_GOOGLE_ENABLED` | Вкл. |
| `MOBILE_WALLET_GOOGLE_ISSUER_ID` | Issuer ID |
| `MOBILE_WALLET_GOOGLE_CLASS_SUFFIX` | Суффикс класса (пусто — `kvn_member`) |
| `MOBILE_WALLET_GOOGLE_SERVICE_ACCOUNT` | JSON сервис-аккаунта (содержимое или путь) |

> Класс `loyaltyClass` создаётся автоматически (`<Issuer ID>.<суффикс>`), объект —
> `<Issuer ID>.<serial>`. Если класс увидят в статусе `UNDER_REVIEW` — это норма до прохождения
> ревью Google.

## 9. Отладка доставки

Вкладка **«Отладка»**:

| Опция | Значение |
|---|---|
| `MOBILE_PUSH_DEBUG` | Вкл. — push не отправляются, а пишутся в **журнал событий Bitrix**. Реальные провайдеры, если настроены, имеют приоритет |

Сквозной путь проверяется так: включить галку, зарегистрировать устройство, вызвать любое
уведомление → в журнале появится строка `KVN push -> <token>: <заголовок>`.

---

## 10. HTTP-эндпоинты и проверка

Все `/api/mobile/*` требуют **авторизованной сессии Bitrix** (cookie `PHPSESSID` + авторизация).
Запросы из WebView приложения используют ту же сессию, что и сайт.

Регистрация устройства:

```bash
curl -X POST 'https://<домен>/api/mobile/device/' \
  -H 'Cookie: PHPSESSID=<...>' \
  --data 'token=<push-token>&platform=ios&provider=apns&deviceId=<id>&appVersion=1.0.0'
```

Удаление устройства:

```bash
curl -X DELETE 'https://<домен>/api/mobile/device/' \
  -H 'Cookie: PHPSESSID=<...>' \
  --data 'token=<push-token>'
```

Конфиг приложения (публично):

```bash
curl 'https://<домен>/api/mobile/config/'
```

Проверка deep-link файлов (публично, ожидается `application/json` и код 200):

```bash
curl -i 'https://<домен>/.well-known/apple-app-site-association'
curl -i 'https://<домен>/.well-known/assetlinks.json'
```

`platform` ∈ `ios | android | web`, `provider` ∈ `apns | rustore | fcm` (или пусто).

---

## 11. Чек-лист запуска

- [ ] Миграция применена, таблица `avn_kvn_mobile_device` существует.
- [ ] Заполнены креды хотя бы одного провайдера (APNs / RuStore / FCM).
- [ ] APNs: `MOBILE_APNS_ENV` соответствует типу сборки (TestFlight/App Store = `production`).
- [ ] Deep links: `MOBILE_APNS_TEAM_ID` + `MOBILE_APNS_TOPIC` и Android-опции заполнены.
- [ ] Заданы версии и ссылки для обновления.
- [ ] Wallet: применена миграция `avn_kvn_pass_device`, заполнены креды Apple/Google, `MOBILE_WALLET_WEB_SERVICE_URL` корректен.
- [ ] Отладочный провайдер выключен перед продом (или оставлен только для теста).

## 12. Ещё не готово (сервер)

- Всё серверное из плана (push, deep links, forced update, Wallet Apple/Google) реализовано.
  Осталось клиентское приложение (Capacitor, нативная интеграция Wallet-кнопок, нативный push).
