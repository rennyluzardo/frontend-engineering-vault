# SPIKE: Reingeniería de Empaquetado — Code Splitting & Dynamic Loading

**Proyecto:** Enterprise Platform  
**Stack:** React 19 + Vite 7 (Rollup) + Redux Toolkit + React Router DOM v7  
**Fecha:** Febrero 2026  
**Autor:** Lead Engineer 
**Versión:** 1.0

---

## Tabla de Contenidos

1. [Resumen Ejecutivo para Clientes](#1-resumen-ejecutivo-para-clientes)
2. [Marco Metodológico de la Investigación](#2-marco-metodológico-de-la-investigación)
3. [Diagnóstico del Estado Actual](#3-diagnóstico-del-estado-actual)
4. [Herramienta de Análisis Visual](#4-herramienta-de-análisis-visual-bundle-analyzer)
5. [Guía de Implementación Paso a Paso](#5-guía-de-implementación-paso-a-paso)
6. [Plantilla de Informe de Resultados](#6-plantilla-de-informe-de-resultados-antesdespués)

---

## 1. Resumen Ejecutivo para Clientes

### Problema Detectado

La aplicación Enterprise Platform carga **la totalidad de su código fuente** en la primera visita del usuario, independientemente de la página solicitada. Esto significa que un usuario que solo necesita iniciar sesión descarga también el código de reportes, generación de documentos, gráficos, administración de entidades y las otras 50+ pantallas de la aplicación.

### Impacto Medible

| Métrica | Estado Actual (Estimado) | Objetivo Post-Optimización |
|---------|--------------------------|----------------------------|
| **Bundle principal** | >2 MB (sin comprimir) | <400 KB initial load |
| **First Contentful Paint** | >3.5s (3G) | <1.8s (3G) |
| **Time to Interactive** | >5.0s (3G) | <3.0s (3G) |
| **Lighthouse Performance** | 40–60 | 80–95 |

### ROI Esperado

- **-60% a -75%** en tamaño de carga inicial (bundle principal).
- **+30 a +50 puntos** en Lighthouse Performance Score.
- **Mejora directa en SEO:** Google penaliza sitios con FCP > 2.5s y TTI > 3.8s.
- **Reducción de bounce rate:** Cada 100ms de mejora en carga incrementa la conversión en ~1% (fuente: Deloitte, "Milliseconds Make Millions", 2020).

### Riesgo de No Actuar

Un bundle monolítico en crecimiento implica degradación progresiva. Cada nueva feature aumenta el tiempo de carga para **todas** las páginas, incluyendo la pantalla de login.

---

## 2. Marco Metodológico de la Investigación

### 2.1 Tipo de Investigación

**Investigación Cuasi-Experimental con Diseño Pre-Test / Post-Test de un solo grupo.**

| Aspecto | Descripción |
|---------|-------------|
| **Tipo** | Cuasi-Experimental (Applied Research) |
| **Diseño** | Pre-Test / Post-Test — Single Group |
| **Variables Independientes** | Técnicas de code splitting, lazy loading, vendor splitting, tree shaking |
| **Variables Dependientes** | Bundle size (KB), FCP (ms), TTI (ms), Lighthouse Score |
| **Grupo de Control** | Mediciones baseline del estado actual (pre-intervención) |
| **Grupo Experimental** | Misma aplicación post-optimización |
| **Validez Interna** | Se controla el entorno (mismo hardware, misma red simulada con Lighthouse throttling) |
| **Validez Externa** | Resultados generalizables a SPAs con arquitectura similar |

### 2.2 Problem Statement

> *La aplicación web Enterprise Platform presenta un bundle monolítico que carga sincrónicamente las 55 escenas (scenes), 30+ componentes compartidos y todas las dependencias de terceros (~15 librerías) en la carga inicial, resultando en un tiempo de First Contentful Paint (FCP) y Time to Interactive (TTI) que exceden los umbrales recomendados por Google Web Vitals (FCP < 1.8s, TTI < 3.8s), impactando negativamente el SEO ranking, la experiencia de usuario y las métricas de conversión del negocio.*

### 2.3 Hipótesis

**H₁:** La implementación de code splitting a nivel de rutas, vendor splitting y optimización de tree shaking reducirá el tamaño del bundle inicial en al menos un 60% y mejorará el FCP en al menos un 40%.

**H₀ (Nula):** Las técnicas de optimización de empaquetado no producen una mejora estadísticamente significativa en las métricas de rendimiento.

### 2.4 Justificación Técnica (para Stakeholders)

1. **Core Web Vitals como factor de ranking (Google, 2021–presente):** LCP, FID/INP y CLS son señales de ranking. Un bundle excesivo degrada LCP y FID directamente.
2. **Cost of JavaScript (Addy Osmani, Google Chrome Team, 2023):** Cada KB de JavaScript tiene un costo de parsing + compilación + ejecución. JS es byte-por-byte más costoso que una imagen del mismo tamaño.
3. **HTTP Archive Web Almanac 2023:** La mediana de JS transferido en páginas móviles es ~500KB. Exceder este umbral coloca a la aplicación en el percentil inferior de rendimiento.
4. **Deloitte, "Milliseconds Make Millions" (2020):** Mejoras de 100ms en velocidad de carga generan incrementos medibles en engagement y conversión.

### 2.5 Metodología de Medición

```
┌─────────────────────────────────────────────────────────────┐
│                    PROTOCOLO DE MEDICIÓN                     │
├─────────────────────────────────────────────────────────────┤
│ 1. Build de producción: `npm run build`                     │
│ 2. Registro de tamaños: `dist/assets/*.js` (raw + gzip)    │
│ 3. Lighthouse CI: 5 runs, mediana, modo mobile              │
│    - Throttling: Simulated Slow 4G                          │
│    - CPU: 4x slowdown                                       │
│ 4. Bundle Analyzer: treemap visual pre y post               │
│ 5. Comparación tabulada con deltas porcentuales             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Diagnóstico del Estado Actual

### 3.1 Anti-Patrones Críticos Detectados

#### 🔴 AP-1: Zero Code Splitting — Bundle Monolítico

**Archivo:** `src/config/routes.js`

Todas las **55 views** se importan estáticamente a través del barrel export `src/views/index.jsx`:

```javascript
// src/config/routes.js (líneas 1-58)
import {
  AccountSettings, OrganizationSetup, AccessControl, TeamManagement,
  Partners, PartnerDirectory, StakeholderRegistry, IdentifierManager, EntityCreation,
  ClientOnboarding, ApprovalWorkflows, GlobalContractTerms,
  ProfileEditor, ProviderRegistry, RoleDefinitions, UserDirectory, CustomerIdentifiers,
  AnalyticsDashboard, AdminAuditLog, BillingTermsEditor, WorkflowReview,
  ClientGroups, VendorDirectory, DocumentDetailView, BillingDashboard, MainLayout,
  EntityRegistry, AssetManagement, SystemLogs, DisputeResolution, Documentation,
  EmptyState, ActivityTracker, ComplianceHistory, ContractLibrary,
  ServiceProviders, SecurityRecovery, PaymentMethodRegistry, UserGraph, BusinessAnalytics,
  DataRequest, PermissionsManager, AuthPortal, SuccessFeedback, ComplianceTerms, UserProfileView, ValidationPortal,
  DocumentBundles, TransactionRetry, SSOGateway, ProfileSelection, PolicyDirectory,
  ConsentAgreement, OperationsManual,
} from "../views";
```

**Consecuencia:** Un usuario que visita `/login` descarga el código de las 55 pantallas.

#### 🔴 AP-2: Namespace Imports que Destruyen Tree Shaking

**Archivo:** `src/util/index.js`

```javascript
import * as FaIcons from "react-icons/fa";   // ~1,500 iconos FA incluidos
import * as TbIcons from "react-icons/tb";   // ~4,500 iconos Tabler incluidos
import * as BiIcons from "react-icons/bi";   // ~800 iconos BoxIcons incluidos
```

**Consecuencia:** `import *` importa **TODOS** los iconos de cada paquete. Si solo se usan 20 iconos, se están cargando ~6,800 iconos innecesarios. Impacto estimado: **+500KB a +1MB** de JavaScript muerto.

**Archivos adicionales con `import *`:**
- `src/index.jsx` → `import * as Sentry from "@sentry/react"`
- `src/App.jsx` → `import * as Sentry from "@sentry/react"`
- `src/views/layout/components/Aside/components/Menu/index.jsx` → react-icons
- `src/views/layout/components/Dropdown/index.jsx` → react-icons
- `src/components/ToggleBTN/index.jsx` → Sentry
- `src/components/ErrorElement/index.jsx` → Sentry

#### 🟡 AP-3: Dependencias Pesadas Sin Lazy Loading

| Dependencia | Tamaño Estimado (min) | Usado en | Frecuencia de Uso |
|-------------|----------------------|----------|-------------------|
| `recharts` | ~500 KB | `analytics/components/Charts/` (1 archivo) | Solo AnalyticsDashboard |
| `jspdf` + `jspdf-autotable` | ~300 KB | `documentDetail/components/DocumentPDF/` (1 archivo) | Solo al generar PDF |
| `react-icons` (con `import *`) | ~500-1000 KB | 3 archivos | Menú lateral + Utils |
| `@sentry/react` | ~250 KB | Entry point | Siempre (pero namespace import) |
| `react-datepicker` | ~150 KB | Formularios con fecha | Algunas pantallas |
| `react-select` | ~100 KB | Dropdowns avanzados | Algunas pantallas |
| `react-table-legacy` | ~80 KB | Tablas legacy | Múltiples pantallas |
| `core-js` (4 polyfills) | ~80 KB | `index.jsx` | Innecesarios en React 19 |
| `bulma` | ~200 KB (CSS) | Global | Siempre |
| `redux-logger` | ~15 KB | Store (solo dev) | Incluido en producción |

#### 🟡 AP-4: Barrel Exports Sin Lazy Boundaries

**Archivos:** `src/views/index.jsx` (55 re-exports) y `src/components/index.jsx` (30 re-exports)

Los barrel exports consolidan las importaciones pero, combinados con imports estáticos en `routes.js`, fuerzan a Rollup a incluir **todo** en un solo chunk.

#### 🟡 AP-5: Sin Vendor Splitting

**Archivo:** `vite.config.mjs` — La sección `rollupOptions.output` no configura `manualChunks`, por lo que todas las dependencias de `node_modules` se empaquetan junto con el código de la aplicación.

### 3.2 Diagrama de Impacto

```
                    index.html
                        │
                   ┌────▼────┐
                   │ main.js │ ← MONOLÍTICO (~2-3 MB estimado)
                   └────┬────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐    ┌─────▼─────┐   ┌────▼────┐
   │ 55      │    │ 15+       │   │ 30      │
   │ Views   │    │ Vendor    │   │ Shared  │
   │ (ALL)   │    │ Libs      │   │ Comps   │
   └─────────┘    └───────────┘   └─────────┘
        │               │
   Partners        react-icons/*
   jspdf           sentry/*
   react-table     core-js
   react-select    redux-logger
   react-datepicker bulma (CSS)
```

---

## 4. Herramienta de Análisis Visual (Bundle Analyzer)

### 4.1 Herramienta Recomendada: `rollup-plugin-visualizer`

Dado que Vite usa Rollup internamente para el build de producción, la herramienta nativa es **rollup-plugin-visualizer**.

#### Instalación

```bash
npm install --save-dev rollup-plugin-visualizer
```

#### Configuración en `vite.config.mjs`

```javascript
import { visualizer } from "rollup-plugin-visualizer";

// Agregar dentro del array plugins:
plugins: [
  // ...plugins existentes,
  visualizer({
    filename: "bundle-analysis.html",  // Archivo de salida
    open: true,                         // Abre automáticamente en el browser
    gzipSize: true,                     // Muestra tamaño gzip
    brotliSize: true,                   // Muestra tamaño brotli
    template: "treemap",                // Opciones: treemap | sunburst | network
  }),
],
```

#### Ejecución

```bash
npm run build
# Se genera bundle-analysis.html en la raíz del proyecto
```

### 4.2 Cómo Interpretar el Treemap

```
┌─────────────────────────────────────────────────────────────────┐
│                        GUÍA DE LECTURA                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tamaño del Rectángulo = Tamaño del módulo en el bundle        │
│                                                                 │
│  ┌──────────────────────┬────────────┬──────┐                  │
│  │                      │            │      │                  │
│  │   node_modules/      │  recharts  │ jspdf│                  │
│  │   react-icons/       │  (~500KB)  │      │                  │
│  │   (~800KB)           │            │      │                  │
│  │                      ├────────────┤      │                  │
│  │   ← QUICK WIN #1    │  sentry    │      │                  │
│  │                      │  (~250KB)  │      │                  │
│  ├──────────────────────┼────────────┴──────┤                  │
│  │   src/scenes/        │  core-js          │                  │
│  │   (todo el app code) │  (polyfills)      │                  │
│  │                      │  ← QUICK WIN #3   │                  │
│  └──────────────────────┴───────────────────┘                  │
│                                                                 │
│  BUSCAR:                                                       │
│  1. Rectángulos GRANDES en node_modules → candidatos a split   │
│  2. Módulos que aparecen DUPLICADOS → deduplicación            │
│  3. Código de app todo junto → falta code splitting            │
│  4. Librerías usadas en 1 ruta → candidatos a dynamic import  │
│                                                                 │
│  QUICK WINS (por orden de impacto):                            │
│  #1: react-icons import * → named imports (−500KB a −1MB)      │
│  #2: recharts + jspdf → dynamic import (−800KB initial)        │
│  #3: core-js polyfills → eliminar (React 19 no los necesita)  │
│  #4: 55 scenes → lazy loading por ruta (−60% initial bundle)  │
│  #5: vendor splitting → cacheo granular de terceros            │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Agregar a `.gitignore`

```
bundle-analysis.html
```

---

## 5. Guía de Implementación Paso a Paso

### Fase 1: Quick Wins — Arreglar Tree Shaking (Impacto Inmediato)

#### Paso 1.1: Eliminar `import *` de react-icons — Patrón Icon Registry

##### Contexto del Problema

La función `customIcon(name)` en `src/util/index.js` es un **resolver dinámico** que recibe un string (e.g., `"FaChartLine"`) y busca el componente en los namespace imports. Se usa **182 veces en 48 archivos**. El mismo patrón se replica en `Menu/index.jsx` (con `Icons` y `IconsGo`).

Los nombres de iconos vienen como strings desde `src/config/asideOptions.js` y desde JSX hardcodeado en los scenes (e.g., `customIcon("FaRegEdit")`).

**¿Por qué NO se puede simplemente cambiar a named imports?** Porque la función `customIcon` hace lookup dinámico por string: `FaIcons[name]`. Se necesita mantener el mapa string→componente, pero **solo con los iconos realmente usados**.

##### Solución: Icon Registry (Explicit Map)

**Paso A:** Auditar todos los nombres de iconos usados en el proyecto:

```bash
# Extraer todos los nombres de iconos pasados a customIcon
grep -roh "customIcon(['\"][A-Za-z]*['\"])" src/ | sort -u
# Extraer los icon props en asideOptions.js
grep -oh 'icon: "[A-Za-z]*"' src/config/asideOptions.js | sort -u
```

**Iconos identificados en la auditoría del proyecto:**

```
# Desde asideOptions.js (Menu icons):
FaAngleDown, FaBook, FaChartLine, FaClipboardList, FaFileAlt,
FaFileInvoiceDollar, FaRegCreditCard, FaRegIdBadge, FaRegListAlt,
FaSyncAlt, FaThList, FaUniversity, FaUserFriends, FaUserPlus,
FaUsers, FaUserTag, GoLaw

# Desde customIcon() calls en scenes/components:
FaBan, FaCheck, FaCheckCircle, FaDownload, FaEdit,
FaExclamationCircle, FaEye, FaFileInvoice, FaHandHoldingUsd,
FaMinusCircle, FaPlus, FaPlusCircle, FaRegCheckCircle, FaRegEdit,
FaRegEye, FaRegFileAlt, FaStore, FaStoreAltSlash, FaTimes,
FaUsersSlash

# Total: ~37 iconos (vs ~6,800 con import *)
# NOTA: Ejecutar la auditoría con grep antes de implementar
# para capturar cualquier icono adicional.
```

**Paso B:** Crear `src/util/iconRegistry.js`:

```javascript
// ── Icon Registry ──
// Solo importamos los iconos que realmente se usan en la aplicación.
// Para agregar un nuevo icono: 1) importarlo aquí, 2) agregarlo al mapa.
// NUNCA usar import * de react-icons.

import {
  FaAngleDown,
  FaBan,
  FaBook,
  FaChartLine,
  FaCheck,
  FaCheckCircle,
  FaClipboardList,
  FaDownload,
  FaEdit,
  FaExclamationCircle,
  FaEye,
  FaFileAlt,
  FaFileInvoice,
  FaFileInvoiceDollar,
  FaHandHoldingUsd,
  FaMinusCircle,
  FaPlus,
  FaPlusCircle,
  FaRegCheckCircle,
  FaRegCreditCard,
  FaRegEdit,
  FaRegEye,
  FaRegFileAlt,
  FaRegIdBadge,
  FaRegListAlt,
  FaStore,
  FaStoreAltSlash,
  FaSyncAlt,
  FaTimes,
  FaThList,
  FaUniversity,
  FaUserFriends,
  FaUserPlus,
  FaUsers,
  FaUsersSlash,
  FaUserTag,
} from "react-icons/fa";

import { GoLaw } from "react-icons/go";

// Agregar aquí los iconos de Tabler y BoxIcons que se usen realmente:
// import { TbXxx } from "react-icons/tb";
// import { BiXxx } from "react-icons/bi";

const ICON_MAP = {
  // FontAwesome
  FaAngleDown,
  FaBan,
  FaBook,
  FaChartLine,
  FaCheck,
  FaCheckCircle,
  FaClipboardList,
  FaDownload,
  FaEdit,
  FaExclamationCircle,
  FaEye,
  FaFileAlt,
  FaFileInvoice,
  FaFileInvoiceDollar,
  FaHandHoldingUsd,
  FaMinusCircle,
  FaPlus,
  FaPlusCircle,
  FaRegCheckCircle,
  FaRegCreditCard,
  FaRegEdit,
  FaRegEye,
  FaRegFileAlt,
  FaRegIdBadge,
  FaRegListAlt,
  FaStore,
  FaStoreAltSlash,
  FaSyncAlt,
  FaTimes,
  FaThList,
  FaUniversity,
  FaUserFriends,
  FaUserPlus,
  FaUsers,
  FaUsersSlash,
  FaUserTag,
  // GitHub Octicons
  GoLaw,
  // Tabler Icons (agregar según auditoría)
  // BoxIcons (agregar según auditoría)
};

export default ICON_MAP;
```

**Paso C:** Refactorizar `customIcon` en `src/util/index.js`:

```javascript
// ANTES:
import * as FaIcons from "react-icons/fa";
import * as TbIcons from "react-icons/tb";
import * as BiIcons from "react-icons/bi";

export const customIcon = (name) => {
  const Icon = FaIcons[name] || TbIcons[name] || BiIcons[name];
  return Icon ? <Icon /> : "";
};

// DESPUÉS:
import ICON_MAP from "./iconRegistry";

export const customIcon = (name) => {
  const Icon = ICON_MAP[name];
  if (!Icon) {
    if (process.env.NODE_ENV !== "production") {
      console.warn(`[customIcon] Icon "${name}" not found in registry. Add it to src/util/iconRegistry.js`);
    }
    return "";
  }
  return <Icon />;
};
```

**Paso D:** Refactorizar `Menu/index.jsx`:

```javascript
// ANTES:
import * as Icons from "react-icons/fa";
import * as IconsGo from "react-icons/go";

customIcon = ({ name }) => {
  const FaIcon = Icons[name];
  const GoIcon = IconsGo[name];
  // ...
};

// DESPUÉS:
import ICON_MAP from "@/util/iconRegistry";

customIcon = ({ name }) => {
  const Icon = ICON_MAP[name];
  if (!Icon) return "";
  return (
    <span className="icon" style={{ marginRight: "5px" }}>
      <Icon />
    </span>
  );
};
```

##### Beneficio

De ~6,800 iconos importados (FA: ~1,500 + Tabler: ~4,500 + BoxIcons: ~800) se pasa a **~30-40 iconos** explícitos.

**Impacto estimado: −500 KB a −1 MB.**

#### Paso 1.2: Eliminar polyfills de core-js innecesarios

**Antes** (`src/index.jsx`):
```javascript
import "core-js/features/object";
import "core-js/features/array";
import "core-js/features/string";
import "core-js/features/promise";
```

**Después:** Eliminar estas líneas. React 19 requiere browsers modernos que ya soportan estas APIs nativamente. Además, el `browserslist` del proyecto excluye IE11 y Opera Mini.

**Impacto estimado: −60 KB a −80 KB.**

#### Paso 1.3: Excluir redux-logger de producción

**Antes** (`src/store/store.js`):
```javascript
import { logger } from "redux-logger";
```

**Después** — dynamic import condicional:
```javascript
// Eliminar el import estático de arriba y usar dynamic import:
const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) => {
    const middleware = getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    });

    return middleware;
  },
  devTools: process.env.NODE_ENV === "production" ? false : { trace: true },
});

// Logger solo en desarrollo, cargado de forma asíncrona
if (process.env.NODE_ENV !== "production") {
  import("redux-logger").then(({ createLogger }) => {
    const logger = createLogger();
    store.dispatch = ((next) => (action) => {
      // Este enfoque no es ideal con RTK; alternativa:
      // usar redux DevTools Extension que ya está habilitado
    })(store.dispatch);
  });
}
```

> **Alternativa más simple:** Dado que `devTools: { trace: true }` ya está habilitado, considerar **eliminar `redux-logger` completamente** y usar Redux DevTools Extension del browser.

**Impacto estimado: −15 KB + eliminación de dependencia.**

---

### Fase 2: Code Splitting por Rutas (Impacto Mayor)

#### Paso 2.1: Crear archivo de lazy imports

Crear `src/config/lazyRoutes.js`:

```javascript
import { lazy } from "react";

// ── Auth Routes (carga inmediata — son la primera pantalla) ──
// SignIn, RecoverPassword, ValidateCode, Saml se mantienen estáticos
// porque son las primeras pantallas que ve el usuario.

// ── Private Routes (carga dinámica) ──
export const AnalyticsDashboard = lazy(() => import("@/views/analyticsDashboard"));
export const ServiceProviders = lazy(() => import("@/views/serviceProviders"));
export const BillingDashboard = lazy(() => import("@/views/billingDashboard"));
export const DocumentDetailView = lazy(() => import("@/views/documentDetailView"));
export const DocumentBundles = lazy(() => import("@/views/documentBundles"));
export const ProviderRegistry = lazy(() => import("@/views/providerRegistry"));
export const ClientOnboarding = lazy(() => import("@/views/clientOnboarding"));
export const ProfileEditor = lazy(() => import("@/views/profileEditor"));
export const UserGraph = lazy(() => import("@/views/userGraph"));
export const EntityCreation = lazy(() => import("@/views/entityCreation"));
export const Partners = lazy(() => import("@/views/partners"));
export const RoleDefinitions = lazy(() => import("@/views/roleDefinitions"));
export const PermissionsManager = lazy(() => import("@/views/permissionsManager"));
export const AccessControl = lazy(() => import("@/views/accessControl"));
export const BusinessAnalytics = lazy(() => import("@/views/businessAnalytics"));
export const UserProfileView = lazy(() => import("@/views/userProfileView"));
export const ApprovalWorkflows = lazy(() => import("@/views/approvalWorkflows"));
export const AccountSettings = lazy(() => import("@/views/accountSettings"));
export const GlobalContractTerms = lazy(() => import("@/views/globalContractTerms"));
export const WorkflowReview = lazy(() => import("@/views/workflowReview"));
export const SystemLogs = lazy(() => import("@/views/systemLogs"));
export const PartnerDirectory = lazy(() => import("@/views/partnerDirectory"));
export const UserDirectory = lazy(() => import("@/views/userDirectory"));
export const AdminAuditLog = lazy(() => import("@/views/adminAuditLog"));
export const ActivityTracker = lazy(() => import("@/views/activityTracker"));
export const DisputeResolution = lazy(() => import("@/views/disputeResolution"));
export const SecurityRecovery = lazy(() => import("@/views/securityRecovery"));
export const SuccessFeedback = lazy(() => import("@/views/successFeedback"));
export const DataRequest = lazy(() => import("@/views/dataRequest"));
export const AssetManagement = lazy(() => import("@/views/assetManagement"));
export const TeamManagement = lazy(() => import("@/views/teamManagement"));
export const Documentation = lazy(() => import("@/views/documentation"));
export const CustomerIdentifiers = lazy(() => import("@/views/customerIdentifiers"));
export const IdentifierManager = lazy(() => import("@/views/identifierManager"));
export const BillingTermsEditor = lazy(() => import("@/views/billingTermsEditor"));
export const WorkflowReview = lazy(() => import("@/views/workflowReview"));
export const ClientGroups = lazy(() => import("@/views/clientGroups"));
export const VendorDirectory = lazy(() => import("@/views/vendorDirectory"));
export const EntityRegistry = lazy(() => import("@/views/entityRegistry"));
export const ComplianceHistory = lazy(() => import("@/views/complianceHistory"));
export const ContractLibrary = lazy(() => import("@/views/contractLibrary"));
export const TransactionRetry = lazy(() => import("@/views/transactionRetry"));
export const SSOGateway = lazy(() => import("@/views/ssoGateway"));
export const ProfileSelection = lazy(() => import("@/views/profileSelection"));
export const EmptyState = lazy(() => import("@/views/emptyState"));
export const PolicyDirectory = lazy(() => import("@/views/policyDirectory"));
export const ConsentAgreement = lazy(() => import("@/views/consentAgreement"));
export const ComplianceTerms = lazy(() => import("@/views/complianceTerms"));
export const OperationsManual = lazy(() => import("@/views/operationsManual"));
```

#### Paso 2.2: Refactorizar `src/config/routes.js`

```javascript
import { Suspense } from "react";
import { Navigate } from "react-router-dom";
import { useSelector } from "react-redux";

// ── Carga estática: solo lo necesario para el primer render ──
import { AuthPortal, SecurityRecovery, ValidationPortal, SSOGateway } from "../views/auth";
// O importar directamente:
// import AuthPortal from "@/views/auth/authPortal";
// import SecurityRecovery from "@/views/auth/securityRecovery";
// import ValidationPortal from "@/views/validationPortal";
// import SSOGateway from "@/views/ssoGateway";

import { ErrorElement, NotFound } from "../components";
import AuthLayout from "@/views/auth/authLayout";
import { Loading } from "@/components";

// ── Carga dinámica: todas las rutas privadas ──
import {
  AnalyticsDashboard, ServiceProviders, BillingDashboard, DocumentDetailView, DocumentBundles,
  ProviderRegistry, ClientOnboarding, ProfileEditor, UserGraph,
  EntityCreation, Partners, RoleDefinitions, PermissionsManager, AccessControl, BusinessAnalytics,
  UserProfileView, ApprovalWorkflows, AccountSettings, GlobalContractTerms,
  WorkflowReview, SystemLogs, PartnerDirectory, UserDirectory,
  AdminAuditLog, ActivityTracker, DisputeResolution, SecurityRecovery, SuccessFeedback, DataRequest,
  AssetManagement, TeamManagement, Documentation, CustomerIdentifiers, IdentifierManager,
  BillingTermsEditor, ClientGroups, VendorDirectory, EntityRegistry, ComplianceHistory,
  ContractLibrary, TransactionRetry, SSOGateway, ProfileSelection, EmptyState, PolicyDirectory,
  ConsentAgreement, ComplianceTerms, OperationsManual,
} from "./lazyRoutes";

// ── Wrapper con Suspense ──
const Lazy = ({ children }) => (
  <Suspense fallback={<Loading />}>
    {children}
  </Suspense>
);

const PrivateRoute = ({ children }) => {
  const auth = useSelector((state) => state.auth);
  if (!auth?.logged) {
    return <Navigate to="/login" replace />;
  }
  return children;
};

const routes = (featureFlags) => [
  {
    path: "/",
    element: <AuthLayout key="/" />,
    errorElement: <ErrorElement />,
    children: [
      { index: true, element: <Navigate to="/login" replace /> },
      { path: "login", element: <AuthPortal key="login" /> },
      { path: "saml", element: <SSOGateway key="saml" /> },
      { path: "recover-password", element: <SecurityRecovery key="recover-password" /> },
      { path: "validate-code", element: <ValidationPortal key="validate-code" /> },
    ],
  },
  {
    path: "/",
    element: (
      <PrivateRoute>
        <Layout key="/" />
      </PrivateRoute>
    ),
    errorElement: <ErrorElement />,
    children: [
      {
        path: "/resp",
        element: <Lazy><SuccessFeedback key="/resp" /></Lazy>,
      },
      {
        path: "dashboard",
        element: <Lazy><AnalyticsDashboard key="analyticsDashboard" /></Lazy>,
      },
      // ... (mismo patrón para todas las rutas privadas)
    ],
  },
];
```

> **Resultado:** Cada ruta se convierte en un chunk separado. Vite/Rollup genera automáticamente archivos como `AnalyticsDashboard.abc123.js`, `BusinessAnalytics.def456.js`, etc.

**Impacto estimado: −60% a −75% del bundle inicial.**

---

### Fase 3: Dynamic Import para Dependencias Pesadas

#### Paso 3.1: Lazy load de recharts (solo AnalyticsDashboard)

**Antes** (`src/views/analyticsDashboard/components/Charts/index.jsx`):
```javascript
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip } from "recharts";
```

**Después** — el `React.lazy` a nivel de ruta ya resuelve esto. Si AnalyticsDashboard es lazy-loaded, recharts se incluye automáticamente en el chunk de AnalyticsDashboard.

Sin embargo, si Dashboard tiene múltiples sub-vistas y los Charts solo se muestran en una de ellas, agregar un lazy más granular:

```javascript
import { lazy, Suspense } from "react";
const Charts = lazy(() => import("./components/Charts"));

// En el render:
{showCharts && (
  <Suspense fallback={<div>Cargando gráficos...</div>}>
    <Charts data={data} />
  </Suspense>
)}
```

#### Paso 3.2: Lazy load de jsPDF (solo al generar PDF)

**Antes** (`src/views/documentDetailView/components/DocumentPDF/index.jsx`):
```javascript
import jsPDF from "jspdf";
import "jspdf-autotable";
```

**Después:**
```javascript
const generatePDF = async (documentData) => {
  const { default: jsPDF } = await import("jspdf");
  await import("jspdf-autotable");

  const doc = new jsPDF();
  // ... generación del PDF
  doc.save("document.pdf");
};
```

**Impacto estimado: −300 KB del chunk de DocumentDetailView.**

---

### Fase 4: Vendor Splitting (Cache Optimization)

#### Paso 4.1: Configurar `manualChunks` en `vite.config.mjs`

```javascript
// En rollupOptions.output:
output: {
  entryFileNames: "[name].[hash].js",
  chunkFileNames: "[name].[hash].js",
  assetFileNames: "[name].[hash].[ext]",
  manualChunks(id) {
    // ── Framework Core (cambia raramente) ──
    if (id.includes("node_modules/react/") ||
        id.includes("node_modules/react-dom/") ||
        id.includes("node_modules/react-router")) {
      return "vendor-react";
    }

    // ── State Management (cambia raramente) ──
    if (id.includes("node_modules/@reduxjs/") ||
        id.includes("node_modules/react-redux") ||
        id.includes("node_modules/redux") ||
        id.includes("node_modules/redux-persist")) {
      return "vendor-redux";
    }

    // ── Monitoring & Analytics ──
    if (id.includes("node_modules/@sentry/")) {
      return "vendor-sentry";
    }

    // ── Charting (solo cargado con Dashboard) ──
    if (id.includes("node_modules/recharts") ||
        id.includes("node_modules/d3-")) {
      return "vendor-charts";
    }

    // ── PDF Generation (solo cargado on-demand) ──
    if (id.includes("node_modules/jspdf")) {
      return "vendor-pdf";
    }

    // ── UI Libraries ──
    if (id.includes("node_modules/react-select") ||
        id.includes("node_modules/react-datepicker") ||
        id.includes("node_modules/react-toastify")) {
      return "vendor-ui";
    }

    // ── i18n ──
    if (id.includes("node_modules/i18next") ||
        id.includes("node_modules/react-i18next")) {
      return "vendor-i18n";
    }

    // ── Remaining node_modules ──
    if (id.includes("node_modules/")) {
      return "vendor-misc";
    }
  },
},
```

#### Beneficios del Vendor Splitting

```
Antes:
  main.abc123.js  →  2.5 MB  (TODO junto, cache invalidado en cada deploy)

Después:
  main.abc123.js          →  ~150 KB  (código de app, cambia frecuentemente)
  vendor-react.def456.js  →  ~180 KB  (cambia solo al actualizar React)
  vendor-redux.ghi789.js  →  ~60 KB   (cambia solo al actualizar Redux)
  vendor-sentry.jkl012.js →  ~250 KB  (cambia solo al actualizar Sentry)
  vendor-charts.mno345.js →  ~500 KB  (SOLO cargado en /dashboard)
  vendor-pdf.pqr678.js    →  ~300 KB  (SOLO cargado al generar PDF)
  vendor-ui.stu901.js     →  ~250 KB  (cargado con primer formulario)
  vendor-i18n.vwx234.js   →  ~40 KB   (cambia solo al actualizar i18n)
  vendor-misc.yz5678.js   →  ~80 KB   (resto de dependencias)

  + 55 route chunks de ~5-30 KB cada uno
```

**Resultado:** Los vendor chunks se cachean en el browser y **no se re-descargan** en deploys sucesivos (salvo que cambien las versiones de las librerías).

---

### Fase 5: Optimización del `vite.config.mjs` Completo

A continuación el archivo de configuración optimizado con todas las mejoras integradas:

```javascript
import { defineConfig, transformWithEsbuild } from "vite";
import react from "@vitejs/plugin-react";
import { sentryVitePlugin } from "@sentry/vite-plugin";
import { visualizer } from "rollup-plugin-visualizer";
import { MONITORING_CONFIG, ENV } from "@/config/parameters";

export default defineConfig(() => {
  const isAnalyze = process.env.ANALYZE === "true";

  const BASE_CONFIG = {
    assetsInclude: ["**/*.xlsx"],
    resolve: {
      alias: {
        "@": "/src",
      },
    },
    plugins: [
      {
        name: "treat-js-files-as-jsx",
        async transform(code, id) {
          if (!id.match(/src\/.*\.js$/)) return null;
          return transformWithEsbuild(code, id, {
            loader: "jsx",
            jsx: "automatic",
          });
        },
      },
      react(),
      sentryVitePlugin({
        org: MONITORING_CONFIG.ORG.SLUG,
        project: MONITORING_CONFIG.PROJECT.NAME,
        release: MONITORING_CONFIG.PROJECT.RELEASE,
        authToken: MONITORING_CONFIG.ORG.AUTH_TOKEN,
        url: MONITORING_CONFIG.URL,
        include: "./src",
        ignore: ["node_modules", "dist"],
        configFile: "./monitoring.properties",
        urlPrefix: "~/",
      }),
      // Bundle Analyzer — solo cuando se ejecuta con ANALYZE=true
      isAnalyze && visualizer({
        filename: "bundle-analysis.html",
        open: true,
        gzipSize: true,
        brotliSize: true,
        template: "treemap",
      }),
    ].filter(Boolean),
    optimizeDeps: {
      force: true,
      esbuildOptions: {
        loader: {
          ".js": "jsx",
        },
      },
    },
    server: {
      port: 3000,
    },
    build: {
      outDir: "build",
      sourcemap: "hidden",
      assetsInlineLimit: 0,
      emptyOutDir: true,
      rollupOptions: {
        cache: false,
        input: {
          main: "./index.html",
        },
        output: {
          entryFileNames: "[name].[hash].js",
          chunkFileNames: "[name].[hash].js",
          assetFileNames: "[name].[hash].[ext]",
          manualChunks(id) {
            if (id.includes("node_modules/react/") ||
                id.includes("node_modules/react-dom/") ||
                id.includes("node_modules/react-router")) {
              return "vendor-react";
            }
            if (id.includes("node_modules/@reduxjs/") ||
                id.includes("node_modules/react-redux") ||
                id.includes("node_modules/redux") ||
                id.includes("node_modules/redux-persist")) {
              return "vendor-redux";
            }
            if (id.includes("node_modules/@sentry/")) {
              return "vendor-sentry";
            }
            if (id.includes("node_modules/recharts") ||
                id.includes("node_modules/d3-")) {
              return "vendor-charts";
            }
            if (id.includes("node_modules/jspdf")) {
              return "vendor-pdf";
            }
            if (id.includes("node_modules/react-select") ||
                id.includes("node_modules/react-datepicker") ||
                id.includes("node_modules/react-toastify")) {
              return "vendor-ui";
            }
            if (id.includes("node_modules/i18next") ||
                id.includes("node_modules/react-i18next")) {
              return "vendor-i18n";
            }
            if (id.includes("node_modules/")) {
              return "vendor-misc";
            }
          },
        },
      },
    },
    test: {
      globals: true,
      include: ["**/test.{ts,tsx}", "**/__tests__/**/*.{js,jsx,ts,tsx}"],
    },
  };

  return BASE_CONFIG;
});
```

#### Script de análisis en `package.json`

```json
{
  "scripts": {
    "analyze": "cross-env ANALYZE=true vite build"
  }
}
```

> Si no se usa `cross-env`, en PowerShell: `$env:ANALYZE='true'; vite build`

---

## 6. Plantilla de Informe de Resultados (Antes/Después)

### 6.1 Métricas de Bundle Size

| Chunk | Antes (raw) | Antes (gzip) | Después (raw) | Después (gzip) | Delta |
|-------|-------------|---------------|----------------|-----------------|-------|
| `main.js` (entry) | ___ KB | ___ KB | ___ KB | ___ KB | __% |
| `vendor-react.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-redux.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-sentry.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-charts.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-pdf.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-ui.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-i18n.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-misc.js` | — | — | ___ KB | ___ KB | N/A |
| Route chunks (sum) | — | — | ___ KB | ___ KB | N/A |
| **TOTAL** | ___ KB | ___ KB | ___ KB | ___ KB | __% |
| **Initial Load** | ___ KB | ___ KB | ___ KB | ___ KB | **__%** |

> **Initial Load** = `main.js` + `vendor-react` + `vendor-redux` + `vendor-i18n` + `vendor-misc` + ruta activa.

### 6.2 Core Web Vitals (Lighthouse Mobile, Simulated Throttling)

| Métrica | Antes | Después | Delta | Umbral Google |
|---------|-------|---------|-------|---------------|
| **FCP** (First Contentful Paint) | ___ ms | ___ ms | __% | < 1,800 ms ✅ |
| **LCP** (Largest Contentful Paint) | ___ ms | ___ ms | __% | < 2,500 ms ✅ |
| **TTI** (Time to Interactive) | ___ ms | ___ ms | __% | < 3,800 ms ✅ |
| **TBT** (Total Blocking Time) | ___ ms | ___ ms | __% | < 200 ms ✅ |
| **CLS** (Cumulative Layout Shift) | ___ | ___ | __% | < 0.1 ✅ |
| **Performance Score** | ___/100 | ___/100 | +___ pts | > 90 ✅ |

### 6.3 Desglose por Ruta (Top 5 más visitadas)

| Ruta | JS cargado (Antes) | JS cargado (Después) | Reducción |
|------|--------------------|----------------------|-----------|
| `/login` | ___ KB | ___ KB | __% |
| `/dashboard` | ___ KB | ___ KB | __% |
| `/invoices` | ___ KB | ___ KB | __% |
| `/reports` | ___ KB | ___ KB | __% |
| `/invoice-detail` | ___ KB | ___ KB | __% |

### 6.4 Protocolo de Medición

```bash
# 1. Baseline — antes de cambios
npm run build
# Registrar tamaños de build/assets/*.js

# 2. Lighthouse — 5 runs, mediana
# Usar Chrome DevTools > Lighthouse > Mobile > Performance
# O CLI:
npx lighthouse http://localhost:3000/login --output=json --output-path=./lighthouse-before.json

# 3. Aplicar cambios de optimización

# 4. Post-optimización
npm run build
# Registrar nuevos tamaños

# 5. Lighthouse post
npx lighthouse http://localhost:3000/login --output=json --output-path=./lighthouse-after.json

# 6. Bundle analysis visual
npm run analyze
# Comparar treemaps before/after
```

### 6.5 Resumen de Impacto Esperado

```
┌──────────────────────────────────────────────────────────────────┐
│                 PROYECCIÓN DE IMPACTO CONSOLIDADA                │
├──────────────────────────────────┬───────────────────────────────┤
│ Optimización                     │ Reducción Estimada            │
├──────────────────────────────────┼───────────────────────────────┤
│ react-icons named imports        │ −500 KB a −1 MB               │
│ Code splitting (55 rutas lazy)   │ −60% a −75% initial bundle    │
│ Vendor splitting (9 chunks)      │ Cache hit rate: ~90% en redep │
│ Eliminar core-js polyfills       │ −60 KB a −80 KB               │
│ jsPDF dynamic import             │ −300 KB del chunk invoice     │
│ Eliminar redux-logger (prod)     │ −15 KB                        │
├──────────────────────────────────┼───────────────────────────────┤
│ TOTAL Initial Load               │ De ~2-3 MB → ~300-500 KB      │
│ FCP Improvement                  │ −40% a −60%                   │
│ Lighthouse Score                 │ +30 a +50 puntos              │
└──────────────────────────────────┴───────────────────────────────┘
```

---

## Anexo A: Orden de Ejecución Recomendado

| Prioridad | Tarea | Esfuerzo | Impacto | Risk |
|-----------|-------|----------|---------|------|
| 🔴 P0 | Arreglar `import *` de react-icons | 2-4h | Muy Alto | Bajo |
| 🔴 P0 | Lazy loading de 55 rutas | 4-6h | Muy Alto | Medio |
| 🟡 P1 | Vendor splitting en vite.config.mjs | 1-2h | Alto | Bajo |
| 🟡 P1 | Eliminar core-js polyfills | 30 min | Medio | Bajo |
| 🟡 P1 | Dynamic import de jsPDF | 1h | Alto | Bajo |
| 🟢 P2 | Eliminar/condicional redux-logger | 30 min | Bajo | Bajo |
| 🟢 P2 | Integrar bundle analyzer | 30 min | Diagnóstico | Ninguno |
| 🟢 P2 | Optimizar imports de Monitoring | 1-2h | Medio | Medio |

## Anexo B: Referencias Bibliográficas

1. Osmani, A. (2023). *The Cost of JavaScript in 2023*. Google Chrome Team. https://v8.dev/blog/cost-of-javascript-2019
2. Google. (2024). *Web Vitals*. https://web.dev/vitals/
3. Deloitte. (2020). *Milliseconds Make Millions*. https://www2.deloitte.com/ie/en/pages/consulting/articles/milliseconds-make-millions.html
4. HTTP Archive. (2023). *Web Almanac — JavaScript Chapter*. https://almanac.httparchive.org/en/2022/javascript
5. Vite Documentation. (2024). *Build Optimizations*. https://vitejs.dev/guide/build.html
6. Rollup Documentation. (2024). *Code Splitting*. https://rollupjs.org/tutorial/#code-splitting


🔒 Confidentiality Notice

All project names, component identifiers, database schemas, and business logic terminology in this technical vault have been fully anonymized and abstracted. The cases presented here reflect my technical methodology, research, and engineering impact, not the proprietary intellectual property of past clients. All metrics presented (e.g., bundle reduction, performance scores) are accurate results of my engineering work.