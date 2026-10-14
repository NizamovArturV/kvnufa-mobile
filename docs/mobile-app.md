# mobile-app.md — мобильное приложение, push и Wallet-карта КВНщика

Статус: **Ф0 (серверная часть) готова** — push, deep links, forced update, Wallet Apple/Google.
Целевое решение: собственное приложение как основной
канал взаимодействия и уведомлений. TG/MAX — временные (арендованные) каналы. В Wallet — **только
карточка КВНщика** (привязанный `Member` + его уровень).

Платформенная часть вынесена в отдельный модуль **`avn.kvn.mobile`** (`Avn\Kvn\Mobile\`) — по аналогии
с `avn.telegram` / `avn.max`. Бизнес-обвязка остаётся в `avn.kvn` (канал `push`, `MemberPassService`).

> Операционная инструкция по настройке (APNs/RuStore/FCM, deep links, forced update, эндпоинты) —
> в [mobile-app-setup.md](mobile-app-setup.md).

## Факт (проверено в коде)

- `Member` (`local/modules/avn.kvn/lib/Domain/Entity/Member/Member.php`): `userId` (привязка к
  пользователю, nullable, `getUserId()` :63-66), `guid`, ФИО, `active`, `level` (`getLevel()` :112-115,
  `setLevel()` :117-121).
- `MemberLevel` (`lib/Domain/Enum/MemberLevel.php`): 13 уровней; у каждого `getName()` :21-38,
  `getColor()` :40-57 (готовая палитра для карты), `getPriority()` :59-76.
- Привязка пользователь↔КВНщик: `MemberRepository::findByUserId(int): ?Member`
  (`lib/Domain/Repository/Member/MemberRepository.php:15`); связывание — сценарий
  `MemberUserLinkRequestScenarioHandler` + `MemberUserLinkAction`.
- Повышение уровня: `AwardAchievementToMember`
  (`lib/Domain/Action/Member/AwardAchievementToMember.php:60-77`) — если приоритет нового уровня выше,
  меняет `level`, ставит `active`, `store()` и шлёт нотфикацию «Уровень профиля повышен».
- Текстовый KVN.ID уже есть: `/kvnid` в Telegram (`lib/App/Command/Telegram/KvnIdCommand.php`) и в
  MAX — вывод «ФИО + уровень».
- Уведомления: enum каналов `email | telegram | max`
  (`lib/Domain/Enum/NotificationChannel.php`); `NotificationDispatcher`
  (`lib/App/Service/NotificationDispatcher.php:33-46`) шлёт ровно в **один** активный канал и выходит;
  fallback-цепочки нет. In-app колокольчик (`avn_kvn_notification`) — отдельный ящик, не канал.
- Каналы data-driven: `NotificationChannelInterface` + `NotificationChannelRegistry` +
  `NotificationChannelSettingDto`; UI/JS при добавлении канала не меняются.
- Сайт: Bitrix (серверный рендер) + BitrixVue-острова, CSS-first адаптив. В корне уже есть
  `manifest.json`, `apple-touch-icon.png`, `android-chrome-192x192.png`, `android-chrome-512x512.png`.

## Решения (продукт)

1. Полноценную веб-версию сохраняем; приложение — для повторных пользователей.
2. Простой deferred deep link: после первой установки открываем главную/логин (не конкретный билет).
3. Сторы на старте: **Google Play + RuStore + App Store** (Huawei AppGallery — по надобности).
4. Обёртка **Capacitor**, **один код** (тот же сайт), без отдельного фронтенда для приложения.
5. TG/MAX остаются fallback; при наличии push-токена `push` — дефолт; email — конечный fallback.
6. В Wallet — только **карточка КВНщика**; обновляется автоматически при смене уровня.

## Архитектура

```
Соцсеть (пост) → ссылка на сайт (лендинг)
     ├─ веб: полный сайт (шаринг/SEO/десктоп)
     └─ баннер «Скачать» → Google Play / RuStore / App Store
                                   │
                     Приложение (Capacitor, тот же сайт)
                      ├─ нативные пуши ← сервер (Bitrix)
                      ├─ Wallet-карта КВНщика
                      └─ Universal/App Links: ссылки домена открываются в приложении
```

## Модуль `avn.kvn.mobile` (платформа)

Структура (как у `avn.telegram` / `avn.max`): таблица устройств, сервис, фасад `Provider`, свой
`.settings.php`. Бизнес-логики КВН тут нет.

- `lib/Orm/PushDeviceTable.php` — таблица `avn_kvn_mobile_device`
  (`ID, USER_ID, PLATFORM, PROVIDER, TOKEN, TOKEN_HASH, DEVICE_ID, APP_VERSION, CREATED_AT,
  UPDATED_AT, LAST_SEEN_AT`). Upsert по `TOKEN_HASH` (sha256), уникальный индекс по хэшу.
- `lib/Service/PushService.php` — регистрация/удаление устройств, `send()` по всем устройствам
  пользователя; невалидные токены, возвращённые провайдером, удаляются. Провайдер выбирается по
  коду, заявленному устройством (`PROVIDER`), иначе — по платформе: на android RuStore- и
  FCM-токены не перепутываются.
- `lib/Push/*` — `PushMessage`, `PushProviderInterface`, `PushProviderRegistry` (+`byCode()`),
  `Jwt` (ES256/RS256 для APNs/FCM).
- `lib/Push/Provider/AbstractPushProvider.php` — общий HTTP POST на cURL с HTTP/2
  (Bitrix HttpClient форсит HTTP/1.x, а APNs/RuStore требуют HTTP/2).
- `lib/Push/Provider/ApnsPushProvider.php` — iOS: `POST api.push.apple.com/3/device/{token}`,
  JWT ES256 из ключа `.p8` (Key ID + Team ID + topic), prod/sandbox.
- `lib/Push/Provider/RustorePushProvider.php` — Android: `POST vkpns.rustore.ru/v1/projects/{id}/messages:send`,
  `Authorization: Bearer <service-token>` (drop-in FCM).
- `lib/Push/Provider/FcmPushProvider.php` — Android (Google Play): FCM HTTP v1, access-token из
  JSON сервис-аккаунта (RS256 JWT → OAuth2), кеш в процессе.
- `lib/Push/Provider/LogPushProvider.php` — отладочный провайдер (пишет в журнал), включается
  опцией `MOBILE_PUSH_DEBUG`; нужен, чтобы проверить сквозной путь без реальных кредов. В реестре
  идёт последним, поэтому включённые реальные провайдеры его перекрывают.
- `lib/Config/Options.php` — чтение настроек `sprint.options`; секреты можно держать как
  содержимое или как путь к файлу (`.p8`, JSON сервис-аккаунта).
- `lib/Controller/DeviceController.php` — HTTP-эндпоинты `POST /api/mobile/device/` (регистрация) и
  `DELETE /api/mobile/device/` (удаление); требуют авторизации.
- `lib/Provider/Provider.php` — фасад для `avn.kvn`.

## План — сервер: push

Состояние шага 1 (сделано):

1. ✅ Модуль `avn.kvn.mobile` + таблица `avn_kvn_mobile_device`.
2. ✅ `NotificationChannel::PUSH = 'push'` + `App/Service/Channel/PushChannel.php` (делегирует в
   `\Avn\Kvn\Mobile\Provider\Provider`), зарегистрирован в `NotificationChannelRegistry`.
3. ✅ Универсальный fallback: `NotificationChannelMetaInterface::getFallbackChannels()` — канал сам
   объявляет резервные каналы (`PushChannel` → `[EMAIL]`, остальные → `[]`); `NotificationDispatcher`
   рекурсивно пробует их при неудаче, не зная конкретики каналов.
4. ✅ Провайдеры доставки: `ApnsPushProvider` (iOS), `RustorePushProvider` (Android/RuStore),
   `FcmPushProvider` (Android/Google Play). Креды — в `sprint.options` (страница «Мобильное
   приложение»): `MOBILE_APNS_*`, `MOBILE_RUSTORE_*`, `MOBILE_FCM_*`, `MOBILE_PUSH_DEBUG`.
   WebPush (VAPID) — не реализован (вне мобильного скоупа).
5. ✅ HTTP-эндпоинт регистрации устройства из приложения: `POST /api/mobile/device/` (поля
   `token, platform, provider, deviceId, appVersion`) и `DELETE /api/mobile/device/` (`token`).
   Требует авторизации; CSRF отключён (вызов из WebView приложения).
6. ✅ Deep-link файлы связывания с доменом отдаются динамически из настроек:
   `GET /.well-known/apple-app-site-association` (Universal Links) и
   `GET /.well-known/assetlinks.json` (App Links).
7. ✅ Эндпоинт конфига/принудительного обновления: `GET /api/mobile/config/`
   (`minSupportedVersion`, `latestVersion`, ссылки на сторы, текст).
8. ✅ RuStore Push — сверено с официальным контрактом (`vkpns.rustore.ru/v1/projects/{id}/messages:send`,
   drop-in FCM); живая проверка требует реального проекта в консоли RuStore.

Опции всех перечисленных подсистем — в `sprint.options`, страница «Мобильное приложение»; полная
инструкция — [mobile-app-setup.md](mobile-app-setup.md).

## План — сервер: Wallet-карта КВНщика

> **Статус: реализовано.** Инструкция по настройке — [mobile-app-setup.md](mobile-app-setup.md), §8.

**Тип pass:** Apple — `storeCard` (лояльность); Google — `loyaltyObject` + `loyaltyClass`.

**Данные карты** — из привязанного `Member`:
- ФИО (`getLastName/getName/getSecondName`);
- уровень (`getLevel()->getName()`);
- цвет уровня (`getLevel()->getColor()`) — палитра уже готова;
- `serial` = `Member.guid` (неугадываемое значение). QR на карте (guid) — есть и в Apple, и в Google; служит просто идентификатором, не защищённым пропуском.

**Обновление при смене уровня (ядро требования):**
- Доменный интерфейс `Domain\Service\MemberPassService` (чистота домена), реализация — `App\Service\MemberPassService`.
- Единственная точка смены уровня — `AwardAchievementToMember` (после `setLevel`/`store`); `UpdateMember` уровень не меняет.
- **Apple Wallet:** pass содержит `webServiceURL` + `authenticationToken`; сервер хранит device-токены
  pass'а (`avn_kvn_pass_device`) и при апдейте шлёт push-to-update в APNs → устройство перезапрашивает pass
  (`/v1/devices/.../registrations/...`). `authenticationToken` — единый секрет на pass type.
- **Google Wallet:** класс и объект создаются/обновляются через Wallet API (PUT, при 404 — POST) — Google
  обновляет карту сам.
- Показ в приложении: кнопка «Добавить в Wallet» (Apple `.pkpass`, Google `Save to Google Wallet`);
  для тех, кто без Wallet — экран KVN.ID внутри приложения.
- Кнопки уже выводятся в профиле, в блоке **KVN.ID** (вкладка «Достижения»): Apple- и
  Google-кнопка показываются **по отдельности и только если соответствующий Wallet настроен**
  (`WalletOptions::appleConfigured()` / `googleConfigured()`) — мёртвых ссылок нет.

## План — клиент (Capacitor)

- Один проект; платформы Android + iOS; тот же сайт. Плагины: push, deep link (`appUrlOpen`),
  биометрия, камера/QR, splash/иконки, preferences.
- Изначально: **shell грузит сайт** (обновления мгновенные, без релизов в стор). Если App Store
  отклонит по правилу 4.2 — переходим на **локальный бандл + OTA** (тот же код, без второго фронта).
- Нативные фичи: нативные пуши, биометрия-вход, Wallet-карта, splash.

## План — снижение риска 4.2 (App Store)

Ранжировано по силе: 1) Wallet-карта КВНщика (нативное, невозможно в Safari); 2) нативные пуши;
3) биометрия/keychain; 4) нативная навигация (tab bar, haptics); 5) локальный бандл вместо «обёртки
вокруг ссылки»; 6) оффлайн-кэш; 7) Sign in with Apple + удаление аккаунта + privacy labels.

## План — принудительное обновление

Эндпоинт с `min_supported_version`; на старте приложение сравнивает версию → блокирующий экран
«Обновите приложение» + ссылка в стор; плюс мягкое уведомление «доступна новая версия».

## План — deep links и онбординг

- iOS: `<meta name="apple-itunes-app" ...>` (Smart App Banner) + Universal Links
  (`/.well-known/apple-app-site-association`).
- Android: баннер/редирект на Google Play/RuStore + App Links (`/.well-known/assetlinks.json`).
- Первый переход (deferred) — простая версия: после установки главная/логин.

## План — сторы и требования

- **RuStore**: аккаунт (юрлицо/ИП РФ), RuStore Push project id + service token.
- **Google Play**: для бесплатного приложения ок; вопросы по монетизации/аккаунту — перепроверить политику 2026.
- **App Store**: Apple Developer Program, bundle id; для Wallet — Pass Type ID + APNs key `.p8`.
- Креды Wallet/push — в `sprint.options`.

## План — комплаенс

- 152-ФЗ: RuStore Push — доставка в РФ; APNs — трансгранично (неизбежно на iOS). Обновить политику
  конфиденциальности, согласие на уведомления и Wallet.

## План — репозитории

- Бэкенд + сайт — текущий репозиторий.
- Приложение — **один** репозиторий (`Capacitor`); различия сторов — build-конфиги и подписи, не отдельные репы.

## План — роадмап

- **Ф0 — сервер:** ✅ модуль `avn.kvn.mobile` + таблица устройств + канал `push` + fallback `push→email`
  (+ миграция); ✅ провайдеры APNs/RuStore/FCM + эндпоинт регистрации устройства; ✅ deep-link
  associated-файлы + forced-update эндпоинт; ✅ `MemberPassService` + генерация pass (Apple/Google)
  + web-service для Apple (`WalletController`, таблица `avn_kvn_pass_device`).
- **Ф1 — MVP-обёртка:** Capacitor; нативные пуши (APNs + RuStore); Wallet «Добавить»; биометрия;
  сборка на устройстве; forced-update UI.
- **Ф2 — публикация:** RuStore → Google Play → App Store; баннеры «Скачать» + Smart App Banner.
- **Ф3 — adoption:** метрики → дефолт `push`; TG/MAX в fallback; опц. PWA.

## Риски и открытые вопросы

1. App Store 4.2 (митигация выше; fallback — локальный бандл + OTA).
2. Google Play — политика 2026 не подтверждена (перепроверить перед публикацией).
3. Apple PassKit web-service (регистрации устройств + APNs pass-push) — реализован на сервере; проверить вживую с реальными кредами.
4. RuStore Push: доставка через дистрибьюторы (RuStore/VK-приложения) → охват проверить.
5. Наличие аккаунтов RuStore/Apple и сроки модерации.

## Метрики успеха

Install rate с лендинга; push opt-in rate; delivery rate; доля каналов (push/telegram/max/email);
добавления карты в Wallet; retention.

## Вне scope

Нативная кодовая база (Flutter/RN), офлайн-first, VK Mini Apps, десктоп-приложения, билеты в Wallet
(в Wallet только карточка КВНщика).
