# КВН Уфа — мобильное приложение

Нативная обёртка **Capacitor** вокруг сайта `https://kvnrb.ru`. Один код: WebView грузит
тот же сайт, нативные возможности (push, deep links, Wallet, биометрия) подключаются
через bridge Capacitor. Отдельного фронтенда у приложения нет.

- Целевые сторы: RuStore → Google Play → App Store.
- Продуктовый/архитектурный план: [docs/mobile-app.md](docs/mobile-app.md).
- Настройка серверной части (APNs/RuStore/FCM, deep links, Wallet): [docs/mobile-app-setup.md](docs/mobile-app-setup.md).

Документы в `docs/` — копии, синхронизируемые с основным репозиторием сайта (Bitrix). Если правите
док — правьте в источнике и переносите сюда.

## Идентификаторы

| Параметр | Значение |
|---|---|
| appId / applicationId | `ru.kvnrb.app` |
| appName | КВН Уфа |
| Адрес shell | `https://kvnrb.ru` |
| Android package | `ru.kvnrb.app` |

## Требования

- Node.js 20+ и npm.
- Java 17 + Android SDK (для сборки Android).
- macOS + Xcode + CocoaPods (для сборки iOS; на Windows платформа не собирается).

## Установка и сборка

```bash
npm install
npx cap add android      # один раз
npx cap add ios          # один раз, только на macOS
npx cap sync             # после npm install / изменения www
npx cap open android     # открыть проект в Android Studio
npx cap open ios         # открыть проект в Xcode (macOS)
```

## Структура

```
capacitor.config.json   # appId, server.url, allowNavigation, plugins
www/                    # локальный fallback-экран (offline); основной контент — сайт
android/                # нативная Android-платформа (генерация: cap add android)
ios/                    # нативная iOS-платформа (генерация: cap add ios, macOS)
```

## Интеграция с сайтом (bridge)

Скрипт интеграции живёт **на сайте** (расширение `kvn.mobile`), а не в этом репозитории: он
определяет `window.Capacitor?.isNativePlatform?.()` и работает как в вебе (no-op), так и внутри
приложения. Контракт серверных эндпоинтов:

| Действие | Запрос |
|---|---|
| Регистрация устройства | `POST /api/mobile/device/` (`token, platform, provider, deviceId, appVersion`, требует авторизации) |
| Удаление устройства | `DELETE /api/mobile/device/` (`token`) |
| Конфиг / forced update | `GET /api/mobile/config/` (`minSupportedVersion, latestVersion, storeUrlIos, storeUrlAndroid, message`) |
| Apple Wallet | `GET /wallet/apple/pass/{serial}/` |
| Google Wallet | `GET /wallet/google/{serial}/` |

Платформы push: `apns` (iOS), `rustore` (Android/RuStore), `fcm` (Android/Google Play).

## Deep links (App Links / Universal Links)

- Android: intent-filter `autoVerify="true"` для хоста `kvnrb.ru` в
  `android/app/src/main/AndroidManifest.xml`. Проверка — файл
  `https://kvnrb.ru/.well-known/assetlinks.json` (формируется из опций сайта).
- iOS: включить Associated Domains `applinks:kvnrb.ru` в Xcode. Файл
  `https://kvnrb.ru/.well-known/apple-app-site-association`.

## Лицензия / приватность

Приватный проект. Политика конфиденциальности и согласие на уведомления — на стороне сайта.
