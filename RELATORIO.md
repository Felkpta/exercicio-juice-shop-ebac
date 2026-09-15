# Relatório de Pentest — OWASP Juice Shop

**Autor:** Felkpta
**Aplicação testada:** OWASP Juice Shop (http://demo.owasp-juice.shop/)
**Data dos testes:** 14/09/2026
**Objetivo:** Identificar e documentar vulnerabilidades de segurança na aplicação, simulando o ponto de vista de um atacante externo.

---

## Resumo Executivo

Durante os testes foram identificadas **3 vulnerabilidades** exploráveis na aplicação, permitindo desde a execução de código no navegador da vítima até o acesso não autorizado a dados pessoais e financeiros de outro usuário (administrador da loja). As falhas encontradas são de severidade **alta**, pois combinadas permitem a um atacante assumir o controle de contas de terceiros e visualizar informações sensíveis sem possuir credenciais válidas.

| # | Vulnerabilidade | Categoria OWASP | Severidade |
|---|---|---|---|
| 1 | SQL Injection no login (bypass de autenticação) | Broken Authentication | Alta |
| 2 | Acesso não autorizado a pedidos e dados de pagamento | Broken Access Control / Sensitive Data Exposure | Alta |
| 3 | Cross-Site Scripting (XSS) Refletido na busca | Injection | Média/Alta |

---

## Vulnerabilidade 1: SQL Injection — Bypass de Autenticação

**Descrição:**
O formulário de login não trata corretamente os caracteres especiais enviados nos campos de e-mail e senha, permitindo a injeção de uma condição SQL que sempre retorna verdadeira, autenticando o atacante sem conhecer nenhuma senha válida.

**Passos para reproduzir (pensando como atacante):**
1. Acessar a tela de login da aplicação.
2. No campo de e-mail, inserir o payload: `' OR 1=1--`
3. No campo de senha, inserir qualquer valor (ex.: `123`).
4. Clicar em "Log in".

**Resultado observado:**
A aplicação autenticou o atacante automaticamente como o usuário **admin@juice-sh.op** — a conta de administrador da loja — sem que nenhuma senha correta tivesse sido informada.

![Conta admin logada](![Evidência 02](Evidencias/02-conta-admin-logada.png))
*Evidência: menu de conta mostrando login efetuado como admin@juice-sh.op após o payload de SQL Injection.*

**Impacto:**
Um atacante consegue assumir a identidade de qualquer usuário cadastrado — inclusive o administrador da plataforma — sem possuir credenciais válidas, comprometendo toda a confidencialidade e integridade da conta.

**Recomendação:**
Utilizar consultas parametrizadas (prepared statements) em vez de concatenação de strings na construção de queries SQL, e aplicar validação/sanitização de entrada em todos os campos do formulário de login.

---

## Vulnerabilidade 2: Acesso Não Autorizado a Pedidos e Dados de Pagamento

**Descrição:**
Como consequência direta da falha de autenticação (Vulnerabilidade 1), o atacante ganha acesso total à área logada da vítima, incluindo histórico de pedidos e métodos de pagamento cadastrados — dados que deveriam ser restritos exclusivamente ao dono da conta.

**Passos para reproduzir:**
1. Realizar o login bypass descrito na Vulnerabilidade 1.
2. Navegar até "Orders & Payment" > "Order History".
3. Navegar até "My Payment Options".

**Resultado observado:**
Foi possível visualizar o histórico completo de pedidos da conta administrativa (produtos comprados, valores e status de entrega) e os cartões de pagamento cadastrados (números mascarados, mas com titular e validade visíveis).

![Histórico de pedidos do admin](![Evidência 03](Evidencias/03-order-history-admin.png))
*Evidência: histórico de pedidos pertencente à conta administradora, acessado sem autorização.*

![Métodos de pagamento do admin](![Evidência 04](Evidencias/04-payment-options-admin.png))
*Evidência: cartões de pagamento cadastrados na conta administradora, expostos ao atacante.*

**Impacto:**
Exposição de dados pessoais e financeiros de terceiros, configurando violação de privacidade e de proteção de dados (o que em um cenário real infringiria legislações como a LGPD/GDPR). Também evidencia falha de segregação de acesso entre contas de usuários comuns e administrativas.

**Recomendação:**
Corrigir a causa raiz (Vulnerabilidade 1) e implementar verificação de autorização em nível de objeto (object-level authorization) em todas as rotas que retornam dados de pedidos e pagamento, garantindo que cada usuário só acesse seus próprios registros.

---

## Vulnerabilidade 3: Cross-Site Scripting (XSS) Refletido

**Descrição:**
O campo de busca de produtos reflete o termo pesquisado na página de resultados sem realizar a sanitização adequada, permitindo a injeção de HTML/JavaScript que é executado no navegador da vítima.

**Passos para reproduzir:**
1. Acessar o campo de busca no topo da aplicação.
2. Inserir o payload: `<iframe src="javascript:alert('XSS')">`
3. Pressionar Enter.

**Resultado observado:**
A aplicação executou o script injetado, exibindo um pop-up de alerta com o texto "XSS", confirmando a execução de código arbitrário no contexto da página.

![Alerta de XSS disparado](![Evidência 01](Evidencias/01-xss-alert-busca.png))
*Evidência: pop-up de alerta disparado pelo payload injetado no campo de busca, confirmando o XSS refletido.*

**Impacto:**
Em um ataque real, o payload poderia ser usado para roubar cookies de sessão, redirecionar a vítima para sites maliciosos, ou executar ações em nome do usuário logado (session hijacking), caso o link malicioso fosse enviado e clicado por outra pessoa.

**Recomendação:**
Sanitizar e/ou codificar (encode) todo conteúdo controlado pelo usuário antes de renderizá-lo na página, e aplicar uma Content Security Policy (CSP) restritiva para bloquear a execução de scripts não autorizados.

---

## Conclusão

Os três testes demonstram que a aplicação OWASP Juice Shop possui falhas críticas de segurança em múltiplas camadas — autenticação, controle de acesso e validação de entrada. A cadeia de exploração observada (SQL Injection → acesso a dados de outro usuário) ilustra como uma única vulnerabilidade mal corrigida pode escalar para um comprometimento muito mais amplo do sistema. Recomenda-se priorizar a correção da falha de autenticação (Vulnerabilidade 1), já que ela é a porta de entrada para o comprometimento das demais.
