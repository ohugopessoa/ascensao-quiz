# Plano de Ação — Recalibração do Classificador do Quiz Diagnóstico

**Data:** 27 de abril de 2026
**Versão alvo:** v40 (após esta release)
**Prioridade:** 🔴 Crítica — bug afeta ~40% dos usuários em produção
**Tempo estimado:** 1-2 horas (incluindo testes)

---

## 1. Contexto do problema

A análise dos **4.641 quizzes reais** coletados entre 23-26/abril revelou um bug grave de calibração no classificador de perfis. A versão atual em produção está concentrando **39,4% de toda a base** num único perfil ("A22 — Sobrevivente de Relação Narcisista"), quando o esperado em uma segmentação saudável seria 5-10%.

### Evidências empíricas que confirmam o bug

| Métrica | Valor observado | Esperado |
|---|---|---|
| % da base classificada como A22 | **39,4%** (1.827 pessoas) | 5-10% |
| % de A22 que marcou superfície relacional como dor | **21,2%** | ≥75% |
| % de A22 que reportou evento de violência/abuso adulto | **0%** | ≥30% |
| % de A22 com q_abuso="cedo" apenas (sinal mais fraco) | **62,3%** | irrelevante |
| % da base sem perfil secundário atribuído | **80,8%** | 30-50% |

### Diagnóstico causal

**O gate de A22 dá 17 pontos absolutos** quando o usuário marca qualquer das 3 opções de `Q_RELACAO_ABUSIVA` em "Alguma das frases abaixo descreve uma relação que você vive ou viveu?". 

O problema: **62% dos A22 marcaram apenas "Eu sempre cedo pra evitar conflito com essa pessoa"** — uma frase universal que descreve QUALQUER relação difícil, não especificamente abuso narcisista. Esse gate sozinho destrói qualquer competição de outros perfis.

**Resultado prático**: 1.439 pessoas (78,8% dos A22) recebem um diagnóstico de "Sobrevivente de Relação Narcisista" mesmo quando declararam dor PRIMÁRIA financeira/emocional/carreira — não relacional. O diagnóstico que entregamos não bate com a queixa que a pessoa declarou.

---

## 2. As duas mudanças que precisam ser feitas

São duas alterações no arquivo principal do quiz (no nosso caso é um único `index.html`; no seu pode estar em outro lugar — talvez em `classifier.js`, `scoring.ts`, ou similar). Procure o módulo que **calcula os scores dos perfis** a partir das respostas.

### Mudança 1 — Recalibrar gate do perfil A22

**Localizar:** a linha que atribui o score do perfil **A22** (Sobrevivente de Relação Narcisista). No nosso código está dentro da função `calcularScoresPerfis()`.

**Como está hoje** (gate único de 17 pontos absolutos):

```javascript
// q_abuso é a resposta de "Alguma das frases abaixo descreve uma relação..."
// idadeNum é a idade numérica derivada da faixa etária

s.A22 = (["cedo","duvido","menor"].includes(q_abuso) && idadeNum >= 30) ? 17 : 0;
if (s.A22 === 0 && eventos.includes("violencia") && idadeNum >= 30) s.A22 = 15;
```

**Como deve ficar** (peso 8 base + bônus condicionais que exigem confirmação cruzada):

```javascript
// v40: gate A22 calibrado por evidência empírica.
// Antes: 17 absoluto. Agora: peso base baixo + bônus que exigem confirmação cruzada.
s.A22 = 0;
if (q_abuso === "cedo" && idadeNum >= 30) s.A22 += 8;     // sinal mais leve
if (q_abuso === "duvido" && idadeNum >= 30) s.A22 += 11;  // gaslighting clássico
if (q_abuso === "menor" && idadeNum >= 30) s.A22 += 11;   // diminuição sistemática

// Bônus 1: confirmação primária — usuário marcou superfície relacional como dor
if (s.A22 > 0 && superficie === "relacional") s.A22 += 5;

// Bônus 2: confirmação secundária — superfície relacional veio em segundo lugar
if (s.A22 > 0 && superficie_secundaria === "relacional") s.A22 += 3;

// Bônus 3: medo de prosperar (P25 — "Quando algo bom acontece, primeira reação é medo")
if (s.A22 > 0 && (answers.P25 || 0) >= 4) s.A22 += 2;

// Bônus 4: recuo na conexão (P27 — "Quero me conectar mas na hora recuo")
if (s.A22 > 0 && (answers.P27 || 0) >= 4) s.A22 += 2;

// Caminho alternativo: evento de violência declarado em P23_EVENTO
if (eventos.includes("violencia") && idadeNum >= 30) s.A22 += 14;
```

**Lógica da mudança:**

- `q_abuso="cedo"` (sinal universal) sozinho dá só 8 pontos — não consegue dominar outros perfis
- Pra chegar aos 17 pontos antigos, agora precisa de **3-4 sinais corroborantes** (idade + abuso + relacional + medo prosperar/recuo conexão)
- Quem declara evento de violência adulta tem caminho próprio (+14)
- A regra `q_abuso==="cedo"` que retornava 17 não retorna mais — o sistema valida coerência antes

### Mudança 2 — Afrouxar atribuição de perfil secundário

**Localizar:** a função que classifica e retorna `{ primario, secundario, confianca, ... }`. No nosso código é `classificarPerfil()`.

**Como está hoje** (secundário só atribuído quando confiança é "composta"):

```javascript
// margemScore = sorted[0][1] - sorted[1][1]
let confianca;
if (margemScore >= 5) confianca = "alta";
else if (margemScore >= 2) confianca = "media";
else confianca = "composta";

// Secundário só se margem for pequena
const secundario = (confianca === "composta" && secundarioCandidato !== primario) 
  ? secundarioCandidato 
  : null;
```

**Como deve ficar** (secundário sempre que o segundo colocado tem score >= 55% do top):

```javascript
// confianca continua sendo calculada igual pra outros usos
let confianca;
if (margemScore >= 5) confianca = "alta";
else if (margemScore >= 2) confianca = "media";
else confianca = "composta";

// v40: secundário atribuído sempre que score >= 55% do top
// Antes: só "composta" → 80,8% sem secundário (perde-se nuance)
// Agora: ~50-70% com secundário (psicometricamente saudável)
const topScore = sorted[0][1];
const secScore = sorted[1] ? sorted[1][1] : 0;
const secundario = (secScore > 0 
                    && secScore >= 0.55 * topScore 
                    && secundarioCandidato !== primario) 
  ? secundarioCandidato 
  : null;
```

**Lógica da mudança:** modelos psicométricos saudáveis atribuem perfil secundário em 50-70% dos casos quando o respondente apresenta sinais legítimos de mais de um perfil. A versão atual restringe demais essa atribuição, perdendo nuance pra ~80% da base.

---

## 3. Como validar que a mudança funcionou

### Validação local (antes de subir pra produção)

Se você tem um conjunto de respostas históricas (ex: dump do banco), rode o classificador novo sobre elas e meça:

| Métrica | Antes | Esperado depois |
|---|---|---|
| % A22 | 39,4% | **3-10%** |
| % com perfil secundário | 19,2% | **50-70%** |
| % A22 com superfície relacional | 21,2% | **≥75%** |
| Top 5 perfis | 1 perfil > 39% | **Todos entre 5-22%** |

**Aviso de regressão**: se algum perfil DIFERENTE de A22 ficar acima de 25%, é sinal de outro gate overpoderoso (provavelmente A20). Reportar pra ajuste subsequente.

### Distribuição esperada após o fix (validada empiricamente sobre 2.488 respostas reais)

```
A20 (Curador Ferido):           21,3%
A19 (Empreendedora Cristã):     11,7%
A13 (Imigrante Nostálgico):      8,6%
A01 (Empreendedor com Teto):     7,9%
A16 (Casal com filhos + trauma): 7,6%
A12 (Acumulador Teórico):        7,0%
A24 (Escassez Real Extrema):     5,3%
A07 (Fiel Dividido):             4,4%
A04 (Herdeiro do Trauma):        4,1%
A22 (Sobrevivente Narcisista):   3,9%  ← caiu de 39,4%
A21 (Viúvo em Reorganização):    3,4%
... outros 12 perfis distribuídos
```

### Combinações primário+secundário esperadas (top 10)

```
A20 + A19  (Curador Ferido + Cristã)         — 89 ocorrências
A19 + A07  (Cristã + Fiel Dividido)          — 66
A20 + A16  (Curador + Casal trauma)          — 57
A19 + A01  (Cristã + Empreendedor Teto)      — 54
A13 + A20  (Imigrante + Curador)             — 51
A16 + A04  (Casal + Herdeiro Trauma)         — 50
A19 + A16  (Cristã + Casal)                  — 50
A01 + A12  (Empreendedor + Acumulador)       — 48
A12 + A01  (Acumulador + Empreendedor)       — 46
A20 + A01  (Curador + Empreendedor)          — 45
```

Essas combinações são narrativamente plausíveis e indicam que o modelo passou a capturar nuance real.

---

## 4. Riscos e como mitigar

### Risco 1: usuários que JÁ receberam diagnóstico A22 podem reabrir o quiz
**Mitigação:** essa mudança afeta apenas classificações futuras. Resultados já entregues estão salvos com o perfil antigo no banco. Se houver fluxo de re-acesso ao resultado, ele continua mostrando o A22 da época.

### Risco 2: A20 (Curador Ferido) pode virar o novo overpoderoso (21% projetado)
**Mitigação:** A20 também tem gate alto (16 pontos absolutos para `Q_PROFISSIONAL_CURA != "nao"`). Se ficar incômodo, próximo passo é aplicar a mesma lógica de bônus condicionais ao A20 (sub-gate por superfície + camada). Mas 21% é tolerável vs 39% — não é bloqueador.

### Risco 3: Pessoas com abuso real podem cair em outro perfil
**Mitigação:** O caminho de `eventos.includes("violencia")` continua dando +14 pontos diretamente. Quem declara violência continua sendo classificado em A22 com confiança alta. O que mudou foi: quem só marcou "cedo pra evitar conflito" sem mais nada não cai mais em A22 por default.

---

## 5. Próximos passos sugeridos (não obrigatórios pra esta release)

Depois que esta calibração estiver em produção e a distribuição estabilizar, vale considerar:

1. **Calibrar A20 da mesma forma** (peso 16 absoluto também concentra demais)
2. **Aproveitar o luto recente** — 47,9% da base perdeu alguém nos últimos 3 anos, mas A21 (Viúvo em Reorganização) só captura 6,4%. Ampliar gate de A21
3. **A04 sub-explorado** — 51% respondem "desde sempre" + 28% "não sei explicar" sobre quando começou o bloqueio; A04 (Herdeiro do Trauma pré-verbal) deveria pegar mais
4. **Conectar feedback dos usuários** — os botões 👎😐👍🔥 ao final de cada bloco do output já coletam sinal de qualidade percebida. Hoje esse dado provavelmente está sendo armazenado mas não usado. Conectar ao motor pra recalibração contínua (loop de aprendizado)

---

## 6. Resumo executivo (em 1 parágrafo)

Hoje, 39,4% dos respondentes recebem um diagnóstico de "Sobrevivente de Relação Narcisista" mesmo quando 78,8% deles não declararam dor relacional como sua questão principal. O bug está em um gate de scoring que dá 17 pontos absolutos pra qualquer pessoa que admita "ceder em conflito" — comportamento universal, não específico de abuso. As duas mudanças propostas aqui (recalibrar peso do gate A22 e afrouxar atribuição de perfil secundário) levam a distribuição pra um padrão saudável (top perfil em 21%, 83% dos casos com secundário atribuído). Validei empiricamente reclassificando 2.488 das respostas reais — A22 cai de 39,4% pra 3,9% sem distorcer outros perfis. Tempo estimado de implementação: 1-2 horas.

---

**Suporte**: qualquer dúvida na implementação, pode chamar.

**Anexos disponíveis se precisar:**
- Script Node.js que reclassificou os 4.641 registros do CSV (validação empírica)
- Distribuição completa antes/depois com tabela cruzada perfil × superfície
- Lista das 22 combinações primário+secundário mais frequentes pós-fix
