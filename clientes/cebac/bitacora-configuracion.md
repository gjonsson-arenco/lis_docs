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
- **Traslocaciones varias (889641).** La lista de Labcore tenía 230 y 300 unidades según la
  subletra.

### Mapeo de los estudios que perdían cotización

Quedaban 119 estudios que hoy se cotizan a PAMI con la lista de Labcore y no tenían
práctica en el set PAMI (106 en ambulatorio y 46 en internados, algunos en los dos). Se
revisaron uno por uno:

- **Aplicados por regla (43 estudios, 73 mapeos):** el mismo código NBU con el prefijo
  PAMI (88 en nivel 1, 77 en nivel 2), la misma determinación y las mismas unidades que hoy.
  Por ejemplo: FISH de RB1, ATM e IGH/FGFR3; traslocaciones 14;18, MLL/AF4, PML-RAR,
  AML1-ETO y TEL-AML1; pro-BNP; proteína C y S; plasminógeno; parainfluenza 1, 2 y 3; ASCA;
  hemoglobinuria paroxística nocturna; hepatitis delta; gliadinas; Polyomavirus BK. El
  dengue por PCR pasa a la práctica PAMI de 110 unidades (hoy 130).
- **Confirmados por Gonzalo (14):** la misma determinación con otro nombre en PAMI. La
  práctica PAMI no tenía precio y tomó las unidades con las que el estudio cotiza hoy: CH50,
  MAR test, azul de anilina, test de MOST, test hiposmótico, 2,5-hexanodiona, p-aminofenol,
  triple test, anti-ribosomal P, litiasis renal, Leishmania, galactomanano, Bordetella por
  PCR y MuSK.

| | Hoy | Con el set PAMI |
|---|---|---|
| Ambulatorio | 940 | 952 |
| Internados | 661 | 819 |

**Para confirmar con CEBAC (16):** son de otro método o variante.
- Fibrosis quística ΔF508 → "19 mutaciones".
- Filiación de adulto → "filiación hasta 3".
- HLA B27 antígeno → HLA B27 molecular.
- HLA DQ2/DQ8: la práctica de nivel 3 (sin precio) o "HLA DQ molecular" (160 unidades).
- Hantavirus totales → Hantavirus IgM.
- *R. conorii* → *R. typhi*.
- Los 4 estudios 669641 (FIP1L1-PDGFRA, MPL, FLT3, NPM1) → "traslocaciones varias", que
  necesita definir 230 o 300 unidades.
- FISH AML1-ETO y PML-RAR → la traslocación por PCR.
- Tiempo de sangría → "coagulación y sangría" (1 unidad, hoy 8).
- TPHA → "FTA-ABS y TPHA" (8 unidades, hoy 15).
- Influenza A+B por PCR → influenza B por PCR.

**Sin práctica PAMI (46):** si PAMI no los paga, se cotizan como particular.
- **Faltan en nivel 2:** cetonuria, Coombs directa e indirecta, fibrinógeno, ácido fólico y
  HLA B27.
- **FISH:** el resto (5q, 7q, centrómero 8, MLL, 1p/1q).
- **Moleculares y paneles:** panel respiratorio, carbapenemasa, BRCA por secuenciación,
  Chikungunya, SARS-CoV-2 por PCR, rinovirus, Zika, JAK2, cultivo de Neisseria, urocultivo
  de primer chorro.
- **Otros:** espermogramas especiales, Helicobacter antígeno, hepatitis E IgM, PRP,
  risperidona.
- **Códigos propios 800xxx:** COVID, inhibidor de factor VIII, screening prenatal, etc.

Registro interno: `operation_logs` #198 (set y nomenclador), #199 y #200 (mapeos), #201
(precios de los 14).

---

## 25/11/2026 · PAMI: una sola cobertura

Pedido por Gonzalo Jonsson. Con la versión del 25/11 (convenios y códigos hacia Labcore
que dependen de la orden), PAMI quedó como una sola cobertura.

**Cobertura "PAMI"** (código 53, la que tenía pacientes y órdenes), con el set PAMI de tres
niveles. El convenio que se aplica depende de la orden:

| Convenio | Se aplica cuando | Nomenclador | Valor de unidad |
|---|---|---|---|
| PAMI ambulatorio | el paciente no es internado y la orden no es de Sanatorio Posadas | PAMI | $693,71 |
| PAMI internados | el paciente es internado y la orden no es de Sanatorio Posadas | PAMI | $564,22 |
| PAMI Sanatorio Posadas | la orden es del origen Sanatorio Posadas | NBU 2016-2019 | $620,65 |

Ejemplo, hemograma: ambulatorio 4 unidades = $2.774,84; internado 3 unidades = $1.692,66;
Sanatorio Posadas $1.861,95. Ambulatorio cotiza 952 estudios, internados 819 y Sanatorio
Posadas 929.

**Bajas:** las coberturas PAMI CEMEBA (cápita y directo), PAMI Nivel III, PAMI Nivel III
extracápita, PAMI Nosiglia y PAMI Posadas, con sus convenios, y los nomencladores 485 PAMI
AMB, 349 PAMI N2 y PAMI NIVEL III. Ninguna tenía pacientes ni órdenes.

**Hacia Labcore**, la cobertura PAMI viaja como:
- 54 (PAMI Nosiglia) cuando la orden es del origen Nosiglia;
- SP023 (PAMI Posadas) cuando es del origen Sanatorio Posadas;
- 53 (PAMI Ambulatorio) en el resto.

**Para definir con CEBAC:** cuándo una orden tiene que llegar a Labcore como PAMI CEMEBA cápita
(PAM_C) o directo (PAM_D), PAMI Nivel III (20040) o extracápita (P_EC). Hasta entonces esas
órdenes llegan como 53, y un paciente que en Labcore tiene alguna de esas coberturas aparece
en admisión sin cobertura, para elegirla a mano.

Registro interno: `operation_logs` #202.

---

## 25/11/2026 · Fusión de coberturas del mismo financiador

Pedido por Gonzalo Jonsson. Criterio: una cobertura por financiador (a quién se le
factura). Cuando lo único que cambia es el tipo de paciente, el plan o el sanatorio, va en
una sola cobertura, y Labcore sigue recibiendo su código de siempre según la orden. Ninguna
de las coberturas dadas de baja tenía pacientes ni órdenes. Quedan 207 coberturas (eran 222).

**Internados con el mismo financiador.** Cada par quedó en una cobertura:

| Cobertura | Se dio de baja | Convenio | Hacia Labcore |
|---|---|---|---|
| OSPECON IOT | OSPECON INTERNADOS IOT | el mismo (NBU 2016, $961,71) | OI; IO024 si el paciente es internado |
| VISITAR IOT | VISITAR IOT INTERNADOS | el mismo (NBU 2016, $938,95) | AV; VII si es internado |
| TRASPLANTE MISIONES | Trasplante Misiones internados | el mismo (293, $730) | TM; TMI si es internado |
| LUZ Y FUERZA | LUZ Y FUERZA INTERNADOS | general N_1002 a $1.428; internados 293 a $1.407,93 | 793; 29 si es internado |

**Planes.** Cada grupo quedó en una cobertura con planes y un convenio para todos:

| Cobertura | Planes | Convenio | Hacia Labcore |
|---|---|---|---|
| UPCN | Accor ambulatorio gravado y no gravado, Plan clásico gravado y no gravado | 340 - NBU 2024, $920 | 1614, 614NG, 20072 y UPCNG según el plan |
| MEDIFE | Obligatorio, Voluntario | 438 - NBU 2016, $1.120,44 | MED_O, MED_V |
| HOSPITAL ALEMAN | Gravado, No gravado | NBU 2016, $1.033,80 | HA_G, HA_NG |

En la admisión hay que elegir el plan del afiliado. Si no se elige, la orden llega a Labcore
con el código del primer plan (Accor gravado, Medife obligatorio, Hospital Alemán gravado).

**Particulares internados.** Las seis coberturas de particulares internados (Boratti,
Camino, CEMEBA, IOT, Nosiglia y Posadas) se dieron de baja. La cobertura PARTICULARES suma el
convenio "particulares internados en sanatorio" (340 - NBU 2024, $1.700), que se aplica
cuando el paciente es internado y la orden viene de uno de esos seis sanatorios. Hacia
Labcore sale el código de cada sanatorio (10001, SC009, 10003, 10005, 10002, 10004). Un
particular ambulatorio, o uno internado en una sede de CEBAC, sigue con el convenio
particular de siempre ($810). Ese mismo precio de particular internado es el que se muestra
como referencia cuando la cobertura de un internado de esos sanatorios no cubre una
práctica.

Registro interno: `operation_logs` #203.

---

## 25/11/2026 · OSDE: se revierten las equivalencias

Pedido por Gonzalo Jonsson al revisar el mapeo de OSDE. Quedan los estudios que son una
variante con letra de una práctica OSDE (por ejemplo, Chlamydia PCR en cada material →
663649, HIV combo → 660063). Se quitaron los 32 mapeos que llevaban un estudio a una
práctica OSDE de otro código, o a varias prácticas:

- **Equivalencias con otro código:** HPV a 6531, HLA DQ2/DQ8 a 6244, panel respiratorio a
  6865, panel de cáncer hereditario a 7191, JAK2 a 6821, Von Willebrand a cofactor de
  ristocetina, TSH neonatal, fenciclidina, porfirinas, protoporfirina, hidatidosis, HIV en
  plasma seminal, Listeria, TPHA y sífilis IgG a FTA/ABS, carbamazepina epóxido, tiempo de
  sangría, MAR test, calcitriol, dengue IgM.
- **Varias prácticas:** excreción fraccional de sodio y de potasio (creatinina + ionograma
  plasmático + ionograma urinario).

Quedan, por ser variantes de su código base: urocultivo + orina completa → urocultivo
(660911), micobacterias atípicas → 667669, ASCA IgA → 668956 y recuento celular en bolsa de
diálisis → 660620.

OSDE y Binaria pasan de cotizar 810 estudios a 782 (59% de lo que cotiza el particular).
Los estudios revertidos vuelven a "usar particular" hasta que CEBAC confirme con qué
práctica los paga OSDE.

Registro interno: `operation_logs` #204.

---

## 25/11/2026 · Set por defecto: prácticas repetidas por subletra

Pedido por Gonzalo Jonsson. El set por defecto (el del particular y de las coberturas sin set
propio) traía una práctica por cada variante con letra de un código (660105A cultivo de LCR,
660105B cultivo de líquido abdominal, …), porque las listas de Labcore ponen precio por
subletra. Cuando todas las variantes de un código valen lo mismo en todas las listas, son la
misma práctica: se juntaron en la práctica base.

- 55 familias, 200 prácticas juntadas. El set pasa de 1.423 a 1.226 prácticas.
- Se crearon tres prácticas base que no existían: 660105 bacteriológico, cultivo e
  identificación del germen (36 cultivos); 666606 IgE específica (69 alergenos); 669101
  fenotipificación de síndromes linfoproliferativos (4 citometrías).
- Ejemplos: ácido base arterial y venoso → 660005; urea sérica, en orina y en diálisis →
  660902; glucosa en diálisis → 660412; FISH oncohematológicos → 664145; drogas de abuso →
  661050.
- Ningún estudio cambió de precio ni dejó de cotizar (verificado en una cobertura por cada
  lista del set). 122 estudios pasaron a cotizar en 12 coberturas donde su variante no tenía
  precio y sus hermanas sí (sobre todo IgE específicas, FISH y Chlamydia, al mismo valor que
  las hermanas en esa lista); confirmado por Gonzalo.

**Quedaron aparte, porque en alguna lista valen distinto que sus hermanas:** cultivos de
loquios, punta de catéter y secreción ótica derecha e izquierda; tipificación de germen y
antibiograma; cultivo de pie diabético; antidepresivos tricíclicos; Chlamydia antígeno en
orina; FISH 8q24 y t(14;18); IgE a carne vacuna, hormiga negra y Penicillium; panel de
pólenes de hierbas. Conviene que CEBAC revise si esas diferencias son a propósito o errores
de carga.

**Sin tocar, porque la práctica base vale distinto que las variantes:** creatinina,
micológicos, filiación y Mycoplasma-Ureaplasma. Tampoco se tocaron las variantes que son otra
determinación con otro precio (hemograma e índices hematimétricos, curvas de glucosa e
insulina, hemocultivos por cantidad de muestras, paneles de subpoblaciones, monitoreo de
fármacos, etc.).

Registro interno: `operation_logs` #205.

---

## 25/11/2026 · MEDIFE: set de prácticas con los códigos del validador

Pedido por Gonzalo Jonsson. MEDIFE valida las prácticas en línea a través de Traditum, y
en esa validación espera sus propios códigos: el código NBU sin el "66" inicial, completado
a seis dígitos (el hemograma 660475 viaja como 000475; la glucemia 660412 como 000412). El
set S_011 MEDIFE, que estaba vacío, se cargó a partir del set por defecto con los códigos
en ese formato, y se asignó a la cobertura MEDIFE (MED_O).

- 733 prácticas, una por cada código NBU del set por defecto: 715 códigos base y 205
  variantes con letra, que para MEDIFE son la misma práctica (todas las IgE específicas
  660606x van como 000606; los cultivos 660105x como 000105).
- 1.111 estudios quedaron con su práctica MEDIFE, copiando el mapeo del set por defecto.
  De las 733 prácticas, 727 tienen al menos un estudio.
- 18 códigos nacieron sólo de variantes porque el set por defecto no tiene la base; 13 toman
  el nombre oficial del NBU 2024 y 5 conservan el nombre de la variante (001134 monitoreo de
  antidepresivos, 006290 HIV-RNA en plasma seminal, 009633 BCR-ABL, 009641 FIP1L1-PDGFRA,
  009936 MLL/AF9).

**Quedan sin práctica MEDIFE 279 estudios**, porque en el set por defecto van con códigos
propios de CEBAC (800xxx y 900xxx, 223 prácticas), veterinarios (44) o administrativos y
fórmulas (39). En la admisión de un afiliado de MEDIFE esos estudios no se van a poder
validar en línea; si alguno se factura a MEDIFE, CEBAC tiene que indicar con qué código.

**Supuesto a confirmar con MEDIFE:** que todo el nomenclador bioquímico siga la misma regla
que los dos códigos que Traditum probó (000412 y 000475). Si un código no existe para
MEDIFE, el validador lo rechaza como "prestación inexistente" y la admisión lo muestra.

Registro interno: `operation_logs` #206 (prácticas y mapeos) y #207 (asignación del set).

---

## 25/11/2026 · MEDIFE: nomenclador sobre el set nuevo

Pedido por Gonzalo Jonsson. El convenio de MEDIFE valorizaba con el nomenclador "438 - NBU
2016", cuyos precios están cargados sobre las prácticas del set por defecto. Como la cobertura
pasó al set MEDIFE, se creó el nomenclador **N_1011 MEDIFE (NBU 2016)** con los mismos
precios, por unidades, y el convenio de MEDIFE (vigente desde el 01/06/2026, unidad
$ 1.120,44) quedó sobre él.

- 667 prácticas del set MEDIFE con el precio que su código NBU tenía en el 438 (el de la
  práctica base; si la base no tenía, el de la variante, en 16 casos). 66 prácticas quedan
  sin precio porque tampoco lo tenían en el 438.
- 27 precios del 438 sobre códigos propios 800/900 del set por defecto se copiaron tal cual,
  para que esos estudios sigan cotizando a MEDIFE por la práctica del particular como hasta
  ahora.
- Estudios que cotizan a MEDIFE: de 997 a 1.062 (del 76% al 81% de lo que cotiza el
  particular); 65 estudios ganados y ninguno perdido.

**Cambian de precio 19 familias, porque en el 438 la variante valía distinto que la base y el
set MEDIFE tiene una sola práctica por código, que toma el precio de la base:** curvas de
somatotrofina (24, 48 y 60 → 12 unidades), hemocultivo automatizado por 2 y 3 muestras (50 y
75 → 25), hemocultivo común por 2 muestras (7,5 → 10), dengue PCR (130 → 110), Epstein Barr
carga viral (250 → 150), HLA DR molecular (160 → 150), BCR-ABL cuantitativo (300 → 360),
índices de función renal (1 → 3), orina completa variante (2,5 → 3), coagulograma variante
(5,5 → 6), VDRL variante (2 → 4), Chagas Strout variante (6 → 3), fosfatasa ácida variante
(3 → 5), recuento de blancos variante (0 → 1), cortisol en saliva variante (12 → 10),
Chlamydia PCR variante C (80 → 90), micológico V (8 → 10), bacteriológico Z (5 → 7) y
FIP1L1-PDGFRA B (300 → 230). Conviene que facturación confirme si MEDIFE paga esas
variantes al valor de la base o si hay que tratarlas aparte.

Registro interno: `operation_logs` #208.

---

## 25/11/2026 · MEDIFE: validación en línea por Traditum activa

Pedido por Gonzalo Jonsson. La cobertura MEDIFE quedó conectada al validador de Traditum,
que es el canal por el que MEDIFE autoriza en línea. Se usa el sitio emisor de CEBAC en la
producción de Traditum, como indicaron Traditum y MEDIFE en el ticket #67232 (MEDIFE no tiene
un ambiente de prueba con datos propios: valida contra su padrón real).

- En la admisión de un afiliado de MEDIFE, el sistema consulta la cobertura (elegibilidad) y
  muestra plan y estado. Con **Registrar** se pide la autorización de las prácticas, que
  MEDIFE aprueba o rechaza práctica por práctica, y una autorización registrada se puede
  anular entera. No hay pre-autorización sin consumo: la autorización se pide cuando el
  operador registra.
- Las prácticas viajan con los códigos del set S_011 MEDIFE (NBU sin el 66, a seis dígitos)
  y nomenclador bioquímico. Prescriptor y efector no son obligatorios para MEDIFE; el
  prescriptor sale de la orden si el médico tiene matrícula cargada.
- Verificado desde el servidor: la elegibilidad del afiliado de prueba vuelve aprobada
  (plan PLATA).
- **Paciente de prueba cargado:** DE PRUEBA, ASOCIADO, DNI 9999999, nacido el 04/12/1968,
  con cobertura MEDIFE y credencial 30646117400000. Es el afiliado de prueba que provee
  MEDIFE: sirve para probar elegibilidad, registro y anulación desde la admisión. Las
  autorizaciones que se registren con él conviene anularlas, y el paciente se borra al
  terminar la homologación.

Configuración interna: provider `TRADITUM_GATEWAY` (seeder), MED_O con ese provider y payer
`medife`, bloque `TRADITUM_*` en el `.env.prod` del broker gateway, release
`cebac/2026-11-25-2342`. Registro interno: `operation_logs` #209 (provider en la cobertura) y
#210 (paciente de prueba).

---

## 25/11/2026 · Sancor Salud: set y nomenclador propios

Pedido por Gonzalo Jonsson. Sancor Salud va a validar en línea por su autorizador, y las
coberturas que validan por un proveedor externo se manejan con un set de prácticas y un
nomenclador propios, aislados del resto. La cobertura SANCOR SALUD (697) facturaba con el set
NBU 2023 (S_004) y el nomenclador "N_1002 NBU 2023 ORIGINAL", que comparten otras siete
coberturas (OSA, Luz y Fuerza, UTA, Avalian, OSFATLYF, Medicus).

- Se creó el set **S_012 SANCOR SALUD** como copia exacta del NBU 2023: 1.374 prácticas con
  los mismos códigos NBU (el autorizador de Sancor los acepta tal cual, sin nomenclador) y los
  1.072 mapeos de estudios (1.064 estudios).
- Se creó el nomenclador **N_1012 SANCOR SALUD (NBU 2023)** con los 1.267 precios por unidad
  del N_1002, práctica por práctica; dos prácticas conservan la marca de "requiere autorización".
- Los cuatro convenios de la cobertura (C_029 gravados y no gravados a $ 1.060 la unidad;
  C_032 plan 800 gravados y no gravados a $ 1.010) pasaron al nomenclador nuevo sin cambiar
  valores ni vigencia.
- Verificado con la valorización real de la admisión: los cuatro planes cotizan los mismos
  1.045 estudios que antes (78,4% de lo que cotiza el particular), ninguno ganado ni perdido.
  A partir de ahora, un cambio de precio o de código para Sancor se hace en N_1012 y S_012 y
  no toca a las otras coberturas.

Registro interno: `operation_logs` #214 (set), #215 (nomenclador) y #216 (cobertura y
convenios).

---

## 25/11/2026 · Sancor Salud: validación en línea activa (modo prueba)

Pedido por Gonzalo Jonsson. La cobertura SANCOR SALUD quedó conectada al autorizador HL7 de
Sancor Salud a través del broker gateway. Sancor no tiene ambiente de prueba: las llamadas van
a su servicio real marcadas como **desarrollo (D)**, que es lo que Sancor pide para la
homologación; el pase a producción (P) se hace cuando Sancor revise las pruebas y acuerde la
fecha, y hasta entonces lo que se autorice no vale como prestación real.

- En la admisión de un afiliado de Sancor, el sistema consulta la elegibilidad (estado, plan),
  **al cotizar pre-valida las prácticas** sin consumirlas y con **Registrar** pide la
  autorización, que Sancor aprueba o rechaza práctica por práctica. Una autorización registrada
  se anula entera.
- Las prácticas que Sancor autoriza de antemano (formulario 4) se registran con ese número: al
  subir la autorización de la práctica en la admisión, el sistema pide el número de formulario
  y lo manda junto con la práctica.
- Sancor exige el token de 4 dígitos del asociado en cada autorización. Mientras dure la
  homologación se usa el comodín 9999 que Sancor admite excepcionalmente, así la admisión no
  pide el token; se le consultó a Sancor si puede seguir así en la puesta en marcha.
- El prestador y el efector viajan con el CUIT de CEBAC; el prescriptor sale de la orden si el
  médico tiene matrícula cargada. Si una orden no trae un diagnóstico codificado, va Z71.
- Verificado desde el servidor: la elegibilidad del asociado de prueba vuelve aprobada (plan
  SANCOR 1500) y la admisión ve a la cobertura con pre-validación, registro, anulación y
  número de autorización previa.
- **Paciente de prueba:** al paciente DE PRUEBA, ASOCIADO (DNI 9999999) se le agregó la
  cobertura SANCOR SALUD, plan no gravados, con la credencial de prueba 121297/00 que provee
  Sancor. Sirve para probar el circuito desde la admisión; las autorizaciones que se registren
  con él conviene anularlas, y la cobertura se borra al terminar la homologación.

Configuración interna: provider `SANCOR_GATEWAY` (seeder), cobertura 697 con ese provider y
payer `sancor`, bloque `SANCOR_*` en el `.env.prod` del broker gateway (backup
`.env.prod.bak-20261126004247-sancor`), contenedor del gateway recreado. Registro interno:
`operation_logs` #219 (provider en la cobertura) y #220 (paciente de prueba).

---

## 26/11/2026 · Swiss Medical: validación en línea activa (ambiente de prueba de SMG)

Pedido por Gonzalo Jonsson. La cobertura SWISS MEDICAL (451) quedó conectada al API de
prestadores de Swiss Medical Group a través del broker gateway, contra el **ambiente de prueba
(PRE)** de SMG y con el prestador 130388 de CEBAC. El pase a producción lo define SMG cuando dé
por homologada la integración y entregue la URL y las credenciales productivas.

- En la admisión de un afiliado de Swiss, el sistema consulta la elegibilidad y con
  **Registrar** registra las prácticas una por una (cada una con su número de transacción de
  SMG), así se sabe cuál aprobó y cuál rechazó y se puede anular de a una. SMG puede exigir el
  token de la credencial digital del afiliado; en ese caso la admisión lo pide.
- **Set y nomenclador propios, ya estaban y no se tocaron:** la cobertura usa el set S_006
  SWISS MEDICAL (1.286 prácticas, 1.081 estudios mapeados) y el convenio C_045 con el
  nomenclador N_1004 SWISS MEDICAL (NBU2016), que tiene precio para las 1.286 prácticas del set
  ($1.458,49 la unidad desde el 01/09/2026). Hoy cotiza 1.078 estudios, el 80,6% de lo que
  cotiza el particular; lo que falta son estudios sin práctica en el set (propios 800/900,
  veterinarios y administrativos).
- Las coberturas ART SWISS MEDICAL (ASW y PSSW) son ART, no van por este API y siguen con sus
  convenios NBU.
- **Estado: el usuario de API que SMG entregó está bloqueado** ("El usuario está bloqueado",
  error 1005) tanto desde el servidor del laboratorio como desde desarrollo; el último ingreso
  exitoso fue el 16/10/2026. Hasta que SMG lo desbloquee, la admisión de un afiliado Swiss
  muestra la validación en línea como no disponible y la cobertura se sigue valorizando por su
  convenio; la admisión no se traba. Se le pidió a SMG el desbloqueo.
- **Paciente de prueba:** al paciente DE PRUEBA, ASOCIADO (DNI 9999999) se le agregó la
  cobertura SWISS MEDICAL con la credencial del socio de prueba que provee SMG
  (8000067180171000161). Sirve para probar el circuito desde la admisión cuando SMG
  desbloquee el usuario; se borra al terminar la homologación.

Configuración interna: provider `SWISS_MEDICAL_GATEWAY` (seeder, id 6), cobertura 451 con ese
provider y payer `swiss-medical`; el bloque `SMG_*` ya estaba en el `.env.prod` del broker
gateway con los mismos valores que desarrollo (no hizo falta tocar el servidor ni recrear
contenedores). Registro interno: `operation_logs` #221 (provider en la cobertura) y #222
(paciente de prueba).

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
   - ConSalud. (Swiss Medical quedó activada el 26/11 contra PRE; falta que SMG desbloquee el
     usuario de API y entregue producción.)
   - Pasaporte de producción de ITC.
   - Credenciales de Sancor, Traditum, SMAUNaM y PAMI Mi Cartilla.
5. **Validación.** Admisiones de prueba por familia de cobertura, comparando con lo que
   factura hoy CEBAC, y el informe final.
