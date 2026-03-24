# BankRecon — Autonomous Bank Statement Parser

Parsea estados de cuenta bancarios en PDF y genera archivos `.xlsx` de conciliación bancaria. **Sin API. Sin backend. 100% en el navegador.**

```
PDF → PDF.js (extrae texto) → Parser determinístico (lee la Bank Key) → SheetJS → .xlsx
```

La inteligencia vive en los archivos `keys/*.json`. La app **nunca cambia**. Solo se agregan keys.

---

## Setup en GitHub Pages

```bash
git init bank-reconciler
cd bank-reconciler
# Copiar todos los archivos
git add .
git commit -m "init"
git remote add origin https://github.com/TU_USUARIO/bank-reconciler.git
git push -u origin main
```

En GitHub → Settings → Pages → Branch: `main` / folder: `/ (root)`

URL: `https://TU_USUARIO.github.io/bank-reconciler/`

---

## Cómo funciona el sistema de keys

### Una key define exactamente cómo leer un estado de cuenta:

```json
{
  "key_id": "chase_business_complete_checking_v1",
  "bank": "Chase",
  "detect_by": ["Chase Business Complete Checking", "CHECKING SUMMARY"],
  "header": {
    "period_regex": "(\\w+ \\d+, \\d{4}) through (\\w+ \\d+, \\d{4})",
    "beginning_balance_regex": "Beginning Balance[\\s\\$]+([\\d,]+\\.\\d{2})",
    "ending_balance_regex": "Ending Balance[\\s\\d]+\\$([\\d,]+\\.\\d{2})"
  },
  "sections": [
    {
      "name": "DEPOSITS AND ADDITIONS",
      "start_marker": "DEPOSITS AND ADDITIONS",
      "end_markers": ["ELECTRONIC WITHDRAWALS", "FEES"],
      "type": "Deposit",
      "row_regex": "(\\d{2}/\\d{2})\\s+(.+?)\\s+\\$?([\\d,]+\\.\\d{2})\\s*$",
      "columns": ["date","description","amount"],
      "normalize": [
        ["Zelle Payment From ([\\w ]+) \\w+$", "Zelle from $1"]
      ]
    }
  ]
}
```

### El parser:
1. Extrae el texto del PDF con PDF.js
2. Usa los regex del `header` para encontrar período, cuenta y saldos
3. Para cada `section`, usa `start_marker` y `end_markers` para aislar el bloque de texto
4. Aplica `row_regex` para extraer cada transacción
5. Aplica `normalize` para limpiar descripciones
6. Calcula el saldo corrido y valida contra el saldo final del estado

---

## Sistema de alertas para secciones no mapeadas

Si el parser encuentra en el PDF una sección que **ninguna key cargada** cubre, automáticamente:

1. **Muestra una alerta** con el nombre exacto de la sección no mapeada
2. **No procesa** esas transacciones (las omite)
3. **Ofrece copiar un prompt** para pedirle a Claude la key complementaria

Esto aplica a **cualquier banco** — no solo Bank of America. Si Chase un día agrega una sección "INTERNATIONAL TRANSFERS" que no está en la key, la alerta se dispara.

---

## Agregar una key complementaria

### Cuando la app lanza una alerta:

1. Click en **"Copy prompt to request complement key"**
2. Abre una nueva conversación con Claude
3. Pega el prompt + adjunta el PDF del estado
4. Claude genera el JSON de la key complementaria
5. Guarda el archivo en `keys/`
6. Agrega la entrada en `keys/index.json`
7. `git add . && git commit -m "add complement key" && git push`

### En la app:
- Selecciona la key base normalmente
- Click en **"+ Load complement key (.json)"** en el sidebar
- Selecciona el archivo de la key complementaria
- El parser re-corre automáticamente combinando ambas keys

O si ya subiste la key al repo: simplemente aparece como una key separada en el listado.

---

## Agregar un banco nuevo

1. Sube el PDF del estado a Claude
2. Pide: *"Genera una bank key JSON para este estado de cuenta siguiendo el formato de las keys en mi repositorio"*  
3. Claude genera el JSON
4. Guarda en `keys/nuevo_banco.json`
5. Agrega en `keys/index.json`:

```json
{
  "key_id": "nuevo_banco_v1",
  "file": "nuevo_banco.json",
  "bank": "Nombre del Banco",
  "product": "Nombre del Producto",
  "status": "active"
}
```

6. Push al repo

---

## Keys incluidas

| Banco | Producto | Key ID |
|-------|---------|--------|
| PNC Bank | Business Checking Plus | `pnc_business_checking_plus_v1` |
| Chase | Business Complete Checking | `chase_business_complete_checking_v1` |
| Regions Bank | LifeGreen Business Checking | `regions_lifegreen_business_checking_v1` |
| Wells Fargo | Optimize Business Checking | `wells_fargo_optimize_business_checking_v1` |
| Bank of America | Business Advantage Fundamentals | `bofa_business_advantage_fundamentals_v1` |
| Mercury | Mercury Checking | `mercury_checking_v1` |
| Truist | Business Value 200 Checking | `truist_business_value_200_checking_v1` |
| Pacific National Bank | Business Regular Checking | `pnb_business_regular_checking_v1` |

---

## Formato del .xlsx

**Hoja:** `[BANCO] [LAST4] - [AÑO]`  
**Columnas:** Date · Reference · Description · Amount · Total Amount · Type · GL Number · GL Account name · Saldo  
**Fuente:** Arial 10pt · Encabezado fondo gris negrita · Primera fila = Beginning Balance  
**Amount:** negativo para pagos, positivo para depósitos  
**Total Amount:** siempre positivo

---

## Notas técnicas

- Cero dependencias de backend o API externa
- PDF.js reconstruye líneas usando posición Y de cada elemento (más preciso que concatenar texto plano)
- Las keys usan regex estándar JavaScript — fáciles de probar en regex101.com
- `amount_sign` puede ser: `positive`, `already_negative`, `suffix_minus`, `from_prefix`
- Para cheques multi-columna: `"layout": "multi_column_checks"` con `check_row_regex`
- El merge de keys es aditivo: la key complementaria agrega secciones a la base
