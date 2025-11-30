# Mapeamento Detalhado de Cenários e Lógica da IA

**Data**: 19 de Novembro de 2025

---

## 1. Estrutura de Resposta da IA (JSON)

A IA sempre responderá com a seguinte estrutura JSON:

```json
{
  "intent": "<string>",
  "sub_intent": {},
  "response_type": "<fixed|generated>",
  "response_text": "<string>",
  "handoff_needed": <boolean>,
  "handoff_reason": "<string>"
}
```

| Campo | Descrição |
|---|---|
| `intent` | A intenção principal do usuário (receita, agendamento, etc.) |
| `sub_intent` | Detalhes da intenção (tipo de receita, especialidade, etc.) |
| `response_type` | Se a resposta é fixa ou gerada pela IA |
| `response_text` | O texto da resposta (se gerada pela IA) |
| `handoff_needed` | Se a conversa precisa ser transferida para um humano |
| `handoff_reason` | O motivo da transferência |

---

## 2. Cenários Detalhados

### Cenário 1: Agendamento de Consulta

#### Fluxo Ideal

1.  **Usuário**: "Quero agendar uma consulta"
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "agendamento",
      "sub_intent": {},
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
3.  **Node `Processar Agendamento` (Resposta Fixa)**: Mensagem com a lista de 13 especialidades.
4.  **Usuário**: "Cardiologia"
5.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "agendamento",
      "sub_intent": { "especialidade": "cardiologia" },
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
6.  **Node `Processar Agendamento` (Resposta Fixa)**: Mensagem personalizada com o nome da Dra. Mariana Leão.

#### Variações de Entrada do Usuário

- "Preciso de um cardiologista"
- "Gostaria de marcar com a Dra. Mariana"
- "Tem horário para nefrologia?"

#### Fluxo de Handoff (Psiquiatria)

1.  **Usuário**: "Quero agendar com um psiquiatra"
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "agendamento",
      "sub_intent": { "especialidade": "psiquiatria" },
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": true,
      "handoff_reason": "Agendamento com Psiquiatria"
    }
    ```
3.  **Node `Processar Agendamento` (Resposta Fixa)**: Mensagem de transferência para atendimento humano.

---

### Cenário 2: Solicitação de Receita

#### Fluxo Ideal (Receita Controlada)

1.  **Usuário**: "Preciso da minha receita azul"
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "receita",
      "sub_intent": { "tipo_receita": "controlada" },
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
3.  **Node `Processar Receita` (Resposta Fixa)**: Mensagem pedindo a localização.
4.  **Usuário**: "Moro em Brasília"
5.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "receita",
      "sub_intent": { "tipo_receita": "controlada", "localizacao": "df" },
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
6.  **Node `Processar Receita` (Resposta Fixa)**: Mensagem de confirmação com número de protocolo.

#### Variações de Entrada do Usuário

- "Minha receita acabou"
- "Preciso de uma receita branca"
- "Sou do Rio de Janeiro e preciso da receita"

---

### Cenário 3: Dúvida sobre Terapia

#### Fluxo Ideal (Resposta Gerada pela IA)

1.  **Usuário**: "O que é o programa VOAR?"
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "terapia",
      "sub_intent": { "tipo_terapia": "voar" },
      "response_type": "generated",
      "response_text": "O Programa VOAR (Vida, Organização, Autonomia e Habilidades) é um protocolo internacional criado pelo Dr. Tiago Figueiredo, focado no tratamento de crianças e adolescentes com TDAH. Ele envolve a participação ativa da família e trabalha habilidades como autonomia, regulação emocional e competência social. Gostaria de saber mais?",
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
3.  **Node `Processar Dúvida` (Resposta Gerada)**: Envia o `response_text` da IA para o paciente.

---

### Cenário 4: Envio de Documento

#### Fluxo Ideal

1.  **Usuário**: (Envia um arquivo PDF)
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "envio_documento",
      "sub_intent": {},
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
3.  **Node `Processar Documento` (Resposta Fixa)**: Mensagem pedindo contexto sobre o arquivo.
4.  **Usuário**: "É um pedido de exame para agendar uma consulta"
5.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "agendamento",
      "sub_intent": {},
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": false,
      "handoff_reason": null
    }
    ```
6.  **Node `Processar Agendamento` (Resposta Fixa)**: Mensagem com a lista de especialidades.

---

### Cenário 5: Reclamação

#### Fluxo Ideal

1.  **Usuário**: "Estou muito insatisfeito com o atendimento"
2.  **IA (Resposta JSON)**:
    ```json
    {
      "intent": "duvida",
      "sub_intent": { "tipo_duvida": "reclamacao" },
      "response_type": "fixed",
      "response_text": null,
      "handoff_needed": true,
      "handoff_reason": "Reclamação do paciente"
    }
    ```
3.  **Node `Handoff Humano` (Resposta Fixa)**: Mensagem de transferência para atendimento humano.

---

## 3. Lógica de Decisão da IA

- **`intent`**: Definido com base em palavras-chave e contexto. Ex: "agendar", "marcar", "consulta" → `agendamento`.
- **`sub_intent`**: Extraído do contexto. Ex: "cardiologista" → `{ "especialidade": "cardiologia" }`.
- **`response_type`**: `fixed` para fluxos críticos (agendamento, receita, handoff), `generated` para dúvidas e terapias.
- **`handoff_needed`**: `true` para Psiquiatria, Neuropsicologia, reclamações ou quando a IA não entende a solicitação.

Este documento serve como um guia detalhado para a implementação e teste do workflow, garantindo que todos os cenários sejam tratados de forma consistente e eficiente.
