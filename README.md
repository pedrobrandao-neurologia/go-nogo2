# Tarefa Go–NoGo (v1.3)

Uma implementação web **estática** e **reprodutível** da tarefa Go–NoGo, com **temporização de alta precisão**, **randomização com semente**, **jitter de intervalos**, cálculo de **d′** e **β** com correção log-linear e **exportação CSV** (detalhada e resumida). Projetado para coleta **em larga escala** (GitHub Pages) em desktop e dispositivos móveis.

> **Arquivo único**: `index.html`
> **Demo local**: abra `index.html` no navegador
> **Produção**: publique via **GitHub Pages** (instruções abaixo)

---

## 1) Visão geral

* **Paradigma**: responder apenas quando o círculo aparece **colorido** (*Go*); **não responder** quando estiver **preto** (*NoGo*).
* **Resposta**: Clique no círculo **ou** pressione **Barra de Espaço**.
* **Modos**:
  – *Reduzido*: 24 Go / 12 NoGo
  – *Completo*: 48 Go / 24 NoGo
* **Precisão temporal**: `performance.now()`; janela de resposta controlada e debouncing por ensaio.
* **Randomização reprodutível**: semente (**seed**) derivada do ID do participante (PRNG `mulberry32`).
* **Jitter**: pré-fixação e ISI com variação pseudoaleatória.
* **Métricas**: acurácia Go, erros (omissão/comissão), RT médio e desvio-padrão (apenas acertos Go), **d′** e **β** (ajuste log-linear).
* **Exportação**: CSV **detalhado** (linha por ensaio) e **resumido** (métricas agregadas) com metadados de sessão.
* **Coleta remota (opcional)**: envio em JSON via `POST_ENDPOINT` (desligado por padrão).

---

## 2) Como executar

### 2.1 Rodar localmente

1. Baixe/clone o repositório.
2. Abra `index.html` no navegador (Chrome/Edge/Firefox/Safari atualizados).

> Dica: para evitar restrições locais de arquivo em alguns navegadores, sirva o diretório com um servidor simples (ex.: `python -m http.server 8000`).

### 2.2 Publicar no GitHub Pages

1. Crie um repositório e adicione `index.html` na raiz.
2. Em **Settings → Pages**, escolha **Deploy from a branch**, selecione **main** e **/ (root)**.
3. Aguarde a publicação e acesse a URL fornecida.

---

## 3) Uso (participante)

1. Digite seu **Identificador** (ex.: `BR-001-A`).
   – O identificador é usado para a **semente** da randomização.
2. Escolha **Teste reduzido** ou **Teste completo**.
3. Clique em **Iniciar**.
   – **Resposta válida**: clique ou **Barra de Espaço** apenas quando o círculo estiver **colorido** (*Go*).
   – **Não responda** quando estiver **preto** (*NoGo*).
4. Ao final, baixe **CSV detalhado** e/ou **CSV resumido**.
   – Opcional: se configurado um endpoint, é possível **Enviar** os dados.

> **Tela cheia** é recomendada (botão “Tela cheia”). Em dispositivos iOS, mantenha o Safari atualizado.

---

## 4) Paradigma e temporização

* **Estímulo**: círculo colorido (*Go*: vermelho, verde, azul) ou preto (*NoGo*).
* **Duração do estímulo / janela de resposta**: 900 ms.
* **Pré-fixação (jitter)**: 250–400 ms.
* **ISI (jitter)**: 600–900 ms.
* **Balanceamento Go**: cores distribuídas ciclicamente e lista embaralhada (Fisher–Yates com PRNG determinístico).

> **Precisão**: tempos baseados em `performance.now()`; resposta aceita somente dentro da janela do estímulo.

---

## 5) Saídas de dados

### 5.1 CSV **detalhado** (`gng_detailed_<ID>.csv`)

* Primeira linha (comentário): metadados de sessão em JSON

  * `version`, `pid`, `seed`, `userAgent`, `lang`, `screen` (`w`,`h`,`dpr`), `tsISO`, `mode`
* Colunas:

  * `pid`: identificador do participante
  * `trial`: número do ensaio (1..N)
  * `cond`: `go` ou `nogo`
  * `stim`: cor do estímulo (hex)
  * `resp`: `go` (respondeu), `miss` (omissão em Go), `withhold` (inibição correta em NoGo)
  * `acc`: acerto (1) / erro (0)
  * `rt_ms`: tempo de reação (ms; apenas quando `resp="go"`)
  * `onset_ms`: timestamp relativo (ms, `performance.now()`) no onset do estímulo
  * `preFix_ms`: pré-fixação do ensaio (ms)
  * `isi_ms`: ISI do ensaio (ms)
  * `seed`: semente numérica usada
  * `version`: versão do task

### 5.2 CSV **resumido** (`gng_summary_<ID>.csv`)

* Primeira linha (comentário): metadados de sessão em JSON (mesmo formato).
* Colunas:

  * `pid`, `mode`, `goN`, `nogoN`, `goHits`, `goMiss`, `commErr`
  * `goAcc_%`, `omiss_%`, `comm_%`
  * `rtMean_ms`, `rtSd_ms`
  * `dprime`, `beta`
  * `hRate`, `faRate`
  * `seed`, `version`

> **RT**: estatísticas calculadas **apenas** sobre acertos Go (média e desvio-padrão).
> **d′ e β**: ver Seção 6 (métodos).

---

## 6) Cálculo de métricas (métodos)

* **Taxas**

  * *Hit rate*: `goHits / goN`
  * *False alarm rate*: `commErr / nogoN`
* **Correção log-linear (Hautus, 1995)**
  Para evitar `0` ou `1` (que levam a valores infinitos de *z*):

  * `hRate = (goHits + 0.5) / (goN + 1)`
  * `faRate = (commErr + 0.5) / (nogoN + 1)`
* **Quantil z**: `z = Φ⁻¹(p)` (implementado via `erfcinv`).
* **d′**: `d′ = z(hRate) − z(faRate)`
* **β**: `β = exp((z(faRate)² − z(hRate)²)/2)`

---

## 7) Controle de qualidade (QC) e exclusões sugeridas

* **Tempo mínimo de exposição total** compatível com o modo (ex.: *Completo* \~ 3–4 min).
* **Atenção/resposta**:

  * *Go* com **acurácia < 60%** ou *NoGo* com **comissão > 40%** → revisar/corroborar repetições.
* **RTs extremos**:

  * Descartar RTs < 120 ms (antecipação) e > 1500 ms (distração), se aplicável à análise (não altera o CSV bruto).
* **Dispositivo/ambiente**:

  * Registre navegador e DPR dos metadados para análises de confounders.

> As regras de QC podem ser adaptadas conforme o protocolo do estudo/CEP.

---

## 8) Personalização rápida

No topo do `<script>`, ajuste o objeto `CONFIG`:

```js
const CONFIG = {
  reduced: { go: 24, nogo: 12 },
  full:    { go: 48, nogo: 24 },
  goColors: ["#ef4444", "#22c55e", "#3b82f6"],
  nogoColor: "#000000",
  stimDurationMs: 900,
  responseWindowMs: 900,
  isiMs: [600, 900],
  preFixMs: [250, 400],
  blockBreakEvery: 9999
};
```

* **Pausas em bloco**: por exemplo, `blockBreakEvery: 24` insere pausa após cada 24 ensaios.
* **Cores**: adicione/remova em `goColors` (mantendo balanceamento).

---

## 9) Coleta automática (opcional)

Para envio automático de dados (JSON), defina a constante:

```js
const POST_ENDPOINT = "https://sua-url-de-coleta.exemplo/endpoint";
```

**Payload**:

```json
{
  "session": { "version": "v1.3", "pid": "BR-001-A", "...": "..." },
  "trials": [
    {"trial":1,"cond":"go","stim":"#ef4444","resp":"go","acc":1,"rt":412,"onset":123456.7,"preFix":312,"isi":712,"...":"..."},
    {"trial":2,"cond":"nogo","stim":"#000000","resp":"withhold","acc":1,"rt":null,"...":"..."}
  ]
}
```

> **Sugestão**: Google Apps Script (Web App) ou Netlify Functions. **Não** inclua identificadores pessoais no endpoint público.

---

## 10) Privacidade, ética e consentimento

* O identificador é livre (ex.: código alfanumérico) e compõe a semente de randomização.
* O repositório **não** armazena dados de participantes por si só.
  – O participante baixa o CSV localmente; o envio remoto só ocorre se **POST\_ENDPOINT** for configurado e acionado.
* Garanta que o protocolo, termo de consentimento e fluxos de dados estejam aprovados por **CEP/IRB**.

---

## 11) Compatibilidade e desempenho

* **Navegadores**: Chrome/Edge/Firefox/Safari recentes.
* **Áudio**: o feedback sonoro requer gesto do usuário para desbloqueio (`AudioContext`); se bloqueado, o teste funciona mesmo sem som.
* **Tela cheia**: recomendada para reduzir distrações.
* **Mobile**: responsivo; em iOS, use Safari atualizado. Desative o *modo baixo consumo* e mantenha a tela ativa quando possível.

---

## 12) Solução de problemas (FAQ)

* **A barra de espaço rola a página**: já impedimos o comportamento padrão; use versão atual do navegador.
* **Sem som**: clique em qualquer botão primeiro (gesto de usuário), depois inicie. Alguns sistemas silenciam o navegador.
* **Página “congela” entre ensaios**: verifique economia de energia/aba em *background*; use **tela cheia** e mantenha a aba ativa.
* **Resultados “vazios”**: somente após concluir o teste (ou abortar) os botões de download aparecem.
* **Ordem previsível de cores**: não — há embaralhamento com semente derivada do ID (reprodutível entre sessões com mesmo ID).

---

## 13) Desenvolvimento

* **Estrutura**: app de página única (HTML/CSS/JS puros).
* **Timing**: `performance.now()`; sem dependências externas.
* **PRNG**: `mulberry32` (semente `FNV-1a` do ID).
* **Cálculo z**: via `erfcinv`/`erf` (aproximações numéricas estáveis).

---

## 14) Licença

Recomendado: **MIT License** (ajuste se necessário ao seu projeto/instituição).

---

## 15) Citação sugerida

> Brandão PRP, *et al*. **Tarefa Go–NoGo web (v1.3)**: implementação estática com temporização de alta precisão, randomização reprodutível e exportação CSV. 2025. Disponível em GitHub Pages.

Formato AMA (exemplo):

> Brandão PRP. Tarefa Go–NoGo (v1.3) \[software]. 2025. Disponível em: URL do GitHub Pages. Acessado em: *data*.

---

## 16) Changelog

* **v1.3**

  * `performance.now()` para RT e onset
  * Correção log-linear para d′/β
  * Jitter (pré-fix e ISI); janela de resposta explícita
  * Seed por ID (PRNG determinístico)
  * Teclado (ESPAÇO) + clique; debouncing por ensaio
  * CSV detalhado e resumido com metadados; botão de **POST** opcional
  * UI responsiva; botão de tela cheia; caixa de diálogo de ajuda

---

## 17) Contato/Contribuição

Sinta-se livre para abrir *issues* e *pull requests* com melhorias metodológicas (parâmetros, QC, novos modos), correções e internacionalização.
