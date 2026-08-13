# Previsão de Risco de Ferrugem Asiática da Soja

Projeto de portfólio que estima o risco de um talhão de soja desenvolver ferrugem asiática, cruzando dados climáticos reais com uma simulação de manejo agrícola. O foco é justamente o tipo de problema que empresas de proteção de cultivo (defensivos agrícolas, sementes) resolvem no dia a dia: identificar onde e quando agir antes que a doença se instale.

Ferrugem asiática é uma das doenças mais custosas da soja no Brasil, e seu risco está diretamente ligado a condições climáticas bem documentadas na literatura agronômica: temperatura entre 15°C e 28°C combinada com muitas horas seguidas de alta umidade favorece o fungo.

## Dados

Duas fontes combinadas, com um critério claro sobre o que é real e o que é simulado:

**Clima (real).** Cinco anos (2021 a 2025) de dados horários da estação meteorológica automática do INMET em Sorriso, Mato Grosso (código A904), um dos maiores polos de produção de soja do Brasil. Os dados foram baixados diretamente do banco de dados histórico oficial do INMET, sem intermediários. Após remover falhas de sensor, restaram cerca de 27 mil registros horários utilizáveis, cobrindo integralmente as safras 2022/23 a 2024/25.

**Talhões e manejo (simulado).** 600 talhões fictícios de soja, cada um com data de plantio real dentro da janela de semeadura (setembro a novembro), cultivar com suscetibilidade variável à ferrugem (baixa, média, alta) e indicação de aplicação preventiva de fungicida. Não existe dado público sobre incidência de doença por talhão ou uso de defensivo por produtor, por isso essa camada é simulada, mas construída sobre o clima real de cada janela específica.

## Construindo o indicador de risco climático

Cada dia do histórico foi marcado como climaticamente favorável à ferrugem quando teve pelo menos 6 horas de umidade relativa acima de 90% e temperatura média entre 15°C e 28°C. Isso é uma aproximação (o ideal seria um sensor de folha molhada, que a estação não tem), mas segue a mesma lógica usada por sistemas reais de alerta fitossanitário quando esse sensor não está disponível.

Resultado: 374 dos 1.168 dias analisados (32%) tiveram condição favorável, um número plausível para o clima de Sorriso.

## Modelagem

Cada talhão foi cruzado com o clima real da sua própria janela crítica (45 a 95 dias após o plantio, que corresponde aproximadamente aos estágios reprodutivos R1 a R5 da soja). A incidência de ferrugem foi então simulada como função de três fatores: dias climaticamente favoráveis na janela, suscetibilidade da cultivar e uso de fungicida preventivo.

**Erro e correção que valem registro.** A primeira versão do modelo usou `LabelEncoder` para transformar a suscetibilidade da cultivar (baixa/média/alta) em número, o que atribuiu valores por ordem alfabética (alta=0, baixa=1, média=2) em vez de por ordem de risco real. Isso quebrou a relação que um modelo linear precisa enxergar, e a acurácia ficou em 57%, quase equivalente a chute. Substituindo pela codificação correta (baixa=0.5, média=1.0, alta=1.6, refletindo a ordem real de risco), a acurácia subiu para 62%, com recall balanceado entre as duas classes (0.63 e 0.62). É um lembrete de que erro de codificação de variável categórica é silencioso, o modelo treina normalmente e só entrega resultado ruim, sem nenhum aviso.

## Resultado final

| Métrica | Valor |
|---|---|
| Acurácia | 0.62 |
| Recall (com ferrugem) | 0.62 |
| AUC | 0.72 |

![Matriz de confusão](matriz_confusao_ferrugem.png)

![Curva ROC](curva_roc_ferrugem.png)

Um AUC de 0.72 com apenas três variáveis é um resultado honesto: o modelo separa bem os dois grupos, mas está longe de ser perfeito, o que é esperado dado o nível de ruído introduzido de propósito na simulação.

## O que mais pesa na decisão

Os coeficientes da regressão confirmam a lógica agronômica que fundamentou a simulação:

- **Suscetibilidade da cultivar** (0.62) e **dias climaticamente favoráveis** (0.61) pesam quase igualmente a favor do risco.
- **Aplicação de fungicida preventivo** (-0.57) reduz o risco de forma consistente.

## Do modelo à ação: ranking de talhões por risco

Como no projeto de churn, a saída mais útil não é a classificação binária, é a probabilidade de cada talhão, ordenada. Dos 15 talhões de maior risco na base de teste, 12 realmente desenvolveram ferrugem, e quase todos compartilham o mesmo perfil: cultivar de alta suscetibilidade sem aplicação preventiva de fungicida. Essa é exatamente a lista que orientaria uma equipe agronômica sobre onde priorizar a aplicação preventiva primeiro. O ranking completo está em `ranking_risco_ferrugem.csv`.

## Limitações

Vale ser direta sobre até onde esse projeto vai:

- A camada de talhões e manejo é simulada. O clima é real e verificável, mas a relação entre clima, suscetibilidade e incidência de doença foi definida por mim de forma simplificada, não medida em campo.
- Apenas uma divisão de treino/teste foi usada. O resultado pode variar com outra divisão; o ideal seria validação cruzada para uma estimativa mais confiável.
- O indicador de umidade alta é um proxy para folha molhada, não a medição direta que a agronomia de precisão usa.
- Uma única estação meteorológica representa uma região inteira, na prática o clima varia dentro do próprio município.

## Próximos passos

- Incorporar mais de uma estação meteorológica por região
- Testar modelos não lineares (Random Forest, XGBoost) e comparar
- Validação cruzada para estimativas mais robustas
- Explorar dados de sensoriamento remoto (NDVI) como proxy de estresse da cultura

## Ferramentas

Python, pandas, scikit-learn, matplotlib, dados históricos oficiais do INMET, Google Colab
