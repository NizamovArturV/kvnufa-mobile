# iOS — сборка и настройка

Платформа iOS **не собирается на Windows** (нужны macOS + Xcode + CocoaPods).
Выполнять на Mac. Android-часть уже готова и собирается на Windows/CI.

## 1. Добавить платформу

```bash
npm install
npx cap add ios      # один раз
npx cap sync         # после изменений www / capacitor.config.json
npx cap open ios     # открыть project в Xcode
```

## 2. Bundle ID и подпись

- В Xcode: таргет **App** → **Signing & Capabilities** → Team (Apple Developer Program),
  Bundle Identifier = `ru.kvnrb.app`.

## 3. Push-уведомления (APNs)

- **+ Capability** → **Push Notifications**.
- **+ Capability** → **Background Modes** → отметить **Remote notifications**.
- На сервере (sprint.options, вкладка «iOS (APNs)») заполнить креды APNs:
  `MOBILE_APNS_KEY`, `MOBILE_APNS_KEY_ID`, `MOBILE_APNS_TEAM_ID`, `MOBILE_APNS_TOPIC`
  (= Bundle ID), `MOBILE_APNS_ENV`.
- `MOBILE_APNS_ENV`: сборки из Xcode (debug на устройстве) → `sandbox`;
  TestFlight и App Store → `production`. Несовпадение = пуши не приходят.

## 4. Universal Links (deep links)

- **+ Capability** → **Associated Domains** → добавить:
  `applinks:kvnrb.ru`
- Файл `https://kvnrb.ru/.well-known/apple-app-site-association` формируется сайтом из
  `MOBILE_APNS_TEAM_ID` + `MOBILE_APNS_TOPIC` (см. `docs/mobile-app-setup.md`, §6).
- Для Smart App Banner на сайте (опционально) в `<head>`: 
  `<meta name="apple-itunes-app" content="app-id=XXXXXXXX, app-argument=https://kvnrb.ru">`.

## 5. Иконки и splash

`resources/icon.png` и `resources/splash.png` — исходники. Сгенерировать нативные ассеты:

```bash
npx capacitor-assets generate --ios
```

> Если `npm install @capacitor/assets` падает на `sharp`/libvips с ошибкой
> `unable to verify the first certificate` (корпоративный VPN/прокси), запускай с
> системным хранилищем сертификатов: `$env:NODE_OPTIONS='--use-system-ca'` (Node 23+).

## 6. Проверка

- Запуск на устройстве из Xcode → открыть сайт, разрешить уведомления.
- Push: отправить уведомление пользователю с сайта → проверить доставку
  (`MOBILE_PUSH_DEBUG` для лога вместо реальной отправки).
- Universal link: открыть `https://kvnrb.ru/...` → должно открыться приложение.

## 7. Что осталось в коде приложения

- Хук `window.__KVN_MOBILE__ = { provider, version }` (опционально) — если нативная
  сборка должна сообщать нестандартный push-провайдер/версию. По умолчанию интеграция
  на сайте определяет `ios → apns`, `android → fcm`, версию — через `App.getInfo()`.
- Биометрия (`@aparajita/capacitor-biometric-auth` уже установлен) — требует серверного
  механизма выдачи device-токена (design TBD); в текущей интеграции не активирована.
