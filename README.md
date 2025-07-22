# **PREDICCIÓN TENDENDIA AL VOTO - DISTRITO 11, PUEBLA**

## **RESUMEN**

Este proyecto implementa un **sistema de análisis electoral** que combina **datos demográficos del INEGI** con **datos electorales del INE** para el Distrito Electoral Federal 11 de Puebla. El sistema incluye tres componentes principales: **consolidación de datos demográficos**, **mapeo geográfico electoral-demográfico**, y **modelado predictivo electoral**.

---

## **ARQUITECTURA DEL SISTEMA**

### **FASE 1: CONSOLIDACIÓN DE DATOS DEMOGRÁFICOS** 
*(Excel_to_CSV_Data_Refactor.ipynb)*

#### **1.1 Contexto y Problemática**
- **Desafío Principal:** No existe información pública oficial que relacione directamente datasets electorales (INE) con datasets demográficos (INEGI)
- **Solución:** Clasificación manual de correspondencia geográfica entre secciones electorales y AGEBs utilizando mapas oficiales obtenidos de los portales del INE y del INEGI:
https://cartografia.ine.mx/sige8/productosCartograficos/planos-mapas 
https://gaia.inegi.org.mx/scince2020/
https://siceen21.ine.mx/home
- **Desafío Técnico:** INEGI proporciona 123 datasets separados (uno por AGEB) que requieren consolidación

#### **1.2 Proceso de Consolidación**
```python
def procesar_archivos_excel():
    """Procesa 123 archivos Excel individuales"""
    for archivo in os.listdir('DatosPorAGEB'):
        if archivo.endswith('.xlsx'):
            workbook = load_workbook(ruta_archivo, data_only=True)
            indicadores = extraer_indicadores(workbook)
            secciones = extraer_secciones(workbook)
            # Consolidar en dataset único
```

**Resultados:**
- **123 AGEBs** procesados del Distrito Electoral 11
- **167 secciones electorales** identificadas
- **355 variables demográficas** optimizadas (de 683 iniciales)
- **Dataset consolidado:** `AGEB_Consolidado_Completo.csv`

#### **1.3 Optimización de Datos**
- **Normalización:** Eliminación de acentos, caracteres especiales
- **Limpieza:** Conversión a formato numérico, eliminación de duplicados
- **Imputación inteligente:** Ceros estructurales para variables específicas, mediana para generales
- **Reducción dimensional:** De 683 a 355 variables relevantes

---

### **FASE 2: MAPEO GEOGRÁFICO ELECTORAL-DEMOGRÁFICO**
*(Mapeo_Seccion_AGEB.ipynb)*

#### **2.1 Objetivo**
Crear datasets integrados que combinen:
- **Datos electorales** por sección (INE)
- **Datos demográficos** por AGEB (INEGI)
- **Mapeo geográfico** entre secciones y AGEBs

#### **2.2 Metodología de Mapeo**
```python
def crear_dataset_por_tipo_eleccion(tipo_eleccion, df_electoral, df_ageb, mapeo_seccion_ageb):
    """Crea dataset integrado para cada tipo de elección"""
    for seccion in df_electoral['SECCION']:
        if seccion in mapeo_seccion_ageb:
            ageb = mapeo_seccion_ageb[seccion]
            # Combinar datos electorales con demográficos
```

**Tipos de Elección Procesados:**
- **Presidencia** (2018, 2024)
- **Senaduría** (2018, 2024) 
- **Diputación Federal** (2018, 2021, 2024)

#### **2.3 Datasets Generados**
- `Dataset_Secciones_PRESIDENCIA.csv`
- `Dataset_Secciones_SENADURIA_MR.csv`
- `Dataset_Secciones_DIPUTACION_FEDERAL_MR.csv`
- Versiones TEST para validación

---

### **FASE 3: MODELADO PREDICTIVO ELECTORAL**
*(Modelo_Electoral_Simplificado.ipynb)*

#### **3.1 Preparación de Datos**
```python
def preparar_datos_simplificado(df):
    """Prepara datos para modelado"""
    # Identificar partidos electorales
    columnas_electorales = [col for col in df.columns if col.startswith('ELECTORAL_')]
    
    # Crear targets (porcentajes)
    for partido in partidos:
        nombre_target = partido.replace('ELECTORAL_', '') + '_PCT'
        df[nombre_target] = (df[partido] / df['TOTAL_VOTOS_VALIDOS'] * 100)
```

**Transformaciones:**
- **Votos absolutos → Porcentajes relativos**
- **Detección automática** de partidos políticos
- **Normalización** por total de votos válidos

#### **3.2 Selección Inteligente de Variables**
```python
def seleccionar_variables_clave(df, variables_demograficas, targets, n_variables=6):
    """Método híbrido de selección"""
    # Método 1: Correlación promedio con targets
    correlaciones = {}
    for var in X.columns:
        corrs = []
        for target in targets[:5]:  # Solo partidos principales
            corr = abs(df[var].corr(df[target]))
            correlaciones[var] = np.mean(corrs)
    
    # Método 2: Importancia Random Forest
    rf_scores = {}
    for target in targets[:3]:
        rf = RandomForestRegressor(n_estimators=50, random_state=42, max_depth=3)
        rf.fit(X_scaled, y)
        rf_scores[var] = rf.feature_importances_[i]
    
    # Combinar scores (60% correlación + 40% RF)
    scores_finales[var] = (score_corr * 0.6) + (score_rf * 0.4)
```

**Estrategia:**
- **Filtrado por varianza:** Elimina variables con poca variabilidad
- **Correlación promedio:** Captura relaciones lineales directas
- **Random Forest:** Captura relaciones no-lineales e interacciones
- **Combinación ponderada:** Balance entre interpretabilidad y poder predictivo

**Resultado:** Top 6 variables más importantes de 355 disponibles

#### **3.3 Arquitectura del Modelo**
```python
modelos = {
    'Ridge': Ridge(alpha=5.0, random_state=42),
    'Lasso': Lasso(alpha=0.1, random_state=42),
    'ElasticNet': ElasticNet(alpha=0.1, l1_ratio=0.5, random_state=42)
}
```

**Características:**
- **Enfoque Multi-Target:** n modelos independientes (uno por partido)
- **Ensemble de Modelos Lineales:** Ridge, Lasso, ElasticNet
- **Selección Automática:** Mejor modelo por partido mediante validación cruzada
- **Regularización:** Previene sobreajuste

#### **3.4 Validación y Evaluación**
```python
cv_method = KFold(n_splits=5, shuffle=True, random_state=42)

for nombre_modelo, modelo in modelos.items():
    cv_scores = cross_val_score(modelo, X_scaled, y, cv=cv_method, scoring='r2')
    cv_mean = cv_scores.mean()
    
    if cv_mean > mejor_cv:
        mejor_cv = cv_mean
        mejor_modelo = modelo
```

**Métricas:**
- **R²:** Coeficiente de determinación
- **MAE:** Error absoluto medio
- **CV:** Validación cruzada (K=5)
- **Diagnóstico de sobreajuste:** Comparación R² vs CV

---

// ... existing code ...

## **LIMITACIONES Y CONSIDERACIONES**

### **1. Limitaciones en el Procesamiento Electoral:**
- **Exclusión de votos nulos:** El modelo solo considera votos válidos, ignorando votos nulos que pueden representar protesta o descontento político
- **Lista nominal no considerada:** No se incluye la lista nominal como variable, perdiendo información sobre participación potencial y abstención
- **Simplificación excesiva:** La conversión a porcentajes relativos puede ocultar patrones importantes en la participación absoluta
- **Falta de contexto temporal:** No se consideran tendencias históricas o cambios en el comportamiento electoral

### **2. Limitaciones en Variables Demográficas:**
- **Representatividad limitada:** Las variables demográficas difícilmente son representativas de la tendencia al voto, ya que el comportamiento electoral está influenciado por múltiples factores no demográficos
- **Falta de variables políticas:** No se incluyen variables como afiliación partidista, historial de votación, o preferencias políticas previas
- **Variables estáticas:** Los datos demográficos son relativamente estáticos y no capturan cambios rápidos en preferencias políticas
- **Falta de variables contextuales:** No se consideran factores como campañas políticas, eventos nacionales, o coyunturas específicas

### **3. Dependencia de Datos:**
- **Calidad:** Depende de la calidad de datos demográficos y electorales
- **Actualización:** Requiere datos recientes para precisión
- **Cobertura:** Limitado a variables disponibles

### **4. Contexto Electoral:**
- **Cambios políticos:** Modelos pueden quedar obsoletos
- **Campañas:** No considera eventos políticos recientes
- **Participación:** Asume patrones de participación similares

### **5. Interpretación Causal:**
- **Correlación ≠ Causalidad:** Variables demográficas no explican todo
- **Factores omitidos:** No considera todos los factores electorales
- **Interacciones complejas:** Relaciones políticas no capturadas

### **6. Limitaciones Geográficas:**
- **Mapeo manual:** Basado en interpretación de mapas oficiales del INE e INEGI
- **Asignación simplificada:** Se descartaron de forma tosca las secciones que quedaban entre varios AGEBs para un manejo simple de la base de datos
- **Precisión limitada:** Las asignaciones de datos demográficos pueden no ser muy precisas debido al mapeo manual "a ojo"
- **Proceso subjetivo:** El mapeo fue realizado manualmente agregando las secciones contenidas a cada AGEB después de descargar los datos, introduciendo potenciales errores de interpretación
- **Falta de validación geográfica:** No se cuenta con validación técnica o automatizada del mapeo realizado
- **Actualizaciones:** Requiere revisión ante cambios geográficos o redistritación


### **7. Limitaciones Metodológicas:**
- **Modelos lineales:** Pueden no capturar relaciones complejas no lineales en el comportamiento electoral
- **Supuestos simplificadores:** Asume que las relaciones demográficas-electorales son consistentes en el tiempo y espacio
- **Falta de validación externa:** No se valida con datos de otras regiones o períodos electorales
- **Sobreajuste potencial:** Aunque se monitorea, el riesgo de sobreajuste persiste con pocas observaciones

// ... existing code ...

---
## **APÉNDICE TÉCNICO**

### **Librerías Utilizadas:**
- **pandas:** Manipulación de datos
- **numpy:** Operaciones numéricas
- **scikit-learn:** Machine learning
- **matplotlib/seaborn:** Visualización
- **joblib:** Persistencia de modelos
- **openpyxl:** Procesamiento de archivos Excel

### **Parámetros Clave:**
- **Variables:** 6 de 355 disponibles
- **Validación:** K-Fold (K=5)
- **Regularización:** Ridge/Lasso/ElasticNet
- **Random Forest:** 50 árboles, profundidad 3

### **Archivos de Salida:**
- **AGEB_Consolidado_Completo.csv:** Dataset demográfico consolidado
- **Dataset_Secciones_*.csv:** Datasets integrados por tipo de elección
- **modelo_electoral_Tendencia_Al_Voto.pkl:** Modelo entrenado
- **resumen_modelo.txt:** Documentación del modelo
- **Visualizaciones:** Gráficas de análisis y comparación

### **Estructura de Datos:**
- **123 AGEBs** del Distrito Electoral 11
- **167 secciones electorales** identificadas
- **355 variables demográficas** optimizadas
- **Múltiples elecciones:** Presidencia, Senaduría, Diputación Federal

---

*Documento técnico del Sistema Integral de Análisis Electoral - Distrito 11, Puebla*
