# BalanceTrack — Full Source Code Review

Every file that ships in production is included below in reading order:
Product Spec → Backend → Frontend Config → App Routes → Src (state / utils / components).

_Auto-generated for pre-publish review. Not part of the shipped build._

## Contents

1. [PRODUCT SPEC — `memory/PRD.md`](#1-memory-prd-md)
2. [BACKEND — API server — `backend/server.py`](#2-backend-server-py)
3. [BACKEND — Python deps — `backend/requirements.txt`](#3-backend-requirements-txt)
4. [BACKEND — env (local Mongo, no secrets) — `backend/.env`](#4-backend--env)
5. [BACKEND — pytest security suite — `backend/tests/test_backend_security.py`](#5-backend-tests-test_backend_security-py)
6. [FRONTEND — Expo app config — `frontend/app.json`](#6-frontend-app-json)
7. [FRONTEND — package.json — `frontend/package.json`](#7-frontend-package-json)
8. [FRONTEND — tsconfig — `frontend/tsconfig.json`](#8-frontend-tsconfig-json)
9. [FRONTEND — metro config — `frontend/metro.config.js`](#9-frontend-metro-config-js)
10. [FRONTEND — eslint config — `frontend/eslint.config.js`](#10-frontend-eslint-config-js)
11. [FRONTEND — env (preview URLs only, no secrets) — `frontend/.env`](#11-frontend--env)
12. [FRONTEND — PWA manifest — `frontend/public/manifest.webmanifest`](#12-frontend-public-manifest-webmanifest)
13. [APP — Root layout (providers, PWA head, gestures) — `frontend/app/_layout.tsx`](#13-frontend-app-_layout-tsx)
14. [APP — Static HTML shell (CSP + PWA meta for production export) — `frontend/app/+html.tsx`](#14-frontend-app-+html-tsx)
15. [APP — Entry: redirect to dashboard — `frontend/app/index.tsx`](#15-frontend-app-index-tsx)
16. [APP — Tab bar layout — `frontend/app/(tabs)/_layout.tsx`](#16-frontend-app-tabs-_layout-tsx)
17. [APP — Dashboard tab (TTB + Day-in-Lieu) — `frontend/app/(tabs)/dashboard.tsx`](#17-frontend-app-tabs-dashboard-tsx)
18. [APP — Timesheet tab (fortnight entry) — `frontend/app/(tabs)/timesheet.tsx`](#18-frontend-app-tabs-timesheet-tsx)
19. [APP — Settings tab — `frontend/app/(tabs)/settings.tsx`](#19-frontend-app-tabs-settings-tsx)
20. [APP — Log detail screen (/logs/ttb, /logs/dil) — `frontend/app/logs/[type].tsx`](#20-frontend-app-logs-type-tsx)
21. [SRC — Design tokens (colours, spacing, type) — `frontend/src/theme/tokens.ts`](#21-frontend-src-theme-tokens-ts)
22. [SRC — BottomSheet component — `frontend/src/components/BottomSheet.tsx`](#22-frontend-src-components-bottomsheet-tsx)
23. [SRC — Icon-font pre-warm hook — `frontend/src/hooks/use-icon-fonts.ts`](#23-frontend-src-hooks-use-icon-fonts-ts)
24. [SRC — App store (AsyncStorage-backed state) — `frontend/src/store/appStore.tsx`](#24-frontend-src-store-appstore-tsx)
25. [SRC — Date / fortnight helpers — `frontend/src/utils/date.ts`](#25-frontend-src-utils-date-ts)
26. [SRC — Timesheet formulas (spreadsheet rules) — `frontend/src/utils/timesheet.ts`](#26-frontend-src-utils-timesheet-ts)

---

<a id="1-memory-prd-md"></a>

## 1. PRODUCT SPEC

**Path:** `memory/PRD.md`

_25 lines, 1363 bytes_

```md
# BalanceTrack — Timesheet & TTB Tracker

Native-feel Expo mobile app that replicates a professional TTB (Time Toward Balance) & fortnight timesheet spreadsheet.

## Tabs
1. **Dashboard** — hero TTB card (hours + days), Day-in-Lieu card, one-tap Add / Use bottom sheets for TTB (15-min increments) and Day-in-Lieu (specify date used).
2. **Timesheet** — 2-week fortnight (Mon–Sun) with prev/next navigation. Default 8:00 → 16:30. Per-day: leave type, start/finish pickers, TTB used, notes. Live weekly + fortnight totals.
3. **Settings** — staff name, manager, role, opening balances, and links to TTB & Day-in-Lieu logs.

## Formulas (matches Pro_Timesheet_System.xlsm)
- Weekday Total = (finish − start) − 0.5 lunch (Work only)
- Sat/Sun Total = (finish − start) (Work only, no lunch)
- Non-Work: PH/RDO/Annual/Sick = 8, TTB = −ttbUsed
- TTB Earned per day: weekday = max(Total − 8, 0); Sat/Sun = full duration
- TTB Balance = Opening + Σ Earned + Σ Manual Adds − Σ Manual Uses − Σ Timesheet TTB Used
- Day in Lieu Balance = Opening + Σ Adds − Σ Uses

## Storage
- 100% local (AsyncStorage). No backend, no accounts.

## Tech
- Expo Router (file-based) + typed routes
- Tabs: Dashboard / Timesheet / Settings + nested `/logs/[type]` screen
- `expo-image`, `expo-linear-gradient`, `@expo/vector-icons`
- Custom BottomSheet component (cross-platform Modal-based)
```

---

<a id="2-backend-server-py"></a>

## 2. BACKEND — API server

**Path:** `backend/server.py`

_49 lines, 1173 bytes_

```python
from fastapi import FastAPI, APIRouter
from dotenv import load_dotenv
from starlette.middleware.cors import CORSMiddleware
import os
import logging
from pathlib import Path


ROOT_DIR = Path(__file__).parent
load_dotenv(ROOT_DIR / '.env')

# Create the main app without a prefix
app = FastAPI()

# Create a router with the /api prefix
api_router = APIRouter(prefix="/api")


@api_router.get("/health")
async def health():
    """Minimal liveness probe — no data returned, no writes."""
    return {"status": "ok"}


# Include the router in the main app
app.include_router(api_router)

# Restrictive CORS: this backend is not used by the client-only app.
# Keep it locked to the app's own preview origin and disable credentials.
_ALLOWED_ORIGINS = [
    o.strip()
    for o in os.environ.get("APP_URL", "").split(",")
    if o.strip()
] or []

app.add_middleware(
    CORSMiddleware,
    allow_credentials=False,
    allow_origins=_ALLOWED_ORIGINS,
    allow_methods=["GET"],
    allow_headers=["*"],
)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)
```

---

<a id="3-backend-requirements-txt"></a>

## 3. BACKEND — Python deps

**Path:** `backend/requirements.txt`

_27 lines, 448 bytes_

```text
fastapi==0.110.1
uvicorn==0.25.0
boto3>=1.34.129
requests-oauthlib>=2.0.0
cryptography>=42.0.8
python-dotenv>=1.0.1
pymongo==4.6.3
pydantic>=2.6.4
email-validator>=2.2.0
pyjwt>=2.10.1
bcrypt==4.1.3
passlib>=1.7.4
tzdata>=2024.2
motor==3.3.1
pytest>=8.0.0
black>=24.1.1
isort>=5.13.2
flake8>=7.0.0
mypy>=1.8.0
python-jose>=3.3.0
requests>=2.31.0
pandas>=2.2.0
numpy>=1.26.0
python-multipart>=0.0.9
jq>=1.6.0
typer>=0.9.0
emergentintegrations==0.2.0
```

---

<a id="4-backend--env"></a>

## 4. BACKEND — env (local Mongo, no secrets)

**Path:** `backend/.env`

_2 lines, 62 bytes_

```bash
MONGO_URL="mongodb://localhost:27017"
DB_NAME="test_database"
```

---

<a id="5-backend-tests-test_backend_security-py"></a>

## 5. BACKEND — pytest security suite

**Path:** `backend/tests/test_backend_security.py`

_102 lines, 4121 bytes_

```python
"""Backend security & hardening tests for BalanceTrack.

Covers:
- /api/health returns 200 {"status":"ok"}
- /api/status GET & POST return 404 (endpoint stripped)
- CORS middleware does NOT emit `access-control-allow-credentials: true`
  when hitting the *internal* backend at localhost:8001 with an unknown Origin.
"""
import os
import pytest
import requests

PUBLIC_URL = os.environ.get("EXPO_PUBLIC_BACKEND_URL", "https://timesheet-pro-99.preview.emergentagent.com").rstrip("/")
INTERNAL_URL = "http://localhost:8001"


@pytest.fixture
def api_client():
    s = requests.Session()
    s.headers.update({"Content-Type": "application/json"})
    return s


# ---------- Health endpoint ---------------------------------------------------

class TestHealth:
    def test_health_public_200(self, api_client):
        r = api_client.get(f"{PUBLIC_URL}/api/health", timeout=15)
        assert r.status_code == 200, r.text
        assert r.json() == {"status": "ok"}

    def test_health_internal_200(self, api_client):
        r = api_client.get(f"{INTERNAL_URL}/api/health", timeout=10)
        assert r.status_code == 200
        assert r.json() == {"status": "ok"}


# ---------- Removed endpoints -------------------------------------------------

class TestStatusEndpointRemoved:
    def test_get_status_public_404(self, api_client):
        r = api_client.get(f"{PUBLIC_URL}/api/status", timeout=15)
        assert r.status_code == 404, f"expected 404, got {r.status_code}: {r.text[:200]}"

    def test_post_status_public_404(self, api_client):
        r = api_client.post(f"{PUBLIC_URL}/api/status", json={"client_name": "TEST_x"}, timeout=15)
        assert r.status_code == 404, f"expected 404, got {r.status_code}: {r.text[:200]}"

    def test_get_status_internal_404(self, api_client):
        r = api_client.get(f"{INTERNAL_URL}/api/status", timeout=10)
        assert r.status_code == 404

    def test_post_status_internal_404(self, api_client):
        r = api_client.post(f"{INTERNAL_URL}/api/status", json={"client_name": "TEST_x"}, timeout=10)
        assert r.status_code == 404


# ---------- CORS hardening ----------------------------------------------------

class TestCorsHardening:
    """The FastAPI CORS middleware must NOT emit
    access-control-allow-credentials: true for any origin.
    Test against the *internal* backend to bypass the K8s ingress which may
    still add a wildcard ACAO header (out of scope)."""

    def test_no_allow_credentials_from_evil_origin(self, api_client):
        r = requests.get(
            f"{INTERNAL_URL}/api/health",
            headers={"Origin": "https://evil.example"},
            timeout=10,
        )
        lowered = {k.lower(): v for k, v in r.headers.items()}
        assert "access-control-allow-credentials" not in lowered, (
            f"CORS credentials header leaked: {lowered.get('access-control-allow-credentials')}"
        )

    def test_no_allow_credentials_from_app_origin(self, api_client):
        r = requests.get(
            f"{INTERNAL_URL}/api/health",
            headers={"Origin": "https://timesheet-pro-99.preview.emergentagent.com"},
            timeout=10,
        )
        lowered = {k.lower(): v for k, v in r.headers.items()}
        assert "access-control-allow-credentials" not in lowered or \
            lowered["access-control-allow-credentials"].lower() != "true", (
            f"CORS credentials true leaked: {lowered.get('access-control-allow-credentials')}"
        )

    def test_preflight_no_allow_credentials(self, api_client):
        r = requests.options(
            f"{INTERNAL_URL}/api/health",
            headers={
                "Origin": "https://evil.example",
                "Access-Control-Request-Method": "GET",
            },
            timeout=10,
        )
        lowered = {k.lower(): v for k, v in r.headers.items()}
        assert "access-control-allow-credentials" not in lowered or \
            lowered["access-control-allow-credentials"].lower() != "true", (
            f"Preflight leaked credentials header: {lowered.get('access-control-allow-credentials')}"
        )
```

---

<a id="6-frontend-app-json"></a>

## 6. FRONTEND — Expo app config

**Path:** `frontend/app.json`

_42 lines, 948 bytes_

```json
{
  "expo": {
    "name": "BalanceTrack",
    "slug": "balancetrack",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/icon.png",
    "scheme": "balancetrack",
    "userInterfaceStyle": "automatic",
    "newArchEnabled": true,
    "ios": {
      "supportsTablet": true
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/adaptive-icon.png",
        "backgroundColor": "#000000"
      },
      "edgeToEdgeEnabled": true
    },
    "web": {
      "bundler": "metro",
      "output": "single",
      "favicon": "./assets/images/favicon.png"
    },
    "plugins": [
      "expo-router",
      [
        "expo-splash-screen",
        {
          "image": "./assets/images/splash-image.png",
          "imageWidth": 200,
          "resizeMode": "contain",
          "backgroundColor": "#000000"
        }
      ]
    ],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

---

<a id="7-frontend-package-json"></a>

## 7. FRONTEND — package.json

**Path:** `frontend/package.json`

_61 lines, 1900 bytes_

```json
{
  "name": "frontend",
  "main": "expo-router/entry",
  "version": "1.0.0",
  "scripts": {
    "preinstall": "./scripts/cmd-guard.js --preinstall",
    "start": "expo start",
    "reset-project": "node ./scripts/reset-project.js",
    "android": "expo start --android",
    "ios": "expo start --ios",
    "web": "expo start --web",
    "lint": "expo lint"
  },
  "dependencies": {
    "@expo/metro-runtime": "6.1.2",
    "@expo/vector-icons": "15.1.1",
    "@react-native-async-storage/async-storage": "2.2.0",
    "date-fns": "4.1.0",
    "dayjs": "1.11.13",
    "expo": "54.0.35",
    "expo-blur": "15.0.8",
    "expo-constants": "18.0.13",
    "expo-font": "14.0.12",
    "expo-haptics": "15.0.8",
    "expo-image": "3.0.11",
    "expo-linear-gradient": "15.0.8",
    "expo-linking": "8.0.12",
    "expo-router": "6.0.24",
    "expo-secure-store": "15.0.8",
    "expo-splash-screen": "31.0.13",
    "expo-status-bar": "3.0.9",
    "expo-symbols": "1.0.8",
    "expo-system-ui": "6.0.9",
    "expo-web-browser": "15.0.11",
    "react": "19.1.0",
    "react-dom": "19.1.0",
    "react-native": "0.81.5",
    "react-native-dotenv": "3.4.11",
    "react-native-gesture-handler": "2.28.0",
    "react-native-reanimated": "4.1.1",
    "react-native-safe-area-context": "5.6.0",
    "react-native-screens": "4.16.0",
    "react-native-web": "0.21.0",
    "react-native-webview": "13.15.0",
    "react-native-worklets": "0.5.1"
  },
  "devDependencies": {
    "@types/react": "19.1.10",
    "eslint": "9.25.0",
    "eslint-config-expo": "10.0.0",
    "expo-doctor": "1.19.8",
    "typescript": "5.9.3"
  },
  "resolutions": {
    "@eslint/plugin-kit": "0.3.4",
    "postcss": "8.5.10",
    "uuid": "11.1.1"
  },
  "private": true,
  "packageManager": "yarn@1.22.22+sha512.a6b2f7906b721bba3d67d4aff083df04dad64c399707841b7acf00f6b133b7ac24255f2652fa22ae3534329dc6180534e98d17432037ff6fd140556e2bb3137e"
}
```

---

<a id="8-frontend-tsconfig-json"></a>

## 8. FRONTEND — tsconfig

**Path:** `frontend/tsconfig.json`

_17 lines, 242 bytes_

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "paths": {
      "@/*": [
        "./*"
      ]
    }
  },
  "include": [
    "**/*.ts",
    "**/*.tsx",
    ".expo/types/**/*.ts",
    "expo-env.d.ts"
  ]
}
```

---

<a id="9-frontend-metro-config-js"></a>

## 9. FRONTEND — metro config

**Path:** `frontend/metro.config.js`

_25 lines, 1001 bytes_

```javascript
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const path = require('path');
const { FileStore } = require('metro-cache');

const config = getDefaultConfig(__dirname);

// Use a stable on-disk store (shared across web/android)
const root = process.env.METRO_CACHE_ROOT || path.join(__dirname, '.metro-cache');
config.cacheStores = [
  new FileStore({ root: path.join(root, 'cache') }),
];


// // Exclude unnecessary directories from file watching
// config.watchFolders = [__dirname];
// config.resolver.blacklistRE = /(.*)\/(__tests__|android|ios|build|dist|.git|node_modules\/.*\/android|node_modules\/.*\/ios|node_modules\/.*\/windows|node_modules\/.*\/macos)(\/.*)?$/;

// // Alternative: use a more aggressive exclusion pattern
// config.resolver.blacklistRE = /node_modules\/.*\/(android|ios|windows|macos|__tests__|\.git|.*\.android\.js|.*\.ios\.js)$/;

// Reduce the number of workers to decrease resource usage
config.maxWorkers = 2;

module.exports = config;
```

---

<a id="10-frontend-eslint-config-js"></a>

## 10. FRONTEND — eslint config

**Path:** `frontend/eslint.config.js`

_10 lines, 237 bytes_

```javascript
// https://docs.expo.dev/guides/using-eslint/
const { defineConfig } = require('eslint/config');
const expoConfig = require('eslint-config-expo/flat');

module.exports = defineConfig([
  expoConfig,
  {
    ignores: ['dist/*'],
  },
]);
```

---

<a id="11-frontend--env"></a>

## 11. FRONTEND — env (preview URLs only, no secrets)

**Path:** `frontend/.env`

_6 lines, 334 bytes_

```bash
EXPO_TUNNEL_SUBDOMAIN=timesheet-pro-99
EXPO_PACKAGER_HOSTNAME=https://timesheet-pro-99.preview.emergentagent.com
EXPO_PUBLIC_BACKEND_URL=https://timesheet-pro-99.preview.emergentagent.com
EXPO_USE_FAST_RESOLVER="1"
METRO_CACHE_ROOT=/app/frontend/.metro-cache
EXPO_PACKAGER_PROXY_URL=https://timesheet-pro-99.preview.emergentagent.com
```

---

<a id="12-frontend-public-manifest-webmanifest"></a>

## 12. FRONTEND — PWA manifest

**Path:** `frontend/public/manifest.webmanifest`

_25 lines, 646 bytes_

```json
{
  "name": "BalanceTrack — Timesheet & TTB",
  "short_name": "BalanceTrack",
  "description": "Track TTB hours & days, days in lieu, and fortnightly timesheets.",
  "start_url": "/dashboard",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#F7F7F7",
  "theme_color": "#5B7C65",
  "icons": [
    {
      "src": "/assets/assets/images/icon.png",
      "sizes": "192x192 512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/assets/assets/images/adaptive-icon.png",
      "sizes": "192x192 512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

---

<a id="13-frontend-app-_layout-tsx"></a>

## 13. APP — Root layout (providers, PWA head, gestures)

**Path:** `frontend/app/_layout.tsx`

_51 lines, 1777 bytes_

```tsx
import { Stack } from "expo-router";
import Head from "expo-router/head";
import * as SplashScreen from "expo-splash-screen";
import { useEffect } from "react";
import { LogBox, StatusBar } from "react-native";
import { SafeAreaProvider } from "react-native-safe-area-context";
import { GestureHandlerRootView } from "react-native-gesture-handler";

import { useIconFonts } from "@/src/hooks/use-icon-fonts";
import { AppProvider } from "@/src/store/appStore";


// Disable logbox errors etc so that users can see the app
// and agent works as expected.
LogBox.ignoreAllLogs(true)

// Keep the native splash visible from cold start until icon fonts register.
// Required because @expo/vector-icons' componentDidMount fallback fires
// Font.loadAsync against a broken vendor path if any <Icon> mounts before
// the family is registered — which throws on Android Expo Go.
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [loaded, error] = useIconFonts();

  useEffect(() => {
    if (loaded || error) {
      SplashScreen.hideAsync();
    }
  }, [loaded, error]);

  // If the CDN is unreachable we fall through on error rather than wedging
  // the app — icons will tofu, but the app still boots.
  if (!loaded && !error) return null;

  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Head.Provider>
        <SafeAreaProvider>
          <AppProvider>
            <StatusBar barStyle="dark-content" />
            <Stack screenOptions={{ headerShown: false }}>
              <Stack.Screen name="(tabs)" />
              <Stack.Screen name="logs/[type]" options={{ presentation: 'card' }} />
            </Stack>
          </AppProvider>
        </SafeAreaProvider>
      </Head.Provider>
    </GestureHandlerRootView>
  );
}
```

---

<a id="14-frontend-app-+html-tsx"></a>

## 14. APP — Static HTML shell (CSP + PWA meta for production export)

**Path:** `frontend/app/+html.tsx`

_99 lines, 4043 bytes_

```tsx
// @ts-nocheck
import { ScrollViewStyleReset } from "expo-router/html";
import type { PropsWithChildren } from "react";

/**
 * Root HTML for every web page during static rendering.
 * Includes PWA / "Add to Home Screen" metadata so the app installs cleanly on
 * iOS Safari and Android Chrome home screens.
 */
export default function Root({ children }: PropsWithChildren) {
  return (
    <html lang="en" style={{ height: "100%" }}>
      <head>
        <meta charSet="utf-8" />
        <meta httpEquiv="X-UA-Compatible" content="IE=edge" />
        <meta
          name="viewport"
          content="width=device-width, initial-scale=1, viewport-fit=cover, shrink-to-fit=no"
        />

        {/* --- Security hardening (SEC-001) --------------------------------
           NB: X-Frame-Options / frame-ancestors are only fully enforceable
           via HTTP headers; browsers ignore them from a <meta> tag.
           - The CSP <meta> here still blunts injection / mixed-content risk.
           - Referrer-Policy via <meta> IS honoured.
           - JS anti-framing below is defence-in-depth against clickjacking.
        ------------------------------------------------------------------- */}
        <meta name="referrer" content="no-referrer" />
        <meta
          httpEquiv="Content-Security-Policy"
          content={
            [
              "default-src 'self'",
              // React Native Web + Metro dev bundler need inline styles/scripts.
              "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
              "style-src 'self' 'unsafe-inline'",
              "img-src 'self' data: blob: https://images.unsplash.com https://plus.unsplash.com",
              "font-src 'self' data: https://cdn.jsdelivr.net https://cdn.expo.dev",
              "connect-src 'self' ws: wss: https:",
              "object-src 'none'",
              "base-uri 'self'",
              "form-action 'self'",
              "frame-ancestors 'none'",
              "manifest-src 'self'",
            ].join('; ')
          }
        />

        <title>BalanceTrack — Timesheet & TTB</title>
        <meta
          name="description"
          content="Track TTB (Time Toward Balance) hours & days, days in lieu, and fortnightly timesheets."
        />
        <meta name="theme-color" content="#5B7C65" />
        <link rel="manifest" href="/manifest.webmanifest" />
        <link rel="apple-touch-icon" href="/assets/assets/images/icon.png" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
        <meta name="apple-mobile-web-app-title" content="BalanceTrack" />
        <meta name="mobile-web-app-capable" content="yes" />
        <meta name="application-name" content="BalanceTrack" />

        {/* JS anti-framing: bust out of any iframe wrap. */}
        <script
          dangerouslySetInnerHTML={{
            __html:
              "if (window.top !== window.self) { try { window.top.location = window.self.location; } catch (e) { document.documentElement.style.display = 'none'; } }",
          }}
        />
        {/*
          Disable body scrolling on web to make ScrollView components work correctly.
        */}
        <ScrollViewStyleReset />
        <style
          dangerouslySetInnerHTML={{
            __html: `
              html, body { background-color: #F7F7F7; }
              body { -webkit-tap-highlight-color: transparent; }
              body > div:first-child { position: fixed !important; top: 0; left: 0; right: 0; bottom: 0; }
              [role="tablist"] [role="tab"] * { overflow: visible !important; }
              [role="heading"], [role="heading"] * { overflow: visible !important; }
            `,
          }}
        />
      </head>
      <body
        style={{
          margin: 0,
          height: "100%",
          overflow: "hidden",
          display: "flex",
          flexDirection: "column",
        }}
      >
        {children}
      </body>
    </html>
  );
}
```

---

<a id="15-frontend-app-index-tsx"></a>

## 15. APP — Entry: redirect to dashboard

**Path:** `frontend/app/index.tsx`

_5 lines, 125 bytes_

```tsx
import { Redirect } from "expo-router";

export default function Index() {
  return <Redirect href="/(tabs)/dashboard" />;
}
```

---

<a id="16-frontend-app-tabs-_layout-tsx"></a>

## 16. APP — Tab bar layout

**Path:** `frontend/app/(tabs)/_layout.tsx`

_65 lines, 1695 bytes_

```tsx
import React from 'react';
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';
import { Platform, StyleSheet } from 'react-native';

import { colors } from '@/src/theme/tokens';

export default function TabsLayout() {
  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: colors.brandPrimary,
        tabBarInactiveTintColor: colors.info,
        tabBarStyle: styles.tabBar,
        tabBarLabelStyle: styles.label,
      }}
    >
      <Tabs.Screen
        name="dashboard"
        options={{
          title: 'Dashboard',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="stats-chart" size={size} color={color} />
          ),
          tabBarButtonTestID: 'tab-dashboard',
        }}
      />
      <Tabs.Screen
        name="timesheet"
        options={{
          title: 'Timesheet',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="calendar" size={size} color={color} />
          ),
          tabBarButtonTestID: 'tab-timesheet',
        }}
      />
      <Tabs.Screen
        name="settings"
        options={{
          title: 'Settings',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="settings-sharp" size={size} color={color} />
          ),
          tabBarButtonTestID: 'tab-settings',
        }}
      />
    </Tabs>
  );
}

const styles = StyleSheet.create({
  tabBar: {
    backgroundColor: colors.surfaceSecondary,
    borderTopColor: colors.border,
    borderTopWidth: StyleSheet.hairlineWidth,
    height: Platform.OS === 'ios' ? 84 : 64,
    paddingTop: 6,
  },
  label: {
    fontSize: 11,
    fontWeight: '500',
  },
});
```

---

<a id="17-frontend-app-tabs-dashboard-tsx"></a>

## 17. APP — Dashboard tab (TTB + Day-in-Lieu)

**Path:** `frontend/app/(tabs)/dashboard.tsx`

_604 lines, 19120 bytes_

```tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ScrollView,
  Pressable,
  TextInput,
  Platform,
} from 'react-native';
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';
import { LinearGradient } from 'expo-linear-gradient';
import { ImageBackground } from 'expo-image';
import { Ionicons } from '@expo/vector-icons';
import Head from 'expo-router/head';

import { colors, spacing, radius, fontSize, fontWeight } from '@/src/theme/tokens';
import { useApp } from '@/src/store/appStore';
import { formatHours, roundTo15 } from '@/src/utils/timesheet';
import { BottomSheet } from '@/src/components/BottomSheet';
import { isoDate, formatDateFull, parseIsoDate } from '@/src/utils/date';

const HERO_BG =
  'https://images.unsplash.com/photo-1618898613684-2cf0ae9a7c19?crop=entropy&cs=srgb&fm=jpg&ixid=M3w4NjA2MTJ8MHwxfHNlYXJjaHwxfHxhYnN0cmFjdCUyMHNhZ2UlMjBncmVlbiUyMG5hdHVyZSUyMHRleHR1cmUlMjBiYWNrZ3JvdW5kfGVufDB8fHx8MTc4Mjk3OTUxMHww&ixlib=rb-4.1.0&q=85';

export default function Dashboard() {
  const {
    ttbBalance,
    dayInLieuBalance,
    ttbEarnedTotal,
    ttbUsedTotal,
    openingTTB,
    openingDayInLieu,
    addTtbLog,
    addDilLog,
    profile,
  } = useApp();
  const insets = useSafeAreaInsets();

  const [ttbSheet, setTtbSheet] = useState<null | 'add' | 'use'>(null);
  const [dilSheet, setDilSheet] = useState<null | 'add' | 'use'>(null);

  const ttbDays = ttbBalance / 8;

  return (
    <SafeAreaView style={styles.safe} edges={['top']} testID="dashboard-screen">
      <Head>
        <title>BalanceTrack — Timesheet & TTB</title>
        <meta
          name="description"
          content="Track TTB (Time Toward Balance) hours & days, days in lieu, and fortnightly timesheets."
        />
        <meta name="theme-color" content="#5B7C65" />
        <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, shrink-to-fit=no, user-scalable=no" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
        <meta name="apple-mobile-web-app-title" content="BalanceTrack" />
        <meta name="mobile-web-app-capable" content="yes" />
        <meta name="application-name" content="BalanceTrack" />
        <link rel="apple-touch-icon" href="/assets/assets/images/icon.png" />
        <link rel="manifest" href="/manifest.webmanifest" />
      </Head>
      <ScrollView
        contentContainerStyle={[styles.container, { paddingBottom: insets.bottom + spacing['3xl'] }]}
        showsVerticalScrollIndicator={false}
      >
        <View style={styles.headerRow}>
          <View>
            <Text style={styles.greeting}>Good day</Text>
            <Text style={styles.name} testID="dashboard-name">
              {profile.staffName || 'Welcome'}
            </Text>
          </View>
        </View>

        {/* TTB Hero card */}
        <View style={styles.heroWrap} testID="ttb-hero-card">
          <ImageBackground source={HERO_BG} style={styles.heroBg} contentFit="cover">
            <LinearGradient
              colors={['rgba(46,64,52,0.55)', 'rgba(46,64,52,0.92)']}
              style={StyleSheet.absoluteFill}
            />
            <View style={styles.heroInner}>
              <Text style={styles.heroLabel}>TIME TOWARD BALANCE</Text>
              <View style={styles.heroRow}>
                <Text style={styles.heroValue} testID="ttb-hours-value">
                  {formatHours(ttbBalance)}
                </Text>
                <Text style={styles.heroSubUnit}>hours</Text>
              </View>
              <View style={[styles.heroRow, { marginTop: spacing.xs }]}>
                <Text style={styles.heroDays} testID="ttb-days-value">
                  {ttbDays.toFixed(2)}
                </Text>
                <Text style={styles.heroSubUnit}>days</Text>
              </View>

              <View style={styles.heroStats}>
                <View style={styles.heroStatItem}>
                  <Text style={styles.heroStatLabel}>Earned</Text>
                  <Text style={styles.heroStatValue}>{formatHours(ttbEarnedTotal)}</Text>
                </View>
                <View style={styles.heroStatDivider} />
                <View style={styles.heroStatItem}>
                  <Text style={styles.heroStatLabel}>Used</Text>
                  <Text style={styles.heroStatValue}>{formatHours(ttbUsedTotal)}</Text>
                </View>
                <View style={styles.heroStatDivider} />
                <View style={styles.heroStatItem}>
                  <Text style={styles.heroStatLabel}>Opening</Text>
                  <Text style={styles.heroStatValue}>{formatHours(openingTTB)}</Text>
                </View>
              </View>
            </View>
          </ImageBackground>
        </View>

        <View style={styles.actionRow}>
          <Pressable
            style={[styles.actionBtn, styles.actionAdd]}
            onPress={() => setTtbSheet('add')}
            testID="ttb-add-button"
          >
            <Ionicons name="add-circle" size={18} color={colors.onBrandPrimary} />
            <Text style={styles.actionAddText}>Add TTB</Text>
          </Pressable>
          <Pressable
            style={[styles.actionBtn, styles.actionUse]}
            onPress={() => setTtbSheet('use')}
            testID="ttb-use-button"
          >
            <Ionicons name="remove-circle" size={18} color={colors.brandPrimary} />
            <Text style={styles.actionUseText}>Use TTB</Text>
          </Pressable>
        </View>

        {/* Day in Lieu card */}
        <View style={styles.dilCard} testID="dil-card">
          <View style={styles.dilHeader}>
            <View>
              <Text style={styles.cardLabel}>DAY IN LIEU</Text>
              <View style={styles.dilRow}>
                <Text style={styles.dilValue} testID="dil-value">
                  {dayInLieuBalance}
                </Text>
                <Text style={styles.dilUnit}>
                  {Math.abs(dayInLieuBalance) === 1 ? 'day' : 'days'}
                </Text>
              </View>
              <Text style={styles.dilOpening}>Opening balance: {openingDayInLieu}</Text>
            </View>
            <View style={styles.dilIconWrap}>
              <Ionicons name="calendar-outline" size={28} color={colors.brandPrimary} />
            </View>
          </View>

          <View style={styles.actionRow}>
            <Pressable
              style={[styles.actionBtn, styles.actionAdd]}
              onPress={() => setDilSheet('add')}
              testID="dil-add-button"
            >
              <Ionicons name="add-circle" size={18} color={colors.onBrandPrimary} />
              <Text style={styles.actionAddText}>Add Day</Text>
            </Pressable>
            <Pressable
              style={[styles.actionBtn, styles.actionUse]}
              onPress={() => setDilSheet('use')}
              testID="dil-use-button"
            >
              <Ionicons name="remove-circle" size={18} color={colors.brandPrimary} />
              <Text style={styles.actionUseText}>Use Day</Text>
            </Pressable>
          </View>
        </View>

        <Text style={styles.footNote}>
          TTB accrues from timesheet overtime (weekday {'>'}8h) plus weekend work. Adjust balances
          any time from these cards.
        </Text>
      </ScrollView>

      <TTBSheet
        mode={ttbSheet}
        onClose={() => setTtbSheet(null)}
        onSubmit={(hours, date, note) => {
          addTtbLog({
            type: ttbSheet === 'add' ? 'add' : 'use',
            hours,
            date,
            note,
            source: 'manual',
          });
          setTtbSheet(null);
        }}
      />
      <DILSheet
        mode={dilSheet}
        onClose={() => setDilSheet(null)}
        onSubmit={(date, note) => {
          addDilLog({
            type: dilSheet === 'add' ? 'add' : 'use',
            date,
            note,
          });
          setDilSheet(null);
        }}
      />
    </SafeAreaView>
  );
}

/** TTB add/use sheet with 15-min increments. */
const TTBSheet: React.FC<{
  mode: null | 'add' | 'use';
  onClose: () => void;
  onSubmit: (hours: number, date: string, note: string) => void;
}> = ({ mode, onClose, onSubmit }) => {
  const [hours, setHours] = useState(1); // increments of 0.25
  const [date, setDate] = useState(isoDate(new Date()));
  const [note, setNote] = useState('');

  React.useEffect(() => {
    if (mode) {
      setHours(1);
      setDate(isoDate(new Date()));
      setNote('');
    }
  }, [mode]);

  const isAdd = mode === 'add';
  const title = isAdd ? 'Add TTB Hours' : 'Use TTB Hours';

  return (
    <BottomSheet visible={!!mode} onClose={onClose} title={title} testID="ttb-sheet">
      <Text style={sheetStyles.label}>Hours (15-min increments)</Text>
      <View style={sheetStyles.stepperRow}>
        <Pressable
          style={sheetStyles.stepperBtn}
          onPress={() => setHours((h) => roundTo15(Math.max(0.25, h - 0.25)))}
          testID="ttb-hours-decrement"
        >
          <Ionicons name="remove" size={20} color={colors.onSurface} />
        </Pressable>
        <View style={sheetStyles.stepperValueWrap}>
          <Text style={sheetStyles.stepperValue} testID="ttb-hours-display">
            {formatHours(hours)}
          </Text>
        </View>
        <Pressable
          style={sheetStyles.stepperBtn}
          onPress={() => setHours((h) => roundTo15(h + 0.25))}
          testID="ttb-hours-increment"
        >
          <Ionicons name="add" size={20} color={colors.onSurface} />
        </Pressable>
      </View>

      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={sheetStyles.chipRow}
      >
        {[0.25, 0.5, 1, 2, 4, 8].map((v) => (
          <Pressable
            key={v}
            onPress={() => setHours(v)}
            style={[sheetStyles.chip, hours === v && sheetStyles.chipActive]}
            testID={`ttb-quick-${v}`}
          >
            <Text style={[sheetStyles.chipText, hours === v && sheetStyles.chipTextActive]}>
              {formatHours(v)}
            </Text>
          </Pressable>
        ))}
      </ScrollView>

      <Text style={sheetStyles.label}>Date</Text>
      <TextInput
        value={date}
        onChangeText={setDate}
        placeholder="YYYY-MM-DD"
        placeholderTextColor={colors.muted}
        style={sheetStyles.input}
        autoCapitalize="none"
        autoCorrect={false}
        testID="ttb-date-input"
      />
      <Text style={sheetStyles.helper}>{safeDateLabel(date)}</Text>

      <Text style={sheetStyles.label}>Notes (optional)</Text>
      <TextInput
        value={note}
        onChangeText={setNote}
        placeholder={isAdd ? 'e.g. Weekend project work' : 'e.g. Left early for appointment'}
        placeholderTextColor={colors.muted}
        style={[sheetStyles.input, { minHeight: 60 }]}
        multiline
        testID="ttb-note-input"
      />

      <Pressable
        style={sheetStyles.submitBtn}
        onPress={() => onSubmit(hours, date, note)}
        testID="ttb-sheet-submit"
      >
        <Text style={sheetStyles.submitText}>{isAdd ? 'Add Hours' : 'Log Usage'}</Text>
      </Pressable>
    </BottomSheet>
  );
};

const DILSheet: React.FC<{
  mode: null | 'add' | 'use';
  onClose: () => void;
  onSubmit: (date: string, note: string) => void;
}> = ({ mode, onClose, onSubmit }) => {
  const [date, setDate] = useState(isoDate(new Date()));
  const [note, setNote] = useState('');

  React.useEffect(() => {
    if (mode) {
      setDate(isoDate(new Date()));
      setNote('');
    }
  }, [mode]);

  const isAdd = mode === 'add';
  const title = isAdd ? 'Add Day in Lieu' : 'Use Day in Lieu';

  return (
    <BottomSheet visible={!!mode} onClose={onClose} title={title} testID="dil-sheet">
      <Text style={sheetStyles.label}>{isAdd ? 'Date awarded' : 'Day being taken'}</Text>
      <TextInput
        value={date}
        onChangeText={setDate}
        placeholder="YYYY-MM-DD"
        placeholderTextColor={colors.muted}
        style={sheetStyles.input}
        autoCapitalize="none"
        autoCorrect={false}
        testID="dil-date-input"
      />
      <Text style={sheetStyles.helper}>{safeDateLabel(date)}</Text>

      <Text style={sheetStyles.label}>Notes (optional)</Text>
      <TextInput
        value={note}
        onChangeText={setNote}
        placeholder={isAdd ? 'e.g. Public holiday worked' : 'e.g. Family day off'}
        placeholderTextColor={colors.muted}
        style={[sheetStyles.input, { minHeight: 60 }]}
        multiline
        testID="dil-note-input"
      />

      <Pressable
        style={sheetStyles.submitBtn}
        onPress={() => onSubmit(date, note)}
        testID="dil-sheet-submit"
      >
        <Text style={sheetStyles.submitText}>{isAdd ? 'Add Day' : 'Log Usage'}</Text>
      </Pressable>
    </BottomSheet>
  );
};

function safeDateLabel(iso: string) {
  const m = /^\d{4}-\d{2}-\d{2}$/.exec(iso);
  if (!m) return 'Enter as YYYY-MM-DD';
  try {
    return formatDateFull(parseIsoDate(iso));
  } catch {
    return 'Invalid date';
  }
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: colors.surface },
  container: { paddingHorizontal: spacing.lg, paddingTop: spacing.md },
  headerRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: spacing.lg,
  },
  greeting: { color: colors.muted, fontSize: fontSize.base },
  name: {
    fontSize: fontSize['2xl'],
    fontWeight: fontWeight.bold,
    color: colors.onSurface,
    marginTop: 2,
  },
  heroWrap: {
    borderRadius: radius.lg,
    overflow: 'hidden',
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOpacity: 0.08,
        shadowRadius: 12,
        shadowOffset: { width: 0, height: 6 },
      },
      android: { elevation: 3 },
      default: {},
    }),
  },
  heroBg: { width: '100%', minHeight: 210 },
  heroInner: {
    padding: spacing.xl,
    paddingBottom: spacing.lg,
  },
  heroLabel: {
    color: 'rgba(255,255,255,0.75)',
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    letterSpacing: 1.2,
    marginBottom: spacing.md,
  },
  heroRow: { flexDirection: 'row', alignItems: 'flex-end', gap: 8 },
  heroValue: {
    color: '#fff',
    fontSize: 40,
    fontWeight: fontWeight.bold,
    lineHeight: 44,
  },
  heroSubUnit: {
    color: 'rgba(255,255,255,0.85)',
    fontSize: fontSize.lg,
    marginBottom: 4,
  },
  heroDays: {
    color: '#fff',
    fontSize: fontSize['2xl'],
    fontWeight: fontWeight.semibold,
  },
  heroStats: {
    flexDirection: 'row',
    alignItems: 'center',
    marginTop: spacing.lg,
    paddingTop: spacing.md,
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: 'rgba(255,255,255,0.25)',
  },
  heroStatItem: { flex: 1 },
  heroStatDivider: {
    width: StyleSheet.hairlineWidth,
    height: 30,
    backgroundColor: 'rgba(255,255,255,0.3)',
  },
  heroStatLabel: {
    color: 'rgba(255,255,255,0.75)',
    fontSize: fontSize.sm,
    marginBottom: 2,
  },
  heroStatValue: { color: '#fff', fontSize: fontSize.lg, fontWeight: fontWeight.semibold },

  actionRow: {
    flexDirection: 'row',
    gap: spacing.md,
    marginTop: spacing.lg,
  },
  actionBtn: {
    flex: 1,
    height: 48,
    borderRadius: radius.md,
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: 6,
  },
  actionAdd: { backgroundColor: colors.brandPrimary },
  actionAddText: {
    color: colors.onBrandPrimary,
    fontWeight: fontWeight.semibold,
    fontSize: fontSize.lg,
  },
  actionUse: {
    backgroundColor: colors.brandTertiary,
    borderWidth: 1,
    borderColor: colors.brandSecondary,
  },
  actionUseText: {
    color: colors.brandPrimary,
    fontWeight: fontWeight.semibold,
    fontSize: fontSize.lg,
  },

  dilCard: {
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.lg,
    padding: spacing.lg,
    marginTop: spacing.xl,
    borderWidth: 1,
    borderColor: colors.border,
  },
  dilHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  cardLabel: {
    color: colors.muted,
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    letterSpacing: 1.2,
  },
  dilRow: { flexDirection: 'row', alignItems: 'baseline', gap: 8, marginTop: 6 },
  dilValue: { fontSize: 36, fontWeight: fontWeight.bold, color: colors.onSurface },
  dilUnit: { fontSize: fontSize.lg, color: colors.muted },
  dilOpening: { color: colors.muted, fontSize: fontSize.sm, marginTop: 4 },
  dilIconWrap: {
    width: 52,
    height: 52,
    borderRadius: radius.md,
    backgroundColor: colors.brandTertiary,
    alignItems: 'center',
    justifyContent: 'center',
  },
  footNote: {
    marginTop: spacing.xl,
    color: colors.muted,
    fontSize: fontSize.sm,
    lineHeight: 20,
  },
});

const sheetStyles = StyleSheet.create({
  label: {
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    color: colors.muted,
    letterSpacing: 0.6,
    marginTop: spacing.md,
    marginBottom: spacing.sm,
    textTransform: 'uppercase',
  },
  stepperRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: spacing.md,
  },
  stepperBtn: {
    width: 44,
    height: 44,
    borderRadius: radius.md,
    backgroundColor: colors.surfaceTertiary,
    alignItems: 'center',
    justifyContent: 'center',
  },
  stepperValueWrap: {
    flex: 1,
    height: 56,
    borderRadius: radius.md,
    backgroundColor: colors.brandTertiary,
    alignItems: 'center',
    justifyContent: 'center',
  },
  stepperValue: {
    fontSize: fontSize['2xl'],
    fontWeight: fontWeight.bold,
    color: colors.onBrandTertiary,
  },
  chipRow: { flexDirection: 'row', gap: 8, paddingVertical: spacing.md },
  chip: {
    height: 36,
    paddingHorizontal: spacing.md,
    borderRadius: radius.pill,
    borderWidth: 1,
    borderColor: colors.border,
    alignItems: 'center',
    justifyContent: 'center',
    flexShrink: 0,
    backgroundColor: colors.surfaceSecondary,
  },
  chipActive: { backgroundColor: colors.brandPrimary, borderColor: colors.brandPrimary },
  chipText: { color: colors.onSurface, fontWeight: fontWeight.medium },
  chipTextActive: { color: colors.onBrandPrimary },
  input: {
    minHeight: 44,
    borderRadius: radius.md,
    borderWidth: 1,
    borderColor: colors.border,
    paddingHorizontal: spacing.md,
    paddingVertical: spacing.sm,
    fontSize: fontSize.lg,
    color: colors.onSurface,
    backgroundColor: colors.surface,
  },
  helper: { color: colors.muted, fontSize: fontSize.sm, marginTop: 4 },
  submitBtn: {
    marginTop: spacing.xl,
    height: 52,
    borderRadius: radius.md,
    backgroundColor: colors.brandPrimary,
    alignItems: 'center',
    justifyContent: 'center',
  },
  submitText: {
    color: colors.onBrandPrimary,
    fontWeight: fontWeight.semibold,
    fontSize: fontSize.lg,
  },
});
```

---

<a id="18-frontend-app-tabs-timesheet-tsx"></a>

## 18. APP — Timesheet tab (fortnight entry)

**Path:** `frontend/app/(tabs)/timesheet.tsx`

_682 lines, 21345 bytes_

```tsx
import React, { useMemo, useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ScrollView,
  Pressable,
  TextInput,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';
import { Ionicons } from '@expo/vector-icons';

import { colors, spacing, radius, fontSize, fontWeight } from '@/src/theme/tokens';
import { useApp, defaultEntry } from '@/src/store/appStore';
import {
  fortnightStart,
  fortnightDates,
  formatDateShort,
  dayName,
  addDays,
  isoDate,
  isoWeekday,
} from '@/src/utils/date';
import {
  LEAVE_TYPES,
  LeaveType,
  TimesheetEntry,
  totalHours,
  formatHours,
  formatTime,
  timeSlots,
} from '@/src/utils/timesheet';
import { BottomSheet } from '@/src/components/BottomSheet';

const DEFAULT_ENTRY = (date: string): TimesheetEntry => defaultEntry(date);

export default function TimesheetScreen() {
  const insets = useSafeAreaInsets();
  const { entries, upsertEntry } = useApp();
  const [anchor, setAnchor] = useState<Date>(() => fortnightStart(new Date()));
  const [expandedDate, setExpandedDate] = useState<string | null>(null);
  const [timePicker, setTimePicker] = useState<
    null | { date: string; field: 'start' | 'finish' }
  >(null);
  const [leavePicker, setLeavePicker] = useState<null | string>(null);

  const days = useMemo(() => fortnightDates(anchor), [anchor]);
  const week1 = days.slice(0, 7);
  const week2 = days.slice(7, 14);

  const getEntry = (d: Date): TimesheetEntry => {
    const key = isoDate(d);
    return entries[key] || DEFAULT_ENTRY(key);
  };

  const week1Total = week1.reduce((sum, d) => sum + totalHours(getEntry(d)), 0);
  const week2Total = week2.reduce((sum, d) => sum + totalHours(getEntry(d)), 0);
  const fortnightTotal = week1Total + week2Total;

  const rangeLabel = `${formatDateShort(anchor)} – ${formatDateShort(addDays(anchor, 13))}`;

  const activeEntry = timePicker
    ? entries[timePicker.date] || DEFAULT_ENTRY(timePicker.date)
    : null;
  const leaveEntry = leavePicker
    ? entries[leavePicker] || DEFAULT_ENTRY(leavePicker)
    : null;

  return (
    <SafeAreaView style={styles.safe} edges={['top']} testID="timesheet-screen">
      <View style={styles.stickyHeader}>
        <View style={styles.headerTopRow}>
          <Pressable
            onPress={() => setAnchor((a) => addDays(a, -14))}
            style={styles.navBtn}
            hitSlop={12}
            testID="fortnight-prev"
          >
            <Ionicons name="chevron-back" size={22} color={colors.brandPrimary} />
          </Pressable>
          <View style={styles.rangeWrap}>
            <Text style={styles.rangeLabel}>FORTNIGHT</Text>
            <Text style={styles.rangeValue} testID="fortnight-range">
              {rangeLabel}
            </Text>
          </View>
          <Pressable
            onPress={() => setAnchor((a) => addDays(a, 14))}
            style={styles.navBtn}
            hitSlop={12}
            testID="fortnight-next"
          >
            <Ionicons name="chevron-forward" size={22} color={colors.brandPrimary} />
          </Pressable>
        </View>
        <Pressable
          onPress={() => setAnchor(fortnightStart(new Date()))}
          style={styles.todayBtn}
          testID="fortnight-today"
        >
          <Text style={styles.todayText}>Jump to today</Text>
        </Pressable>
      </View>

      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        keyboardVerticalOffset={80}
      >
        <ScrollView
          contentContainerStyle={[
            styles.container,
            { paddingBottom: insets.bottom + spacing['3xl'] },
          ]}
          showsVerticalScrollIndicator={false}
          keyboardShouldPersistTaps="handled"
        >
          <WeekSection
            title="Week 1"
            subtitle={`${formatDateShort(week1[0])} – ${formatDateShort(week1[6])}`}
            total={week1Total}
            days={week1}
            getEntry={getEntry}
            upsertEntry={upsertEntry}
            expandedDate={expandedDate}
            setExpandedDate={setExpandedDate}
            onOpenTime={(d, f) => setTimePicker({ date: isoDate(d), field: f })}
            onOpenLeave={(d) => setLeavePicker(isoDate(d))}
          />

          <WeekSection
            title="Week 2 (pay week)"
            subtitle={`${formatDateShort(week2[0])} – ${formatDateShort(week2[6])}`}
            total={week2Total}
            days={week2}
            getEntry={getEntry}
            upsertEntry={upsertEntry}
            expandedDate={expandedDate}
            setExpandedDate={setExpandedDate}
            onOpenTime={(d, f) => setTimePicker({ date: isoDate(d), field: f })}
            onOpenLeave={(d) => setLeavePicker(isoDate(d))}
          />

          <View style={styles.totalCard} testID="fortnight-total-card">
            <View style={styles.totalRow}>
              <Text style={styles.totalLabel}>Week 1 total</Text>
              <Text style={styles.totalValue}>{formatHours(week1Total)}</Text>
            </View>
            <View style={styles.totalRow}>
              <Text style={styles.totalLabel}>Week 2 total</Text>
              <Text style={styles.totalValue}>{formatHours(week2Total)}</Text>
            </View>
            <View style={[styles.totalRow, styles.totalRowStrong]}>
              <Text style={styles.totalLabelStrong}>Fortnight total</Text>
              <Text style={styles.totalValueStrong} testID="fortnight-total-value">
                {formatHours(fortnightTotal)}
              </Text>
            </View>
          </View>
        </ScrollView>
      </KeyboardAvoidingView>

      {/* Time picker sheet */}
      <BottomSheet
        visible={!!timePicker}
        onClose={() => setTimePicker(null)}
        title={timePicker?.field === 'start' ? 'Start time' : 'Finish time'}
        testID="time-picker-sheet"
      >
        <View style={pickerStyles.grid}>
          {timeSlots().map((t) => {
            const selected =
              activeEntry &&
              (timePicker!.field === 'start' ? activeEntry.start : activeEntry.finish) === t;
            return (
              <Pressable
                key={t}
                style={[pickerStyles.timeCell, selected && pickerStyles.timeCellActive]}
                onPress={() => {
                  if (!activeEntry) return;
                  upsertEntry({ ...activeEntry, [timePicker!.field]: t });
                  setTimePicker(null);
                }}
                testID={`time-slot-${t}`}
              >
                <Text
                  style={[pickerStyles.timeText, selected && pickerStyles.timeTextActive]}
                >
                  {formatTime(t)}
                </Text>
              </Pressable>
            );
          })}
        </View>
      </BottomSheet>

      {/* Leave type picker sheet */}
      <BottomSheet
        visible={!!leavePicker}
        onClose={() => setLeavePicker(null)}
        title="Leave type"
        testID="leave-picker-sheet"
      >
        {LEAVE_TYPES.map((lt) => {
          const selected = leaveEntry?.leaveType === lt;
          return (
            <Pressable
              key={lt}
              style={[pickerStyles.leaveRow, selected && pickerStyles.leaveRowActive]}
              onPress={() => {
                if (!leaveEntry) return;
                upsertEntry({ ...leaveEntry, leaveType: lt });
                setLeavePicker(null);
              }}
              testID={`leave-option-${lt}`}
            >
              <Text style={pickerStyles.leaveText}>{lt}</Text>
              {selected ? (
                <Ionicons name="checkmark" size={22} color={colors.brandPrimary} />
              ) : null}
            </Pressable>
          );
        })}
      </BottomSheet>
    </SafeAreaView>
  );
}

interface WeekSectionProps {
  title: string;
  subtitle: string;
  total: number;
  days: Date[];
  getEntry: (d: Date) => TimesheetEntry;
  upsertEntry: (e: TimesheetEntry) => void;
  expandedDate: string | null;
  setExpandedDate: (d: string | null) => void;
  onOpenTime: (d: Date, f: 'start' | 'finish') => void;
  onOpenLeave: (d: Date) => void;
}

const WeekSection: React.FC<WeekSectionProps> = ({
  title,
  subtitle,
  total,
  days,
  getEntry,
  upsertEntry,
  expandedDate,
  setExpandedDate,
  onOpenTime,
  onOpenLeave,
}) => {
  return (
    <View style={styles.weekSection}>
      <View style={styles.weekHeader}>
        <View>
          <Text style={styles.weekTitle}>{title}</Text>
          <Text style={styles.weekSubtitle}>{subtitle}</Text>
        </View>
        <View style={styles.weekTotalPill}>
          <Text style={styles.weekTotalText}>{formatHours(total)}</Text>
        </View>
      </View>

      {days.map((d) => {
        const entry = getEntry(d);
        const key = isoDate(d);
        const expanded = expandedDate === key;
        const hours = totalHours(entry);
        const wd = isoWeekday(d);
        const weekend = wd === 6 || wd === 7;
        return (
          <View key={key} style={[styles.dayCard, weekend && styles.dayCardWeekend]}>
            <View style={styles.dayHeaderRow}>
              <View style={styles.dayDateBlock}>
                <Text style={styles.dayDayName}>{dayName(d)}</Text>
                <Text style={styles.dayDate}>{formatDateShort(d)}</Text>
              </View>
              <View style={styles.dayTotalBlock}>
                <Text style={styles.dayTotalLabel}>Hours</Text>
                <Text
                  style={[styles.dayTotalValue, hours < 0 && { color: colors.error }]}
                  testID={`day-total-${key}`}
                >
                  {formatHours(hours)}
                </Text>
              </View>
            </View>

            <View style={styles.dayControls}>
              <Pressable
                style={styles.pill}
                onPress={() => onOpenTime(d, 'start')}
                disabled={entry.leaveType !== 'Work'}
                testID={`day-start-${key}`}
              >
                <Text style={styles.pillLabel}>Start</Text>
                <Text
                  style={[
                    styles.pillValue,
                    entry.leaveType !== 'Work' && styles.pillValueDisabled,
                  ]}
                >
                  {formatTime(entry.start)}
                </Text>
              </Pressable>
              <View style={styles.pillDivider} />
              <Pressable
                style={styles.pill}
                onPress={() => onOpenTime(d, 'finish')}
                disabled={entry.leaveType !== 'Work'}
                testID={`day-finish-${key}`}
              >
                <Text style={styles.pillLabel}>Finish</Text>
                <Text
                  style={[
                    styles.pillValue,
                    entry.leaveType !== 'Work' && styles.pillValueDisabled,
                  ]}
                >
                  {formatTime(entry.finish)}
                </Text>
              </Pressable>
            </View>

            <Pressable
              style={styles.leaveRow}
              onPress={() => onOpenLeave(d)}
              testID={`day-leave-${key}`}
            >
              <Text style={styles.leaveLabel}>Leave type</Text>
              <View style={styles.leaveValueRow}>
                <View
                  style={[
                    styles.leaveBadge,
                    { backgroundColor: badgeColor(entry.leaveType) },
                  ]}
                >
                  <Text style={styles.leaveBadgeText}>{entry.leaveType}</Text>
                </View>
                <Ionicons name="chevron-forward" size={16} color={colors.muted} />
              </View>
            </Pressable>

            <Pressable
              onPress={() => setExpandedDate(expanded ? null : key)}
              style={styles.expandBtn}
              testID={`day-expand-${key}`}
            >
              <Text style={styles.expandText}>
                {expanded ? 'Hide notes / TTB' : 'Add notes or TTB used'}
              </Text>
              <Ionicons
                name={expanded ? 'chevron-up' : 'chevron-down'}
                size={16}
                color={colors.brandPrimary}
              />
            </Pressable>

            {expanded ? (
              <View style={styles.expandedBlock}>
                {entry.leaveType === 'TTB' ? (
                  <View style={styles.expandedField}>
                    <Text style={styles.fieldLabel}>TTB hours used</Text>
                    <TextInput
                      style={styles.textInput}
                      value={String(entry.ttbUsed || '')}
                      onChangeText={(v) => {
                        const n = parseFloat(v);
                        upsertEntry({
                          ...entry,
                          ttbUsed: Number.isFinite(n) ? n : 0,
                        });
                      }}
                      keyboardType="decimal-pad"
                      placeholder="0"
                      placeholderTextColor={colors.muted}
                      testID={`day-ttb-input-${key}`}
                    />
                  </View>
                ) : null}
                <View style={styles.expandedField}>
                  <Text style={styles.fieldLabel}>Notes</Text>
                  <TextInput
                    style={[styles.textInput, { minHeight: 60 }]}
                    value={entry.notes}
                    onChangeText={(v) => upsertEntry({ ...entry, notes: v })}
                    placeholder="Optional note"
                    placeholderTextColor={colors.muted}
                    multiline
                    testID={`day-notes-input-${key}`}
                  />
                </View>
              </View>
            ) : null}
          </View>
        );
      })}
    </View>
  );
};

function badgeColor(t: LeaveType) {
  switch (t) {
    case 'Work':
      return colors.brandTertiary;
    case 'Public Holiday':
      return '#FFE9C7';
    case 'RDO':
      return '#D6ECFF';
    case 'TTB':
      return '#FFD9D6';
    case 'Annual Leave':
      return '#E4D7FF';
    case 'Sick Leave':
      return '#F9D6E0';
    default:
      return colors.surfaceTertiary;
  }
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: colors.surface },
  stickyHeader: {
    backgroundColor: colors.surface,
    paddingHorizontal: spacing.lg,
    paddingBottom: spacing.md,
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: colors.divider,
  },
  headerTopRow: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingTop: spacing.sm,
  },
  navBtn: {
    width: 44,
    height: 44,
    borderRadius: radius.md,
    backgroundColor: colors.brandTertiary,
    alignItems: 'center',
    justifyContent: 'center',
  },
  rangeWrap: { alignItems: 'center' },
  rangeLabel: {
    color: colors.muted,
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    letterSpacing: 1.2,
  },
  rangeValue: {
    color: colors.onSurface,
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    marginTop: 2,
  },
  todayBtn: { alignSelf: 'center', marginTop: spacing.sm, padding: spacing.xs },
  todayText: { color: colors.brandPrimary, fontWeight: fontWeight.semibold, fontSize: fontSize.sm },

  container: {
    paddingHorizontal: spacing.lg,
    paddingTop: spacing.md,
  },
  weekSection: { marginBottom: spacing.lg },
  weekHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: spacing.sm,
    paddingHorizontal: spacing.xs,
  },
  weekTitle: {
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    color: colors.onSurface,
  },
  weekSubtitle: {
    fontSize: fontSize.sm,
    color: colors.muted,
    marginTop: 2,
  },
  weekTotalPill: {
    paddingHorizontal: spacing.md,
    paddingVertical: 6,
    borderRadius: radius.pill,
    backgroundColor: colors.brandPrimary,
  },
  weekTotalText: {
    color: colors.onBrandPrimary,
    fontWeight: fontWeight.semibold,
    fontSize: fontSize.base,
  },

  dayCard: {
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.md,
    padding: spacing.md,
    marginBottom: spacing.sm,
    borderWidth: 1,
    borderColor: colors.border,
  },
  dayCardWeekend: { backgroundColor: '#FAF6F0' },
  dayHeaderRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'flex-start',
    marginBottom: spacing.sm,
  },
  dayDateBlock: { flex: 1 },
  dayDayName: {
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    color: colors.onSurface,
  },
  dayDate: { fontSize: fontSize.sm, color: colors.muted, marginTop: 2 },
  dayTotalBlock: { alignItems: 'flex-end' },
  dayTotalLabel: { fontSize: fontSize.sm, color: colors.muted },
  dayTotalValue: {
    fontSize: fontSize.lg,
    fontWeight: fontWeight.bold,
    color: colors.brandPrimary,
    marginTop: 2,
  },
  dayControls: {
    flexDirection: 'row',
    alignItems: 'stretch',
    backgroundColor: colors.surface,
    borderRadius: radius.md,
    borderWidth: 1,
    borderColor: colors.border,
    padding: 4,
  },
  pill: {
    flex: 1,
    paddingHorizontal: spacing.sm,
    paddingVertical: spacing.sm,
    borderRadius: radius.sm,
    alignItems: 'flex-start',
    minHeight: 48,
    justifyContent: 'center',
  },
  pillLabel: { color: colors.muted, fontSize: fontSize.sm },
  pillValue: {
    color: colors.onSurface,
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    marginTop: 2,
  },
  pillValueDisabled: { color: colors.muted },
  pillDivider: {
    width: StyleSheet.hairlineWidth,
    backgroundColor: colors.border,
    alignSelf: 'stretch',
    marginVertical: 6,
  },
  leaveRow: {
    marginTop: spacing.sm,
    paddingVertical: spacing.sm,
    paddingHorizontal: spacing.sm,
    borderRadius: radius.sm,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: colors.surface,
    borderWidth: 1,
    borderColor: colors.border,
    minHeight: 44,
  },
  leaveLabel: { color: colors.muted, fontSize: fontSize.base },
  leaveValueRow: { flexDirection: 'row', alignItems: 'center', gap: 6 },
  leaveBadge: {
    paddingHorizontal: spacing.sm,
    paddingVertical: 4,
    borderRadius: radius.pill,
  },
  leaveBadgeText: { fontSize: fontSize.sm, fontWeight: fontWeight.semibold, color: '#3A3A3C' },
  expandBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 4,
    marginTop: spacing.sm,
    paddingVertical: 6,
  },
  expandText: { color: colors.brandPrimary, fontWeight: fontWeight.medium, fontSize: fontSize.sm },
  expandedBlock: {
    marginTop: spacing.sm,
    paddingTop: spacing.sm,
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: colors.divider,
    gap: spacing.sm,
  },
  expandedField: {},
  fieldLabel: {
    fontSize: fontSize.sm,
    color: colors.muted,
    marginBottom: 4,
    fontWeight: fontWeight.medium,
  },
  textInput: {
    minHeight: 44,
    borderRadius: radius.sm,
    borderWidth: 1,
    borderColor: colors.border,
    paddingHorizontal: spacing.md,
    paddingVertical: spacing.sm,
    fontSize: fontSize.base,
    color: colors.onSurface,
    backgroundColor: colors.surface,
  },

  totalCard: {
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.lg,
    padding: spacing.lg,
    borderWidth: 1,
    borderColor: colors.border,
    marginTop: spacing.md,
  },
  totalRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 6,
  },
  totalRowStrong: {
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: colors.divider,
    marginTop: 6,
    paddingTop: spacing.sm,
  },
  totalLabel: { color: colors.muted, fontSize: fontSize.base },
  totalValue: { color: colors.onSurface, fontSize: fontSize.base, fontWeight: fontWeight.medium },
  totalLabelStrong: {
    color: colors.onSurface,
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
  },
  totalValueStrong: {
    color: colors.brandPrimary,
    fontSize: fontSize.xl,
    fontWeight: fontWeight.bold,
  },
});

const pickerStyles = StyleSheet.create({
  grid: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,
  },
  timeCell: {
    width: '31%',
    height: 44,
    borderRadius: radius.sm,
    borderWidth: 1,
    borderColor: colors.border,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: colors.surface,
  },
  timeCellActive: {
    backgroundColor: colors.brandPrimary,
    borderColor: colors.brandPrimary,
  },
  timeText: { color: colors.onSurface, fontSize: fontSize.base, fontWeight: fontWeight.medium },
  timeTextActive: { color: colors.onBrandPrimary },
  leaveRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: spacing.md,
    paddingHorizontal: spacing.md,
    borderRadius: radius.sm,
    minHeight: 52,
  },
  leaveRowActive: { backgroundColor: colors.brandTertiary },
  leaveText: { color: colors.onSurface, fontSize: fontSize.lg, fontWeight: fontWeight.medium },
});
```

---

<a id="19-frontend-app-tabs-settings-tsx"></a>

## 19. APP — Settings tab

**Path:** `frontend/app/(tabs)/settings.tsx`

_379 lines, 12320 bytes_

```tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ScrollView,
  Pressable,
  TextInput,
  Platform,
} from 'react-native';
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';
import { Image } from 'expo-image';
import { Ionicons } from '@expo/vector-icons';
import { useRouter } from 'expo-router';

import { colors, spacing, radius, fontSize, fontWeight } from '@/src/theme/tokens';
import { useApp } from '@/src/store/appStore';
import { formatHours } from '@/src/utils/timesheet';

const AVATAR =
  'https://images.unsplash.com/photo-1560250097-0b93528c311a?crop=entropy&cs=srgb&fm=jpg&ixid=M3w4NjA1NDh8MHwxfHNlYXJjaHwxfHxwcm9mZXNzaW9uYWwlMjBlbXBsb3llZSUyMGhlYWRzaG90JTIwYXZhdGFyfGVufDB8fHx8MTc4Mjk3OTUxMHww&ixlib=rb-4.1.0&q=85';

export default function SettingsScreen() {
  const insets = useSafeAreaInsets();
  const router = useRouter();
  const {
    profile,
    setProfile,
    openingTTB,
    setOpeningTTB,
    openingDayInLieu,
    setOpeningDayInLieu,
    ttbBalance,
    dayInLieuBalance,
    ttbLog,
    dilLog,
    resetAll,
  } = useApp();

  const [staffName, setStaffName] = useState(profile.staffName);
  const [managerName, setManagerName] = useState(profile.managerName);
  const [role, setRole] = useState(profile.role);
  const [openingTtbStr, setOpeningTtbStr] = useState(String(openingTTB));
  const [openingDilStr, setOpeningDilStr] = useState(String(openingDayInLieu));
  const [confirmReset, setConfirmReset] = useState(false);

  React.useEffect(() => {
    setStaffName(profile.staffName);
    setManagerName(profile.managerName);
    setRole(profile.role);
  }, [profile.staffName, profile.managerName, profile.role]);

  const commitProfile = () => {
    setProfile({ staffName, managerName, role });
  };

  const commitOpeningTtb = () => {
    const n = parseFloat(openingTtbStr);
    setOpeningTTB(Number.isFinite(n) ? n : 0);
  };
  const commitOpeningDil = () => {
    const n = parseFloat(openingDilStr);
    setOpeningDayInLieu(Number.isFinite(n) ? n : 0);
  };

  return (
    <SafeAreaView style={styles.safe} edges={['top']} testID="settings-screen">
      <ScrollView
        contentContainerStyle={[
          styles.container,
          { paddingBottom: insets.bottom + spacing['3xl'] },
        ]}
        showsVerticalScrollIndicator={false}
      >
        <Text style={styles.screenTitle}>Settings</Text>

        <View style={styles.profileCard}>
          <Image source={AVATAR} style={styles.avatar} contentFit="cover" />
          <View style={{ flex: 1 }}>
            <Text style={styles.profileName} testID="settings-current-name">
              {staffName || 'Enter staff name'}
            </Text>
            <Text style={styles.profileRole}>{role || 'No role set'}</Text>
            <Text style={styles.profileManager}>
              Manager: {managerName || 'not set'}
            </Text>
          </View>
        </View>

        <Section title="Employee">
          <Field
            label="Staff name"
            value={staffName}
            onChangeText={setStaffName}
            onBlur={commitProfile}
            placeholder="e.g. David Andrews"
            testID="staff-name-input"
          />
          <Field
            label="Manager name"
            value={managerName}
            onChangeText={setManagerName}
            onBlur={commitProfile}
            placeholder="e.g. Connor Wilson"
            testID="manager-name-input"
          />
          <Field
            label="Role"
            value={role}
            onChangeText={setRole}
            onBlur={commitProfile}
            placeholder="e.g. Site Supervisor"
            testID="role-input"
            last
          />
        </Section>

        <Section title="Opening balances">
          <Field
            label="Opening TTB (hours)"
            value={openingTtbStr}
            onChangeText={setOpeningTtbStr}
            onBlur={commitOpeningTtb}
            placeholder="0"
            keyboardType="decimal-pad"
            testID="opening-ttb-input"
          />
          <Field
            label="Opening Day in Lieu"
            value={openingDilStr}
            onChangeText={setOpeningDilStr}
            onBlur={commitOpeningDil}
            placeholder="0"
            keyboardType="number-pad"
            testID="opening-dil-input"
            last
          />
          <View style={styles.balanceRow}>
            <Text style={styles.balanceLabel}>Current TTB balance</Text>
            <Text style={styles.balanceValue}>{formatHours(ttbBalance)}</Text>
          </View>
          <View style={styles.balanceRow}>
            <Text style={styles.balanceLabel}>Current Day in Lieu</Text>
            <Text style={styles.balanceValue}>
              {dayInLieuBalance} {Math.abs(dayInLieuBalance) === 1 ? 'day' : 'days'}
            </Text>
          </View>
        </Section>

        <Section title="Logs">
          <NavRow
            label="TTB usage log"
            trailing={`${ttbLog.length}`}
            onPress={() => router.push('/logs/ttb')}
            testID="ttb-log-link"
          />
          <NavRow
            label="Day in Lieu log"
            trailing={`${dilLog.length}`}
            onPress={() => router.push('/logs/dil')}
            last
            testID="dil-log-link"
          />
        </Section>

        <Section title="Danger zone">
          {confirmReset ? (
            <View style={styles.resetConfirm}>
              <Text style={styles.resetWarn}>
                This will erase all timesheet entries, logs, and settings on this device.
              </Text>
              <View style={{ flexDirection: 'row', gap: spacing.sm, marginTop: spacing.sm }}>
                <Pressable
                  style={[styles.dangerBtn, styles.dangerCancel]}
                  onPress={() => setConfirmReset(false)}
                  testID="reset-cancel"
                >
                  <Text style={styles.dangerCancelText}>Cancel</Text>
                </Pressable>
                <Pressable
                  style={[styles.dangerBtn, styles.dangerConfirm]}
                  onPress={() => {
                    resetAll();
                    setConfirmReset(false);
                  }}
                  testID="reset-confirm"
                >
                  <Text style={styles.dangerConfirmText}>Reset everything</Text>
                </Pressable>
              </View>
            </View>
          ) : (
            <Pressable
              style={styles.resetBtn}
              onPress={() => setConfirmReset(true)}
              testID="reset-open"
            >
              <Ionicons name="trash-outline" size={18} color={colors.error} />
              <Text style={styles.resetBtnText}>Reset all data</Text>
            </Pressable>
          )}
        </Section>

        <Text style={styles.version}>BalanceTrack v1.0</Text>
      </ScrollView>
    </SafeAreaView>
  );
}

const Section: React.FC<{ title: string; children: React.ReactNode }> = ({ title, children }) => (
  <View style={styles.section}>
    <Text style={styles.sectionTitle}>{title}</Text>
    <View style={styles.sectionCard}>{children}</View>
  </View>
);

const Field: React.FC<{
  label: string;
  value: string;
  onChangeText: (v: string) => void;
  onBlur?: () => void;
  placeholder?: string;
  keyboardType?: 'default' | 'decimal-pad' | 'number-pad';
  last?: boolean;
  testID?: string;
}> = ({ label, value, onChangeText, onBlur, placeholder, keyboardType, last, testID }) => (
  <View style={[styles.fieldRow, !last && styles.fieldRowDivider]}>
    <Text style={styles.fieldLabel}>{label}</Text>
    <TextInput
      style={styles.fieldInput}
      value={value}
      onChangeText={onChangeText}
      onBlur={onBlur}
      placeholder={placeholder}
      placeholderTextColor={colors.muted}
      keyboardType={keyboardType || 'default'}
      autoCapitalize={keyboardType ? 'none' : 'words'}
      autoCorrect={false}
      testID={testID}
    />
  </View>
);

const NavRow: React.FC<{
  label: string;
  trailing?: string;
  onPress: () => void;
  last?: boolean;
  testID?: string;
}> = ({ label, trailing, onPress, last, testID }) => (
  <Pressable
    style={[styles.navRow, !last && styles.fieldRowDivider]}
    onPress={onPress}
    testID={testID}
  >
    <Text style={styles.navLabel}>{label}</Text>
    <View style={styles.navTrailing}>
      {trailing ? <Text style={styles.navCount}>{trailing}</Text> : null}
      <Ionicons name="chevron-forward" size={18} color={colors.muted} />
    </View>
  </Pressable>
);

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: colors.surface },
  container: { paddingHorizontal: spacing.lg, paddingTop: spacing.md },
  screenTitle: {
    fontSize: fontSize['2xl'],
    fontWeight: fontWeight.bold,
    color: colors.onSurface,
    marginBottom: spacing.md,
  },
  profileCard: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: spacing.md,
    padding: spacing.md,
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.lg,
    borderWidth: 1,
    borderColor: colors.border,
    marginBottom: spacing.lg,
  },
  avatar: { width: 60, height: 60, borderRadius: 30, backgroundColor: colors.surfaceTertiary },
  profileName: { fontSize: fontSize.lg, fontWeight: fontWeight.semibold, color: colors.onSurface },
  profileRole: { fontSize: fontSize.base, color: colors.brandPrimary, marginTop: 2 },
  profileManager: { fontSize: fontSize.sm, color: colors.muted, marginTop: 4 },

  section: { marginBottom: spacing.lg },
  sectionTitle: {
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    color: colors.muted,
    letterSpacing: 1.2,
    textTransform: 'uppercase',
    marginBottom: spacing.sm,
    paddingHorizontal: spacing.xs,
  },
  sectionCard: {
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.md,
    borderWidth: 1,
    borderColor: colors.border,
    overflow: 'hidden',
  },

  fieldRow: {
    paddingHorizontal: spacing.md,
    paddingVertical: spacing.sm,
    minHeight: 56,
    justifyContent: 'center',
  },
  fieldRowDivider: {
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: colors.divider,
  },
  fieldLabel: { fontSize: fontSize.sm, color: colors.muted, marginBottom: 2 },
  fieldInput: {
    fontSize: fontSize.lg,
    color: colors.onSurface,
    paddingVertical: Platform.OS === 'android' ? 2 : 6,
  },

  balanceRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    paddingHorizontal: spacing.md,
    paddingVertical: spacing.md,
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: colors.divider,
    backgroundColor: colors.brandTertiary,
  },
  balanceLabel: { color: colors.onBrandTertiary, fontWeight: fontWeight.medium },
  balanceValue: { color: colors.onBrandTertiary, fontWeight: fontWeight.bold },

  navRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingHorizontal: spacing.md,
    minHeight: 52,
  },
  navLabel: { fontSize: fontSize.lg, color: colors.onSurface },
  navTrailing: { flexDirection: 'row', alignItems: 'center', gap: 6 },
  navCount: {
    backgroundColor: colors.surfaceTertiary,
    color: colors.onSurfaceTertiary,
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: radius.pill,
    fontSize: fontSize.sm,
    fontWeight: fontWeight.semibold,
    overflow: 'hidden',
  },

  resetBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: 8,
    paddingVertical: spacing.md,
  },
  resetBtnText: { color: colors.error, fontWeight: fontWeight.semibold, fontSize: fontSize.lg },
  resetConfirm: { padding: spacing.md },
  resetWarn: { color: colors.onSurface, fontSize: fontSize.base },
  dangerBtn: {
    flex: 1,
    height: 44,
    borderRadius: radius.md,
    alignItems: 'center',
    justifyContent: 'center',
  },
  dangerCancel: { backgroundColor: colors.surfaceTertiary },
  dangerCancelText: { color: colors.onSurface, fontWeight: fontWeight.medium },
  dangerConfirm: { backgroundColor: colors.error },
  dangerConfirmText: { color: colors.onError, fontWeight: fontWeight.semibold },

  version: { textAlign: 'center', color: colors.muted, marginTop: spacing.md },
});
```

---

<a id="20-frontend-app-logs-type-tsx"></a>

## 20. APP — Log detail screen (/logs/ttb, /logs/dil)

**Path:** `frontend/app/logs/[type].tsx`

_167 lines, 5850 bytes_

```tsx
import React from 'react';
import { View, Text, StyleSheet, ScrollView, Pressable } from 'react-native';
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';
import { Stack, useLocalSearchParams, useRouter } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

import { colors, spacing, radius, fontSize, fontWeight } from '@/src/theme/tokens';
import { useApp } from '@/src/store/appStore';
import { formatHours } from '@/src/utils/timesheet';
import { formatDateFull, parseIsoDate } from '@/src/utils/date';

export default function LogScreen() {
  const insets = useSafeAreaInsets();
  const router = useRouter();
  const { type } = useLocalSearchParams<{ type: string }>();
  const { ttbLog, dilLog, deleteTtbLog, deleteDilLog } = useApp();

  const isTtb = type === 'ttb';
  const title = isTtb ? 'TTB usage log' : 'Day in Lieu log';
  const items = isTtb ? ttbLog : dilLog;
  const empty = items.length === 0;

  return (
    <SafeAreaView style={styles.safe} edges={['top']}>
      <Stack.Screen options={{ headerShown: false }} />
      <View style={styles.header}>
        <Pressable
          onPress={() => router.back()}
          hitSlop={12}
          style={styles.backBtn}
          testID="log-back"
        >
          <Ionicons name="chevron-back" size={22} color={colors.brandPrimary} />
          <Text style={styles.backText}>Settings</Text>
        </Pressable>
        <Text style={styles.title} testID="log-title">{title}</Text>
      </View>

      <ScrollView
        contentContainerStyle={[
          styles.container,
          { paddingBottom: insets.bottom + spacing.xl },
        ]}
      >
        {empty ? (
          <View style={styles.emptyBox} testID="log-empty">
            <Ionicons name="document-text-outline" size={40} color={colors.muted} />
            <Text style={styles.emptyTitle}>No entries yet</Text>
            <Text style={styles.emptySub}>
              {isTtb
                ? 'Add or use TTB from the Dashboard to see entries here.'
                : 'Add or use a Day in Lieu from the Dashboard to see entries here.'}
            </Text>
          </View>
        ) : (
          items.map((it: any) => {
            const dateLabel = safeDateLabel(it.date);
            const isAdd = it.type === 'add';
            return (
              <View key={it.id} style={styles.row} testID={`log-item-${it.id}`}>
                <View
                  style={[
                    styles.badge,
                    { backgroundColor: isAdd ? colors.brandTertiary : '#FFE0DE' },
                  ]}
                >
                  <Ionicons
                    name={isAdd ? 'add' : 'remove'}
                    size={18}
                    color={isAdd ? colors.brandPrimary : colors.error}
                  />
                </View>
                <View style={{ flex: 1 }}>
                  <Text style={styles.rowTitle}>
                    {isAdd ? 'Added ' : 'Used '}
                    {isTtb ? formatHours((it as any).hours) : '1 day'}
                  </Text>
                  <Text style={styles.rowMeta}>{dateLabel}</Text>
                  {it.note ? <Text style={styles.rowNote}>{it.note}</Text> : null}
                  {isTtb && (it as any).source === 'timesheet' ? (
                    <Text style={styles.rowSource}>via timesheet</Text>
                  ) : null}
                </View>
                <Pressable
                  onPress={() => (isTtb ? deleteTtbLog(it.id) : deleteDilLog(it.id))}
                  hitSlop={10}
                  style={styles.deleteBtn}
                  testID={`log-delete-${it.id}`}
                >
                  <Ionicons name="close" size={18} color={colors.muted} />
                </Pressable>
              </View>
            );
          })
        )}
      </ScrollView>
    </SafeAreaView>
  );
}

function safeDateLabel(iso: string) {
  try {
    return formatDateFull(parseIsoDate(iso));
  } catch {
    return iso;
  }
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: colors.surface },
  header: {
    paddingHorizontal: spacing.lg,
    paddingTop: spacing.sm,
    paddingBottom: spacing.md,
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: colors.divider,
  },
  backBtn: { flexDirection: 'row', alignItems: 'center', gap: 2, paddingVertical: 4 },
  backText: { color: colors.brandPrimary, fontSize: fontSize.lg },
  title: {
    fontSize: fontSize['2xl'],
    fontWeight: fontWeight.bold,
    color: colors.onSurface,
    marginTop: spacing.sm,
  },
  container: { paddingHorizontal: spacing.lg, paddingTop: spacing.md },
  emptyBox: {
    alignItems: 'center',
    paddingVertical: spacing['3xl'],
    gap: spacing.sm,
  },
  emptyTitle: {
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    color: colors.onSurface,
  },
  emptySub: { color: colors.muted, textAlign: 'center', paddingHorizontal: spacing.xl },
  row: {
    flexDirection: 'row',
    alignItems: 'flex-start',
    gap: spacing.md,
    padding: spacing.md,
    backgroundColor: colors.surfaceSecondary,
    borderRadius: radius.md,
    borderWidth: 1,
    borderColor: colors.border,
    marginBottom: spacing.sm,
  },
  badge: {
    width: 36,
    height: 36,
    borderRadius: radius.pill,
    alignItems: 'center',
    justifyContent: 'center',
  },
  rowTitle: { fontSize: fontSize.lg, fontWeight: fontWeight.semibold, color: colors.onSurface },
  rowMeta: { color: colors.muted, fontSize: fontSize.sm, marginTop: 2 },
  rowNote: { color: colors.onSurface, fontSize: fontSize.base, marginTop: 6 },
  rowSource: { color: colors.brandPrimary, fontSize: fontSize.sm, marginTop: 4 },
  deleteBtn: {
    width: 32,
    height: 32,
    borderRadius: radius.pill,
    alignItems: 'center',
    justifyContent: 'center',
  },
});
```

---

<a id="21-frontend-src-theme-tokens-ts"></a>

## 21. SRC — Design tokens (colours, spacing, type)

**Path:** `frontend/src/theme/tokens.ts`

_59 lines, 1072 bytes_

```ts
// Design tokens for BalanceTrack app
export const colors = {
  surface: '#F7F7F7',
  onSurface: '#1C1C1E',
  surfaceSecondary: '#FFFFFF',
  onSurfaceSecondary: '#1C1C1E',
  surfaceTertiary: '#EAEAEA',
  onSurfaceTertiary: '#3A3A3C',
  surfaceInverse: '#1C1C1E',
  onSurfaceInverse: '#FFFFFF',
  brand: '#5B7C65',
  brandPrimary: '#5B7C65',
  onBrandPrimary: '#FFFFFF',
  brandSecondary: '#89A291',
  onBrandSecondary: '#FFFFFF',
  brandTertiary: '#E6EBE7',
  onBrandTertiary: '#2E4034',
  success: '#34C759',
  warning: '#FF9500',
  error: '#FF3B30',
  info: '#8E8E93',
  muted: '#8E8E93',
  border: '#E5E5EA',
  borderStrong: '#C7C7CC',
  divider: '#E5E5EA',
};

export const spacing = {
  xs: 4,
  sm: 8,
  md: 12,
  lg: 16,
  xl: 24,
  '2xl': 32,
  '3xl': 48,
};

export const radius = {
  sm: 6,
  md: 12,
  lg: 20,
  pill: 999,
};

export const fontSize = {
  sm: 12,
  base: 14,
  lg: 16,
  xl: 20,
  '2xl': 24,
  '3xl': 32,
};

export const fontWeight = {
  regular: '400' as const,
  medium: '500' as const,
  semibold: '600' as const,
  bold: '700' as const,
};
```

---

<a id="22-frontend-src-components-bottomsheet-tsx"></a>

## 22. SRC — BottomSheet component

**Path:** `frontend/src/components/BottomSheet.tsx`

_107 lines, 2835 bytes_

```tsx
import React from 'react';
import { Modal, View, Text, StyleSheet, Pressable, ScrollView, Platform } from 'react-native';
import { colors, spacing, radius, fontSize, fontWeight } from '@/src/theme/tokens';

interface Props {
  visible: boolean;
  onClose: () => void;
  title?: string;
  children: React.ReactNode;
  testID?: string;
}

/** Simple bottom-sheet–style modal that works on iOS/Android/Web. */
export const BottomSheet: React.FC<Props> = ({ visible, onClose, title, children, testID }) => {
  return (
    <Modal
      visible={visible}
      transparent
      animationType="slide"
      onRequestClose={onClose}
      testID={testID}
    >
      <Pressable style={styles.backdrop} onPress={onClose} testID="sheet-backdrop" />
      <View style={styles.sheet} testID="sheet-container">
        <View style={styles.handle} />
        {title ? (
          <View style={styles.header}>
            <Text style={styles.title}>{title}</Text>
            <Pressable
              onPress={onClose}
              style={styles.closeBtn}
              hitSlop={12}
              testID="sheet-close-button"
            >
              <Text style={styles.closeText}>Done</Text>
            </Pressable>
          </View>
        ) : null}
        <ScrollView
          style={styles.body}
          contentContainerStyle={styles.bodyContent}
          keyboardShouldPersistTaps="handled"
          showsVerticalScrollIndicator={false}
        >
          {children}
        </ScrollView>
      </View>
    </Modal>
  );
};

const styles = StyleSheet.create({
  backdrop: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.4)',
  },
  sheet: {
    position: 'absolute',
    left: 0,
    right: 0,
    bottom: 0,
    maxHeight: '85%',
    backgroundColor: colors.surfaceSecondary,
    borderTopLeftRadius: radius.lg,
    borderTopRightRadius: radius.lg,
    paddingBottom: Platform.OS === 'ios' ? spacing.xl : spacing.lg,
  },
  handle: {
    width: 40,
    height: 4,
    borderRadius: 2,
    backgroundColor: colors.borderStrong,
    alignSelf: 'center',
    marginTop: spacing.sm,
    marginBottom: spacing.sm,
  },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingHorizontal: spacing.lg,
    paddingVertical: spacing.md,
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: colors.divider,
  },
  title: {
    fontSize: fontSize.lg,
    fontWeight: fontWeight.semibold,
    color: colors.onSurface,
  },
  closeBtn: {
    paddingHorizontal: spacing.sm,
    paddingVertical: spacing.xs,
  },
  closeText: {
    fontSize: fontSize.lg,
    color: colors.brandPrimary,
    fontWeight: fontWeight.semibold,
  },
  body: {
    flexGrow: 0,
  },
  bodyContent: {
    padding: spacing.lg,
    paddingBottom: spacing.xl,
  },
});
```

---

<a id="23-frontend-src-hooks-use-icon-fonts-ts"></a>

## 23. SRC — Icon-font pre-warm hook

**Path:** `frontend/src/hooks/use-icon-fonts.ts`

_52 lines, 2092 bytes_

```ts
// Icon font loader for Expo apps. Fonts are loaded from a CDN only under
// Expo Go (StoreClient) — that's where @expo/vector-icons' .ttf files come
// back as 0 bytes from Metro's asset resolver on Android. Native dev/prod
// builds and web pass an empty map, so useFonts resolves to [true, null]
// immediately via react-native-vector-icons autolinking / web stubs.
// ICON_VECTOR_VERSION must match @expo/vector-icons in package.json.
// Usage: const [loaded, error] = useIconFonts();

import Constants, { ExecutionEnvironment } from "expo-constants";
import { useFonts } from "expo-font";

const ICON_VECTOR_VERSION = "15.1.1";

// short internal fontName (what the library queries) -> CDN .ttf file name
const ICON_FAMILIES: Record<string, string> = {
  anticon: "AntDesign",
  entypo: "Entypo",
  evilicons: "EvilIcons",
  feather: "Feather",
  FontAwesome: "FontAwesome",
  Fontisto: "Fontisto",
  foundation: "Foundation",
  ionicons: "Ionicons",
  "material-community": "MaterialCommunityIcons",
  material: "MaterialIcons",
  octicons: "Octicons",
  "simple-line-icons": "SimpleLineIcons",
  zocial: "Zocial",
  // FontAwesome5 style variants (key = `FontAwesome5Free-<style>`)
  "FontAwesome5Free-Regular": "FontAwesome5_Regular",
  "FontAwesome5Free-Solid": "FontAwesome5_Solid",
  "FontAwesome5Free-Brand": "FontAwesome5_Brands",
  // FontAwesome6 style variants (key = `FontAwesome6Free-<style>`)
  "FontAwesome6Free-Regular": "FontAwesome6_Regular",
  "FontAwesome6Free-Solid": "FontAwesome6_Solid",
  "FontAwesome6Free-Brand": "FontAwesome6_Brands",
};

const cdnUrl = (file: string): string =>
  `https://cdn.jsdelivr.net/npm/@expo/vector-icons@${ICON_VECTOR_VERSION}/build/vendor/react-native-vector-icons/Fonts/${file}.ttf`;

const iconFontMap = (): Record<string, string> =>
  Object.fromEntries(
    Object.entries(ICON_FAMILIES).map(([key, file]) => [key, cdnUrl(file)]),
  );

export const useIconFonts = (): readonly [boolean, Error | null] =>
  useFonts(
    Constants.executionEnvironment === ExecutionEnvironment.StoreClient
      ? iconFontMap()
      : {},
  );
```

---

<a id="24-frontend-src-store-appstore-tsx"></a>

## 24. SRC — App store (AsyncStorage-backed state)

**Path:** `frontend/src/store/appStore.tsx`

_229 lines, 6593 bytes_

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import React, { createContext, useContext, useEffect, useMemo, useState, useCallback } from 'react';

import { TimesheetEntry, LeaveType, ttbEarned } from '@/src/utils/timesheet';
import { isoDate, parseIsoDate } from '@/src/utils/date';

export interface Profile {
  staffName: string;
  managerName: string;
  role: string;
}

export interface TtbLogEntry {
  id: string;
  type: 'add' | 'use';
  hours: number; // stored as positive number
  date: string; // day the action applies to (YYYY-MM-DD)
  note: string;
  createdAt: string; // ISO
  source: 'manual' | 'timesheet'; // where it came from
}

export interface DayInLieuLogEntry {
  id: string;
  type: 'add' | 'use';
  date: string; // day awarded or used
  note: string;
  createdAt: string;
}

interface AppState {
  profile: Profile;
  openingTTB: number; // hours
  openingDayInLieu: number; // count
  entries: Record<string, TimesheetEntry>; // key = date
  ttbLog: TtbLogEntry[];
  dilLog: DayInLieuLogEntry[];
  hasSeeded: boolean;
}

interface AppContextValue extends AppState {
  ready: boolean;
  setProfile: (p: Profile) => void;
  setOpeningTTB: (n: number) => void;
  setOpeningDayInLieu: (n: number) => void;
  upsertEntry: (e: TimesheetEntry) => void;
  addTtbLog: (entry: Omit<TtbLogEntry, 'id' | 'createdAt'>) => void;
  deleteTtbLog: (id: string) => void;
  addDilLog: (entry: Omit<DayInLieuLogEntry, 'id' | 'createdAt'>) => void;
  deleteDilLog: (id: string) => void;
  ttbBalance: number; // hours (opening + earned + manual adds - manual uses - timesheet TTB used)
  dayInLieuBalance: number;
  ttbEarnedTotal: number;
  ttbUsedTotal: number;
  resetAll: () => void;
}

const STORAGE_KEY = '@balancetrack:v1';

const DEFAULT_STATE: AppState = {
  profile: { staffName: '', managerName: '', role: '' },
  openingTTB: 0,
  openingDayInLieu: 0,
  entries: {},
  ttbLog: [],
  dilLog: [],
  hasSeeded: false,
};

const AppCtx = createContext<AppContextValue | null>(null);

function genId() {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
}

export const AppProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, setState] = useState<AppState>(DEFAULT_STATE);
  const [ready, setReady] = useState(false);

  // Load on mount
  useEffect(() => {
    (async () => {
      try {
        const raw = await AsyncStorage.getItem(STORAGE_KEY);
        if (raw) {
          const parsed = JSON.parse(raw) as AppState;
          setState({ ...DEFAULT_STATE, ...parsed });
        }
      } catch (e) {
        console.warn('Failed to load app state', e);
      } finally {
        setReady(true);
      }
    })();
  }, []);

  // Persist on change
  useEffect(() => {
    if (!ready) return;
    AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(state)).catch(() => {});
  }, [state, ready]);

  const setProfile = useCallback((p: Profile) => {
    setState((s) => ({ ...s, profile: p }));
  }, []);

  const setOpeningTTB = useCallback((n: number) => {
    setState((s) => ({ ...s, openingTTB: n }));
  }, []);

  const setOpeningDayInLieu = useCallback((n: number) => {
    setState((s) => ({ ...s, openingDayInLieu: n }));
  }, []);

  const upsertEntry = useCallback((e: TimesheetEntry) => {
    setState((s) => ({ ...s, entries: { ...s.entries, [e.date]: e } }));
  }, []);

  const addTtbLog = useCallback((entry: Omit<TtbLogEntry, 'id' | 'createdAt'>) => {
    setState((s) => ({
      ...s,
      ttbLog: [
        { ...entry, id: genId(), createdAt: new Date().toISOString() },
        ...s.ttbLog,
      ],
    }));
  }, []);

  const deleteTtbLog = useCallback((id: string) => {
    setState((s) => ({ ...s, ttbLog: s.ttbLog.filter((l) => l.id !== id) }));
  }, []);

  const addDilLog = useCallback((entry: Omit<DayInLieuLogEntry, 'id' | 'createdAt'>) => {
    setState((s) => ({
      ...s,
      dilLog: [
        { ...entry, id: genId(), createdAt: new Date().toISOString() },
        ...s.dilLog,
      ],
    }));
  }, []);

  const deleteDilLog = useCallback((id: string) => {
    setState((s) => ({ ...s, dilLog: s.dilLog.filter((l) => l.id !== id) }));
  }, []);

  const resetAll = useCallback(() => {
    setState(DEFAULT_STATE);
  }, []);

  // Derived totals
  const { ttbBalance, dayInLieuBalance, ttbEarnedTotal, ttbUsedTotal } = useMemo(() => {
    const entriesArr = Object.values(state.entries);
    const earned = entriesArr.reduce((sum, e) => sum + ttbEarned(e), 0);
    const timesheetTtbUsed = entriesArr.reduce(
      (sum, e) => (e.leaveType === 'TTB' ? sum + (e.ttbUsed || 0) : sum),
      0,
    );
    const logAdds = state.ttbLog
      .filter((l) => l.type === 'add')
      .reduce((s, l) => s + l.hours, 0);
    const logUses = state.ttbLog
      .filter((l) => l.type === 'use')
      .reduce((s, l) => s + l.hours, 0);
    const ttbUsedTotal = timesheetTtbUsed + logUses;
    const ttbEarnedTotal = earned + logAdds;
    const ttbBalance = state.openingTTB + ttbEarnedTotal - ttbUsedTotal;

    const dilAdds = state.dilLog
      .filter((l) => l.type === 'add')
      .length;
    const dilUses = state.dilLog
      .filter((l) => l.type === 'use')
      .length;
    const dayInLieuBalance = state.openingDayInLieu + dilAdds - dilUses;

    return { ttbBalance, dayInLieuBalance, ttbEarnedTotal, ttbUsedTotal };
  }, [state]);

  const value: AppContextValue = {
    ...state,
    ready,
    setProfile,
    setOpeningTTB,
    setOpeningDayInLieu,
    upsertEntry,
    addTtbLog,
    deleteTtbLog,
    addDilLog,
    deleteDilLog,
    ttbBalance,
    dayInLieuBalance,
    ttbEarnedTotal,
    ttbUsedTotal,
    resetAll,
  };

  return <AppCtx.Provider value={value}>{children}</AppCtx.Provider>;
};

export function useApp(): AppContextValue {
  const v = useContext(AppCtx);
  if (!v) throw new Error('useApp must be used inside <AppProvider>');
  return v;
}

/** Convenience: get (or default) a timesheet entry for a date. */
export function useEntryForDate(date: Date): TimesheetEntry {
  const { entries } = useApp();
  const key = isoDate(date);
  return entries[key] || defaultEntry(key);
}

/** Default entry: 08:00–16:30 on weekdays, 08:00–08:00 on Sat/Sun (0 hrs). */
export function defaultEntry(dateIso: string): TimesheetEntry {
  const d = parseIsoDate(dateIso);
  const wd = d.getUTCDay(); // 0=Sun, 6=Sat
  const isWeekend = wd === 0 || wd === 6;
  return {
    date: dateIso,
    start: '08:00',
    finish: isWeekend ? '08:00' : '16:30',
    leaveType: 'Work',
    ttbUsed: 0,
    notes: '',
  };
}

export { isoDate, parseIsoDate };
```

---

<a id="25-frontend-src-utils-date-ts"></a>

## 25. SRC — Date / fortnight helpers

**Path:** `frontend/src/utils/date.ts`

_67 lines, 2273 bytes_

```ts
// Fortnight date helpers. A fortnight starts on Monday and lasts 14 days.
// Anchor Monday: 2024-01-01 (Mon).
const MS_PER_DAY = 24 * 60 * 60 * 1000;
const ANCHOR = new Date(Date.UTC(2024, 0, 1)); // Mon 2024-01-01

/** Returns the Monday date (YYYY-MM-DD) that starts the fortnight containing `d`. */
export function fortnightStart(d: Date = new Date()): Date {
  const local = new Date(Date.UTC(d.getFullYear(), d.getMonth(), d.getDate()));
  const daysSince = Math.floor((local.getTime() - ANCHOR.getTime()) / MS_PER_DAY);
  const fortnightIndex = Math.floor(daysSince / 14);
  return new Date(ANCHOR.getTime() + fortnightIndex * 14 * MS_PER_DAY);
}

export function addDays(d: Date, days: number): Date {
  return new Date(d.getTime() + days * MS_PER_DAY);
}

export function fortnightDates(start: Date): Date[] {
  return Array.from({ length: 14 }, (_, i) => addDays(start, i));
}

const MONTHS = [
  'Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
  'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec',
];
const DAYS = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
const DAYS_SHORT = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

export function formatDateShort(d: Date): string {
  return `${d.getUTCDate()} ${MONTHS[d.getUTCMonth()]}`;
}

export function formatDateFull(d: Date): string {
  return `${d.getUTCDate()} ${MONTHS[d.getUTCMonth()]} ${d.getUTCFullYear()}`;
}

export function dayName(d: Date): string {
  return DAYS[d.getUTCDay()];
}

export function dayShort(d: Date): string {
  return DAYS_SHORT[d.getUTCDay()];
}

/** ISO date string YYYY-MM-DD (UTC-based to avoid tz issues). */
export function isoDate(d: Date): string {
  const y = d.getUTCFullYear();
  const m = String(d.getUTCMonth() + 1).padStart(2, '0');
  const day = String(d.getUTCDate()).padStart(2, '0');
  return `${y}-${m}-${day}`;
}

export function parseIsoDate(s: string): Date {
  const [y, m, day] = s.split('-').map(Number);
  return new Date(Date.UTC(y, m - 1, day));
}

export function isWeekend(d: Date): boolean {
  const wd = d.getUTCDay();
  return wd === 0 || wd === 6;
}

/** JS getUTCDay: 0=Sun,1=Mon... We want ISO weekday: 1=Mon..7=Sun */
export function isoWeekday(d: Date): number {
  const wd = d.getUTCDay();
  return wd === 0 ? 7 : wd;
}
```

---

<a id="26-frontend-src-utils-timesheet-ts"></a>

## 26. SRC — Timesheet formulas (spreadsheet rules)

**Path:** `frontend/src/utils/timesheet.ts`

_115 lines, 3475 bytes_

```ts
import { isoWeekday, parseIsoDate } from './date';

export type LeaveType =
  | 'Work'
  | 'Public Holiday'
  | 'RDO'
  | 'TTB'
  | 'Annual Leave'
  | 'Sick Leave';

export const LEAVE_TYPES: LeaveType[] = [
  'Work',
  'Public Holiday',
  'RDO',
  'TTB',
  'Annual Leave',
  'Sick Leave',
];

export interface TimesheetEntry {
  date: string; // YYYY-MM-DD
  start: string; // "HH:MM" 24h
  finish: string; // "HH:MM" 24h
  leaveType: LeaveType;
  ttbUsed: number; // hours
  notes: string;
}

/** Parse "HH:MM" -> decimal hours (24h clock). */
export function parseTimeHours(t: string): number {
  const [h, m] = t.split(':').map(Number);
  return h + m / 60;
}

export function formatTime(t: string): string {
  // Convert HH:MM to friendly display "H:MM AM/PM"
  const [h, m] = t.split(':').map(Number);
  const period = h >= 12 ? 'PM' : 'AM';
  const displayH = h === 0 ? 12 : h > 12 ? h - 12 : h;
  return `${displayH}:${String(m).padStart(2, '0')} ${period}`;
}

/** Generate all 30-minute time slots from 00:00 to 23:30. */
export function timeSlots(): string[] {
  const slots: string[] = [];
  for (let h = 0; h < 24; h++) {
    for (const m of [0, 30]) {
      slots.push(`${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`);
    }
  }
  return slots;
}

/** Duration in hours (finish - start), assuming same-day. */
function duration(start: string, finish: string): number {
  return parseTimeHours(finish) - parseTimeHours(start);
}

/**
 * Total hours (F column) — matches spreadsheet:
 *   If leaveType != 'Work' -> PH=8, RDO=8, TTB=-ttbUsed, else 0
 *   If Work weekday -> (finish-start) - 0.5 (lunch)
 *   If Work Sat/Sun -> (finish-start), no lunch, no multiplier
 */
export function totalHours(entry: TimesheetEntry): number {
  if (entry.leaveType !== 'Work') {
    if (entry.leaveType === 'Public Holiday') return 8;
    if (entry.leaveType === 'RDO') return 8;
    if (entry.leaveType === 'TTB') return -entry.ttbUsed;
    if (entry.leaveType === 'Annual Leave') return 8;
    if (entry.leaveType === 'Sick Leave') return 8;
    return 0;
  }
  if (!entry.start || !entry.finish) return 0;
  const d = duration(entry.start, entry.finish);
  const wd = isoWeekday(parseIsoDate(entry.date));
  if (wd === 6 || wd === 7) return d;
  return Math.max(d - 0.5, 0);
}

/** Base hours (K col) = min(total,8), only for the "on-the-clock" bit. */
export function baseHours(entry: TimesheetEntry): number {
  return Math.min(totalHours(entry), 8);
}

/**
 * OT / TTB Earned (L column):
 *   Weekday: max(F - 8, 0)  (only when > 8 hours worked)
 *   Sat/Sun: full duration (finish-start), all hours are TTB
 * Only applies to actual Work days.
 */
export function ttbEarned(entry: TimesheetEntry): number {
  if (entry.leaveType !== 'Work') return 0;
  if (!entry.start || !entry.finish) return 0;
  const wd = isoWeekday(parseIsoDate(entry.date));
  if (wd === 6 || wd === 7) {
    return Math.max(duration(entry.start, entry.finish), 0);
  }
  return Math.max(totalHours(entry) - 8, 0);
}

/** Round to nearest 15-minute increment (0.25 hour). */
export function roundTo15(hours: number): number {
  return Math.round(hours * 4) / 4;
}

/** Formats decimal hours as "Hh Mm" like "7h 30m". */
export function formatHours(hours: number): string {
  const sign = hours < 0 ? '-' : '';
  const abs = Math.abs(hours);
  const h = Math.floor(abs);
  const m = Math.round((abs - h) * 60);
  if (m === 0) return `${sign}${h}h`;
  return `${sign}${h}h ${m}m`;
}
```

---

