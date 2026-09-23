# Guía Técnica de Desarrollo LWC en Salesforce OmniStudio / Vlocity

> Cheat sheet de referencia técnica y patrones de arquitectura para desarrollo con Lightning Web Components (LWC) en entornos OmniStudio.

---

## Tabla de Contenidos

1. [Imports Necesarios](#imports-necesarios)
2. [Clase Base del Componente](#clase-base-del-componente)
3. [Meta XML Estándar](#meta-xml-estándar)
4. [Llamar a un Integration Procedure (IP)](#llamar-a-un-integration-procedure-ip)
5. [Llamar a un DataRaptor / DataMapper (DR)](#llamar-a-un-dataraptor--datamapper-dr)
6. [Llamar a Apex (omniRemoteCall)](#llamar-a-apex-omniremotecall)
7. [Llamar a Apex con OmniscriptActionCommonUtil](#llamar-a-apex-con-omniscriptactioncommonutil)
8. [Métodos del OmniScript](#métodos-del-omniscript)
9. [Navegación entre Steps](#navegación-entre-steps)
10. [Validación del Componente desde el OmniScript](#validación-del-componente-desde-el-omniscript)
11. [Comunicación Padre-Hijo (Custom Events)](#comunicación-padre-hijo-custom-events)
12. [Imports Útiles Adicionales](#imports-útiles-adicionales)
13. [Patrones Comunes](#patrones-comunes)
14. [FlexCards](#flexcards)
15. [Tips y Buenas Prácticas](#tips-y-buenas-prácticas)

---

## Imports Necesarios

```javascript
// === SIEMPRE ===
import { LightningElement, api, track } from "lwc";
import { OmniscriptBaseMixin } from "vlocity_cmt/omniscriptBaseMixin";
import { NavigationMixin } from "lightning/navigation";

// === PARA INTEGRATION PROCEDURES ===
import { getNamespaceDotNotation } from "vlocity_cmt/omniscriptInternalUtils";
import { OmniscriptActionCommonUtil } from "vlocity_cmt/omniscriptActionUtils";

// === PARA DATARAPTORS / DATAMAPPERS ===
import { getDataHandler } from "vlocity_cmt/utility";

// === OPCIONALES ===
import { cloneDeep } from "vlocity_cmt/lodash";              // clonar objetos profundos
import { loadStyle } from "lightning/platformResourceLoader";  // cargar CSS custom
import { ShowToastEvent } from "lightning/platformShowToastEvent"; // mostrar toasts
```

---

## Clase Base del Componente

```javascript
// Forma estándar (la más utilizada)
export default class CustomOmniLwcComponent extends OmniscriptBaseMixin(NavigationMixin(LightningElement)) {
    // ...
}

// Sin navegación (componentes de presentación simples)
export default class CustomOmniLwcComponent extends OmniscriptBaseMixin(LightningElement) {
    // ...
}

// Orden invertido de Mixins
export default class CustomOmniLwcComponent extends NavigationMixin(OmniscriptBaseMixin(LightningElement)) {
    // ...
}
```

---

## Meta XML Estándar

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>60.0</apiVersion>
    <isExposed>true</isExposed>
    <runtimeNamespace>vlocity_cmt</runtimeNamespace>
    <masterLabel>CustomOmniLwcComponent</masterLabel>
</LightningComponentBundle>
```

---

## Llamar a un Integration Procedure (IP)

Se usa `OmniscriptActionCommonUtil` + `IntegrationProcedureService`.

### Setup (en connectedCallback)

```javascript
connectedCallback() {
    this._actionUtil = new OmniscriptActionCommonUtil();
    this._ns = getNamespaceDotNotation();
}
```

### Llamada al IP

```javascript
callIntegrationProcedure() {
    const ipInput = {
        cardNumber: "4111111111111111",
        accountId: "001XXXXXXXXXXXXXXX"
    };

    const params = {
        input: JSON.stringify(ipInput),
        sClassName: `${this._ns}IntegrationProcedureService`,
        sMethodName: "Type_ProcedureName",  // Ej: "Payment_ValidateCard", "Account_FetchDetails"
        options: JSON.stringify({})
    };

    this._actionUtil.executeAction(params, null, this, null, null)
        .then(response => {
            // La respuesta se obtiene en response.result.IPResult
            let result = response.result.IPResult;
            console.log("Resultado IP:", result);
        })
        .catch(error => {
            console.error("Error en IP:", error);
        });
}
```

### Variante con omniRemoteCall (alternativa)

```javascript
invokeIntegrationProcedure() {
    const input = { body: { accountNumber: "ACC-12345" } };
    const params = {
        input,
        sClassName: 'vlocity_cmt.IntegrationProcedureService',
        sMethodName: 'Type_ProcedureName',
        options: '{}',
    };

    this.omniRemoteCall(params, true)
        .then(response => {
            let result = response.result.IPResult;
            console.log("IP Result:", result);
        })
        .catch(error => {
            console.error(error);
        });
}
```

---

## Llamar a un DataRaptor / DataMapper (DR)

Se usa `getDataHandler` de `vlocity_cmt/utility`.

```javascript
callDataRaptor() {
    let drRequest = {
        type: "DataRaptor",
        value: {
            bundleName: "DR_ExtractAccountDetails",  // Nombre del DataRaptor / DataMapper
            inputMap: {
                "AccountId": "001XXXXXXXXXXXXXXX",
                "IsActive": true
            },
            optionsMap: "{}",
        },
    };

    getDataHandler(JSON.stringify(drRequest))
        .then(result => {
            // El resultado se devuelve como un JSON serializado en string
            const jsonResult = JSON.parse(result);
            console.log("DR Result:", jsonResult);
        })
        .catch(error => {
            console.error("Error en DataRaptor:", error);
        });
}
```

---

## Llamar a Apex (omniRemoteCall)

Se usa `this.omniRemoteCall` expuesto por `OmniscriptBaseMixin`.

```javascript
callApex() {
    const params = {
        input: JSON.stringify({}),                     // Parámetros de entrada
        sClassName: 'PaymentUtilsController',          // Clase Apex
        sMethodName: 'getDocumentTypes',               // Método de la clase
        options: JSON.stringify({                      // Opciones o argumentos adicionales
            paymentMethodIds: ["id1", "id2"],
            orderId: "801XXXXXXXXXXXXXXX"
        }),
    };

    this.omniRemoteCall(params, true)
        .then(response => {
            let result = response.result;
            console.log("Apex Result:", result);
        })
        .catch(error => {
            console.error("Error en Apex:", error);
        });
}
```

### Ejemplo: Ejecución secuencial (Async/Await)

```javascript
async executeSequentialApexCalls() {
    // Primera ejecución
    let params1 = {
        input: "{}",
        sClassName: "OrderController",
        sMethodName: "cancelReservedStock",
        options: JSON.stringify({ "orderId": this.orderId })
    };
    await this.omniRemoteCall(params1, true).then(response => {
        console.log("Paso 1 completado:", response.result);
    });

    // Segunda ejecución
    let params2 = {
        input: "{}",
        sClassName: "OrderController",
        sMethodName: "rollbackOrderInvoicing",
        options: JSON.stringify({ "orderId": this.orderId })
    };
    await this.omniRemoteCall(params2, true).then(response => {
        console.log("Paso 2 completado:", response.result);
    });
}
```

---

## Llamar a Apex con OmniscriptActionCommonUtil

Alternativa recomendada cuando se mantiene el mismo patrón de interacción que los IPs.

```javascript
callApexViaActionUtil() {
    this._actionUtil = new OmniscriptActionCommonUtil();

    const inputData = { OrderData: this.currentOrder };
    const options = { OrderId: this.currentOrder.Id };

    const params = {
        input: JSON.stringify(inputData),
        sClassName: 'InventoryManagerController',
        sMethodName: 'reassignSerialNumber',
        options: JSON.stringify(options)
    };

    this._actionUtil.executeAction(params, null, this, null, null)
        .then(resp => {
            let result = resp.result;
            console.log("Apex Action Result:", result);
        })
        .catch(error => {
            console.error("Error en Apex Action:", error);
        });
}
```

---

## Métodos del OmniScript

### omniApplyCallResp — Inyectar datos al JSON del OmniScript

```javascript
// Actualizar una clave simple
this.omniApplyCallResp({ selectedPaymentType: "CreditCard" });

// Actualizar múltiples claves simultáneamente
this.omniApplyCallResp({
    selectedPaymentType: "CreditCard",
    paymentLabel: "Tarjeta de Crédito",
    isSpecialOperation: true
});

// Actualizar objetos anidados
this.omniApplyCallResp({
    StepAccountDetails: {
        State: "Buenos Aires",
        City: "Capital Federal"
    }
});

// Enviar arrays
this.omniApplyCallResp({ selectedInvoices: [invoice1, invoice2] });
```

### omniUpdateDataJson — Sobrescribir el DataJson

```javascript
// Actualizar con un payload completo
this.omniUpdateDataJson(this.selectedPayload);

// Reiniciar o limpiar
this.omniUpdateDataJson(null);
```

### omniJsonData — Lectura del JSON del OmniScript

```javascript
connectedCallback() {
    // Leer el objeto completo del DataJson
    const fullData = this.omniJsonData;
    
    // Clonar para evitar la mutación directa del estado
    this.localState = JSON.parse(JSON.stringify(this.omniJsonData));
    
    // Lectura de valores de nodos específicos
    const accountId = this.omniJsonData.accountId;
    const channel = this.omniJsonData.OrderSummary?.Channel;
}
```

### omniScriptHeaderDef — Definiciones de Cabecera del OmniScript

```javascript
connectedCallback() {
    // Obtener el nombre del Step activo
    const currentStepName = this.omniScriptHeaderDef.asName;
    
    // Validar si el mapa de etiquetas incluye la acción CANCEL
    const hasCancelButton = this.omniScriptHeaderDef.labelMap.hasOwnProperty('CANCEL');
}
```

---

## Navegación entre Steps

```javascript
// Avanzar al siguiente paso
this.omniNextStep();

// Regresar al paso anterior
this.omniPrevStep();

// Navegar directamente a un Step por su nombre exacto
this.omniNavigateTo("SelectPaymentMethodsStep");

// Disparar un evento de avance automático
const autoAdvanceEvent = new CustomEvent('omniautoadvance', {
    bubbles: true,
    cancelable: true,
    composed: true,
    detail: { moveToStep: 'next' }
});
this.dispatchEvent(autoAdvanceEvent);
```

### Navegación mediante NavigationMixin

```javascript
// Navegar a un componente wrapper de OmniStudio
this[NavigationMixin.Navigate]({
    type: 'standard__component',
    attributes: {
        componentName: 'vlocity_cmt__vlocityLWCOmniWrapper'
    },
    state: {
        c__target: "c:accountManagementEnglish",
        c__layout: "lightning",
        c__tabIcon: "custom:custom18",
        c__tabLabel: "AccountManagement"
    }
}, [true]);

// Navegar a un registro específico de Salesforce
this[NavigationMixin.Navigate]({
    type: 'standard__recordPage',
    attributes: {
        recordId: this.accountId,
        objectApiName: 'Account',
        actionName: 'view'
    }
}, { replace: true });

// Navegar a una página web o App Page
this[NavigationMixin.Navigate]({
    type: 'standard__webPage',
    attributes: { url: '/lightning/page/home' }
});
```

---

## Validación del Componente desde el OmniScript

Permite asegurar que el LWC cumpla las condiciones necesarias antes de que el usuario avance de paso.

```javascript
// Habilitar la validación en el ciclo de renderizado
renderedCallback() {
    this.omniValidate(true);
}

// Exponer el método 'checkValidity' para que el OmniScript pueda invocarlo
@api checkValidity() {
    return this.componentIsValid;
}

// Lógica de evaluación de validez
get componentIsValid() {
    return this.isFormCompleted === true;
}
```

---

## Comunicación Padre-Hijo (Custom Events)

### Componente Hijo → Notificación al Padre

```javascript
const paymentSelectedEvent = new CustomEvent("paymentselected", {
    detail: {
        cardNumber: this.cardNumber,
        cardType: this.cardType,
        paymentMethod: "CreditCard",
        last4Digits: this.last4Digits
    },
    bubbles: true
});
this.dispatchEvent(paymentSelectedEvent);
```

### Componente Padre → Manejo en el Template HTML

```html
<c-custom-payment-form
    onpaymentselected={handlePaymentSelection}>
</c-custom-payment-form>
```

### Patrón Avanzado: Despacho de Acciones desde el Hijo

```javascript
// Modificar propiedades del padre
updateParentProperty(labelProperty, valueProperty) {
    let customEvent = new CustomEvent("actiondispatch", {
        detail: {
            eventName: 'updateProperty',
            dataChange: {
                labelProperty: labelProperty,
                valueProperty: valueProperty,
                index: `${this.index}`
            }
        }
    });
    this.dispatchEvent(customEvent);
}

// Invocación de funciones del padre
executeParentFunction(func) {
    let customEvent = new CustomEvent("actiondispatch", {
        detail: {
            eventName: 'executeFunction',
            dataFunction: {
                func: func,
                index: `${this.index}`
            }
        }
    });
    this.dispatchEvent(customEvent);
}
```

---

## Imports Útiles Adicionales

```javascript
// Copia profunda de estructuras de datos (Lodash)
import { cloneDeep } from 'vlocity_cmt/lodash';
const copyList = cloneDeep(this.paymentList);

// Carga dinámica de Static Resources para estilos CSS
import { loadStyle } from 'lightning/platformResourceLoader';
renderedCallback() {
    loadStyle(this, '/resource/CustomAppStyles/css/main.css');
}

// Toast Notifications
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
this.dispatchEvent(new ShowToastEvent({
    title: 'Notificación',
    message: 'Operación realizada correctamente',
    variant: 'success'  // success, error, warning, info
}));

// Utilidades personalizadas LWC a LWC
import { validateIBAN } from "c/paymentUtils";
```

---

## Patrones Comunes

### Inicialización Estándar

```javascript
connectedCallback() {
    this._actionUtilClass = new OmniscriptActionCommonUtil();
    this._ns = getNamespaceDotNotation();
    this.localData = JSON.parse(JSON.stringify(this.omniJsonData));
    
    this.initializeComponent();
}
```

### Manejo del DOM y Eventos de Teclado

```javascript
// Búsqueda por data-attributes
let element = this.template.querySelector('lightning-input[data-id="fieldId"]');

// blur en evento 'Enter'
handleKeyDown(event) {
    if (event.key === 'Enter') {
        event.target.blur();
    }
}
```

---

## FlexCards

```javascript
import { FlexCardMixin } from "vlocity_cmt/flexCardMixin";
import { OmniscriptBaseMixin } from "vlocity_cmt/omniscriptBaseMixin";
import data from "./definition";
import styleDef from "./styleDefinition";

export default class CustomFlexCardComponent extends FlexCardMixin(OmniscriptBaseMixin(LightningElement)) {
    @api recordId;
    @api objectApiName;
    @track _omniSupportKey = 'cfCustomFlexCardComponent';

    connectedCallback() {
        super.connectedCallback();
        this.setThemeClass(data);
        this.setStyleDefinition(styleDef);
        data.Session = {};
        this.setDefinition(data);
        this.registerEvents();
    }

    disconnectedCallback() {
        super.disconnectedCallback();
        this.omniSaveState(this.records, this.omniSupportKey, true);
        this.unregisterEvents();
    }
}
```

---

## Tips y Buenas Prácticas

1. **Evitar mutación de estado:** Desvincular referencias de `omniJsonData` realizando una copia profunda antes de manipular los datos (`JSON.parse(JSON.stringify(...))`).
2. **Utilizar Getters:** Facilitan la lectura y mantienen limpia la lógica de renderizado condicional en la plantilla HTML.
3. **Validación de Arrays:** Garantizar la comprobación previa mediante `Array.isArray()` antes de ejecutar bucles o iteraciones.
4. **Desacoplamiento:** Los componentes que no requieran interacción con el árbol de OmniScript pueden prescindir del `OmniscriptBaseMixin` y comunicarse únicamente mediante llamadas a IPs vía `OmniscriptActionCommonUtil`.

---

## Resumen de Integraciones y Métodos

| Necesidad | Enfoque Recomendado | Módulo / Import |
| :--- | :--- | :--- |
| **Integration Procedure** | `_actionUtil.executeAction(params, ...)` | `OmniscriptActionCommonUtil` + `getNamespaceDotNotation` |
| **DataRaptor / DataMapper** | `getDataHandler(JSON.stringify(req))` | `getDataHandler` (`vlocity_cmt/utility`) |
| **Apex Class Directo** | `this.omniRemoteCall(params, true)` | `OmniscriptBaseMixin` |
| **Actualizar JSON OmniScript** | `this.omniApplyCallResp({...})` | `OmniscriptBaseMixin` |
| **Avanzar / Retroceder Step** | `this.omniNextStep()` / `this.omniPrevStep()` | `OmniscriptBaseMixin` |
| **Navegación Org/App** | `this[NavigationMixin.Navigate](...)` | `NavigationMixin` |
| **Validar LWC en Step** | `this.omniValidate(true)` + `@api checkValidity()` | `OmniscriptBaseMixin` |