# Documentação Completa - Workflow Cognus v5.0

**Data**: 19 de Novembro de 2025  
**Autor**: Manus AI

---

## Sumário Executivo

Este documento apresenta a reavaliação completa da lógica e estrutura do workflow de automação WhatsApp da Clínica Cognus. O projeto foi reestruturado em uma arquitetura de 5 camadas, garantindo clareza de responsabilidades, facilidade de manutenção e escalabilidade para futuras expansões.

A nova versão 5.0 do workflow implementa todos os fluxos de negócio definidos em reunião com o cliente, incluindo mensagens personalizadas com os nomes dos profissionais do corpo clínico, integração com Trello para gestão de receitas controladas e handoff inteligente para atendimento humano quando necessário.

---

## 1. Arquitetura de 5 Camadas

O workflow foi organizado em 5 camadas lógicas, cada uma com responsabilidades bem definidas:

### Camada 1: Entrada e Parser

Esta camada é responsável por receber as mensagens do WhatsApp e transformá-las em um formato estruturado que pode ser processado pelo restante do workflow. O node **`playload`** é o coração desta camada, extraindo informações como número de telefone, nome do paciente, tipo de mensagem (texto, áudio, imagem, documento) e criando o campo crítico **`sessionId`** que permite ao Redis Chat Memory manter o histórico da conversa.

### Camada 2: Contexto e IA

Esta camada prepara o contexto necessário para que a Inteligência Artificial possa tomar decisões precisas. O sistema de buffer (nodes `push`, `Wait`, `msgs`, `junta_msgs`) coleta todas as mensagens recentes do paciente e as organiza em um histórico formatado. O **`AI Agent`**, equipado com o prompt v5.0, analisa este histórico e classifica a intenção do paciente em categorias como receita, agendamento, terapia, relatório ou dúvida. A IA também determina se a solicitação é complexa o suficiente para exigir transferência para um atendente humano.

### Camada 3: Roteamento

O **`Switch por Intenção`** é o roteador principal do workflow. Com base na classificação feita pela IA, ele direciona o fluxo para o node de negócio apropriado. Esta camada garante que cada tipo de solicitação seja tratada pelo código especializado correto.

### Camada 4: Negócio

Esta é a camada onde as regras de negócio específicas da Clínica Cognus são implementadas. Cada node desta camada é responsável por um tipo de solicitação. O **`Processar Receita`** lida com receitas brancas e controladas, criando cards no Trello quando necessário. O **`Processar Agendamento`** apresenta mensagens personalizadas com os nomes dos profissionais e decide quando transferir para atendimento humano. O **`Processar Terapia`** apresenta a lista completa de terapias e programas especiais da clínica.

### Camada 5: Saída

O **`Enviar texto`** é o ponto de saída único do workflow. Todos os fluxos convergem para este node, que envia a mensagem final para o paciente através da Evolution API.

![Diagrama de Arquitetura](https://private-us-east-1.manuscdn.com/sessionFile/oNehb8G6BqjzlTNypn5ASq/sandbox/QRvEEn7WmTpvxIhqdgLg2n-images_1764432840356_na1fn_L2hvbWUvdWJ1bnR1L2RpYWdyYW1hX2FycXVpdGV0dXJhX3Y1.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvb05laGI4RzZCcWp6bFROeXBuNUFTcS9zYW5kYm94L1FSdkVFbjdXbVRwdnhJaHFkZ0xnMm4taW1hZ2VzXzE3NjQ0MzI4NDAzNTZfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyUnBZV2R5WVcxaFgyRnljWFZwZEdWMGRYSmhYM1kxLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=J24yz77XJW6rJEDDlqQV5RShXMGyoxXbfU6hM1F~QIB76fVxImtrCHOphtU4QQ7AXkacyGuWKrclR-Sm9E26O9sQsxFGplh8PP~ZwGhVNQ46LTwdV71ZEIT2bNxYgrj~2MWTxpIDWCABmktLk~WLMSUV5rqIdCnmvTXICiKNWpeBbiD-XttKteXH6p3yEfKJD1lcO3m8pgadtNiCszETz7oAkkrYWrjFeEx6z5yB9zVC3uO5QbU25SKpXqPkm-3NPlytjkx9ySKZHWBjFTsJ3ZTj2QyfFYrrSq-wTDcFFYGxXVxsrQEaiXdrCs9btCshJy~6Yhc9saF9g3syC8MxqA__)

---

## 2. Responsabilidades dos Nós

A tabela abaixo detalha a responsabilidade de cada nó no workflow:

| Camada | Nó | Responsabilidade |
|---|---|---|
| **1. Entrada e Parser** | Webhook WhatsApp Evolution | Recebe todas as mensagens do WhatsApp |
| | playload | Transforma o payload bruto em JSON estruturado, cria o `sessionId` |
| **2. Contexto e IA** | Switch1 | Roteia por tipo de mensagem (texto ou áudio) |
| | Download Audio | Baixa o arquivo de áudio |
| | transcript | Transcreve áudio para texto |
| | push (Redis) | Adiciona mensagem ao histórico |
| | Wait | Aguarda mais mensagens |
| | msgs (Redis) | Recupera histórico completo |
| | junta_msgs | Formata histórico para a IA |
| | AI Agent | Classifica intenção e gera respostas |
| | Validate AI Output | Valida formato do JSON da IA |
| **3. Roteamento** | Switch por Intenção | Direciona para o node de negócio correto |
| **4. Negócio** | Processar Receita | Lida com receitas (brancas e controladas) |
| | Processar Agendamento | Lida com agendamentos (com handoff se necessário) |
| | Processar Terapia | Apresenta terapias e programas |
| | Informar E-mail | Informa e-mail para relatórios |
| | Handoff Humano | Prepara transferência para humano |
| | Processar Documento | Lida com envio de arquivos |
| | Processar Dúvida | Responde dúvidas gerais |
| | Create Card Trello | Cria cards para receitas controladas |
| **5. Saída** | Enviar texto | Envia mensagem final ao paciente |

---

## 3. Cenários Mapeados

O workflow foi projetado para lidar com 6 cenários principais:

### Cenário 1: Agendamento de Consulta

O paciente solicita agendar uma consulta. A IA identifica a especialidade desejada e apresenta uma mensagem personalizada com o nome do profissional responsável. Para Psiquiatria e Neuropsicologia, o sistema automaticamente transfere para atendimento humano, conforme solicitado pelo cliente.

### Cenário 2: Solicitação de Receita Controlada

O paciente solicita uma receita controlada (azul ou amarela). O sistema pergunta a localização e, com base na resposta (Brasília, Rio de Janeiro ou outra cidade), cria um card no Trello na lista apropriada (Receitas DF, Receitas RJ ou Receitas Correios) e retorna um número de protocolo ao paciente.

### Cenário 3: Dúvida sobre Terapia

O paciente pergunta sobre uma terapia ou programa específico, como o Programa VOAR. O sistema apresenta a lista completa de terapias e profissionais, e a IA gera uma resposta personalizada sobre o programa solicitado.

### Cenário 4: Envio de Documento

O paciente envia um arquivo (PDF, imagem, etc.) sem explicação. O sistema solicita contexto sobre o documento e, com base na resposta, direciona para o fluxo apropriado.

### Cenário 5: Dúvida sobre Convênio

O paciente pergunta se a clínica aceita determinado convênio. O sistema responde que a clínica atende apenas particular, mas fornece recibo para reembolso.

### Cenário 6: Reclamação

O paciente expressa insatisfação ou faz uma reclamação. A IA detecta o tom negativo e automaticamente transfere para atendimento humano para garantir um tratamento adequado da situação.

---

## 4. Plano de Implementação

O projeto será implementado em 5 fases ao longo de 9 dias:

**Fase 1 (1 dia)**: Estrutura e Parser - Garantir que todos os dados de entrada sejam processados corretamente.

**Fase 2 (2 dias)**: Contexto e IA - Implementar o prompt v5.0 e garantir que a IA tenha todo o contexto necessário.

**Fase 3 (3 dias)**: Roteamento e Fluxos de Negócio - Implementar a lógica de negócio para cada cenário.

**Fase 4 (1 dia)**: Integrações e Saída - Garantir que o workflow se comunique corretamente com sistemas externos.

**Fase 5 (2 dias)**: Testes e Documentação - Executar testes de ponta a ponta e finalizar a documentação.

---

## 5. Entregáveis

Ao final do projeto, serão entregues:

1.  **Cognus_WhatsApp_Agent_v5.0_FINAL.json**: Arquivo do workflow pronto para importação no n8n.
2.  **Documento_Executivo_Cognus_v5.0.md**: Documento completo para o cliente com todas as mensagens, lógicas e fluxos.
3.  **Guia_Implementacao_v5.0.md**: Guia técnico para implementação do workflow.
4.  **Changelog_v5.0.md**: Detalhamento de todas as mudanças da v4.0 para v5.0.
5.  **Diagrama_Arquitetura_v5.0.png**: Diagrama visual da arquitetura de 5 camadas.

---

## 6. Próximos Passos

Após a aprovação deste planejamento, a equipe de desenvolvimento iniciará a implementação seguindo o cronograma estabelecido. Recomenda-se uma reunião de alinhamento ao final de cada fase para garantir que o projeto está seguindo conforme o esperado e que eventuais ajustes possam ser feitos rapidamente.
