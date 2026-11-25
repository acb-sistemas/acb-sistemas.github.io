# ACB Sistema — Contexto do Projeto

## Visão Geral
Sistema de gestão para a ONG **Associação Cristã de Base (ACB)**, Crato/CE: vários programas sociais, municípios e estados do Nordeste.

**Desenvolvido por:** Falcioni Consultoria
**Cliente:** Associação Cristã de Base
**URL do app:** https://acb-sistemas.github.io
**Tecnologia:** PWA (HTML/CSS/JS em arquivo único) + Firebase (Firestore, Auth e Storage)

> Este arquivo fica público no GitHub. Não colocar aqui senhas, tokens ou dados pessoais.

---

## Infraestrutura

### Firebase
- **Projeto:** `acb-sistema` (região southamerica-east1, São Paulo)
- **Auth:** e-mail/senha (equipe) + login anônimo automático (portal de projetos)
- **Storage:** fotos dos beneficiários (`beneficiarios/{id}/fotos/...`)
- A configuração do Firebase (apiKey etc.) está no `index.html`. Ela é pública por natureza: quem protege os dados são as regras do Firestore/Storage no console.

### Hospedagem
- **GitHub Pages**, repositório `acb-sistemas/acb-sistemas.github.io`, branch `main`.
- O Netlify (acbcrato-app.netlify.app) foi usado antes e não é mais o endereço oficial.

---

## Arquivos
```
index.html      → App completo (single-file PWA)
manifest.json   → Configuração PWA
sw.js           → Service Worker (versão atual: acb-v39)
logo-acb.png    → Logo ACB (splash, login, sidebar)
Falcioni.jpg    → Logo Falcioni (rodapé da sidebar)
logo-falcioni.png
icon-192.png / icon-512.png → Ícones PWA
CLAUDE.md       → Este arquivo
```

---

## As duas partes do sistema

### 1. Portal de Projetos (entrada pública)
- Ao abrir o site, entra com login anônimo e mostra os projetos na barra lateral, cada um com cadeado.
- Projetos ficam fixos no código, no objeto `PROJETOS` (plano de trabalho: objeto, concedente, metas, etapas, itens, valores, cronograma).
- Projetos atuais: Ceará Sem Fome, Cisternas 16m³ (PI), Cisterna Calçadão 52m³ (PI), Mulheres Cultivando Vidas, P1+2 Maranhão, P1MC Ceará, Quintais das Marias, SDA 1, SDA 2 (Cisternas Araripe e Salitre, Contrato 052/2025), Sementes do Futuro.
- Cada projeto tem senha própria (libera só ele, com edição). Senhas master (`MASTER`) liberam todos + "Gestão Geral" (visão consolidada).
- Abas do projeto: Visão Geral, Metas & Despesas, Cronograma de Desembolso, Dashboard (acompanhamento).
- Acompanhamento salvo na coleção `acompanhamento/{projetoId}`: etapas (%), metasPerc, parcelas recebidas, info (período, responsável, meta realizada), cronograma editável, aditivos por meta, observações por meta. Cópia também no localStorage.
- Lançamento do andamento por meta: `qtdInput` = por quantidade, `pctInput` = por %, padrão = por valor (R$).
- Projetos com `indicadoresSig` (hoje só o SDA 2) mostram no Dashboard o card "Indicadores Sig Cisterna", lançado à mão e salvo em `acompanhamento/{id}.sig` (valores iniciais em `seedSig`). Não registrar CPF nem nomes.
- Há migrações pontuais em `carregarAcomp()` (Ceará Sem Fome e P1MC Ceará, 08/07/2026) e `seedEtapas` no P1MC Ceará.

### 2. Sistema com login (botão "Sou Técnico / Administrador")
- Admin/coordenador/financeiro: Dashboard, Programas, Contratos, Beneficiários, Municípios, Execução Físico-Financeira (com repasses em parcelas), Relatórios PDF. Admin/coordenador também veem Técnicos, Usuários, Evolução por Área, Avaliar Técnicos.
- Perfil `tecnico` (modo campo): Meus Beneficiários (ficha com Dados, Fase, Localização GPS, Fotos, Material), Minha Agenda, Questionário Marias, Minhas Avaliações.

---

## Coleções Firestore
```
/acompanhamento/{projetoId}   → andamento dos projetos do portal
/programas/{id}               → nome, sigla, descricao, financiador, status
/contratos/{id}               → programaId, numero, sub, financiador, objeto, valor, inicio, fim, metaFisica, unidade, status, municipiosStr
/atividades/{id}              → contratoId, nome
/execucoes/{id}               → contratoId, atividadeNome, municipioId, municipioNome, data, qtd, valor, tecnicoId, status, obs
/repasses/{id}                → contratoId, valor, data, desc, criadoEm
/beneficiarios/{id}           → nome, cpf, telefone, nascimento, estado, municipioId, municipioNome, comunidade, programaId, programaNome, contratoId, status, fase, localizacao, dataCadastro, obs, createdBy, tecnicoResponsavelId
   /fotos/{id}                → url, desc, criadoEm, criadoPor
   /materiais/{id}            → nome, status (pendente/ok/avariado), obs, criadoEm
/municipios/{id}              → nome, estado, status
/tecnicos/{id}                → nome, cargo, email, telefone, cpf, status
/usuarios/{uid}               → email, nome, perfil (admin/coordenador/financeiro/tecnico), ativo
/agenda/{id}                  → tecnicoId, data, hora, tipo, beneficiarioId, titulo, obs, feito
/questionarios_marias/{id}    → tecnicaId, nome, data, respostas{p1..p6}, obs
/avaliacoes_tecnico/{id}      → tecnicoId, avaliadorId, data, criterios[], notaFinal, obs
```

---

## Regras de Negócio

### Beneficiários
- Fase em 10 etapas (`FASE_ORDEM`): Reuniões → Cadastro → Capacitação → Visita Técnica → Recebimento de Material → Construção → Finalização → Pagamento Pedreiro → Pagamento Beneficiário → Assinatura do Termo. O campo antigo `status` (5 valores) continua gravado por compatibilidade.
- Beneficiário cadastrado por técnico grava `tecnicoResponsavelId` = uid do técnico. "Meus Beneficiários" mostra os com `tecnicoResponsavelId` ou `createdBy` do técnico.
- % Física = beneficiários do programa ÷ soma de `metaFisica` dos contratos do programa.
- % Financeira = gastos (execuções) ÷ orçamento dos contratos do programa.

### Execução — 5 KPIs por contrato
Orçamento Total · Repasse Recebido (soma das parcelas) · A Receber (orçamento − recebido) · Gastos (soma das execuções) · Saldo Disponível (recebido − gastos).

### Perfis
- `admin` / `coordenador` → veem Administração.
- `financeiro` → sistema sem Administração.
- `tecnico` → só o modo campo.
- Primeiro usuário logado sem cadastro em `usuarios` recebe perfil admin.

---

## Publicação
1. Subir a versão no `sw.js` (acb-v38, v39...) a cada publicação. Sem isso, o celular pode continuar com a versão antiga.
2. Enviar os arquivos alterados para a branch `main` do repositório (pelo site do GitHub ou git).
3. O GitHub Pages atualiza em 1 a 2 minutos.

---

## Pendências conhecidas
- Senhas do portal (projetos e master) estão escritas no `index.html`: migrar para conferência pelo Firebase.
- Questionário Quintais das Marias com perguntas provisórias, aguardando o modelo oficial da ACB.
- Projetos do portal fixos no código: migrar para o Firestore com tela de edição.
- Pedidos da reunião de 25/11/2026 na ACB:
  - Área da Assistente Social: plano de ação 2026 (metas de assistência social dentro de cada projeto, ligadas à certificação que garante cerca de 20% de isenção), planejado x realizado, encaminhamentos (quantos e para quais órgãos, ex.: CRAS) e atendimentos por grupo. Ela presta contas à presidência toda segunda-feira.
  - Área do Ramon (indicadores).
  - Portal abrir documentos direto de links do Google Drive da própria ACB (cada área informa o seu link).
  - Responsável técnico pelo portal a partir da entrega: Ramon (ACB).

---

## Histórico
- jun/2026: % Física e % Financeira nos beneficiários; KPI "A Receber"; repasse em parcelas.
- 08/07/2026: login anônimo no portal; reestruturação de metas do Ceará Sem Fome e P1MC Ceará.
- 15/07/2026: modo técnico (ficha, agenda, questionário, avaliações, evolução por área); botão de acesso técnico mais visível.
- 25/11/2026: beneficiário cadastrado pelo técnico passa a aparecer em "Meus Beneficiários"; "Editar" na ficha do técnico abre preenchido; correção no tratamento de erro ao iniciar o app; este arquivo atualizado.
- 25/11/2026 (v39): novo projeto SDA 2 no portal (337 cisternas 16 mil L, Araripe e Salitre, R$ 2.216.122,11; vigência 17/04/2021 a 17/04/2026 no Sig) a partir do plano de trabalho do Drive, com card de indicadores do Sig Cisterna.
