# Engenharia de Privacidade na Prática: Implementando o Direito ao Apagamento (RGPD) e Proteção de Segredos no Django

**Tags:** `#AnaliseFuncional` `#Cybersecurity` `#RGPD` `#PrivacyByDesign` `#Python` `#Django` `#Git`

---

## 🎯 O Contexto / Problema
Durante o desenvolvimento de um CRM Imobiliário em Python/Django, o foco inicial concentrou-se nas funcionalidades comerciais (gestão de contactos, imóveis e campanhas de comunicação). Contudo, na revisão final antes da entrada em produção e da publicação do código no GitHub, surgiram dois obstáculos críticos de cibersegurança e conformidade legal:

1. **Lacuna de Conformidade Legal (Artigo 17.º do RGPD):** O sistema possuía bloqueio preventivo de comunicações sem consentimento (*opt-in*) e links de cancelamento (*opt-out*), mas faltava um fluxo formal e seguro para executar o **Direito ao Apagamento** ("direito a ser esquecido") quando solicitado pelo titular, garantindo simultaneamente que o cliente recebia um comprovativo oficial dessa eliminação.
2. **Exposição de Credenciais no Controlo de Versões:** O ficheiro `settings.py` continha chaves sensíveis em texto limpo (`SECRET_KEY`, `FIELD_ENCRYPTION_KEY` para criptografia dos dados na base de dados e credenciais SMTP), além da existência de ficheiros locais (`db.sqlite3`) que jamais poderiam ser enviados para um repositório remoto.

---

## 🧠 Decisão Arquitetural: Visão de Análise Funcional (Porquê Python / Django?)
Sob a perspetiva de **Análise Funcional**, a escolha do ecossistema Python/Django baseou-se no princípio **DRY (*Don't Repeat Yourself*)** e na necessidade de traduzir regras de negócio e requisitos legais do RGPD em fluxos automatizados, auditáveis e livres de retrabalho manual:

* **Eliminação de Retrabalho e Erro Humano:** Em vez de depender de verificações manuais em folhas de cálculo para validar consentimentos ou formatar comunicações, o sistema automatiza a validação de regras (*opt-in*), gera tokens únicos de revogação (*opt-out*) e regista o histórico numa única operação.
* **Segurança e Produtividade (*"Batteries Included"*):** A legibilidade do Python aproxima a lógica técnica das regras de negócio, enquanto o Django oferece proteção nativa contra vulnerabilidades comuns (CSRF, SQL Injection), autenticação integrada e mapeamento objeto-relacional (ORM), permitindo concentrar o esforço na conformidade normativa e na integridade dos dados.

---

## 🔍 A Investigação
Ao analisar a ordem de execução necessária para cumprir o Artigo 17.º do RGPD, identifiquei um problema lógico clássico em sistemas de informação: **se eliminarmos o registo do cliente da base de dados em primeiro lugar, perdemos o endereço de e-mail necessário para lhe enviar a notificação formal de apagamento**. Por outro lado, se enviarmos o e-mail e ocorrer uma falha na eliminação, geramos uma inconsistência jurídica.

Paralelamente, na preparação para o deploy em PostgreSQL (macOS) e sincronização com o GitHub, verifiquei que:
* Hardcoding de senhas no `settings.py` compromete todo o histórico de commits do Git, mesmo que as senhas sejam apagadas em commits posteriores.
* Ficheiros de ambiente virtual (`venv/`), bases de dados locais (`db.sqlite3`) e dumps de backup (`*.sql.gz`) exigem regras explícitas no `.gitignore` antes de qualquer `git add`.

---

## 🛠️ A Solução

### 1. Fluxo Atómico para o Direito ao Apagamento (`views.py` e Template)
Implementei uma *view* dedicada (`apagar_dados_cliente_rgpd`) protegida por autenticação (`@login_required`), método `POST` e confirmação explícita via modal Bootstrap para evitar eliminações acidentais:

```python
# 1. Extração e sanitização dos contactos antes da eliminação
nome_limpo = " ".join(str(cliente.nome_completo).splitlines()).strip()
email_limpo = "".join(str(cliente.email_principal or '').splitlines()).strip()

# 2. Envio da notificação formal (Art. 17.º RGPD) enquanto o contacto existe
if email_limpo and '@' in email_limpo:
    email_msg = EmailMultiAlternatives(
        subject="Confirmação de Eliminação de Dados Pessoais (RGPD)",
        body=corpo_texto,
        from_email=settings.DEFAULT_FROM_EMAIL,
        to=[email_limpo]
    )
    email_msg.attach_alternative(corpo_html, "text/html")
    email_msg.send()

# 3. Eliminação definitiva em cascata (cliente, negócios e histórico)
cliente.delete()