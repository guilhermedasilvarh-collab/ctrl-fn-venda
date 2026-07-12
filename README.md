# Controle Financeiro — versão para venda

Cópia genérica do app pessoal (`../index.html`), sem nenhum dado real embutido.
O app original continua intocado — esta pasta é a base do produto.

## `demo.html` — para mostrar a clientes agora, sem depender do Firebase

`demo.html` é uma versão à parte, sem login e sem backend: abre direto no
dashboard, já preenchido com dados **fictícios** (um estúdio de design PJ
fictício, com contas, cartões, reserva de emergência e um veículo completos)
representando o mês atual e os últimos meses. Nada é salvo em servidor —
tudo fica só na memória da aba, e um botão "↺ reiniciar demo" no topbar
restaura os dados originais a qualquer momento (um reload da página faz o
mesmo). Um aviso fixo no topo deixa claro pro cliente que é uma demonstração.

Como usar: abra `venda/demo.html` direto no navegador (localmente ou hospedado
em qualquer lugar estático — não precisa de Firebase configurado) e navegue
à vontade durante a apresentação. Clique nas abas, marque contas como pagas,
mude o tema — é interativo de verdade, só não persiste para a próxima sessão.

Quando o cliente fechar negócio, ele cria a própria conta na versão real
(`index.html`, depois que você configurar o Firebase — veja abaixo) e começa
do zero, sem nenhum dado fictício.

## O que já foi feito nesta versão

- Estado inicial em branco: nenhuma conta, cartão, reserva ou dado de veículo
  pré-cadastrado (o app original tinha despesas, RENAVAM/chassi e financiamento
  reais como padrão).
- Cartões deixaram de ser uma lista fixa (Nubank/Itaú/Rafaela) — agora cada
  usuário cadastra os próprios cartões em **Configurar → Meus cartões**, com
  cor atribuída automaticamente.
- Removido o card "Financiamento" com dados reais de um veículo específico.
- Caminhos de manifest/service worker/ícones viraram relativos, para funcionar
  em qualquer subpasta ou domínio.
- Onboarding guiado de 3 passos no primeiro login (receita, cartões, meta da
  reserva) — tudo opcional/pulável, marcado como concluído por usuário
  (`onboardingDone`) para não repetir em logins seguintes.
- Termos de Uso (`termos-de-uso.html`) e Política de Privacidade
  (`politica-de-privacidade.html`), com aceite obrigatório por checkbox no
  cadastro — sem marcar, `createUserWithEmailAndPassword` nem é chamado.
  **Minuta inicial, não é aconselhamento jurídico** — tem campos marcados em
  vermelho (`[PREENCHER]`) com a identidade do responsável (nome/CPF ou CNPJ),
  cidade do foro e prazo de retenção de dados. Revise com um advogado antes do
  primeiro usuário pagante — inclusive se vale a pena formalizar CNPJ/MEI, já
  que pessoa física tratando dados comercialmente de terceiros sai da exceção
  de "uso pessoal" da LGPD.
- **Perfil personalizável** (onboarding + editável depois em Configurar →
  Perfil): tipo de regime (PJ / CLT / Autônomo), com labels dinâmicos em todo
  o app ("Custos PJ" vira "Custos do trabalho" fora do regime PJ); cronograma
  de recebimento configurável — 1 ou 2 pagamentos por mês, com dia e rótulo
  livres (não é mais fixo em "dia 15 / dia 2", pode ser qualquer combinação,
  ex: dia 5 e dia 20); e toggle "tenho veículo", que mostra/esconde a aba
  Veículo inteira.
- **Tema claro/escuro**: botão no topbar (🌙/☀️), persiste por usuário
  (`S.config.tema`) e também no `localStorage` do navegador para não piscar o
  tema errado no primeiro carregamento. Gráficos (Chart.js) leem as cores via
  `getComputedStyle` das variáveis CSS, então se adaptam automaticamente.
- **Trial de 14 dias + cobrança manual (Fase 1)** — só em `index.html`, o
  `demo.html` nunca mostra nada disso. Veja a seção própria abaixo.

## Trial + cobrança manual (Fase 1)

Cada conta nova recebe `S.config.plano='trial'` e `S.config.trialInicio`
(data ISO do primeiro login), com `TRIAL_DIAS=14`. Contas que já existiam
antes deste recurso ganham um trial novo a partir de hoje (não nascem
expiradas) — ver o tratamento em `loadData()`.

- `diasTrialRestantes()` e `contaAtiva()` (`index.html`) calculam se a conta
  ainda pode usar o app. `aplicarPlano()` roda depois de `loadData()` no
  `onAuthStateChanged` (e é chamada de novo dentro de `renderPlano()`/
  `renderCfg()`) e decide o que aparece:
  - Faixa `#trial-strip` no topo enquanto o trial está ativo (fica âmbar/
    "perto" nos últimos 3 dias).
  - Tela `#paywall` no lugar do app (`#main-app`) quando o trial acaba e
    `plano` ainda não é `'pro'` — os dados continuam salvos no Firestore,
    só o acesso é bloqueado.
  - Card **"🎟️ Meu plano"** em Configurar e o modal `#modal-planos` (Pix +
    aviso manual por e-mail) mostram o status atual.
- **Não existe gateway de pagamento.** O fluxo é: usuário faz Pix → manda
  comprovante pro e-mail (link "Já paguei — avisar" já monta o e-mail) → você
  confirma o pagamento manualmente e libera a conta.
- **Como liberar um usuário manualmente**: Firebase Console → Firestore
  Database → coleção `usuarios` → documento do UID do cliente (dá pra achar
  o e-mail em Authentication → Users) → edite o campo `config.plano` de
  `"trial"` para `"pro"` → salve. Na próxima vez que o usuário abrir o app
  (ou recarregar a página), `aplicarPlano()` libera o acesso automaticamente,
  sem precisar de nenhum deploy.
- **Pendente antes de vender de verdade**: preencher os dois `[PREENCHER]`
  do `#modal-planos` em `index.html` — o preço mensal e a chave Pix.
- Fase 2 (futura, não iniciada): integração automática com Mercado Pago,
  provavelmente via Cloud Functions validando webhook no servidor (o cliente
  não pode simplesmente "avisar que pagou" sem validação server-side).

## Firebase do projeto de venda

`index.html` já aponta para um projeto Firebase próprio (`ctrl-fn-venda`),
separado do app pessoal. Authentication (e-mail/senha) e Firestore precisam
estar ativados no console — se algum dia precisar recriar do zero, os passos
são: **Authentication → Sign-in method → E-mail/senha** e **Firestore
Database → Criar banco de dados** (modo produção).

## Regras de segurança do Firestore (obrigatório)

O arquivo `venda/firestore.rules` tem o conteúdo exato a colar em
**Firestore Database → Regras** no console:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /usuarios/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

Isso garante que cada usuário só lê/escreve o próprio documento — sem isso,
qualquer conta autenticada pode acessar dados financeiros de outra.
**Confirme no console que essas regras estão publicadas antes de divulgar o
link para clientes reais.**

## Ainda pendente (próximas fases)

- Cobrança automática (Fase 2): gateway Mercado Pago com validação de webhook
  server-side (Cloud Functions). Hoje a cobrança é manual — ver seção acima.
- Preencher os campos `[PREENCHER]` dos termos/política/planos (identidade do
  responsável, cidade do foro, prazo de retenção, preço mensal e chave Pix) e
  revisar com advogado.
- Categorias de despesa ainda são uma lista fixa (Moradia, Alimentação, etc.) —
  funcional para MVP, mas não editável pelo usuário.
