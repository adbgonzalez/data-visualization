# Práctica: `superstore-dataset`

## 1. Obxectivo

Nesta práctica imos construír un proxecto completo en Power BI a partir dun dataset realista de vendas tipo `Superstore`.

O traballo debe cubrir o fluxo completo visto ao longo da unidade:

1. importación de datos
2. limpeza e transformación básica en Power Query
3. modelado en estrela
4. creación de medidas en DAX
5. deseño dun informe con visualizacións útiles
6. definición e validación de roles RLS

---

## 2. Dataset de partida

Empregaremos o dataset `superstore-dataset`, gardado no cartafol:

`power-bi/data/superstore-dataset/`

Neste cartafol deberán estar os ficheiros ou táboas necesarios para construír o modelo. Para esta práctica asumiremos que o dataset inclúe polo menos:

- `Orders`
- `People`

Nesta práctica, a táboa `People` non se empregará no modelo final, porque só achega unha relación simple entre persoa e rexión e non é necesaria para cumprir os obxectivos do traballo.

O traballo centrarase en construír o modelo a partir de `Orders`.

Neste contexto, o importante é ser capaz de:

- identificar a táboa de feitos principal
- separar dimensións útiles
- renomear consultas con criterio
- crear un modelo limpo e coherente

---

## 3. Resultado esperado

Ao rematar a práctica deberías ter:

- un ficheiro `.pbix` funcional
- consultas limpas e renomeadas de forma clara
- un modelo en estrela con relacións correctas
- un conxunto mínimo de medidas DAX útiles para análise
- unha páxina de informe principal ben organizada
- polo menos un rol RLS definido e probado

---

## 4. Preparación inicial

Antes de empezar:

1. crea un ficheiro novo en Power BI Desktop
2. importa os datos desde `power-bi/data/superstore-dataset/`
3. revisa que campos e táboas trae o dataset
4. identifica que parte do dataset representa transaccións, pedidos ou vendas
5. comproba que `Orders` contén, como mínimo, campos de cliente, produto, xeografía, datas e métricas de venda

---

## 5. Limpeza e transformación

Realiza polo menos estas tarefas en Power Query:

1. renomear `Orders` como base de traballo para construír as consultas finais
2. revisar tipos de datos en importes, cantidades, descontos e datas
3. eliminar columnas irrelevantes se non achegan valor analítico
4. corrixir valores nulos, baleiros ou inconsistentes cando sexa necesario
5. comprobar que os nomes das categorías e segmentos son coherentes
6. revisar que as columnas que actuarán como claves teñen un comportamento estable
7. nas dimensións creadas desde `Orders`, eliminar duplicados para que quede unha soa fila por clave

Transformacións mínimas esperadas:

- renomeado de consultas e columnas principais
- revisión e corrección de tipos
- eliminación dalgunha columna innecesaria
- tratamento básico dalgún problema de calidade do dato

Se a consulta `People` está cargada, podes desactivar a súa carga ao modelo final.

Para eliminar duplicados:

1. entra en `Inicio -> Transformar datos` para abrir Power Query
2. selecciona a consulta da dimensión correspondente
3. marca só as columnas que definen de forma única cada fila desa dimensión
4. aplica `Inicio -> Quitar filas -> Quitar duplicados`

Tamén podes facer clic dereito sobre as columnas seleccionadas e escoller `Quitar duplicados`.

---

## 6. Modelo en estrela

Constrúe un modelo en estrela a partir de `Orders`, creando estas táboas:

- `FactSales`
- `DimDate`
- `DimProduct`
- `DimCustomer`
- `DimGeography`

### 6.1. Crear `FactSales`

Parte da consulta `Orders` e deixa nela o detalle transaccional.

Campos recomendados:

- `Row ID`
- `Order ID`
- `Order Date`
- `Ship Date`
- `Ship Mode`
- `Customer ID`
- `Product ID`
- `Region`
- `Sales`
- `Quantity`
- `Discount`
- `Profit`

### 6.2. Crear `DimCustomer`

Crea unha consulta de referencia a partir de `Orders`, conserva só estes campos e elimina duplicados:

- `Customer ID`
- `Customer Name`
- `Segment`

En Power Query:

1. crea a referencia desde `Orders`
2. conserva só esas tres columnas
3. selecciona as tres columnas
4. aplica `Inicio -> Quitar filas -> Quitar duplicados`

### 6.3. Crear `DimProduct`

Crea outra consulta de referencia a partir de `Orders`, conserva só estes campos e elimina duplicados:

- `Product ID`
- `Category`
- `Sub-Category`
- `Product Name`

En Power Query:

1. crea a referencia desde `Orders`
2. conserva só esas catro columnas
3. selecciona esas catro columnas
4. aplica `Inicio -> Quitar filas -> Quitar duplicados`

### 6.4. Crear `DimGeography`

Crea outra consulta de referencia a partir de `Orders`, conserva só estes campos e elimina duplicados:

- `Region`

En Power Query:

1. crea a referencia desde `Orders`
2. conserva só esas columnas
3. selecciona `Region`
4. aplica `Inicio -> Quitar filas -> Quitar duplicados`

Nesta práctica, `Region` será o campo recomendado para segmentación territorial e para RLS.

### 6.5. Crear `DimDate`

`DimDate` crearase como táboa nova en DAX, non en Power Query.

En `Modelado -> Nueva tabla`, crea:

```DAX
DimDate =
ADDCOLUMNS(
    CALENDAR(
        MIN(FactSales[Order Date]),
        MAX(FactSales[Order Date])
    ),
    "Year", YEAR([Date]),
    "Quarter", "Q" & FORMAT([Date], "Q"),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMMM"),
    "MonthKey", YEAR([Date]) * 100 + MONTH([Date])
)
```

Despois:

1. marca `DimDate` como táboa de datas usando a columna `Date`
2. ordena `Month` por `Month Number` ou por `MonthKey`

### 6.6. Relacións do modelo

Configura estas relacións:

- `FactSales[Customer ID] -> DimCustomer[Customer ID]`
- `FactSales[Product ID] -> DimProduct[Product ID]`
- `FactSales[Region] -> DimGeography[Region]`
- `FactSales[Order Date] -> DimDate[Date]`

Opcionalmente, podes crear tamén esta relación adicional:

- `FactSales[Ship Date] -> DimDate[Date]`

Se a creas, debe quedar inactiva, para evitar ambigüidade temporal no modelo.

Requisitos do modelo:

1. as consultas deben quedar renomeadas con criterio (`Fact...`, `Dim...`)
2. as relacións principais deben estar en cardinalidade `*:1`
3. a dirección de filtro debe ser a habitual nun modelo en estrela (`Dim` -> `Fact`)
4. `Order Date` debe ser a data principal de análise temporal
5. nesta práctica, `DimGeography` simplifícase a nivel de `Region` para evitar problemas de duplicados e facilitar o RLS

---

## 7. Medidas DAX

Crea como mínimo estas medidas:

```DAX
Total Sales = SUM(FactSales[Sales])
Total Profit = SUM(FactSales[Profit])
Total Quantity = SUM(FactSales[Quantity])
Total Orders = DISTINCTCOUNT(FactSales[Order ID])
Profit Margin % = DIVIDE([Total Profit], [Total Sales], 0)
Average Order Value = DIVIDE([Total Sales], [Total Orders], 0)
```

Ademais, crea polo menos dúas medidas adicionais escollidas entre estas opcións:

- `Sales YTD`
- `Sales PY`
- `YoY %`
- vendas dunha categoría concreta con `CALCULATE`
- participación dunha categoría no total
- beneficio medio por pedido

Requisitos:

1. aplica formato monetario cando corresponda
2. aplica formato porcentaxe en medidas relativas
3. valida as medidas nunha táboa ou matriz antes de pasar aos visuais finais

---

## 8. Visualizacións

Constrúe unha páxina principal de informe coas seguintes pezas mínimas:

### 8.1. KPIs

Inclúe polo menos 4 indicadores:

- `Total Sales`
- `Total Profit`
- `Profit Margin %`
- `Total Orders` ou `Average Order Value`

### 8.2. Visuais obrigatorios

Inclúe polo menos estes visuais:

1. gráfico de columnas ou barras por categoría de produto
2. gráfico temporal por mes ou ano
3. matriz ou táboa de detalle
4. segmentador para filtrar por categoría, segmento ou rexión
5. un visual adicional escollido entre `Treemap`, `Scatter`, `Ribbon`, `Gauge`, `Mapa`, `Embudo` ou `Barras apiladas`

### 8.3. Requisitos de deseño

1. todos os visuais deben ter título claro
2. o informe debe ser lexible e manter certa coherencia visual
3. as interaccións entre visuais deben funcionar correctamente
4. a páxina non debe estar saturada de elementos innecesarios

---

## 9. RLS

Define e proba polo menos un rol de seguridade RLS.

Nesta práctica, o RLS farase preferentemente sobre `DimGeography[Region]`.

Exemplos posibles:

- un rol `West` co filtro `[Region] = "West"`
- un rol `East` co filtro `[Region] = "East"`
- dous ou máis roles para comparar comportamentos por rexión

Requisitos:

1. crea o rol en `Modelado -> Administrar roles`
2. define o filtro DAX sobre `DimGeography`
3. valida o comportamento con `Ver como`
4. comproba que os KPIs e visuais cambian segundo o rol

---

## 10. Entrega

A entrega da práctica debe incluír:

- o ficheiro `.pbix`
- un PDF co informe final
- un pequeno documento ou PDF con capturas que demostren:
  - as transformacións principais en Power Query
  - o modelo en estrela
  - as medidas DAX creadas
  - a proba do rol RLS con `Ver como`

---

## 11. Rúbrica de corrección

A práctica avaliarase sobre 10 puntos.

### 11.1. Importación, limpeza e transformación

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 2 puntos | Os datos están ben importados, as consultas están renomeadas e hai transformacións claras e útiles de limpeza e preparación. |
| Parcial | 1 punto | A importación e a limpeza están parcialmente resoltas, pero hai carencias de nomes, tipos ou transformacións. |
| Insuficiente | 0 puntos | A carga dos datos é incorrecta ou apenas hai preparación das consultas. |

### 11.2. Modelo en estrela

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 2 puntos | O modelo é claro, coherente, con táboa de feitos e dimensións ben separadas, relacións correctas e nomes adecuados. |
| Parcial | 1 punto | O modelo funciona, pero presenta problemas de estrutura, nomes pouco claros ou relacións mellorables. |
| Insuficiente | 0 puntos | O modelo non segue unha lóxica analítica clara ou ten relacións incorrectas. |

### 11.3. Medidas DAX

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 2 puntos | As medidas obrigatorias están creadas, funcionan correctamente e hai medidas adicionais ben resoltas. |
| Parcial | 1 punto | Hai medidas básicas, pero faltan varias, teñen erros ou non están ben formatadas. |
| Insuficiente | 0 puntos | Apenas hai medidas ou non son válidas para a análise. |

### 11.4. Visualizacións e deseño do informe

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 2 puntos | O informe é claro, útil e contén os visuais obrigatorios ben configurados e ben organizados. |
| Parcial | 1 punto | O informe funciona, pero faltan visuais, hai problemas de claridade ou o deseño é mellorable. |
| Insuficiente | 0 puntos | O informe está incompleto, é pouco lexible ou non permite unha análise correcta. |

### 11.5. RLS

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 2 puntos | O rol está ben definido, probado con `Ver como` e afecta correctamente aos visuais e KPIs. |
| Parcial | 1 punto | Hai un intento válido de RLS, pero con problemas de definición, proba ou impacto real. |
| Insuficiente | 0 puntos | Non hai RLS ou non funciona. |

---

## 12. Ideas clave

Nesta práctica o importante non é só construír un informe bonito, senón demostrar que sabes percorrer o fluxo completo dun proxecto de análise en Power BI:

- entender e preparar os datos
- estruturar un modelo analítico
- crear medidas útiles
- escoller visuais axeitados
- e aplicar seguridade básica cando o escenario o require

Un traballo correcto debe ser tecnicamente consistente e tamén fácil de interpretar por outra persoa.
