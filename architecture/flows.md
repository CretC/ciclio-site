# Fluxurile critice

Scris de mână, din cod (2026-10-11). Pentru graful de foldere vezi `modules.md`
(generat). **Regula:** o schimbare care modifică un drum de mai jos actualizează
acest fișier în același commit (`plan/README.md`). Căile sunt relative la `src/`.
Pașii „Pe telefon” intră în QA-ul de release când fluxul e atins de diff
(`plan/RELEASE.md`, „Analiza de impact”).

## 1. Notificare programată și reconstruire de la zero
**Scop:** notificările sunt funcție pură de baza de date; reconstruirea dă mereu aceeași stare.
**Drum:** `app/startup.ts` (`runStartup`, la fiecare pornire) → `notifications/scheduler.ts`
(`reconcileNotifications`) → `data/obligations.ts`, `data/settings.ts` → `domain/` (praguri,
`toUtcInstant` în `domain/dates.ts`). După fiecare modificare: `shared/stores/obligationsStore.ts`
→ `refreshAfterChange` (scheduler). În fundal: `notifications/backgroundTask.ts`
(`ensureBackgroundReconcile`, task `obligatii-reconcile-notifications`). Jurnal: `data/sentNotifications.ts`.
**Pe telefon:**
1. Adaugă o obligație cu scadența peste 2 zile; Setări → Notificări programate arată pragurile.
2. Schimbă un prag în Remindere; lista programată se actualizează.
3. Închide forțat aplicația și redeschide: aceleași notificări, fără dubluri.
4. Repornește telefonul: notificările rămân programate.

## 2. Tap pe notificare → pagina bunului cu obligația deschisă
**Scop:** tapul duce la cardul obligației, nu la o listă.
**Drum:** `notifications/responses.ts` (`resolveNotificationResponse`) → `app/useNotificationNavigation.ts`
(așteaptă `navigationReady`; `acknowledgeSentNotifications` din `data/`; `prepareExternalNavigation` din
`app/closeOverlays.ts`) → ruta `Detail` din `features/obligations/ObligationsNavigator.tsx` →
`ObligationRedirect.tsx` (`assetPageFor` din `domain/`) → `navigation.replace('Asset')` →
`AssetDetailScreen.tsx` (cardul `focus` se deschide).
**Pe telefon:**
1. Cu aplicația închisă, tap pe o notificare: se deschide pagina bunului, cardul obligației e deschis.
2. Același lucru cu aplicația în fundal și cu panoul de notificări deschis (panoul se închide).
3. Tap pe o notificare a unei obligații șterse: revine fără eroare.
4. Back de pe pagina bunului nu intră în Setări.

## 3. „Reînnoiește” din notificare
**Scop:** butonul din notificare deschide direct reînnoirea; „Amână” și „Rezolvat” nu deschid nimic.
**Drum:** acțiunile declarate în `notifications/scheduler.ts` → `responses.ts` (`renew|postpone|resolve`) →
`useNotificationNavigation.ts` (`action: 'renew'` → `Detail`) → `ObligationRedirect.tsx` → `Asset` cu
`action` → `AssetDetailScreen.tsx` (efectul pe `action === 'renew'`). Fundal fără aplicație:
`notifications/actionTask.ts` (`postponeFromNotification`).
**Pe telefon:**
1. „Reînnoiește” din notificare: pagina bunului cu formularul de reînnoire deschis.
2. „Amână 7 zile”: notificarea dispare, nu se deschide aplicația, reamintirea se mută.
3. „Rezolvat”: lista, obligația nu mai e activă.

## 4. Deep link `ciclio://`
**Scop:** `ciclio://obligation/<id>` și `ciclio://add` deschid ținta, restul e ignorat.
**Drum:** `domain/deepLink.ts` (`parseDeepLink`) → `app/useDeepLinkNavigation.ts` (`getInitialURL` + eveniment
`url`) → `closeOverlays.ts` → `add`: `List` cu `choose` (deschide `AddChooser.tsx`); `obligation`: `Detail` →
fluxul 2. Schema: `app.json` (`scheme`).
**Pe telefon:**
1. `adb shell am start -a android.intent.action.VIEW -d "ciclio://add"`: apare alegerea „ce adaugi”.
2. `ciclio://obligation/<id real>`: pagina bunului cu cardul deschis.
3. `ciclio://obligation/nu-e-uuid`: nu se întâmplă nimic.

## 5. Widget (randare + tap + „+”)
**Scop:** widgetul arată următoarele scadențe și trimite în aplicație.
**Drum:** `index.ts` (`registerWidgetTaskHandler`) → `widget/widgetTaskHandler.ts` → `widget/nextDueWidgetData.ts`
(`loadWidgetData`, din `data/`+`domain/`) → `renderNextDueWidget.tsx` (`NextDueWidget.tsx`,
`UpcomingWidget.tsx`, culori `colors.ts`). Reîmprospătare: `widget/refreshWidget.ts` apelat din
`shared/stores/obligationsStore.ts`, `assetsStore.ts` și `features/settings/ResetDataSection.tsx`. Tap/„+”:
`clickAction="OPEN_URI"` cu `ciclio://…` → fluxul 4.
**Pe telefon:**
1. Pune ambele widgeturi: arată datele reale, culori corecte în temă luminoasă și întunecată.
2. Adaugă/rezolvă o obligație: widgetul se actualizează fără deschiderea aplicației.
3. Tap pe un rând: pagina bunului; tap pe „+”: alegerea de adăugare.
4. Resetează datele: widgetul se golește.

## 6. Share intent
**Scop:** textul partajat din altă aplicație ajunge în notițele unei obligații noi.
**Drum:** `expo-share-intent` → `app/App.tsx` (provider) → `app/useShareIntentNavigation.ts`
(`clampText`, `TEXT_LIMITS.sharedText` din `domain/`) → `Add` cu `initialNotes` →
`features/obligations/AddObligationScreen.tsx`.
**Pe telefon:**
1. Din Chrome/Mesaje: Partajează text → Ciclio: formularul „Adaugă” cu textul în notițe.
2. Cu aplicația deja deschisă pe o pagină din Setări: formularul se deschide, Setările se închid.
3. Text foarte lung: se taie la limită, fără blocaj.

## 7. Adăugare obligație (formular, „+” după context, card nou pe pagina bunului)
**Scop:** o obligație nouă ajunge corect în bunul potrivit și în notificări.
**Drum:** „+” din bara de jos (`app/RootTabs.tsx`, `AddTabButton` → `List` cu `choose`) sau FAB/widget/deep link →
`AddChooser.tsx` → `AddObligationScreen.tsx` (`obligationSave.ts`: `saveUnlessDuplicate`) sau
`QuickAddBundleScreen.tsx` → `shared/stores/obligationsStore.ts` (`create`) → `data/obligations.ts` →
`refreshAfterChange` (fluxul 1) + `refreshNextDueWidget` (fluxul 5) + sincronizare (fluxul 9). Pe pagina bunului:
„+ Obligație” → `edit.request({kind:'open', target:{kind:'newObligation'}})` → `ObligationEditor.tsx` (card nou).
Din pagina bunului „Adaugă” primește `assetId` (`ObligationsNavigator.tsx`, ruta `Add`).
**Pe telefon:**
1. „+” din bară: alegerea; salvează o obligație: apare în listă, notificarea e programată.
2. Din pagina unui bun, „+ Obligație”: apare un card nou în pagină, salvat rămâne în același bun.
3. Prima obligație salvată: cererea de permisiune pentru notificări (o singură dată).
4. Duplicat: apare întrebarea, nu se salvează dublu.

## 8. Editare pe pagina bunului (sesiune unică, salvare atomică)
**Scop:** un singur lucru deschis la editare; salvarea scrie obligația și detaliile într-o tranzacție.
**Drum:** `features/obligations/AssetDetailScreen.tsx` → `useEditSession.ts` (reducer `domain/editSession.ts`,
gardă Back) → `ObligationEditor.tsx` / `AssetHeaderEditor.tsx` → `obligationsStore.update` →
`data/saveObligation.ts` (`saveObligationWithFields`, o tranzacție) → `refreshAfterChange` + widget + sync.
**Pe telefon:**
1. Deschide editarea unui card, schimbă scadența, Salvează: cardul și notificarea se actualizează.
2. Cu modificări nesalvate, deschide alt card: apare dialogul de renunțare, nu pierzi tăcut.
3. Back cu modificări nesalvate: dialog, nu ieșire.
4. Edit pe două câmpuri, salvează: ambele persistă după repornire.

## 9. Sincronizarea cu contul
**Scop:** doar cine are cont atinge serverul; notificările nu depind de sync.
**Drum:** `features/account/AccountScreen.tsx` / `auth.ts` → `features/account/autoSync.ts` (`startAutoSync`
din `app/startup.ts`; după autentificare, la prim-plan, debounce 4 s după schimbări în stores) →
`features/account/sync.ts` (`syncNow`: push apoi pull) → `data/sync/*` (`client.ts`, `obligations.ts`,
`assets.ts`, `appSettings.ts`, …). Fără cont: nicio rețea.
**Pe telefon:**
1. Fără cont: modul avion nu schimbă nimic; nicio cerere (fără erori de rețea).
2. Autentifică-te pe un telefon cu date: datele urcă; pe un al doilea telefon coboară.
3. Editează o obligație, așteaptă ~5 s: apare pe celălalt telefon după sincronizare.
4. Ieșire din cont: datele locale rămân, notificările la fel.

## 10. Backup și import
**Scop:** copie manuală completă a bazei de date, restaurabilă fără pierdere.
**Drum:** `features/settings/BackupScreen.tsx` → `data/backup.ts` (`exportDatabaseFile`, `importDatabaseFile`:
validare, migrare pe copie, comparare de schemă) + `data/backupValidation.ts`, `data/rowWarnings.ts` →
`reconcileNotifications` (fluxul 1) → reîncărcarea stores.
**Pe telefon:**
1. Exportă, șterge o obligație, importă fișierul: obligația revine, notificările sunt reprogramate.
2. Importă un fișier străin (alt `.db`/imagine): mesaj de eroare clar, datele neatinse.
3. După import, widgetul și lista arată datele restaurate.

## 11. Resetare
**Scop:** ștergerea completă a datelor locale, fără rămășițe.
**Drum:** `features/settings/ResetDataSection.tsx` (cuvânt de confirmare: `domain/resetData.ts`,
`runResetSteps`: `cancelNotifications` → `signOut` → `wipeLocalData` = `data/reset.ts` → `refreshWidgets`) →
`Updates.reloadAsync()`.
**Pe telefon:**
1. Resetează cu cuvântul de confirmare greșit: butonul rămâne inactiv.
2. Resetează: aplicația repornește în onboarding, Setări → Notificări programate e gol, widgetul gol.
3. Cont conectat: după reset ești deconectat.

## 12. Bara de jos, panoul de notificări și Setările
**Scop:** navigarea de bază, fără suprapuneri.
**Drum:** `app/RootTabs.tsx` (tab-uri Obligations / „+” / Assets) → `HeaderActions.tsx` (clopot `HeaderBell.tsx`,
cog → `Settings` pe stiva tab-ului, `closeSettingsStacks`) → `NotificationsPanel.tsx` +
`notificationsPanelStore.ts` (`SentNotificationsScreen` din `features/reminders`) → `closeOverlays.ts`
(`prepareExternalNavigation`) → `SettingsNavigator.tsx`. Insigna: `shared/stores/sentNotificationsBadgeStore.ts`.
**Pe telefon:**
1. Clopot: panoul se deschide deasupra barei; tap pe un tab sau pe „+” îl închide.
2. Cog: Setările se deschid; în celălalt tab nu rămâne o a doua instanță.
3. Din panou, tap pe o notificare: panoul se închide, se deschide pagina bunului.
4. Insigna clopotului dispare după vizită.

## 13. Actualizarea în aplicație
**Scop:** Play In-App Updates, mod flexibil, o dată pe zi, fără a bloca utilizatorul.
**Drum:** `app/App.tsx` (efect la pornire) → `shared/appUpdate.ts` (`checkForAppUpdate`) →
`domain/appUpdate.ts` (`shouldCheckForAppUpdate`) → snackbar (`shared/ui/Snackbar.tsx`,
`shared/stores/snackbarStore.ts`). Data OTA în `features/settings/SettingsScreen.tsx`.
**Pe telefon:**
1. Instalare internă cu versiune mai veche și una nouă în Play: apare „Actualizare disponibilă”.
2. Descarcă: apare „Repornește”; repornirea instalează.
3. A doua deschidere în aceeași zi: nu se mai întreabă.

## 14. Onboarding
**Scop:** prima pornire explică aplicația o singură dată; instalările cu date îl sar.
**Drum:** `app/App.tsx` → `app/StartupGate.tsx` (`runStartup`) → `app/OnboardingGate.tsx`
(`isOnboardingDone`/`markOnboardingDone` din `data/settings.ts`, `countObligationsByStatus`) →
`features/onboarding/OnboardingScreen.tsx` (`OnboardingChoice` → navigarea inițială din `App.tsx`).
**Pe telefon:**
1. Instalare curată: onboarding-ul apare; paginile descriu aplicația curentă (Bunuri, „+”, clopot).
2. Termină: nu mai apare la repornire.
3. Update peste o instalare cu obligații: nu apare.
4. Alegerea de la final duce unde zice (adaugă / listă).
