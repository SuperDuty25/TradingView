# SMC Pro: estructura, liquidez, Order Blocks e ineficiencias

Este paquete tiene dos scripts de Pine Script v6 que comparten **el mismo motor y las mismas reglas**:

| Archivo | Uso |
|---|---|
| `SMC_Pro_Indicador.pine` | Muestra en el gráfico la estructura, la liquidez, los OB, los FVG, las señales, las alertas y un panel. |
| `SMC_Pro_Estrategia.pine` | Ejecuta el backtest con capital de 200 $, tamaño por riesgo, apalancamiento máximo x10 y cierre a las 48 h. |

Configuración de partida: **gráfico de 15m con sesgo y OB en 4H**, para BTC, acciones y forex.

---

## 1. Definiciones exactas (lo que el código hace de verdad)

### Estructura
- **Swing alto o bajo:** se usa `ta.pivothigh` o `ta.pivotlow` con longitud *N*. Un swing es la vela más alta (o más baja) entre las *N* velas anteriores y las *N* siguientes, así que **se confirma *N* velas después**.
- **BOS:** una vela **cierra** por encima del último swing alto (o por debajo del último swing bajo) a favor de la tendencia actual.
- **CHoCH:** es la misma ruptura, pero **en contra** de la tendencia anterior. Es la primera señal de cambio.
- Las mechas no rompen estructura. Solo cuentan los cierres.

### Liquidez
- Cada swing confirmado se guarda como un **pool de liquidez**: BSL por encima de los máximos y SSL por debajo de los mínimos. Se guardan como máximo 40 por lado.
- Un pool queda **tomado** cuando el precio lo supera con la mecha o con el cuerpo. En ese momento se elimina de la lista.
- **Barrido (✕ BSL / ✕ SSL):** la mecha toma el nivel pero la vela **cierra de vuelta** al otro lado.
- **EQH y EQL:** un swing nuevo queda a ≤ *tolerancia × ATR(14)* de un pool que sigue sin tomar.

### Ineficiencias (FVG)
- **FVG alcista:** `low > high[2]`. **FVG bajista:** `high < low[2]`.
- Hay un filtro de tamaño mínimo de 0,25 × ATR(14) por defecto, para descartar huecos irrelevantes.
- La línea punteada central es el **CE (Consequent Encroachment)**, es decir, el 50% del hueco.
- El hueco se recorta a medida que se rellena. **Solo se invalida cuando una vela cierra al otro lado.**

### Order Block (con tus reglas)
Un OB se crea **únicamente** cuando hay una ruptura de estructura (BOS o CHoCH) y además:
1. **La vela del OB** es la última vela opuesta del tramo, en concreto la de mínimo más bajo (en el caso alcista), entre el swing roto y la ruptura. Su rango es la **vela completa con mechas**.
2. **Hay FVG en el desplazamiento:** existe al menos un FVG a favor entre la vela del OB y la ruptura.
3. **Hubo toma de liquidez previa:** dentro del mismo tramo (después del swing que se rompe) el precio tomó un pool del lado contrario.
4. **Mitigación:** el OB se invalida **solo cuando una vela cierra al otro lado** (por debajo del mínimo en un OB alcista).

Puedes desactivar las condiciones 2 y 3 desde los inputs para comparar resultados.

---

## 2. Modelo de entrada (tu modelo, convertido en reglas)

**Largo** (el corto es exactamente al revés):

1. **Sesgo 4H alcista:** el último quiebre en 4H fue alcista. Se puede desactivar.
2. Existe un **OB alcista 4H válido**, que es el más reciente.
3. En 15m, el precio **toca el 50% del OB** (`low <= 50%`) sin cerrar por debajo de su base. El setup queda **ARMADO**.
4. Antes de 48 velas (12 h) aparece un **CHoCH alcista en 15m**. Ahí está la **SEÑAL**, con entrada a mercado al cierre de esa vela. En el backtest la orden se llena en la apertura de la vela siguiente.
5. **SL:** la base del OB 4H menos 0,2 × ATR(14) de 15m.
6. **TP:** se elige según el modo.
   - *3R fijo:* entrada + 3 × riesgo.
   - *Siguiente liquidez:* el pool BSL sin tomar más cercano que dé al menos el R:B mínimo (2R). Si no hay ninguno, usa 3R.
   - *Más conservador* (por defecto): el **más cercano** entre 3R y esa liquidez. Es la traducción literal de *"buscar 3R o la siguiente liquidez, lo más seguro"*.
7. **Filtros:**
   - Se descarta la operación si el R:B no llega al mínimo.
   - Se descarta si el stop queda más allá del 80% de la distancia aproximada de liquidación (100 % / apalancamiento).
8. **Un solo trade por OB.** Después de operar un OB, no se vuelve a usar.
9. **Cierre forzado** a las 48 h si no se tocó ni el SL ni el TP.

---

## 3. Longitud de pivote (no la sabías, aquí va la explicación)

Es el número de velas a cada lado que necesita un swing para considerarse confirmado.

| Valor | En 15m | En 4H | Efecto |
|---|---|---|---|
| 3 | confirma en 45 min | confirma en 12 h | Más estructura, más señales y más ruido. |
| **5 (defecto)** | 75 min | 20 h | Equilibrio habitual como punto de partida. |
| 8–10 | 2–2,5 h | 32–40 h | Solo estructura mayor y menos señales. Con trades de 2 días como máximo puede quedarse corto. |

No hay un valor "correcto" universal. **Pruébalo en el backtest** (3, 5 y 8) por cada instrumento y quédate con el que mejor funcione **fuera de muestra**.

---

## 4. Gestión de riesgo y apalancamiento

- **Capital:** 200 $. **Riesgo por operación:** 1% del capital por defecto (2 $).
- **Tamaño:** `cantidad = (capital × riesgo%) / (entrada − SL)`, con un límite de `capital × apalancamiento / precio`.
- **El apalancamiento no cambia cuánto arriesgas** si el stop se respeta. Solo reduce el margen que bloqueas. Con x10, un movimiento de ~10% en contra liquida la posición. La cifra exacta depende del exchange y del margen de mantenimiento, por eso el filtro es **aproximado**.
- **Comisión por defecto:** 0,05% por lado y 2 ticks de slippage. **Ajusta las dos a tu broker o exchange**, porque en forex y acciones cambian mucho.
- **Acciones:** activa *"Cantidad en unidades enteras"*. Con 200 $ muchas acciones caras quedarán en 0 unidades y la operación no se ejecuta.

---

## 5. Cómo usarlo

1. En TradingView abre el **Editor de Pine**, pega `SMC_Pro_Indicador.pine`, guárdalo y añádelo al gráfico. Haz lo mismo con `SMC_Pro_Estrategia.pine`.
2. Pon el gráfico en **15m**. El panel avisa si la temporalidad del gráfico no es menor que la superior.
3. **Alertas:** crea una alerta con la condición *"SMC Pro → Cualquier llamada de alert()"*. Así recibes el mensaje con entrada, SL y TP. También están las alertas individuales de CHoCH, toque del 50% y barridos.
4. **Backtest:** abre el *Probador de estrategias*. Revisa el número de operaciones, el factor de beneficio, el drawdown máximo y la duración media.

### Cómo validar en serio
- Separa **en muestra** (por ejemplo 2024) y **fuera de muestra** (por ejemplo 2025). Optimiza solo con el primer periodo.
- Necesitas **muchas operaciones** (idealmente >100) antes de sacar conclusiones. Con pocas operaciones, el resultado es ruido.
- Prueba cada mercado por separado. BTC, forex y acciones se comportan distinto, y las acciones tienen huecos de apertura que pueden saltarse el stop.

---

## 6. Limitaciones conocidas (honestas)

- **SMC/ICT no tiene evidencia académica de ventaja estadística.** Este script define las reglas de forma objetiva para que **tú lo midas**. No garantiza resultados.
- De la temporalidad superior se sigue **solo el OB válido más reciente** por dirección. Los OB 4H anteriores sin mitigar se descartan.
- Los swings se confirman con retraso (*N* velas). Es inevitable si no se quiere repintar.
- Los datos 4H se leen de la **vela 4H ya cerrada** (`[1]` + `lookahead_on`), así que **no repinta**. A cambio, el OB 4H aparece en 15m cuando cierra la vela 4H que lo confirma.
- La entrada del backtest se llena en la apertura de la vela siguiente a la señal, por lo que el R:B real puede diferir un poco del calculado.
- La cantidad de historial en 15m depende de tu plan de TradingView.
- El código **no se ha compilado fuera de TradingView**. Si el editor marca algún error, copia el mensaje exacto y lo corrijo.

## 7. Próximas mejoras opcionales
Premium/discount y OTE, IFVG y Breaker Blocks, killzones por sesión, mover el stop a break-even en 1R, varios OB 4H activos a la vez, y FVG de 4H como POI alternativo.
