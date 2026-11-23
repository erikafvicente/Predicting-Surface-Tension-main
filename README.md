# Introducción
El objetivo de este proyecto es el modelado de la tensión superficial.   
La tensión superficial (TS) es una propiedad física de los líquidos y es la fuerza que hace que la superfície de dicho líquido se comporte como una membrana elástica debido a la atracción entre moléculas. Sus unidades son mN/m, y es una propiedad que depende de la temperatura.  
Es una propiedad importante en la industria química, ya que debe controlarse para aquellos productos que deben interactuar con una superfici, dispersarse, formar una película, generar gotas o penetrar en un sustrato.  
En Lubrizol, tenemos varios productos en el sector de Surface Sciences, que necesitan control de esta propiedad. Entre ellos:  
- Dispersiones, dispersantes, resinas y modificadores de superficie para tintas digitales sobre textiles, films, papel y cerámica.
- Pinturas y coatings.
- Formulaciones de activos cosméticos.

## ¿Qué aporta este proyecto?
Esta propiedad suele medirse usando un tensiómetro y empleando distintos métodos experimentales, pero lo que tienen en común todos ellos es que el producto ya debe estar acabado para que se puedan hacer estas mediciones. Es decir, si queremos ver qué formulación cumpliría con las necesidades de tensión superficial del uso destinado, tendríamos que hacer todas las formulaciones y luego medir la propiedad. Por lo tanto, eso nos limita a probar unas pocas formulaciones que creemos que son mejores candidatos, ya que cada fabricación y medición implica un gasto cuantioso de tiempo y de dinero.  
Si tenemos un modelo de esta propiedad, podemos pasar por él cientos o miles de formulaciones, para encontrar aquellos mejores candidatos para llevarlos a fabricación y validación con el método experimental apropiado.  
Entonces, a efectos de empresa, este proyecto permitiría hacer un screening amplio de formulaciones en minutos, y solo aplicar las pruebas experimentales a aquellas más prometedoras, ayudando así a agilizar la innovación de productos reduciendo el tiempo y el coste.

# Dataset
Para este proyecto, usaremos los datos de una base propietaria, con lo cual no publicaremos los datos en este repositorio, ni tampoco el modelo resultante.  
Los campos que cogemos de la base de datos son:
- ChemID: identificador del compuesto químico.
- Name: nombre del compuesto.
- Formula
- SMILES: código SMILES del compuesto
- Structure: estructura del compuesto
- Family: clasificación de familia a la que pertenece el compuesto
- Std_State: indica el estado de la materia en la que se encuentra el compuesto.
- T: temperatura
- Surface_tension: nuestro target, la tensión superficial.
- TC: temperatura crítica. Es una variable importante de cara a crear un modelo que no incumpla leyes físicas.
- Error: incertidumbre asociada al valor guardado en la base de datos.
- Acceptance: estado de aceptación de los datos por parte del staff.
  - A = Accepted. Datos aceptados por el staff del proyecto.
  - N = Not used (but acceptable). Datos aceptables, pero con una precisión menor o igual a la de los valores aceptados. Dentro del doble de la incertidumbre o no usados debido a tendencias inconsistentes.
  - U = Unevaluated. Datos no evaluados por el staff del proyecto.
  - R = Rejected. Datos rechazados debido a que se desvían más del doble de la incertidumbre o se consideran poco fiables.

## Limpieza y transformaciones
Para simplificar el estudio, solo cogeremos aquellos valores que hayan sido verificados y aceptados por el staff (Acceptance == "A") y los que tengan incertidumbres menores del 25%. Como estamos estudiando una propiedad de los líquidos, solo cogeremos aquellos compuestos cuyo estado de la materia sea líquido (Std_state == "L").  
Al obervar los datos, vemos que tenemos más nombres únicos que códigos SMILE, cosa que no es posible ya que a cada compuesto se le asigna un SMILE. Revisando más a fondo, podemos ver que hay compuestos que, después de una revisión, se han guardado añadiendo a su nombre la coletilla del año de revisión. Eliminaremos esa coletilla y auditaremos los valores para evitar casos como el de la imagen.  
![Revisión del 1,3-Dicloro-2-propanol](images/paste-2.png)
Nos quedaremos con el valor de revisión, cuando este difiera mucho con el anterior. Nos quedaremos con ambos cuando sean muy similares.  
Los códigos SMILES contienen los elementos que componen el compuesto. Gracias a eso, podemos usarlos para detectar cuáles de ellos llevan los elementos más usados en las formulaciones de la empresa: Carbono, Oxígeno, Sulfuro, Fósforo y Nitrógeno.  
Con éstas limpiezas, el dataset que empezó con 52225 registros, pasa a 25649.  
El último cambio que hay que hacer tiene que ver con las unidades de la TS. En la base de datos las unidades son N/m, y como he dicho al inicio, las unidades típicas son mN/m. Así que, realizamos una simple conversión de unidades.  
![Distribución de los datos](images/paste-3.png)
  
# Creación de modelos
A modo de exploración, debemos empezar con modelos más sencillos y que sigan los métodos que típicamente usan los técnicos e ingenieros.

## 1. Uso de la ecuación de Yaws

La ecuación de Yaws describe la tensión superficial como: $\sigma = A(1-T_r)^n$ donde $T_r = \frac{T}{T_C}$
Usando esta ecuación, podemos hacer una regresión lineal:
$$
\log\sigma = \log A+n\log(1-T_r)
$$
![Regresión lineal usando los datos de la ecuación de Yaws](images/paste-4.png)

### ¿Qué problema hay con este modelo?
Como usa la Temperatura crítica, nos obligaría a tenerla para todos los nuevos compuestos.

## 2. Generando variables químicas
Como el uso de las variables actuales no nos permite crear un modelo sin data leakage, usamos los SMILES para obtener:
- Contribución de grupos: es un método donde cogemos una molécula, la descomponemos en grupos funcionales, y miramos cómo la suma de dichos grupos contribuye a la propiedad X de dicha molécula. ![](images/paste-5.png) Con esto creamos 29 predictores.
- Descriptores químicos. Escogemos los que tienen mayor relevancia del paquete RDkit:
  - Descriptores de polaridad e interacciones intermoleculares.
    - TPSA (Topological Polar Surface Area)
    - LogP: mide el balance entre el comportamiento hidrofóbico e hidrofílico. Nos da información de la polaridad y la hidrofobicidad.
  - Cargas parciales. Para poder mirar las cargas parciales, debemos calcular las cargas de cada átomo de la molécula, calculando las cargas parciales de Gasteiger para cada átomo. Así, obtendremos los valores que nos darán las cargas máximas y mínimas de cada compuesto.
  - E-State: el estado electrotopológico combina la información topologica molecular y el entorno electrónico.
  - Descriptores de tamaño y dispersión: peso molecular y contaje de átomos pesados.
  - Descriptores de enlaces de hidrógeno: importantes ya que este tipo de enlaces influencian las fuerzas intermoleculares cohesivas en un líquido.
  - LabuteASA: nos da una aproximación de la superficie molecular describiendo lo grande que es la molécula y el tipo de superficie que tiene.
  - SlogP_VSA: nos divide el área de la superficie molecular según la contribución de la superficie de van der Waals al valor de LogP.
  - PEOE_VSA: se refiere a la información que aportan las cargas parciales. Lo que nos mira es cómo los rangos de diferencias entre cargas parciales de los átomos, afecta a la superficie molecular.
  - SMR_VSA: nos dan información de la polarizabilidad y de las características de volumen y dispersión de la molécula.
  - Descriptores estructurales: proporcionan características estructurales básicas.
    - Contaje de anillos, que nos da información de la ciclicidad.
    - Anillos aromáticos, que captura todas aquellas estructuras aromáticas.
    - Enlaces rotables, que nos proporciona una medida de la flexibilidad de la molécula.
- Morgan Fingertips: es una codificación de fragmentos estructurales de las moléculas. Hay 2048 fragmentos. Si el valor para un fingertip es 1, entonces es que este fragmento está presente en la molécula.

## 3. Separación de los datos
Como tenemos un dataset químico y dependiente de la temperatura, tenemos muchas filas que corresponden al mismo compuesto pero a diferentes temperaturas, debemos tener en cuenta dos cosas para dividir los datos:
1. No debe existir la misma molécula tanto en los datos de entreno como en los de test.
2. Los diferentes grupos químicos deben estar representados de manera balanceada entre los distintos dataset, así nos aseguramos de no estar entrenando para una familia química en concreto y probándolo en familias químicas que nunca ha visto.

Lo primero se puede conseguir con un simple GroupShuffleSplit (de scikit-learn), agrupando por el ChemID. Pero, con ese método no resolveríamos lo segundo. Para ese segundo, probamos dos estrategias:
- Separar los datos agrupando por la columna **Family** con StratifiedGroupKFold y n_split=3 (para coger 2 partes como train y 1 como test). 
- Separar siguiendo la estrategia del paper de Aouichaoui et al. (2025).
  1. Usamos los Fingertips de Morgan para comparar si dos moléculas son similares. La forma de compararlos es usando la similaridad de Tanimoto.
  2. Con la similaridad, podemos agrupar las moléculas en clústers usando el algoritmo de Butina.
  3. Agrupamos por esos clústers usando StratifiedGroupKFold.

## 4. Resumen de resultados

> Metrics: **MSE · RMSE · MAE · sd · NRMSE(%) · R²**  
> 🟢 Best R² per table · 🔴 Worst R² per table

---

## Descriptors

### BUTINA

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| MLP_PyTorch | 11.172621 | 3.342547 | 1.953643 | 3.342645 | 15.170166 | **0.9115** 🟢 |
| XGBoost | 12.132120 | 3.483119 | 1.971473 | 3.477951 | 15.808154 | 0.9039 |
| GradientBoosting | 12.422363 | 3.524537 | 1.960977 | 3.516391 | 15.996130 | 0.9016 |
| SVR | 13.123600 | 3.622651 | 2.012031 | 3.614743 | 16.441419 | 0.8960 |
| MLP | 13.593887 | 3.686989 | 2.004373 | 3.681039 | 16.733417 | 0.8923 |
| RandomForest | 14.653414 | 3.827978 | 2.170210 | 3.817407 | 17.373296 | 0.8839 |
| GaussianProcess | 19.079359 | 4.367993 | 2.442907 | 4.354140 | 19.824155 | 0.8489 |
| DecisionTree | 24.764026 | 4.976347 | 3.087593 | 4.967554 | 22.585174 | 0.8038 |
| Ridge | 28.529893 | 5.341338 | 3.687888 | 5.341507 | 24.241689 | 0.7740 |
| LinearRegression | 28.750779 | 5.361975 | 3.695160 | 5.362161 | 24.335351 | 0.7723 🔴 |

### FAMILY

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| XGBoost | 12.673114 | 3.559932 | 2.032720 | 3.549482 | 16.272529 | **0.8918** 🟢 |
| RandomForest | 13.155799 | 3.627092 | 2.159858 | 3.623848 | 16.579522 | 0.8877 |
| MLP_PyTorch | 14.108286 | 3.756100 | 2.111376 | 3.730603 | 17.169218 | 0.8796 |
| GradientBoosting | 14.562817 | 3.816126 | 2.176133 | 3.801668 | 17.443599 | 0.8757 |
| MLP | 15.323083 | 3.914471 | 2.188752 | 3.912758 | 17.893136 | 0.8692 |
| GaussianProcess | 15.982873 | 3.997859 | 2.359143 | 3.995966 | 18.274303 | 0.8636 |
| SVR | 16.394766 | 4.049045 | 2.144289 | 4.046999 | 18.508277 | 0.8601 |
| DecisionTree | 23.626783 | 4.860739 | 3.144635 | 4.858931 | 22.218549 | 0.7983 |
| Ridge | 32.552508 | 5.705481 | 3.751229 | 5.705417 | 26.079882 | 0.7221 |
| LinearRegression | 33.195143 | 5.761523 | 3.744843 | 5.761313 | 26.336052 | 0.7166 🔴 |

---

## Group Contribution

### BUTINA

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| MLP_PyTorch | 13.226527 | 3.636829 | 2.086570 | 3.626986 | 16.505767 | **0.8952** 🟢 |
| MLP | 14.814753 | 3.848994 | 2.167495 | 3.840811 | 17.468677 | 0.8827 |
| XGBoost | 15.092460 | 3.884902 | 2.237017 | 3.876862 | 17.631645 | 0.8805 |
| RandomForest | 16.050737 | 4.006337 | 2.384896 | 3.986430 | 18.182781 | 0.8729 |
| GradientBoosting | 16.576912 | 4.071475 | 2.365504 | 4.065056 | 18.478411 | 0.8687 |
| DecisionTree | 25.795908 | 5.078967 | 3.137712 | 5.074884 | 23.050918 | 0.7957 |
| Ridge | 26.600284 | 5.157546 | 3.637393 | 5.156574 | 23.407549 | 0.7893 |
| LinearRegression | 26.663289 | 5.163651 | 3.631778 | 5.163952 | 23.435254 | 0.7888 |
| SVR | 27.757404 | 5.268530 | 3.207996 | 5.268720 | 23.911247 | 0.7801 |
| GaussianProcess | 29.627999 | 5.443161 | 3.060249 | 5.420297 | 24.703812 | 0.7653 🔴 |

### FAMILY

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| MLP_PyTorch | 11.498182 | 3.390897 | 2.157482 | 3.371110 | 15.499867 | **0.9019** 🟢 |
| MLP | 13.384342 | 3.658462 | 2.181232 | 3.647587 | 16.722912 | 0.8857 |
| XGBoost | 13.737128 | 3.706363 | 2.253807 | 3.702378 | 16.941871 | 0.8827 |
| RandomForest | 14.668975 | 3.830010 | 2.420258 | 3.828464 | 17.507062 | 0.8748 |
| GradientBoosting | 14.738294 | 3.839049 | 2.316814 | 3.833018 | 17.548379 | 0.8742 |
| LinearRegression | 20.465634 | 4.523896 | 3.260465 | 4.521152 | 20.678831 | 0.8253 |
| Ridge | 20.465634 | 4.523896 | 3.260465 | 4.521152 | 20.678832 | 0.8253 |
| DecisionTree | 22.120460 | 4.703239 | 3.094653 | 4.703369 | 21.498614 | 0.8112 |
| GaussianProcess | 24.782497 | 4.978202 | 2.955500 | 4.965265 | 22.755476 | 0.7885 |
| SVR | 27.548477 | 5.248664 | 3.210159 | 5.246005 | 23.991764 | 0.7648 🔴 |

---

## Morgan Fingerprints

### BUTINA

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| MLP | 19.384876 | 4.402826 | 2.818431 | 4.381843 | 19.982247 | **0.8465** 🟢 |
| GradientBoosting | 19.916526 | 4.462794 | 2.883908 | 4.456376 | 20.254410 | 0.8422 |
| XGBoost | 20.210580 | 4.495618 | 2.902629 | 4.493778 | 20.403383 | 0.8399 |
| MLP_PyTorch | 20.713358 | 4.551193 | 2.977344 | 4.504174 | 20.655612 | 0.8359 |
| GaussianProcess | 22.678285 | 4.762172 | 3.207619 | 4.739786 | 21.613142 | 0.8204 |
| RandomForest | 25.699677 | 5.069485 | 3.603383 | 5.062489 | 23.007882 | 0.7964 |
| DecisionTree | 29.704283 | 5.450164 | 3.649443 | 5.446199 | 24.735594 | 0.7647 |
| Ridge | 30.405665 | 5.514133 | 3.909181 | 5.503500 | 25.025921 | 0.7592 |
| LinearRegression | 61.165389 | 7.820830 | 5.580721 | 7.811435 | 35.494877 | 0.5155 🔴 |

### FAMILY

| Algorithm | MSE | RMSE | MAE | sd | NRMSE (%) | R² |
|---|---:|---:|---:|---:|---:|---:|
| MLP | 16.497235 | 4.061679 | 2.709585 | 4.050823 | 18.566027 | **0.8592** 🟢 |
| XGBoost | 17.098327 | 4.135012 | 2.835504 | 4.132859 | 18.901236 | 0.8540 |
| GradientBoosting | 17.518400 | 4.185499 | 2.765037 | 4.173993 | 19.132010 | 0.8505 |
| MLP_PyTorch | 19.836301 | 4.453796 | 3.045490 | 4.452758 | 20.358404 | 0.8307 |
| GaussianProcess | 21.715518 | 4.659991 | 3.228836 | 4.659582 | 21.300926 | 0.8146 |
| RandomForest | 22.793254 | 4.774228 | 3.415461 | 4.774309 | 21.823106 | 0.8054 |
| DecisionTree | 26.931426 | 5.189550 | 3.528235 | 5.186095 | 23.721550 | 0.7701 |
| Ridge | 27.787990 | 5.271432 | 3.806959 | 5.271323 | 24.095834 | 0.7628 |
| LinearRegression | 87.125437 | 9.334101 | 6.478444 | 9.334545 | 42.666388 | 0.2563 🔴 |

Estos resultados nos demuestran que la separación usando la clasificación de Family de la base de datos es mejor. Pero antes de ir a mirar el por qué nos están produciendo NRMSE(%) tan altos, vamos a comprobar si  las predicciones hechas cumplen una ley física que dice que: la tensión superficial en la temperatura crítica debe ir a 0.
Esta comprobación nos permite ver que esa ley es violada por todos los modelos en varias ocasiones.

## 5. Physics-Informed ML
Vemos que si solo aplicamos los algoritmos de ML, no coseguiremos una consistenciafísica. Necesitamos añadir constraints físicos.
