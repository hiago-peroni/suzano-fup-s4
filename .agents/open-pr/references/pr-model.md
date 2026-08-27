# 🧹 Overview
Make the plant field always editable and decouple cart button visibility from plant selection state.

### ✅ Definition of Done

- [x] `plantSelectionEnabled` is always `true` regardless of the presence of the `/bs/plant1` URL parameter
- [x] A new `cartEnabled` property controls cart button visibility, set to `true` only when `/bs/plant1` is present in the URL
- [x] Cart buttons remain visible when the catalog is opened via external integration (SAP GUI / Cockpit OM)
- [x] Plant field remains editable in all access scenarios
- [x] Both initialization paths (`Component.js` and `model/models.js`) are updated consistently

## 🔧 Problem Overview

### Issue Description
The catalog's plant selection field was locked (disabled) whenever the application was opened via external integration systems such as SAP GUI or Cockpit OM. These integrations pass a `/bs/plant1` URL parameter to pre-set the plant. When this parameter was present, `plantSelectionEnabled` in `UserPreferences` was set to `false`, which disabled the `MultiInput` field bound to `enabled="{UserPreferences>/plantSelectionEnabled}"` in `CatalogSearchConfigure.fragment.xml`. This prevented users from changing the plant even when needed.

### Root Cause
The `plantSelectionEnabled` flag was serving two conflicting purposes simultaneously:
- It controlled the `enabled` state of the plant `MultiInput` field in the configuration dialog
- It controlled the `visible` state of all cart-related buttons via inverted binding expressions (`!${UserPreferences>/plantSelectionEnabled}`) across 7 view files

When `plantSelectionEnabled` was `false` (triggered by `/bs/plant1`), the plant was locked AND the cart buttons were visible. Setting it to `true` would unlock the plant but hide all cart buttons. There was no way to satisfy both requirements with a single property.

## 🗃 Data Flow

### Input Sources
- **URL parameter `/bs/plant1`**: Passed by external integrations (SAP GUI, Cockpit OM) to pre-set the plant ID when opening the catalog
- **`UserPreferences` JSON model**: Runtime model that stores user settings and feature flags consumed by all views via SAPUI5 data binding

### Output Formats
- **`UserPreferences>/plantSelectionEnabled`**: Boolean — always `true` after this change, keeping the plant field enabled in all scenarios
- **`UserPreferences>/cartEnabled`**: New boolean — `true` only when `/bs/plant1` is present in the URL, controlling visibility of all cart-related buttons

### Key Components
- **`Component.js`**: `webapp/Component.js` — primary initialization path; sets both `plantSelectionEnabled` and `cartEnabled` in `setUserPreferencesModel()`
- **`models.js`**: `webapp/model/models.js` — legacy initialization path via `_setUserPreferencesJSONModel()`; also sets both properties
- **`CatalogSearchConfigure.fragment.xml`**: plant `MultiInput` field bound to `plantSelectionEnabled`
- **Cart views**: `ItemSearch.view.xml`, `CatalogMaterialList.fragment.xml`, `CatalogServiceList.fragment.xml`, `FavoriteListManagement.view.xml`, `ItemDetail.view.xml` — cart buttons rebound to `cartEnabled`

## 🧪 Test Script

> [!NOTE]
>
> You need to have access to the development environment to test.

### Test 1: Plant field is editable via external integration
```text
1. Open the catalog with the URL parameter /bs/plant1=<plant-id>
2. Click the "Configurations" button (gear icon) in the page header
3. Observe the plant MultiInput field in the dialog

Expected: The plant field is enabled and the user can open the plant selection
dialog and change the plant freely
```

### Test 2: Cart buttons are visible via external integration
```text
1. Open the catalog with the URL parameter /bs/plant1=<plant-id>
2. Perform a search to load catalog items

Expected:
- "Cart" button is visible in the top-right area of the page (above "Favorites")
- "Add to Cart" button and quantity field are visible on each material/service item
- Toolbar multiselect and cart buttons are visible in the result list header
- "Add to Cart" button is visible in the Favorites management page
- Cart and Checkout buttons are visible in the Item Detail page
```

### Test 3: Direct access without URL parameter
```text
1. Open the catalog directly without the /bs/plant1 URL parameter
2. Observe the top-right page area
3. Click "Configurations" and inspect the plant field

Expected:
- "Cart" button is NOT visible (cartEnabled = false)
- Plant field in the configuration dialog IS editable (plantSelectionEnabled = true)
- No regression in existing direct-access behavior
```

## 📋 System Impact

### Business Impact
- **Plant flexibility**: Users accessing the catalog via SAP GUI or Cockpit OM can now change the plant during their session, removing a restriction that previously locked the plant to the one pre-set by the external integration
- **Cart workflow preserved**: All cart-related actions remain fully available when the catalog is opened via external integration, maintaining the purchasing workflow intact

### Technical Impact
- **Property decoupling**: `UserPreferences` now carries a dedicated `cartEnabled` property independent of `plantSelectionEnabled`, allowing each UI concern to be controlled separately
- **Two initialization paths updated**: Both `Component.js` and `model/models.js` are updated to ensure consistent behavior regardless of which path initializes the model
- **Binding simplification**: Cart button bindings changed from `{= !${UserPreferences>/plantSelectionEnabled} }` to `{UserPreferences>/cartEnabled}`

### Integration Points
- **SAP GUI / Cockpit OM**: These systems pass `/bs/plant1` in the URL. The cart workflow remains intact while the plant field lock is removed
- **SAPUI5 data binding**: 12 binding expressions across 5 view files updated from `!plantSelectionEnabled` to `cartEnabled`

## 🔍 Code Changes

### Modified Files

#### Application Bootstrap
- `webapp/Component.js`
  - In `setUserPreferencesModel()`: replaced the conditional `plantSelectionEnabled` assignment with `true` always, and added `cartEnabled = !!urlPlantId`

#### Model Initialization
- `webapp/model/models.js`
  - In `_setUserPreferencesJSONModel()`: replaced the `if/else` block with a direct `setProperty("/plantSelectionEnabled", true)` and added `setProperty("/cartEnabled", !!oComponent._getParameterByName("/bs/plant1"))`

#### Views — Cart Button Visibility
- `webapp/view/ItemSearch.view.xml` — Cart button `visible` binding updated to `{UserPreferences>/cartEnabled}`
- `webapp/view/CatalogMaterialList.fragment.xml` — 5 bindings updated: toolbar multiselect, toolbar cart button, `ToolbarSpacer`, quantity `HBox`, and item cart button
- `webapp/view/CatalogServiceList.fragment.xml` — 4 bindings updated: toolbar multiselect, toolbar cart button, `ToolbarSpacer`, and item cart button
- `webapp/view/FavoriteListManagement.view.xml` — `PositiveAction` "Add to Cart" button visibility updated
- `webapp/view/ItemDetail.view.xml` — Cart action button and Checkout action button visibility updated

### Code Snippets

#### Component.js — Before/After
```javascript
// BEFORE
userPreferences['plantSelectionEnabled'] = urlPlantId ? false : true;

// AFTER
userPreferences['plantSelectionEnabled'] = true;
userPreferences['cartEnabled'] = !!urlPlantId;
```

#### models.js — Before/After
```javascript
// BEFORE
if (oComponent._getParameterByName("/bs/plant1")) {
    oUserPreferencesJSONModel.setProperty("/plantSelectionEnabled", false);
} else {
    oUserPreferencesJSONModel.setProperty("/plantSelectionEnabled", true);
}

// AFTER
oUserPreferencesJSONModel.setProperty("/plantSelectionEnabled", true);
oUserPreferencesJSONModel.setProperty("/cartEnabled", !!oComponent._getParameterByName("/bs/plant1"));
```

#### View Binding — Before/After
```xml
<!-- BEFORE -->
<custom:ZVSButton visible="{= !${UserPreferences>/plantSelectionEnabled} }" ... />

<!-- AFTER -->
<custom:ZVSButton visible="{UserPreferences>/cartEnabled}" ... />
```

---
Pull Request opened by [Augment Code](https://www.augmentcode.com/) with guidance from the PR author