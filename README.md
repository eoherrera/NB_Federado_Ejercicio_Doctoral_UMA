# EJD-UMA-003 v8.8 · Naive Bayes Federado con MoG Real + Modelo Híbrido

**Ejercicio doctoral** | Programa de Doctorado en Tecnologías Informáticas
Universidad de Málaga

**Autor:** Ing. Edgar O. Herrera Logroño, M.Sc. en Inteligencia Artificial, VIU España
**Directores propuestos:** Prof. Ezequiel López Rubio · Prof. Juan Miguel Ortiz de Lazcano

---

## De qué trata este ejercicio

Cuando una institución financiera, un hospital y una entidad gubernamental colaboran para detectar ciberataques sin compartir sus datos, cada nodo aprende un modelo local y el servidor debe combinarlos de forma inteligente.

Este ejercicio implementa una **Mixtura de Gaussianas (MoG) real** en la inferencia federada: el servidor no colapsa las distribuciones locales — mantiene las k gaussianas vivas y las combina ponderadamente en el momento de la predicción:

```
P(x | c) = sum_k  w_k * N(x ; mu_k(c), sigma2_k(c))
```

En esta versión se incorporan las correcciones aprobadas por ambos directores y la verificación de integridad solicitada por el Prof. López Rubio.

---

## Correcciones incorporadas en esta versión

**Corrección 1 — Prof. Ortiz de Lazcano (21-abr-2026):**
Las variables categóricas (protocol_type, service, flag) se procesan con CategoricalNB en lugar de LabelEncoder + GaussianNB. LabelEncoder asignaba códigos enteros que GaussianNB interpretaba como distancias numéricas reales, introduciendo un sesgo sin fundamento matemático. La solución combina ambos modelos multiplicando sus probabilidades:

```
P(x|c) = P_cat(x_qual|c) * P_gauss(x_quant|c)
```

**Corrección 2 — Prof. López Rubio (20-abr-2026):**
Se amplían los valores de alpha de 3 a 7 niveles [0.05, 0.1, 0.2, 0.3, 0.5, 0.7, 1.0] para observar el gradiente suave en el comportamiento de las cuatro propuestas al variar la heterogeneidad.

**Verificación — Prof. López Rubio (24-abr-2026):**
Se confirma que los conjuntos de validación, test y OOD no contienen categorías no vistas en entrenamiento. OrdinalEncoder fue ajustado sobre el conjunto completo antes de la división de datos, lo que permite afirmar con fundamento que todas las categorías presentes en val, test y OOD fueron vistas durante el ajuste del codificador. No es necesario repetir los experimentos.

---

## Las cuatro propuestas comparadas

Siguiendo el orden solicitado por el Prof. López Rubio:

| # | Nombre | Descripción | Pesos |
|---|--------|-------------|-------|
| 1 | **Centralizado** | Un solo NB entrenado con todos los datos | N/A — referencia teórica |
| 2 | **Baseline (FedAvg)** | Promedio ponderado de parámetros por tamaño de dataset | n_k / n |
| 3 | **Mezcla Entropía** | Pesos inversamente proporcionales a la entropía local | 1/H(k) normalizado |
| 4 | **Mezcla Aprendida** | Pesos aprendidos desde validación con regularización ICC | Nelder-Mead + L2→ICC |

---

## Variables de riesgo CRISC

| Variable | Qué mide | Rango |
|----------|----------|-------|
| CMM | Madurez del proceso de gestión de riesgos | 1 a 5 |
| KCI | Proporción de controles de seguridad implementados | 0 a 1 |
| KRI | Frecuencia de activación de indicadores de riesgo | 0 a 1 (menor es mejor) |
| CVSS | Puntuación media de vulnerabilidades | 0 a 10 |
| ICC | Índice de Coherencia Contextual | 0 a 1 |

**Fórmula:**
```
ICC = (CMM / 5) * KCI * (1 - KRI) * (1 - CVSS / 10)
```

**Valores por nodo:**

| Nodo | CMM | KCI | KRI | CVSS | ICC |
|------|-----|-----|-----|------|-----|
| Financiero | 4 | 0.82 | 0.12 | 3.2 | 0.393 |
| Salud | 3 | 0.70 | 0.25 | 5.1 | 0.154 |
| Gobierno | 2 | 0.55 | 0.40 | 6.8 | 0.042 |

---

## Resultados principales — NSL-KDD, evaluación OOD

| Alpha | JS | Aprendida | Baseline | Delta | McNemar |
|-------|----|-----------|----------|-------|---------|
| 0.05 | 0.63 | 0.2799 | 0.2678 | +0.012 | chi2=242.5, p<0.001 |
| 0.1 | 0.67 | 0.3287 | 0.3292 | -0.001 | chi2=4.5, p=0.034 |
| 0.2 | 0.52 | 0.3243 | 0.3321 | -0.008 | chi2=140.7, p<0.001 |
| 0.3 | 0.40 | 0.2458 | 0.2492 | -0.003 | chi2=44.1, p<0.001 |
| 0.5 | 0.32 | 0.4278 | 0.4201 | +0.008 | chi2=297.9, p<0.001 |
| 0.7 | 0.28 | 0.4663 | 0.4665 | -0.000 | chi2=3.0, p=0.082 |
| 1.0 | 0.18 | 0.3630 | 0.3619 | +0.001 | chi2=24.5, p<0.001 |

El gradiente es continuo al variar alpha, respondiendo la solicitud del Prof. López Rubio.

---

## PROTOCOLO-STRESS · Resumen (v8.8)

| Verificación | Resultado |
|-------------|-----------|
| Tamaño dataset (125,973 registros) | OK |
| Clases presentes en todos los conjuntos | OK |
| Heterogeneidad real en 7 niveles de alpha | OK |
| Prueba ácida alpha=0.01 | OK |
| Pesos suman 1.0000 en todos los niveles | OK |
| Predicciones diversas (5/5 clases) | OK |
| Categorías nuevas en val/test/OOD | OK — ninguna detectada |
| McNemar significativo en 6/7 niveles | OK |

---

## Limitaciones declaradas

**Limitación 1 — Dataset:** NSL-KDD es un dataset de laboratorio de 2009. Sus distribuciones no reflejan la complejidad de tráfico real moderno. La extensión a CIC-IDS2017 y UNSW-NB15 está planificada como trabajo siguiente.

**Limitación 2 — Gradiente no uniforme:** La Mezcla Aprendida supera al Baseline en alta heterogeneidad pero pierde en heterogeneidad moderada, lo que indica sobreajuste del optimizador al conjunto de validación cuando las distribuciones son similares.

**Limitación 3 — Variables CRISC estáticas:** Los valores de ICC se definen al inicio y no evolucionan por ronda de entrenamiento.

---

## Pregunta abierta para el siguiente ejercicio

La verificación confirma que OrdinalEncoder funciona correctamente en NSL-KDD. La pregunta que abre el siguiente ejercicio es: cuando en CIC-IDS2017 o UNSW-NB15 aparezcan valores de Protocol o Flags nunca vistos en entrenamiento, ¿sería suficiente asignar max_categorías + 1, o convendría explorar estrategias de open-set recognition para capturar los patrones propios de ataques genuinamente nuevos? Esta pregunta conecta este ejercicio con la línea de trabajo siguiente.

---

## Cómo ejecutar en Google Colab

1. Abrir `EJD_UMA_003_v8_8.ipynb` en Google Colab
2. Ejecutar **Runtime > Run all**
3. Tiempo estimado: 30-40 minutos en CPU de Colab
4. Al finalizar suena un beep doble de 432 Hz
5. En caso de error suena un beep triple descendente

Todos los resultados son reproducibles con SEMILLA=42.

---

## Control de cambios

| Versión | Fecha | Cambio principal |
|---------|-------|-----------------|
| v7.2 | Mar 2026 | CRISC completas, pesos aprendidos, KDDTest+21 OOD, McNemar |
| v8.0 | Abr 2026 | MoG real aprobada por Prof. López Rubio |
| v8.5 | Abr 2026 | ICC como regularización — pesos alineados con CRISC |
| v8.6 | Abr 2026 | Modelo híbrido CategoricalNB+GaussianNB + 7 alphas |
| v8.7 | Abr 2026 | Verificación de categorías nuevas en val/test/OOD |
| **v8.8** | **Abr 2026** | **Conclusiones completas (puntos 6 y 7) + pregunta reflexiva actualizada** |

---

## Repositorios relacionados

| Código | Descripción | Enlace |
|--------|-------------|--------|
| EJD-UMA-001 | Fed-TRUST: Random Forest Federado con Coeficiente de Veracidad Vi | [RF_Federado_Ejercicio_Doctoral_UMA_v8](https://github.com/eoherrera/RF_Federado_Ejercicio_Doctoral_UMA_v8) |
| EJD-UMA-002 | Tree Edit Distance + MDS para comparación de estructuras | [TED_MDS_Ejercicio_Doctoral_UMA](https://github.com/eoherrera/TED_MDS_Ejercicio_Doctoral_UMA) |
| EJD-UMA-003 | Este ejercicio | Repositorio actual |
