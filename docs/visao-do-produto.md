> **Nota:** o nome definitivo do produto é **Orcamentos Brasil**. Este documento foi escrito com o nome provisório "Orçamento Fácil"; onde aparecer esse nome, leia "Orcamentos Brasil".

# Orçamento Fácil
## Documento de visão do produto e planejamento do MVP

**Tipo de produto:** Micro-SaaS de criação e gestão de orçamentos  
**Mercado inicial:** Brasil  
**Público inicial:** Técnicos de ar-condicionado, eletricistas e prestadores de serviços  
**Modelo de negócio:** Assinatura mensal, com plano gratuito de entrada

---

## 1. Visão geral

O **Orçamento Fácil** é um sistema web para profissionais autônomos e pequenas empresas criarem, enviarem e acompanharem orçamentos de serviços e produtos.

A proposta é substituir processos manuais feitos em Word, Excel ou mensagens de WhatsApp por um fluxo simples:

1. Cadastrar o cliente.
2. Adicionar serviços e materiais.
3. Calcular automaticamente os valores.
4. Gerar um orçamento profissional em PDF.
5. Compartilhar o orçamento por WhatsApp ou por um link público.
6. Acompanhar se o cliente visualizou, aprovou, recusou ou pediu alterações.

O produto não precisa começar como um ERP completo. O foco é resolver muito bem o processo de orçamento e acompanhamento comercial.

## 2. Problema que o produto resolve

Muitos prestadores de serviços ainda:

- Montam orçamentos manualmente em documentos ou planilhas.
- Repetem informações de clientes e serviços.
- Têm dificuldade para manter um padrão visual profissional.
- Perdem o histórico de propostas enviadas.
- Não sabem facilmente quais orçamentos foram aprovados ou estão pendentes.
- Precisam cobrar respostas dos clientes manualmente.
- Não têm uma maneira simples de apresentar uma proposta pelo celular.

O Orçamento Fácil centraliza essas tarefas em um único lugar.

## 3. Público-alvo

### Segmento inicial recomendado

Começar com prestadores de serviços que fazem orçamentos com frequência e trabalham com mão de obra, materiais ou ambos:

- Técnicos de ar-condicionado.
- Eletricistas.
- Encanadores.
- Profissionais de manutenção.
- Empresas de instalação.
- Prestadores de serviços residenciais.
- Pequenas empresas de reformas e construção.

### Estratégia de entrada no mercado

Não tentar atender todos os segmentos logo no lançamento. Começar com técnicos de ar-condicionado e eletricistas, oferecendo modelos e catálogos iniciais de serviços e materiais relevantes para essas atividades.

Depois de validar o produto, expandir para outros segmentos.

## 4. Proposta de valor

**Criar um orçamento profissional em poucos minutos e acompanhar a resposta do cliente sem depender de planilhas.**

Principais benefícios:

- Economia de tempo.
- Apresentação mais profissional.
- Menos retrabalho.
- Histórico centralizado.
- Facilidade para compartilhar pelo WhatsApp.
- Acompanhamento de aprovações.
- Acesso pelo computador ou celular.

## 5. Fluxo principal de uso

1. O profissional cria sua conta e cadastra os dados da empresa.
2. Cadastra um cliente ou seleciona um cliente existente.
3. Adiciona serviços e produtos ao orçamento.
4. Informa quantidades, preços, descontos, condições e validade.
5. Confere o resumo e gera o orçamento.
6. O sistema disponibiliza um PDF e um link público exclusivo.
7. O profissional compartilha o link ou PDF com o cliente.
8. O cliente visualiza a proposta e pode aprovar ou solicitar alterações.
9. O profissional acompanha a atualização no painel.

## 6. Funcionalidades

### 6.1 Painel inicial

Exibir indicadores e atalhos para as atividades mais comuns:

- Quantidade de orçamentos criados.
- Orçamentos aguardando resposta.
- Orçamentos aprovados.
- Orçamentos recusados.
- Valor total em negociação.
- Valor total aprovado.
- Lista dos últimos orçamentos.
- Atalho para criar novo orçamento.

Os números devem ser calculados a partir dos dados reais da empresa autenticada.

### 6.2 Cadastro de clientes

Campos sugeridos:

- Nome ou razão social.
- CPF/CNPJ (opcional no MVP).
- Telefone/WhatsApp.
- E-mail (opcional).
- Endereço (opcional).
- Observações internas.

Permitir criar, editar, consultar e arquivar clientes.

### 6.3 Catálogo de produtos e serviços

Permitir que o usuário salve itens usados frequentemente para adicioná-los rapidamente aos orçamentos.

Campos:

- Nome/descrição.
- Tipo: produto ou serviço.
- Unidade de medida.
- Preço padrão.
- Ativo/inativo.

Exemplos:

- Instalação de ar-condicionado.
- Manutenção preventiva.
- Tubulação de cobre por metro.
- Suporte de condensadora.
- Mão de obra elétrica.

O usuário deve poder alterar a descrição, quantidade e preço de cada item dentro de um orçamento sem alterar automaticamente o cadastro original.

### 6.4 Criação e edição de orçamentos

Campos e recursos:

- Número sequencial ou identificador legível.
- Cliente.
- Data de criação.
- Validade.
- Itens, quantidades e valores unitários.
- Desconto em valor ou percentual.
- Subtotal e total calculados automaticamente.
- Condições de pagamento.
- Prazo estimado de execução.
- Observações e termos.
- Status do orçamento.

Fórmula básica:

`subtotal = soma(quantidade × valor_unitário)`

`total = subtotal - desconto`

No MVP, manter impostos e cálculos fiscais fora do escopo, salvo se forem apenas campos informativos configuráveis. O produto não deve se apresentar como sistema fiscal ou contábil.

### 6.5 Geração de PDF

Gerar um documento com:

- Logotipo e dados da empresa.
- Dados do cliente.
- Número e data do orçamento.
- Lista de produtos e serviços.
- Quantidades, preços e totais.
- Descontos.
- Condições e validade.
- Observações.
- Contatos da empresa.

O PDF deve ser legível em celular e computador e ter aparência profissional.

### 6.6 Compartilhamento por WhatsApp

Disponibilizar uma ação para abrir o WhatsApp com uma mensagem pronta contendo o link público do orçamento.

Exemplo de mensagem:

> Olá! Preparei seu orçamento. Você pode visualizar os detalhes e responder por este link: [link do orçamento].

O compartilhamento inicial pode usar o mecanismo de link do WhatsApp, sem integração oficial com a API do WhatsApp.

### 6.7 Página pública do orçamento

Cada orçamento terá um link exclusivo, difícil de adivinhar e sem exigir que o cliente crie uma conta.

A página deve apresentar:

- Identificação do prestador.
- Dados pertinentes do orçamento.
- Itens e valores.
- Total.
- Condições e validade.
- Botão para baixar PDF.
- Ação para aprovar.
- Ação para solicitar alteração.
- Contato pelo WhatsApp.

A página pública deve expor somente os dados necessários para a proposta. Não deve mostrar notas internas, outros clientes ou informações administrativas.

### 6.8 Acompanhamento e histórico

Status sugeridos:

- Rascunho.
- Enviado.
- Visualizado.
- Aguardando resposta.
- Aprovado.
- Recusado.
- Alteração solicitada.
- Expirado.
- Cancelado.

Registrar eventos importantes, como criação, envio, visualização e resposta do cliente, com data e hora.

A aprovação deve registrar a ação e o momento em que ocorreu. Para usos que exijam validade jurídica ou assinatura formal, será necessário avaliar posteriormente um fluxo próprio de assinatura eletrônica e seus requisitos.

## 7. Planos e monetização

Valores iniciais sugeridos para validação comercial. Não são preços comprovados nem garantia de receita.

### Gratuito — R$ 0/mês

- Até 5 orçamentos por mês.
- Cadastro de clientes.
- Criação de orçamentos.
- PDF com marca do Orçamento Fácil.
- Compartilhamento básico.

### Profissional — R$ 29,90/mês

- Orçamentos ilimitados ou limite alto sujeito a política de uso.
- PDF personalizado com logotipo.
- Link público de aprovação.
- Histórico completo.
- Cadastro de produtos e serviços.
- Controle de status.

### Empresa — R$ 59,90/mês

- Todos os recursos do Profissional.
- Mais de um usuário.
- Relatórios.
- Modelos personalizados.
- Gestão básica da equipe.

Antes de implementar cobrança recorrente, validar disposição de pagamento com potenciais clientes. Definir limites claros por plano e evitar prometer funcionalidades que ainda não existem.

## 8. Arquitetura técnica sugerida

### Stack

| Camada | Tecnologia |
|---|---|
| Frontend | React + Vite + TypeScript |
| Componentes e estilos | Tailwind CSS + shadcn/ui |
| Ícones | Lucide |
| Backend | Python + FastAPI |
| Banco de dados | PostgreSQL |
| Autenticação | Sessão segura ou JWT com estratégia de refresh |
| Geração de PDF | Biblioteca Python apropriada ou HTML para PDF |
| Pagamentos | Asaas ou Mercado Pago |
| Hospedagem inicial | VPS Ubuntu + Docker |
| Armazenamento de arquivos | Armazenamento compatível com S3, como Cloudflare R2 |

A escolha final de bibliotecas e provedores deve considerar custo, manutenção, segurança e facilidade de operação.

### Componentes da aplicação

- Aplicação web para o prestador.
- API para autenticação e operações de negócio.
- Banco PostgreSQL.
- Serviço de geração de PDF.
- Página pública para visualização e resposta do orçamento.
- Integração de cobrança para assinaturas, em uma etapa posterior.

## 9. Multiempresa e estrutura de dados

O sistema deve nascer preparado para múltiplas empresas (multi-tenant).

Cada empresa terá seus próprios usuários, clientes, itens e orçamentos. Toda consulta e operação de dados deve respeitar o contexto da empresa autenticada.

Tabelas iniciais sugeridas:

| Tabela | Responsabilidade |
|---|---|
| `companies` | Empresas/contas clientes do SaaS |
| `users` | Usuários e credenciais |
| `company_users` | Relação entre usuários e empresas, se houver múltiplos usuários |
| `clients` | Clientes de cada empresa |
| `catalog_items` | Produtos e serviços cadastrados |
| `quotes` | Dados principais dos orçamentos |
| `quote_items` | Itens de cada orçamento |
| `quote_events` | Histórico de eventos e alterações de status |
| `subscriptions` | Plano e situação da assinatura |

Campos importantes:

- `id` como identificador primário.
- `company_id` nas entidades pertencentes a uma empresa.
- `created_at` e `updated_at`.
- `created_by` quando for útil identificar o usuário responsável.

### Regras importantes de isolamento

- Nunca confiar em um `company_id` enviado livremente pelo frontend.
- Determinar a empresa a partir da sessão/usuário autenticado e das permissões.
- Validar autorização em todas as operações da API.
- Aplicar isolamento também em downloads, PDFs e arquivos.
- Usar tokens públicos aleatórios e revogáveis para páginas de orçamento.
- Testar explicitamente que uma empresa não consegue acessar dados de outra.

## 10. Segurança, privacidade e confiabilidade

O sistema armazenará dados pessoais e comerciais. Desde o MVP:

- Coletar somente os dados necessários.
- Proteger senhas com hash seguro.
- Usar HTTPS.
- Validar e autorizar todas as operações no backend.
- Evitar expor informações internas em links públicos.
- Limitar tentativas e proteger endpoints públicos contra abuso.
- Registrar eventos relevantes.
- Fazer backups regulares e testar restauração.
- Definir política de retenção e exclusão de dados.
- Preparar aviso de privacidade e termos de uso.
- Avaliar obrigações aplicáveis da LGPD.

Não armazenar dados de cartão. Utilizar um provedor de pagamento para processar cobranças.

## 11. Escopo do MVP

### Essencial para a primeira versão

- [ ] Cadastro e login.
- [ ] Cadastro dos dados da empresa.
- [ ] Cadastro de clientes.
- [ ] Cadastro de produtos e serviços.
- [ ] Criar, editar e consultar orçamentos.
- [ ] Cálculo de subtotal, desconto e total.
- [ ] Geração de PDF.
- [ ] Compartilhamento por WhatsApp.
- [ ] Página pública do orçamento.
- [ ] Aprovação ou solicitação de alteração.
- [ ] Histórico e status.
- [ ] Isolamento multiempresa.
- [ ] Layout responsivo para celular e desktop.

### Deixar para versões futuras

- [ ] Cobrança recorrente e gestão completa de assinaturas.
- [ ] Integração oficial com WhatsApp Business.
- [ ] Notificações automáticas por e-mail ou WhatsApp.
- [ ] IA para sugerir descrições e estruturar propostas.
- [ ] Assinatura eletrônica avançada.
- [ ] Integração com emissão de nota fiscal.
- [ ] Gestão financeira e fluxo de caixa.
- [ ] Aplicativo mobile nativo.
- [ ] Estoque e compras.
- [ ] Relatórios avançados.

Manter o MVP concentrado em criar, enviar e acompanhar orçamentos. Evitar transformar o primeiro lançamento em um ERP.

## 12. Plano de desenvolvimento sugerido: 7 dias

Estimativa agressiva para um protótipo funcional, considerando escopo controlado e desenvolvimento concentrado. Não inclui necessariamente testes extensivos, cobrança em produção ou adequação completa de segurança.

| Dia | Entrega |
|---|---|
| 1 | Estrutura do projeto, layout, autenticação e empresas |
| 2 | Cadastro e gerenciamento de clientes |
| 3 | Catálogo de produtos e serviços |
| 4 | Criação, edição e cálculo dos orçamentos |
| 5 | Geração de PDF e compartilhamento |
| 6 | Página pública e ações de resposta |
| 7 | Histórico, testes, correções e deploy inicial |

### Critérios mínimos antes de disponibilizar a usuários

- Uma empresa não consegue consultar dados de outra.
- Totais e descontos são calculados corretamente no backend.
- Um orçamento enviado não muda silenciosamente após o cliente visualizá-lo; alterações relevantes devem gerar uma nova versão ou registro claro.
- Links públicos não permitem descobrir outros orçamentos.
- PDFs mostram os dados corretos e funcionam em dispositivos móveis.
- Aprovação, recusa e solicitação de alteração são registradas.
- Backups e recuperação foram testados.

## 13. Validação comercial

Antes de investir em funcionalidades adicionais:

1. Conversar com pelo menos 10 prestadores do segmento escolhido.
2. Perguntar como criam e enviam orçamentos atualmente.
3. Entender quantos orçamentos fazem por mês.
4. Descobrir o que mais toma tempo ou causa perda de vendas.
5. Apresentar um protótipo clicável.
6. Tentar conseguir de 3 a 5 usuários-piloto.
7. Medir uso real e disposição de pagar.
8. Ajustar o produto antes de ampliar o público.

### Perguntas para entrevistas

- Como você prepara um orçamento hoje?
- Quanto tempo leva para criar e enviar um orçamento?
- Você reutiliza serviços e materiais de orçamentos anteriores?
- Como acompanha se o cliente aprovou?
- Quantos orçamentos costuma enviar por mês?
- O que mais incomoda no processo atual?
- Pagaria por uma ferramenta que simplificasse esse processo? Quanto?

## 14. Simulação de receita

Com uma mensalidade hipotética de R$ 29,90:

| Clientes pagantes | Receita bruta mensal |
|---:|---:|
| 10 | R$ 299,00 |
| 30 | R$ 897,00 |
| 50 | R$ 1.495,00 |
| 100 | R$ 2.990,00 |
| 200 | R$ 5.980,00 |

Valores brutos ilustrativos, antes de taxas de pagamento, impostos, hospedagem, suporte, marketing, inadimplência e outros custos. Não representam previsão de vendas ou lucro.

## 15. Indicadores para acompanhar

- Visitantes da página de vendas.
- Cadastros iniciados e concluídos.
- Empresas que criam o primeiro orçamento.
- Orçamentos criados por empresa.
- Percentual de orçamentos compartilhados.
- Percentual de orçamentos visualizados.
- Taxa de aprovação.
- Conversão do plano gratuito para pago.
- Cancelamentos (churn).
- Receita recorrente mensal (MRR).
- Custo de aquisição de clientes (CAC).
- Retenção por coorte.

O primeiro indicador de ativação pode ser: **uma empresa cadastrada que cria e compartilha seu primeiro orçamento**.

## 16. Posicionamento e mensagem comercial

### Mensagem principal

**Crie orçamentos profissionais em minutos e acompanhe a aprovação dos seus clientes em um só lugar.**

### Benefícios para a comunicação

- Chega de montar orçamento manualmente.
- Envie propostas profissionais pelo WhatsApp.
- Saiba quais clientes aprovaram ou ainda precisam responder.
- Reutilize serviços e materiais cadastrados.
- Organize seus orçamentos sem planilhas.

### Exemplo de chamada para a landing page

**Seus orçamentos mais profissionais. Seu trabalho muito mais organizado.**

Crie propostas, envie pelo WhatsApp e acompanhe as respostas dos clientes com praticidade.

**CTA:** Começar gratuitamente.

## 17. Próximos passos recomendados

1. Validar o segmento inicial com entrevistas.
2. Definir identidade visual e nome definitivo.
3. Criar protótipo das telas principais.
4. Implementar o MVP com isolamento multiempresa.
5. Testar com prestadores reais.
6. Ajustar a experiência com base no uso.
7. Só então ativar planos pagos e ampliar a aquisição.

---

## Resumo executivo

O Orçamento Fácil é um Micro-SaaS focado em simplificar a criação, o envio e o acompanhamento de orçamentos para prestadores de serviços e pequenas empresas.

A melhor estratégia inicial é atender um nicho específico — começando por técnicos de ar-condicionado e eletricistas — e entregar muito bem quatro capacidades: **cadastrar clientes, montar orçamentos, gerar PDFs e acompanhar aprovações por link**.

O diferencial não é apenas gerar um documento bonito. É transformar o orçamento em um processo organizado, rastreável e fácil de compartilhar, criando valor recorrente suficiente para sustentar uma assinatura mensal.
