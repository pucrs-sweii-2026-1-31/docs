# ADR — Documento de Decisão de Arquitetura
### Sistema de Autoavaliação de Competências para Pessoas Idosas

| Campo | Valor |
|---|---|
| **Projeto** | Sistema de autoavaliação de competências para pessoas idosas |
| **Disciplina** | Engenharia de Software II — 98802-02 |
| **Turma** | 30 — PUCRS / 2026-1 |
| **Professor** | Prof. José Pedro Schardosim Simão |
| **Parceira** | Letícia Sophia Rocha Machado |
| **Versão / Data** | v1.0 — Abril de 2026 |
| **Status** | ✔ Aceita |

---

## 1. Contexto e Problema

O projeto consiste no desenvolvimento de um Sistema de Autoavaliação de Competências voltado para pessoas idosas. O sistema deve permitir a criação de modelos de competências baseados no framework CHA (Conhecimentos, Habilidades e Atitudes), a aplicação de questionários de autoavaliação com interface acessível e adaptada, e a geração de relatórios para acompanhamento individual e coletivo.

Este ADR documenta as decisões sobre a arquitetura de microsserviços, microfrontends, autenticação, comunicação entre serviços, tecnologias e infraestrutura adotadas pela turma.

---

## 2. Decisão Arquitetural

Adotar uma arquitetura de microsserviços com microfrontend, composta por unidades deployáveis independentes: 1 serviço de Autenticação/Autorização e 5 serviços de domínio, cada um com seu respectivo microfrontend React.

> **Decisão Central:** 5 microsserviços de domínio + 5 microfrontends React, orquestrados por um Shell App via AWS API Gateway. Comunicação assíncrona via SQS/SNS. Infra: ECS Fargate (MSs), RDS (banco), S3 + CloudFront (frontend), Ministack (dev local). O MS Auth será escolhido por votação após a P1.

---

## 3. Mapeamento dos Microsserviços e Microfrontends

| ID | Nome | Responsabilidade do MS | Responsabilidade do MFE |
|---|---|---|---|
| **MS Auth** | Autenticação / Autorização | Login, OAuth2/OIDC, emissão e refresh de JWT, RBAC por roles, revogação de tokens | Telas de login/logout, gerenciamento de sessão, contexto de auth compartilhado no Shell |
| **MS1** | Competências | Cadastro e gerenciamento de modelos de competências (CHA), competências gerais, específicas e seus elementos constituintes | Cadastro de competências, modelos CHA, elementos constituintes, visualização de estrutura |
| **MS2** | Questionários | Criação e gestão de questionários de autoavaliação associados aos modelos de competências | Criação de questionários, associação a modelos, gestão de versões e publicação |
| **MS3** | Usuários e Grupos | Cadastro e gerenciamento de usuários e grupos, organização de participantes e controle de acessos | Cadastro de participantes, gestão de grupos, administração de perfis e acessos |
| **MS4** | Aplicação da Avaliação | Aplicação dos questionários com interface adaptada para pessoas idosas, registro de respostas | Interface acessível para idosos, aplicação passo a passo, navegação simples e legível |
| **MS5** | Relatórios | Geração de relatórios individuais, por grupo e gerais; análise e acompanhamento de competências avaliadas | Relatórios individuais e coletivos, painel administrativo, exportação de dados e análises |

---

## 4. Visão da Arquitetura

### 4.1 Fluxo Geral

O browser carrega o **Shell App** (S3 + CloudFront), que gerencia roteamento, design system e contexto de auth. Cada MFE é carregado via Module Federation (lazy loading).

As requisições passam pelo **AWS API Gateway**, que valida o JWT por introspecção e roteia para o MS correspondente no **ECS Fargate**. Os MSs recebem os claims do JWT via header, sem revalidar o token.

Comunicação assíncrona entre MSs (ex: questionário respondido → geração de relatório) é feita via **SQS/SNS**. No desenvolvimento local, o **Ministack** emula todos os serviços AWS sem custo.

### 4.2 Camadas da Arquitetura

| Camada | Serviço AWS | Descrição |
|---|---|---|
| **Shell App** | S3 + CloudFront | Build React servido via CDN; gerencia rotas, design system (MUI), shared state (AuthContext) |
| **MFEs** | S3 + CloudFront | 5 microfrontends React + TypeScript, publicados como pacotes NPM, integrados via Module Federation |
| **API Gateway** | AWS API Gateway | Roteamento para microsserviços, rate limit, validação JWT por introspecção, contrato OpenAPI |
| **MS Auth** | ECS Fargate + RDS | Autenticação e autorização; emite e revoga JWTs; User Store em banco relacional (RDS) |
| **Microsserviços (5)** | ECS Fargate + RDS/S3 | Cada MS expõe uma API REST documentada com Swagger; recebe claims do JWT via header |
| **Event Bus** | Amazon SQS / SNS | Comunicação assíncrona entre microsserviços de domínio |
| **Banco de dados** | Amazon RDS (PostgreSQL) | Banco relacional por microsserviço (padrão database-per-service) |
| **Dev local** | Ministack | Emulação local de todos os serviços AWS sem custo |

---

## 5. Serviços AWS Utilizados

| Serviço AWS | Papel na Arquitetura | Ministack |
|---|---|---|
| **Amazon ECS (Fargate)** | Execução dos containers dos microsserviços sem gerenciar servidores | ✔ |
| **Amazon ECR** | Registro privado de imagens Docker dos microsserviços | ✔ |
| **AWS API Gateway** | Ponto de entrada único, rate limit, roteamento e validação JWT | ✔ |
| **Amazon RDS (PostgreSQL)** | Banco de dados relacional, um por microsserviço (database-per-service) | ✔ |
| **Amazon S3** | Hospedagem do Shell App e MFEs, armazenamento de exports | ✔ |
| **Amazon CloudFront** | CDN para distribuição do frontend com baixa latência | — |
| **Amazon SQS** | Filas de mensagens para comunicação assíncrona entre MSs | ✔ |
| **Amazon SNS** | Publicação de eventos/notificações | ✔ |
| **AWS IAM** | Controle de permissões entre serviços AWS (roles por MS) | ✔ |
| **AWS CloudWatch** | Logs centralizados e métricas dos microsserviços | ✔ |

---

## 6. Tecnologias Adotadas

### 6.1 Microfrontend

- **Obrigatório:** React + TypeScript
- **Design system:** MUI (Material UI)
- **Module Federation:** Webpack 5 ou Vite
- **CI/CD:** GitHub Actions
- **Distribuição:** pacotes publicados no NPM; hospedagem via S3 + CloudFront
- Testes unitários com cobertura mínima definida pela equipe

### 6.2 Microsserviços

- **Linguagens recomendadas:** TypeScript (Node.js), Python ou Ruby
- **Desenvolvimento local:** Docker + Ministack (emulação de serviços AWS)
- **CI/CD:** GitHub Actions (testes unitários + geração de release)
- **Release:** imagens Docker publicadas no Amazon ECR
- **Documentação:** Swagger (OpenAPI 3.0)

### 6.3 Infraestrutura e DevOps

- Repositórios: GitHub Org da disciplina, 1 repositório por MS/MFE
- CI/CD: GitHub Actions com pipeline de testes, build e push para ECR
- Ambiente local: Docker Compose + Ministack
- Deploy obrigatório: **AWS** (ECS Fargate + API Gateway + S3 + RDS + SQS/SNS)
- Testes de integração (ponto extra): suite separada da unitária

---

## 7. Microsserviço de Autenticação e Autorização (MS Auth)

### 7.1 Escopo

Único serviço responsável pela identidade dos usuários. Roda no **ECS Fargate** com User Store no **RDS (PostgreSQL)**. Os demais MSs confiam nos claims do JWT entregues pelo API Gateway via header, sem revalidar o token.

### 7.2 Responsabilidades

- Login com usuário/senha (e opcionalmente OAuth2/OIDC)
- Emissão de JWT (access token) e refresh token
- RBAC: controle de permissões por role (admin, participante, gestor)
- Revogação de tokens (logout, invalidação forçada)
- Persistência dos usuários em User Store (RDS PostgreSQL)
- Exposição de endpoint de introspecção para uso pelo AWS API Gateway

### 7.3 Processo de Seleção

Na P1 (13/05), cada grupo entrega seu MS Auth + MFE. Após a entrega, a turma vota o melhor par, que se torna a solução oficial para a entrega final (24/06).

---

## 8. Alternativas Consideradas

| Alternativa | Prós | Contras / Motivo da Rejeição |
|---|---|---|
| Monólito único | Simples de implementar inicialmente; sem overhead de rede entre módulos | Não atende ao requisito pedagógico; não permite desenvolvimento paralelo; baixa escalabilidade |
| Monólito modular | Mais organizado que monólito puro; sem latência interna | Não permite deploys independentes; conflitos de merge entre grupos; não cumpre requisito de microsserviços |
| MFE sem Module Federation (iframes) | Isolamento total de contexto | UX degradada; sem compartilhamento de estado; não permite compartilhar design system |
| Serviço de auth externo (Auth0, Keycloak) | Pronto para produção; alta segurança | Não cumpre o requisito pedagógico de implementar o MS Auth; limita o aprendizado da turma |
| Comunicação síncrona pura (REST entre MSs) | Simplicidade de implementação | Cria acoplamento temporal; falhas em cascata; não escala bem para operações assíncronas |
| Localstack no lugar de Ministack | Mais conhecido e com documentação ampla | Footprint maior, mais difícil de configurar localmente; Ministack oferece setup mais simples |

---

## 9. Consequências

### 9.1 Positivas

- **Desenvolvimento paralelo:** cada grupo trabalha de forma independente no seu MS + MFE
- **Escalabilidade:** ECS Fargate escala cada serviço de forma independente
- **Resiliência:** falha em um serviço não derruba a plataforma inteira
- **Aprendizado pedagógico completo:** cada grupo domina um fluxo ponta-a-ponta com serviços AWS reais
- **Padronização:** AWS API Gateway + JWT + Swagger garante contrato claro entre equipes
- **Interface acessível:** arquitetura desacoplada permite otimizar o MFE de aplicação para idosos sem impactar outros serviços

### 9.2 Negativas / Riscos

- **Overhead de comunicação:** latência adicional nas chamadas entre serviços via API Gateway
- **Complexidade operacional:** orquestração de múltiplos serviços AWS (mesmo no Ministack)
- **Consistência eventual:** operações assíncronas via SQS/SNS exigem tratamento de falhas e idempotência
- **Curva de aprendizado:** AWS SDK, ECS, IAM roles e Module Federation podem ser novidade para parte da turma
- **Custo AWS:** deploy real na AWS gera custos; grupos devem usar Free Tier e monitorar via CloudWatch

---

## 10. Histórico de Revisões

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | Abr/2026 | Prof. | Versão inicial do ADR com mapeamento completo dos 5 MSs + MFEs, tecnologias e cronograma |

---

*Decisões arquiteturais relevantes que desviem deste documento devem ser registradas como nova versão do ADR, com justificativa, data e autores.*
