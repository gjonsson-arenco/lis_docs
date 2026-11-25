# CEBAC · Bitácora de configuración del LIS

Registro de lo que se revisó y se cambió en la configuración de coberturas, convenios,
sets de prácticas y mapeo de estudios del LIS de CEBAC (instancia on-premise,
`qa.cebac.ankra-lis.com`). Cada entrada dice qué se hizo, qué cambia en la admisión y qué
queda pendiente de una decisión de CEBAC.

**Cómo se mide "cotiza".** Se simula la cotización de los 1.411 estudios activos con la
misma lógica que usa la admisión: convenio vigente, nomenclador y set de prácticas. Un
estudio "cotiza" para una cobertura cuando todas sus prácticas tienen precio en el convenio.
El porcentaje se calcula sobre los 1.283 estudios que cotiza el particular.

---

## 25/11/2026 · Auditoría de coberturas

Sólo lectura: no se modificó nada.

- 373 coberturas activas. 167 cotizan bien (60% o más de los estudios), 99 cotizan poco
  y 91 no cotizan nada o casi nada. Además hay 14 coberturas internas (mantenimiento y
  control de calidad) y 2 de prueba.
- 27 coberturas no cotizan nada porque su convenio usa un nomenclador cargado sobre un set
  de prácticas que la cobertura no tiene asignado. Ejemplos: OSPSIP, OSSEG, UTA, Luz y
  Fuerza, IOSFA, Omint y Binaria Seguros.
- El set NBU 2016 (S_003) tiene sólo 120 estudios mapeados. Por eso Galeno, SMAUNaM,
  OSDEPYM y otras 5 coberturas cotizan el 9%.
- 55 coberturas activas no tienen ningún convenio.
- Sólo OSDE tiene configurado un provider (ITC), y apunta al ambiente de prueba de ITC.

Detalle completo: informe "Coberturas CEBAC QA".

---

## 25/11/2026 · OSDE y Binaria Seguros: set S_002-OSDE

**Coberturas afectadas.** OSDE (convenio C_071) y Binaria Seguros (convenio C_033). Las
dos usan el nomenclador N_1007-OSDE.

**Resultado**

| | Antes | Después |
|---|---|---|
| Estudios con código OSDE | 755 | 811 |
| Estudios que cotiza OSDE | 755 (57,3%) | 810 (61,0%) |
| Estudios que cotiza Binaria Seguros | 1 (0,1%) | 810 (61,0%) |

Registro interno de los cambios: `operation_logs` #187, #188 y #189.

### 1. Estudios mapeados por código o variante (27)

Reglas aplicadas:

- el estudio tiene el mismo código que la práctica OSDE;
- el estudio es una variante con letra de un código (660063C, 663649A…) y se factura con
  la práctica base (660063, 663649).

| Estudio | Nombre | Práctica OSDE |
|---|---|---|
| 660063C | HIV AG-AB COMBO SCREENING REACTIVO | 660063 HIV ANTICUERPOS (METODO DE ELISA) |
| 660063R | RATIFICACION - HIV AG-AB COMBO - USO BIOQUIMICO | 660063 HIV ANTICUERPOS (METODO DE ELISA) |
| 660356B | FOSFATASA ACIDA EN LIQUIDO SEMINAL | 660356 FOSFATASA ACIDA TOTAL |
| 660409A | RECUENTO GLOBULOS BLANCOS (CD) | 660409 GLOBULOS BLANCOS RECUENTO DE |
| 660540A | INMUNOGLOBULINA G EN LCR | 660540 INMUNOGLOBULINA G -IGG- CADENA ESPECIFICA GAMMA |
| 660862A | TALIO EN SANGRE | 660862 TALIO EN ORINA (sin precio en el convenio, ver pendientes) |
| 661086R | RATIFICACION - HEPATITIS B, Ag de Sup. (HBSAG) | 661086 HEPATITIS B,HBSAG |
| 661095R | RATIFICACION - HEPATITIS C, AC. IGG ANTI- (HCV AC IGG) | 661095 HEPATITIS C, AC O AC IGG |
| 662273A | ACIDO METIL MALONICO - SERICO | 662273 ACIDO METIL MALONICO AMM (ORINA DE 24HS) |
| 662410A | AQUAPORINA 4 AC - (NMO) - LCR | 662410 ACUAPORINA 4 AC - (NMO) - SUERO |
| 662418A | ADENOSIN DEAMINASA EN LIQUIDO ASCITICO | 662418 ADENOSIN DEAMINASA - liquido pleural |
| 662444B | ADENOVIRUS Ag. (HISOPADO NASOFARINGEO) | 662444 ADENOVIRUS, AG |
| 662627 | AMINOACIDOS URINARIOS (Cromatografía cuantitativa) | 662627 AMINOACIDOS - urinarios |
| 663307 | CALCULO URINARIO - ESTUDIO CRISTALOGRAFICO | 663307 CALCULO URINARIO, ESTUDIO CRISTALOGRAFICO |
| 663649A a 663649E | CHLAMYDIA TRACHOMATIS - PCR (esperma, materiales varios, conjuntiva, orina, endocervical) | 663649 CHLAMYDIA TRACHOMATIS, GENOMA (PCR) |
| 666332A | HLA B27 ANTIGENO | 666332 HLA B27 |
| 667503A | MICROGLOBULINA BETA 2 EN ORINA | 667503 MICROGLOBULINA BETA 2 |
| 668281A | PEPTIDO C URINARIO | 668281 PEPTIDO C |
| 668463A | POLYOMAVIRUS BK (CARGA VIRAL) - ORINA | 668463 POLIOMA VIRUS, BK - CARGA VIRAL |
| 669127A | STREPTOCOCO B PERIANAL | 669127 STREPTOCOCCUS BETA-HEMOLITICO GRUPO B - PRENATAL |
| 669443B | TIROGLOBULINA EN LIQUIDO DE PUNCION | 669443 TIROGLOBULINA (TGS) |
| 669633A, 669633B | T (9-22) BCR-ABL P210 / P230 - CUALITATIVO | 669633 REARREGLO (9,22) BCR/ABL LMC |

### 2. Equivalencias (25 estudios)

Estudios que no tienen el mismo código que una práctica OSDE pero corresponden a una. Varias
son códigos propios de OSDE que no usaba ningún estudio. Otras son equivalencias que CEBAC
ya usa con otros financiadores.

| Estudio | Práctica OSDE | Por qué |
|---|---|---|
| 665116 FACTOR VON WILLEBRAND, FUNCIONAL (COFACTOR DE RISTOCETINA) | 236108 COFACTOR DE RISTOCETINA | misma determinación; código propio OSDE |
| 669905 1,25-DIHIDROXICALCIFEROL - CALCITRIOL | 6001 VITAMINA D,1-25 DI(OH) | misma determinación; código propio OSDE |
| 666401 HLA DQ MOLECULAR DQ2 DQ8 - CELIAQUIA | 6244 CELIACA, ALELOS HLA-DQ 2 Y DQ 8 (PCR) | misma determinación; código propio OSDE |
| 668011, 668011B, 668011C, 668011E HPV GENOTIPIFICACION (distintas muestras) | 6531 HPV DETECCION DE ADN VIRAL | código propio OSDE |
| 669102 JAK 2 MUTACION V617F | 6821 MUTACION JAK 2 -EXON 14-V617 | código propio OSDE |
| 664363 DENGUE IgM (ELISA) | 6830 DENGUE VIRUS, AC. ANTI IGM | código propio OSDE |
| 662749 PANEL DE VIRUS RESPIRATORIOS | 6865 MODULO DE PANEL VIRAL RESPIRATORIO | código propio OSDE |
| 800156 / 800198 PANEL SINDROME CANCER HEREDITARIO (básico / ampliado) | 7191 PANEL DE CANCER HEREDITARIO | código propio OSDE |
| 664749 ESPERMOGRAMA (MAR TEST D.) | 664845 MAR TEST DIRECTO | misma determinación |
| 667785 TSH, NEONATAL SCREENING | 661196 MOD. SCREENING NEONATAL TSH/PKU | misma determinación |
| 668493 PORFIRINAS TOTALES - ERITROCITARIAS | 660751 PORFIRINAS | |
| 669773 UROPORFIRINAS - URINARIAS | 660751 PORFIRINAS | así está en otros 4 financiadores |
| 668682 PROTOPORFIRINA ERITROCITARIA | 660768 PROTOPORFIRINAS | así está en Swiss Medical |
| 666163 HIDATIDOSIS, AC. IGG | 661100 ARCO 5 PARA HIDATIDOSIS | así está en otros 4 financiadores |
| 666173 HIDATIDOSIS, AC. IGM | 666161 HIDATIDOSIS, ECHINOCOCCUS AC (ELISA) | |
| 666290A HIV-RNA CUANTITATIVO EN PLASMA SEMINAL | 661105 HIV, CARGA VIRAL | así está en NBU 2024 |
| 667240A LISTERIA MONOCITOGENES AC IGG | 660622 LISTERIAS POR INMUNOFLUORESCENCIA | así está en Swiss Medical y PAMI |
| 800008 TREPONEMA PALLIDUM HA (TPHA) | 660371 FTA/ABS INMUNOFLUORESCENCIA | prueba treponémica; así está en ConSalud y NBU 2023 |
| 669054 SIFILIS TP - IGG | 660371 FTA/ABS INMUNOFLUORESCENCIA | prueba treponémica; así está en Swiss Medical y PAMI |
| 663362 CARBAMAZEPINA EPOXIDO | 661136 MONITOREO DE FARMACOS ANTICONVULSIVANTES | así está en otros 3 financiadores |
| 668977 SANGRIA, TIEMPO DE | 660169 COAGULACION Y SANGRIA TIEMPO DE | la práctica OSDE incluye sangría |

### 3. Otras altas y correcciones

| Estudio | Antes | Ahora | Por qué |
|---|---|---|---|
| 660911C UROCULTIVO + ORINA COMPLETA | sin práctica | 660911 UROCULTIVO | igual que en NBU 2023, NBU 2024, Swiss Medical y ConSalud |
| 667669A MICOBACTERIAS ATIPICAS - PCR | sin práctica | 667669 MYCOBACTERIUM TUBERCULOSIS, GENOMA (PCR) | igual que en NBU 2024 y ConSalud |
| 668956A SACCHAROMYCES CEREVISIAE AC IgA (ASCA) | sin práctica | 668956 SACCHAROMYCES CEREVISIAE AC IgG (ASCA) | igual que en NBU 2024 y ConSalud |
| 666711F EFNA - EXCRECION FRACCIONAL DE SODIO | sin práctica | 660192 CREATININA + 660546 IONOGRAMA PLASMATICO + 660547 IONOGRAMA URINARIO | mismo criterio que los otros índices de función renal en OSDE |
| 666711G EFK - EXCRECION FRACCIONAL DE POTASIO | sin práctica | 660192 CREATININA + 660546 IONOGRAMA PLASMATICO + 660547 IONOGRAMA URINARIO | ídem |
| 660620B RECUENTO CELULAR EN BOLSA DE DIALISIS PERITONEAL | 660409 GLOBULOS BLANCOS RECUENTO | 660620 MODULO DE LIQUIDO DE PUNCION | variante de 660620; igual que en NBU 2023, NBU 2024 y PAMI |
| 665136 FENCICLIDINA (drogas de abuso) | 661136 MONITOREO DE ANTICONVULSIVANTES | 661050 SCREENING PARA DROGAS DE ABUSO | era un mapeo erróneo |
| 666879 INTERLEUQUINA 6 - IL6 | 660546 IONOGRAMA PLASMATICO | sin práctica | era un mapeo erróneo, y OSDE no tiene práctica de IL-6. En la admisión aparece como "no cubierto, cobrar particular" |

### 4. Binaria Seguros

Binaria tenía convenio con el nomenclador de OSDE pero no tenía asignado el set OSDE, así
que no cotizaba ningún estudio. Se le asignó el set S_002-OSDE y ahora cotiza lo mismo que
OSDE.

### Pendiente de decisión de CEBAC

**A. Estudios sin código en el set OSDE (500 estudios).** El set OSDE trae 589 códigos y
el nomenclador de OSDE 588 precios. El resto de los estudios aparece en la admisión como
"no cubierto, cobrar particular": son estudios con código NBU que no están en el set
(unos 290), códigos propios de CEBAC y derivaciones CEMIC (800xxx, 900xxx) y estudios
veterinarios. **¿OSDE cubre sólo esos 589 códigos, o hay que cargar más prácticas al
convenio?**

**B. Mapeos que siguen como estaban y conviene confirmar.** El estudio factura con una
práctica de otra determinación o de otra muestra, porque OSDE no tiene la exacta:

- 669725 TROPONINA I DE ALTA SENSIBILIDAD → 669734 TROPONINA T (CUALITATIVO)
- 664012, 664012A, 664012B CORTISOL EN SALIVA → 660189 CORTISOL PLASMATICO
- 660244 CHAGAS, PARASITEMIA MICRO STROUT → 660241 CHAGAS, AGLUTINACION DIRECTA
- 662649 AMONIO EN ORINA DE 24 HS → 660031 AMONEMIA
- 660417A GLUCOSA EN ORINA DE 24 HS → 660412 GLUCEMIA
- 666896 IONOGRAMA EN MATERIA FECAL → 660546 MODULO DE IONOGRAMA PLASMATICO
- Sodio y potasio (660839, 660839A, 660839B, 660753, 660754, 660754A) → ionograma
  plasmático o urinario

**C. Talio en orina (660862)** está en el set OSDE pero no tiene precio en el convenio:
ni talio en orina ni talio en sangre cotizan.

**D. Variantes de urocultivo, micobacterias y ASCA** (punto 3): se mapearon igual que
con los otros financiadores. Si OSDE no las acepta así, se sacan.

---

## 25/11/2026 · Análisis de convenios y nomencladores

Sólo lectura: no se modificó nada. Lista completa de coberturas con su estado:
`coberturas-convenios-2026-11-25.csv`.

### Coberturas

| Estado | Coberturas |
|---|---|
| En convenio (convenio vigente para la cobertura) | 315 |
| En convenio por plan (cada plan tiene su convenio: Avalian, Prevención Salud, Sancor Salud) | 3 |
| Sin convenio | 55 |
| **Total de coberturas activas** | **373** |

Ninguna de las 55 coberturas sin convenio tuvo convenio alguna vez en el LIS, y ninguna
tiene pacientes cargados. Por grupo:

- **Derivantes instituciones (19):** Cahamai, Crucero del Sur, Distribuidora Cristal, Dr/a
  Cano Ciro, El Territorio, Enriquez, Facultad de Ciencias Exactas, Gendarmería Nacional
  (prest. esp.), Genesia, Giovinazzo, Grupo Nordeste, Hospital Italiano de Buenos Aires,
  IPLIC, IPS No Convenido, Laboratorio Genia, Lazarte Construcciones, Lodemario, Milto
  Rodríguez, Newmont.
- **COVID empresas (13):** Dossuarez, Droguería Génesis, Hidra, Illex Fishing, Legal y
  Técnica de Gobernación, Milicic, Montecine, Research, Rovella Carranza, San Lucas
  Sanatorio, Sombracine, Teamsys, Vigencia Duque.
- **Derivantes lab colegas (10):** Dr/a Aguirre, Espíndola, La Cava, Medina (patólogo),
  Molina Laura, Molina Sabrina, Sanabria, Simois (veterinaria), Stuht, Valdovino.
- **Sanatorio Posadas (6):** Assitravel, Círculo Médico (LC), Cobertura Médico Deportiva,
  Sanatorio Posadas (LC), Travel Ace, Universal Assistance.
- **Programas (3):** Contingencia, Foundation One, Gempre Genómica.
- **Círculo (2):** Industria Metalúrgica, OSSALARA Círculo.
- **Directas (1):** Trabajo ALAC Glicosilada.
- **ConSalud (1):** Gematec (es un proveedor de equipos, como las coberturas de mantenimiento).

### Convenios

Hay 347 convenios cargados. Ninguno está vencido por fecha, porque ninguno tiene fecha de
fin. Los que sobran son de otro tipo:

| Situación | Convenios | Qué pasa hoy |
|---|---|---|
| En uso | 323 | Es el convenio que toma la admisión para su cobertura o plan |
| Reemplazados | 20 | Siguen activos, pero la cobertura tiene un convenio más nuevo que siempre gana. La admisión nunca los usa |
| Inactivos | 2 | El OSDE anterior (nomenclador 285 - NBU 2012 OSDE) y el de PRUEBA FACTURACION |
| De coberturas borradas | 2 | Jerárquicos Salud Internado-Boratti y Prev. Salud (no grav.), coberturas que ya no existen |

**Los 20 reemplazados** son el convenio que vino de Labcore, que quedó activo cuando se
cargó el nuevo:

- 14 con el nomenclador 340 - NBU 2024, reemplazados por N_1001 NBU 2024: Meld Salud,
  Jerárquicos Salud, APTM Asoc. Plantadores de Tabaco, APTM Mutual Celo, OSPIDA, Asis
  Salud, OSPES, OSPAPEL, OSPM, Poder Judicial, OSUTHGRA, Instituto de Cardiología, OSECAC y
  Dr/a Rivarola (preocupacionales). Con el convenio nuevo estas coberturas cotizan el 79%.
- 5 con el nomenclador NBU 2016 de Labcore, reemplazados por N_1003 NBU 2016: Galeno,
  SMAUNaM, Su Salud, OPSA y Domin.
- 1 con el nomenclador 293 - NBU 2023, reemplazado por N_1002 NBU 2023: Luz y Fuerza.

Descartar los 24 convenios no cambia nada en la admisión. **Ojo con los últimos 6**
(Galeno, SMAUNaM, Su Salud, OPSA, Domin y Luz y Fuerza): hoy cotizan peor con el convenio
nuevo que lo que cotizaría el viejo. Los cinco de NBU 2016 cotizan el 9% porque falta mapear
los estudios al set S_003, y Luz y Fuerza cotiza 0% porque no tiene asignado el set S_004.
Eso se arregla con el mapeo y la asignación del set, no volviendo al convenio viejo.

### Nomencladores

Hay 109 nomencladores: 43 ya estaban descartados, 43 los usa algún convenio vigente y 23
están en desuso.

**En desuso, para descartar (21):**

| Nomenclador | Precios | Por qué |
|---|---|---|
| NO SIRVE BORRAR | 1.390 | marcado para borrar |
| 272 - NBU 2012 SWISS MEDICAL 10_2023 | 914 | Swiss Medical usa N_1004 |
| 341 - NBU 2024 (IOSFA) | 777 | IOSFA usa N_1006 |
| 462 - NBU 2023 (Avalian) | 948 | Avalian usa N_1002 |
| 459 - NBU 2024 y 460 - NBU 2024 | 949 y 948 | ningún convenio |
| 297 - DERIVANTES ENERO 2024 | 467 | reemplazado por 475 - DERIVANTES MAYO 2026 |
| 445 - DERIVANTES OCTUBRE 2025 | 517 | reemplazado por 475 - DERIVANTES MAYO 2026 |
| 5 - NBU CÍRCULO - NBU REDUCIDO | 217 | ningún convenio |
| 334 - SANAT PARTICULARES MAYO 2024 | 27 | ningún convenio |
| 335 - TOMA DE MUESTRA Y ENVÍO | 5 | reemplazado por 481 y 471 |
| 325 - DIFER CITOMETRÍAS PARQUE - PEDIATRÍA | 4 | ningún convenio |
| 306 - IPS COVID y 76 - SAN COVID 19 | 2 y 9 | programas COVID |
| 333 - IPS GASES 03_2024 | 0 | vacío; reemplazado por 487 - IPS GASES 06-2026 |
| 12 - INOS PARA ARTs, 20 - AZISTIA lista 122017, 21 - ARANCEL VETERINARIO | 228, 8 y 42 | ya estaban inactivos |
| 285 - NBU 2012 OSDE | 828 | sólo lo usa el convenio OSDE inactivo |
| 441 - NBU 2023 | 948 | sólo lo usa un convenio de una cobertura borrada |
| 478 - DIF APTM 04/2026 | 0 | vacío; sólo es excepción de dos convenios reemplazados |

**A consultar antes de descartar (2).** Se importaron el 01/10/2026 y no los usa ningún
convenio. Puede ser que estén preparados para un cambio que no se terminó:

- N_1008-AVALIAN NBU2023 (1.259 precios). Hoy Avalian usa N_1002 NBU 2023.
- 003-JERARQUICOS NBU2024 (1.281 precios). Hoy Jerárquicos Salud usa N_1001 NBU 2024.

**Nomencladores en uso con observaciones:**

- 109 - PREOCUPACIONALES IOT está inactivo pero lo usa el convenio vigente de
  Preocupacionales IOT, y la admisión cotiza igual.
- MANT-1 lo usan las 14 coberturas internas de mantenimiento y no tiene precios.

### Para decidir

1. Descartar los 24 convenios: 20 reemplazados, 2 inactivos y 2 de coberturas borradas.
2. Descartar los 21 nomencladores en desuso. Tres de ellos (285, 441 y 478) hoy sólo los
   usan convenios a descartar, así que se descartan después de esos convenios.
3. N_1008-AVALIAN y 003-JERARQUICOS: ¿estaban preparados para reemplazar al convenio
   actual?
4. Las 55 coberturas sin convenio: ¿cuáles se siguen usando y necesitan convenio, y
   cuáles se desactivan?

"Descartar" es una baja lógica: el registro deja de aparecer pero queda en la base y se
puede recuperar.

---

## 25/11/2026 · Limpieza de convenios, nomencladores y coberturas

Aprobado por Gonzalo Jonsson. Baja lógica (se puede recuperar), sólo en la base del LIS.

| Se descartaron | Cantidad | Detalle |
|---|---|---|
| Convenios | 24 | Los 20 reemplazados, el OSDE inactivo, el de PRUEBA FACTURACION y los 2 de coberturas borradas (ver análisis anterior) |
| Nomencladores | 21 | Los de la tabla "En desuso, para descartar" del análisis anterior |
| Coberturas | 55 | Las 55 sin convenio listadas en el análisis anterior. Ninguna tenía pacientes, presupuestos ni planes |

No se descartaron N_1008-AVALIAN ni 003-JERARQUICOS: quedan a confirmar con CEBAC.

**Resultado.**
- Quedan 318 coberturas activas, todas con convenio: 315 propio y 3 por plan.
- Quedan 323 convenios, todos en uso.
- De los nomencladores, quedan 43 en uso más los 2 a confirmar.
- La admisión cotiza igual que antes: cada cobertura y cada plan siguen tomando el mismo
  convenio. Lo verificamos comparando antes y después.

Registro interno: `operation_logs` #190.

---

## 25/11/2026 · Asignación del set de prácticas a 26 coberturas

Aprobado por Gonzalo Jonsson. El convenio vigente de estas coberturas usa un nomenclador
nuevo (N_1001, N_1002, N_1003 o N_1006) cuyos precios están cargados sobre las prácticas de
un set, pero la cobertura no tenía ese set asignado. Por eso no cotizaba ningún estudio. A
cada cobertura se le asignó el set de su nomenclador.

| Set asignado | Coberturas | Antes | Después |
|---|---|---|---|
| S_005 NBU 2024 (nomencladores N_1001 y N_1006) | Centro Patrones, OSPSIP, Bramed, OSVARA, Farmacia, Andar Visitar (Círculo), OPDEA, OSTPCHPYARA, MeP Life, Asoc. Protección Familiar, OSPIP-Plásticos, AAPM, OSSACRA, OSSEG Seguros, OSPPRA, OSPEDYC UTEDYC, Clero, IOSFA y OSJERA (19) | 0% | 79% |
| S_004 NBU 2023 (nomenclador N_1002) | UTA, Luz y Fuerza, OSFATLYF y OSA (ConSalud) (4) | 0% | 78% |
| S_003 NBU 2016 (nomenclador N_1003) | Cámara del Tabaco, Omint y ACTIM (3) | 0% | 9% |

OSA (ConSalud) tenía asignado el set CONSALUD, pero su convenio usa N_1002. Se le asignó
S_004, que es el set de ese nomenclador. La regla que agrega el acto bioquímico (660001)
sigue funcionando para OSA, porque esa práctica existe en el set S_004.

Las tres coberturas del set S_003 recién van a cotizar bien cuando se mapeen los estudios a
ese set, que hoy tiene 120 estudios mapeados. Lo mismo pasa con Galeno, SMAUNaM, OSDEPYM, Su
Salud, OPSA, Domin, OSPATCA y Global Empresaria.

Registro interno: `operation_logs` #192, #193 y #194.

---

## 25/11/2026 · Uso de los sets de prácticas

Además de cuántos estudios cotiza cada cobertura, medimos qué parte de cada set de
prácticas está en uso: cuántas de sus prácticas tienen al menos un estudio asignado.

- Un **set corto con uso alto**, como OSDE (589 prácticas, 79% en uso), ya no tiene mucho
  más para asignar. Los estudios que no cotizan no están en el set, y la pregunta es de
  convenio.
- Un **set NBU completo** usa alrededor de la mitad de sus prácticas, y es normal: el NBU
  trae unos 1.375 códigos y CEBAC realiza unos 700.
- Un **set largo con uso muy bajo**, como el NBU 2016 (S_003) antes de esta jornada, es
  un mapeo pendiente.

La tabla de sets del informe "Coberturas CEBAC QA" muestra este número para cada set.

---

## 25/11/2026 · Set NBU 2016 (S_003): mapeo de estudios

**Coberturas afectadas (11).** Galeno, SMAUNaM, OSDEPYM, Omint, Su Salud, OPSA, Domin,
OSPATCA, Global Empresaria, Cámara del Tabaco y ACTIM. Todas tienen convenio con el
nomenclador N_1003 NBU 2016.

**Qué se hizo.** Se asignaron 989 estudios a su práctica del set, con 995 asignaciones en
total. Algunos estudios se facturan con más de una práctica. Las reglas aplicadas fueron:

- el mismo código: 618 asignaciones;
- variante con letra hacia el código base: 320 asignaciones;
- equivalencias que ya usan al menos otros dos financiadores NBU (NBU 2023, NBU 2024, Swiss
  Medical o ConSalud): 57 asignaciones.

Antes de aplicar, cada par se comparó con los sets NBU 2023, NBU 2024, Swiss Medical y
ConSalud: el 95% coincide con todos y el resto con al menos uno.

**Resultado**

| | Antes | Después |
|---|---|---|
| Estudios con práctica en el set | 120 | 1.109 |
| Uso del set (prácticas con estudios) | 4,1% | 52,6% |
| Estudios que cotiza cada una de las 11 coberturas | 120 (9,1%) | 1.092 (81,5%) |

Registro interno: `operation_logs` #195.

### Pendiente de confirmación de CEBAC (26 estudios)

Equivalencias que propone un solo financiador, o estudios que necesitan varias prácticas:

| Estudio | Práctica NBU 2016 propuesta | Por qué |
|---|---|---|
| CD4 SUB POBLACION LINFOCITARIA | 661011 CD4 por citometría de flujo | así en Swiss Medical |
| CD8 SUB POBLACION LINFOCITARIA | 661015 CD8 por citometría de flujo | así en Swiss Medical |
| CHAG_X2 CHAGAS Ac - 2 METODOS | 660242 CHAGAS (HAI) + 660243 CHAGAS Ac. Totales/IgG | así en OSDE y Swiss Medical |
| CD34 POR CITOMETRIA DE FLUJO | 663495 CD34 POSITIVAS | misma determinación |
| 800080 GLICOSAMINOGLICANOS EN ORINA DE 24 HS | 665621 GLICOSAMINOGLICANOS, cuantitativo | misma determinación |
| 800048 TAENIA SOLIUM Ac. IGG | 669270 TAENIA SOLIUM - Ac. Anti- IgG | así en ConSalud |
| 800035 POLYOMAVIRUS JC CUALITATIVO | 668467 POLIOMAVIRUS JC, PCR | así en ConSalud |
| 800050 / 800251 GALACTOMANANO (suero / BAL) | 665508 GALACTOMANANO de ASPERGILLUS, Ag. | así en ConSalud |
| 800060 AC ANTI MUSK | 667636 MuSK, Ac. Anti- | así en ConSalud |
| 800078 HEPATITIS E - RNA CUALITATIVO | 666011 HEPATITIS E - PCR | así en ConSalud |
| 800079 SCREENING NEONATAL AMPLIADO NEUROMETABOLICO | 667775 NEONATAL SCREENING AMPLIADO | así en ConSalud |
| 800023 MORFOLOGIA ESPERMATICA | 664868 ESPERMA, MORFOLOGÍA CRITERIO ESTRICTO | así en ConSalud |
| 666290A HIV-RNA CUANTITATIVO EN PLASMA SEMINAL | 661105 HIV, CARGA VIRAL | así en OSDE y NBU 2024 |
| 800036 INFLUENZA A, B ANTIGENO | 666725 INFLUENZA A Ag. + 666768 INFLUENZA B Ag. | son dos determinaciones |
| DENGUE (PCR + IgG + IgM) | 664369 DENGUE PCR + 664361 IgG + 664363 IgM | combo de tres determinaciones |
| 669641A a 669641F (estudios moleculares: FIP1L1-PDGFRA, MPL, FLT3, NPM-1, E2A-PBX1, CBFB-MYH11) | 669637 TRANSLOCACIÓN, varias - cualitativa | así en ConSalud |

Siguen sin práctica en el set unos 260 estudios, en su mayoría códigos propios de CEBAC
(800xxx), derivaciones CEMIC (900xxx) y estudios veterinarios.

---

## 25/11/2026 · Mapeos hacia Labcore de las coberturas descartadas

Aprobado por Gonzalo Jonsson. El LIS tiene una tabla propia que traduce cada cobertura a su
código en Labcore. Se desactivaron las 55 entradas de las coberturas descartadas, porque ya
no son necesarias. Así, si en la admisión se busca un paciente de Labcore con una de esas
coberturas, el formulario no se precarga con una cobertura que ya no existe: el operador
elige una vigente. El cambio es sólo en la base del LIS; en Labcore no se modificó nada.

Registro interno: `operation_logs` #196.

---

## 25/11/2026 · Baja de coberturas de mantenimiento, control de calidad y colegas

Pedido por Gonzalo Jonsson. Estas coberturas no son obras sociales: son proveedores de
mantenimiento, el control de calidad y los laboratorios colegas que derivan muestras.
Se dieron de baja 90 coberturas, ninguna con pacientes, órdenes ni presupuestos en el LIS:

- **Laboratorios colegas y veterinarias (74):** todo el grupo DERIVANTES LAB COLEGAS. Son
  58 bioquímicos y laboratorios (Biolac, Laboratorio Cabrera, Segovia, etc.) y 16
  veterinarias.
- **Mantenimiento y control de calidad (16):** Abbott, Roche, Siemens, BD, bioMérieux, BG
  Analizadores, Bioars, Montebio, Tecnolab, General Filter, MS Equipos, Mogesen
  (refrigeración), Mildenberg Leonel, control interno y control externo de calidad, y
  Sanatorio San Miguel. Cinco de los proveedores estaban cargados por error en el grupo
  CONSALUD.

Junto con ellas se dieron de baja sus 90 convenios y los 6 nomencladores que quedaron sin
uso: las listas de derivantes 475 (mayo 2026) y 364 (agosto 2024), el NBU 2024
veterinario, MANT-1 (vacío), 332 - PRP y 480 - Trabajo FCEQyN. También se desactivaron sus
90 traducciones de código hacia Labcore, sólo en la base del LIS.

Todo es baja lógica: se puede recuperar.

| | Antes | Después |
|---|---|---|
| Coberturas | 318 | 228 |
| Convenios | 323 | 233 |
| Nomencladores | 45 (43 en uso) | 39 (37 en uso) |

**Para confirmar:** Sanatorio San Miguel estaba en el grupo de mantenimiento, pero su convenio
es con el nomenclador del particular y cotizaba como una cobertura normal. Si es un cliente,
se restaura.

**Para definir:** cómo se van a admitir y valorizar las muestras que derivan los colegas.

Además, desde la pantalla de sets se renombraron los códigos de los sets: DEFAULT pasó a
S_001, OSDE a S_002, Swiss Medical a S_006, ConSalud a S_010 y los PAMI a S_007, S_008 y
S_009. También se creó el set S_011 MEDIFE, todavía vacío.

Registro interno: `operation_logs` #197.

---

## 25/11/2026 · PAMI: set de prácticas único con los tres niveles

Pedido por Gonzalo Jonsson. Objetivo: una sola cobertura PAMI, con un solo set de
prácticas, y que el nivel con el que se factura salga del tipo de paciente de la orden.

**Set S_007, ahora "PAMI (niveles 1, 2 y 3)".** Se juntaron los tres sets PAMI en uno:

| Nivel | Prácticas | Cuándo se factura |
|---|---|---|
| Nivel 1 (ambulatorio) | 1.326 | Tipo de paciente distinto de internado (ambulatorio, guardia, domicilio, etc.) |
| Nivel 2 (internados) | 1.299, las del ex S_008 (códigos 77) | Tipo de paciente internado |
| Nivel 3 (alta complejidad) | 30, las del ex S_009 | Siempre, con cualquier tipo de paciente |

S_008 y S_009 quedaron dados de baja. Se borró una fila de S_009 que era el encabezado del
archivo ("CODIGO / DETERMINACIONES") y dos prácticas repetidas con S_007.

**Mapeo de estudios.**
- Nivel 2: los estudios que tenían práctica de nivel 1 suman su equivalente de nivel 2
  (mismo código con 77 adelante): 940 mapeos nuevos. 24 prácticas de nivel 1 no tienen
  equivalente en la lista de nivel 2 (Coombs, crioaglutininas, factores de coagulación,
  electroforesis de hemoglobina, etc.).
- Nivel 3: 46 estudios de alta complejidad (cargas virales, HLA molecular, FISH, BCR-ABL,
  inmunofijación, subpoblaciones linfocitarias, etc.) se mapearon a su práctica de nivel 3,
  y se les quitó la de nivel 1, para que se facturen como nivel 3 siempre.

**Nomenclador único "PAMI"**, por unidades. Cada práctica tiene las unidades de su nivel,
tomadas de las listas que venían de Labcore: 485 PAMI AMB (nivel 1), 349 PAMI N2 (nivel 2) y
PAMI NIVEL III (nivel 3). El valor de la unidad no está en el nomenclador sino en el
convenio, así que ambulatorio e internados comparten nomenclador y cada uno tendrá su
convenio con su valor. Hay 1.230 prácticas con precio. Cuando la lista de Labcore traía el
precio por subletra (660105AO, 660105B…) y PAMI tiene una sola práctica, la práctica tomó
ese valor si todas las subletras coincidían; cuando PAMI usa otro código que el NBU
(albúmina 660015 es 660948 en PAMI), tomó el precio del estudio.

**Simulación** con todos los estudios activos, contra lo que cotiza hoy PAMI con la lista
de Labcore:

| | Hoy | Con el set único | Gana | Pierde |
|---|---|---|---|---|
| Ambulatorio | 940 | 901 | 104 | 143 |
| Internados | 661 | 783 | 168 | 46 |

Todavía no lo usa ninguna cobertura: se asigna junto con la cobertura única y los
convenios por tipo de paciente.

**Para decidir con CEBAC:**
- **Cultivos.** PAMI tiene una sola práctica de cultivo (660105) y la lista de nivel 1 de
  Labcore les daba 7 unidades a la mayoría de los cultivos y 5 a otros. Hay que definir
  cuántas unidades lleva; mientras tanto, 37 cultivos no cotizan como ambulatorio.
- **Estudios sin práctica PAMI.** 106 estudios que hoy se cotizan a PAMI ambulatorio con la
  lista de Labcore no tienen práctica en el set PAMI (FISH, PCR, espermogramas
  especiales, los códigos propios 800xxx, etc.), y 46 en internados. Hay que confirmar
  cuáles paga PAMI y con qué práctica; los que no, se cotizan como particular.
- **Traslocaciones varias (889641).** La lista de Labcore tenía 230 y 300 unidades según la
  subletra.

Registro interno: `operation_logs` #198.

---

## Próximos pasos propuestos

El detalle está en el informe "Coberturas CEBAC QA", sección "Próximos pasos".

1. **Decisiones de CEBAC.**
   - OSDE: si cubre sólo los 589 códigos del set y qué pasa con las equivalencias viejas.
   - Las 26 equivalencias del set NBU 2016.
   - Los nomencladores N_1008-AVALIAN y 003-JERARQUICOS.
   - Borrar las coberturas de prueba (ARENCO IT, PRUEBA FACTURACION) y confirmar Sanatorio
     San Miguel.
   - Cómo se admiten las derivaciones de colegas.
   - El precio de las derivaciones CEMIC en el particular.
2. **Correcciones técnicas.**
   - Que la admisión valorice los nomencladores por honorarios y gastos (ART Reconquista
     hoy cotiza en $0).
   - Los nomencladores inactivos que siguen en uso.
3. **Más estudios cotizados.**
   - Pasar los 117 convenios con nomencladores de Labcore a los NBU nuevos, o heredar el
     precio del código base para las subletras.
   - Completar sodio, potasio y los códigos propios en los NBU nuevos.
   - Definir PAMI.
4. **Providers.**
   - ConSalud y Swiss Medical.
   - Pasaporte de producción de ITC.
   - Credenciales de Sancor, Traditum, SMAUNaM y PAMI Mi Cartilla.
5. **Validación.** Admisiones de prueba por familia de cobertura, comparando con lo que
   factura hoy CEBAC, y el informe final.
