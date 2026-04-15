# Sobre o Trabalho

Ao longo da disciplina, iremos desenvolver o **Chave — Sistema de Autoavaliação de Competências para Pessoas Idosas**, em parceria com a Prof. Letícia Sophia Rocha Machado da UFRGS.

O sistema permite que pessoas idosas avaliem suas próprias competências, recebendo planos de desenvolvimento personalizados e acompanhamento de evolução ao longo do tempo.

O sistema é construído em arquitetura de **microsserviços com microfrontends**, onde cada grupo é responsável por um par MS + MFE de um domínio específico. Os serviços se comunicam via **AWS API Gateway** e **SQS/SNS**, e são deployados no **Amazon ECS (Fargate)**.

O projeto é dividido em duas entregas:

- **T1 — MS Auth + MFE de Autenticação:** cada grupo implementa o serviço de autenticação; a turma vota o melhor, que é adotado por todos na T2.
- **T2 — MS de Domínio + MFE:** cada grupo desenvolve seu microsserviço de domínio (Perfil, Competências, Avaliação, Resultados etc.) integrado ao MS Auth eleito.

O trabalho aplica na prática conceitos de arquitetura distribuída, CI/CD, Module Federation, deploy em cloud e desenvolvimento colaborativo em equipes independentes.
