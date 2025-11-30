# Guia Técnico Completo para Geração do Workflow n8n v5.0 - Clínica Cognus

**Versão**: 1.0  
**Data**: 19 de Novembro de 2025  
**Objetivo**: Fornecer especificações técnicas detalhadas para geração do workflow completo no n8n

---

## 1. Visão Geral

Este documento contém todas as especificações técnicas necessárias para gerar o workflow `Cognus_WhatsApp_Agent_v5.0_FINAL.json` no n8n 1.115.3 ou superior.

### Arquitetura de 5 Camadas

1. **Entrada e Parser**: Recebe e valida dados
2. **Contexto e IA**: Prepara contexto e classifica intenções
3. **Roteamento**: Decide o caminho a seguir
4. **Negócio**: Executa ações específicas
5. **Saída**: Envia resposta ao paciente

### Metadados do Workflow

```json
{
  "name": "Cognus WhatsApp Agent v5.0",
  "nodes": [],
  "connections": {},
  "active": false,
  "settings": {},
  "versionId": "v5.0"
}
```

---

## 2. CAMADA 1: ENTRADA E PARSER

### Node 1: Webhook WhatsApp Evolution

**Tipo**: `n8n-nodes-base.webhook`

**Configuração**:
```json
{
  "httpMethod": "POST",
  "path": "cognus-whatsapp",
  "responseMode": "lastNode",
  "options": {}
}
```

**Posição**: `[250, 300]`

---

### Node 2: playload (Parser)

**Tipo**: `n8n-nodes-base.code`

**Código JavaScript**:
```javascript
// Parser v5.0 - Extrai e valida dados do payload Evolution API
const items = $input.all();
const results = [];

for (const item of items) {
  try {
    const body = item.json.body || item.json;
    const data = body.data || body;
    
    // Extração de dados
    const from = data.key?.remoteJid || data.from || '';
    const fromName = data.pushName || data.fromName || 'Paciente';
    const messageType = data.message?.conversation ? 'text' : 
                       data.message?.extendedTextMessage ? 'text' :
                       data.message?.audioMessage ? 'audio' :
                       data.message?.imageMessage ? 'image' :
                       data.message?.videoMessage ? 'video' :
                       data.message?.documentMessage ? 'document' : 'unknown';
    
    // Extração de texto
    let text = '';
    if (data.message?.conversation) {
      text = data.message.conversation;
    } else if (data.message?.extendedTextMessage?.text) {
      text = data.message.extendedTextMessage.text;
    }
    
    // Extração de URL de áudio
    let audioUrl = null;
    if (messageType === 'audio' && data.message?.audioMessage?.url) {
      audioUrl = data.message.audioMessage.url;
    }
    
    // Validação de número de telefone
    const phoneRegex = /^\d{10,15}@/;
    if (!phoneRegex.test(from)) {
      throw new Error(`Número de telefone inválido: ${from}`);
    }
    
    // Criação do sessionId (crítico para o Redis Chat Memory)
    const sessionId = from.split('@')[0];
    
    // Timestamp
    const timestamp = data.messageTimestamp || Math.floor(Date.now() / 1000);
    
    // Resultado
    results.push({
      json: {
        sessionId: sessionId,  // CAMPO CRÍTICO
        dataw: {
          event: body.event || 'messages.upsert',
          instance: body.instance || 'cognus',
          from: from,
          fromName: fromName,
          message: text,
          type: messageType,
          audioUrl: audioUrl,
          timestamp: timestamp
        }
      }
    });
    
  } catch (error) {
    // Em caso de erro, retorna um objeto de erro
    results.push({
      json: {
        error: true,
        errorMessage: error.message,
        originalPayload: item.json
      }
    });
  }
}

return results;
```

**Posição**: `[450, 300]`

---

## 3. CAMADA 2: CONTEXTO E IA

### Node 3: Switch1 (Por Tipo de Mensagem)

**Tipo**: `n8n-nodes-base.switch`

**Configuração**:
```json
{
  "mode": "rules",
  "rules": {
    "rules": [
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.dataw.type }}",
              "operation": "equals",
              "value2": "text"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "texto"
      },
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.dataw.type }}",
              "operation": "equals",
              "value2": "audio"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "audio"
      }
    ]
  },
  "options": {}
}
```

**Posição**: `[650, 300]`

---

### Node 4: Download Audio

**Tipo**: `n8n-nodes-base.httpRequest`

**Configuração**:
```json
{
  "url": "={{ $json.dataw.audioUrl }}",
  "method": "GET",
  "responseFormat": "file",
  "options": {
    "timeout": 30000
  }
}
```

**Posição**: `[850, 400]`

**Conexão**: Recebe de `Switch1` (saída "audio")

---

### Node 5: transcript (Transcrição de Áudio)

**Tipo**: `n8n-nodes-base.openAi`

**Configuração**:
```json
{
  "resource": "audio",
  "operation": "transcribe",
  "model": "whisper-1",
  "binaryPropertyName": "data",
  "options": {}
}
```

**Posição**: `[1050, 400]`

---

### Node 6: Set Transcription (Adiciona transcrição ao dataw)

**Tipo**: `n8n-nodes-base.set`

**Configuração**:
```json
{
  "values": {
    "string": [
      {
        "name": "dataw.message",
        "value": "={{ $json.text }}"
      }
    ]
  },
  "options": {
    "keepOnlySet": false
  }
}
```

**Posição**: `[1250, 400]`

---

### Node 7: push (Redis - Adiciona mensagem ao buffer)

**Tipo**: `n8n-nodes-base.redis`

**Configuração**:
```json
{
  "operation": "push",
  "list": "={{ $json.sessionId }}:buffer",
  "messageData": "={{ $json.dataw.message }}",
  "options": {
    "ttl": 300
  }
}
```

**Posição**: `[1050, 300]`

**Conexão**: Recebe de `Switch1` (saída "texto") e de `Set Transcription`

---

### Node 8: Wait (Aguarda mais mensagens)

**Tipo**: `n8n-nodes-base.wait`

**Configuração**:
```json
{
  "resume": "after",
  "amount": 5,
  "unit": "seconds"
}
```

**Posição**: `[1250, 300]`

---

### Node 9: msgs (Redis - Recupera buffer)

**Tipo**: `n8n-nodes-base.redis`

**Configuração**:
```json
{
  "operation": "lrange",
  "list": "={{ $json.sessionId }}:buffer",
  "start": 0,
  "end": -1
}
```

**Posição**: `[1450, 300]`

---

### Node 10: junta_msgs (Formata histórico)

**Tipo**: `n8n-nodes-base.code`

**Código JavaScript**:
```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const messages = item.json.list || [];
  const formattedHistory = messages.map((msg, index) => {
    return `Mensagem ${index + 1}: ${msg}`;
  }).join('\n');
  
  results.push({
    json: {
      ...item.json,
      conversationHistory: formattedHistory
    }
  });
}

return results;
```

**Posição**: `[1650, 300]`

---

### Node 11: AI Agent (Classificação e Resposta)

**Tipo**: `n8n-nodes-base.openAi`

**Configuração**:
```json
{
  "resource": "chat",
  "operation": "message",
  "model": "gpt-4o",
  "messages": {
    "values": [
      {
        "role": "system",
        "content": "VEJA O PROMPT COMPLETO NA SEÇÃO 4"
      },
      {
        "role": "user",
        "content": "={{ $json.conversationHistory }}"
      }
    ]
  },
  "options": {
    "temperature": 0.3,
    "responseFormat": "json_object"
  }
}
```

**Posição**: `[1850, 300]`

---

## 4. PROMPT DA IA v5.0

```
Você é o assistente virtual da Clínica Cognus, uma clínica especializada em saúde mental e desenvolvimento infantil. Sua função é classificar as solicitações dos pacientes e fornecer respostas precisas e humanizadas.

## CORPO CLÍNICO (12 profissionais)

### Médicos
- Dr. Tiago Figueiredo, MD, Ph.D. - Psiquiatra (Co-fundador, Criador do Programa VOAR)
- Dra. Mariana Leão, MD. - Cardiologista
- Dr. Diêgo Figueiredo, MD. - Nefrologista
- Dra. Anna Gabriela, MD. - Endocrinologista
- Dr. Allan Marinho - Psiquiatra

### Terapeutas e Psicólogos
- Karla Valente Sanches - Neuropsicóloga
- Ana Carolina Caputo Aucélio Rabetti - Fonoaudióloga
- Wesley Oliveira - Terapeuta Ocupacional
- Juliana Nogueira, MSc. - Psicóloga
- Gleisse Nunes Pires da Silva - Psicóloga, Educadora Parental
- Ana Paula Gheller - Psicóloga
- Camila Barros Ribeiro - Neuropsicóloga

## ESPECIALIDADES (13)
Psiquiatria, Cardiologia, Nefrologia, Endocrinologia, Cirurgia Vascular, Psicologia, Educação Parental, Fonoaudiologia, Neuropsicologia, Neuropsicopedagogia, Terapia Ocupacional, Avaliação do Perfil Cognitivo, Avaliação da Cognição Social

## TERAPIAS E PROGRAMAS (6 + 2)
- Fonoaudiologia
- Psicoterapia
- Psicopedagogia
- Terapia Ocupacional
- Educação Parental
- Neuropsicologia
- **Programa VOAR**: Vida, Organização, Autonomia e Habilidades - Protocolo internacional para crianças e adolescentes com TDAH, criado pelo Dr. Tiago Figueiredo
- **Programa de Leitura e Escrita**: Protocolo para dificuldades de aprendizagem

## REGRAS DE NEGÓCIO

1. **Agendamentos**: Psiquiatria e Avaliação Neuropsicológica SEMPRE exigem handoff humano.
2. **Receitas Controladas**: Perguntar a localização (DF, RJ ou outra).
3. **Convênio**: A clínica atende apenas particular.
4. **Relatórios/Laudos**: Devem ser solicitados por e-mail: contato@clinicacognus.com.br

## ESTRUTURA DE RESPOSTA (JSON)

Você DEVE responder SEMPRE em JSON com a seguinte estrutura:

{
  "intent": "<receita|agendamento|terapia|relatorio|envio_documento|duvida>",
  "sub_intent": {},
  "response_type": "<fixed|generated>",
  "response_text": "<string ou null>",
  "handoff_needed": <boolean>,
  "handoff_reason": "<string ou null>"
}

### Campos:
- **intent**: A intenção principal do usuário
- **sub_intent**: Detalhes (ex: {"especialidade": "cardiologia", "tipo_receita": "controlada", "localizacao": "df"})
- **response_type**: "fixed" para fluxos críticos (agendamento, receita), "generated" para dúvidas e terapias
- **response_text**: Texto da resposta (apenas se response_type = "generated")
- **handoff_needed**: true para Psiquiatria, Neuropsicologia, reclamações
- **handoff_reason**: Motivo do handoff

## EXEMPLOS

Usuário: "Quero agendar com um cardiologista"
Resposta:
{
  "intent": "agendamento",
  "sub_intent": {"especialidade": "cardiologia"},
  "response_type": "fixed",
  "response_text": null,
  "handoff_needed": false,
  "handoff_reason": null
}

Usuário: "Preciso de uma receita azul, moro em Brasília"
Resposta:
{
  "intent": "receita",
  "sub_intent": {"tipo_receita": "controlada", "localizacao": "df"},
  "response_type": "fixed",
  "response_text": null,
  "handoff_needed": false,
  "handoff_reason": null
}

Usuário: "O que é o programa VOAR?"
Resposta:
{
  "intent": "terapia",
  "sub_intent": {"tipo_terapia": "voar"},
  "response_type": "generated",
  "response_text": "O Programa VOAR (Vida, Organização, Autonomia e Habilidades) é um protocolo internacional criado pelo Dr. Tiago Figueiredo, focado no tratamento de crianças e adolescentes com TDAH. Ele envolve a participação ativa da família e trabalha habilidades como autonomia, regulação emocional e competência social. Gostaria de saber mais?",
  "handoff_needed": false,
  "handoff_reason": null
}
```

---

## 5. CAMADA 3: ROTEAMENTO

### Node 12: Switch por Intenção

**Tipo**: `n8n-nodes-base.switch`

**Configuração**:
```json
{
  "mode": "rules",
  "rules": {
    "rules": [
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.message.content.intent }}",
              "operation": "equals",
              "value2": "receita"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "receita"
      },
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.message.content.intent }}",
              "operation": "equals",
              "value2": "agendamento"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "agendamento"
      },
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.message.content.intent }}",
              "operation": "equals",
              "value2": "terapia"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "terapia"
      },
      {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.message.content.intent }}",
              "operation": "equals",
              "value2": "relatorio"
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "relatorio"
      },
      {
        "conditions": {
          "boolean": [
            {
              "value1": "={{ $json.message.content.handoff_needed }}",
              "operation": "equals",
              "value2": true
            }
          ]
        },
        "renameOutput": true,
        "outputKey": "handoff"
      }
    ]
  },
  "options": {}
}
```

**Posição**: `[2050, 300]`

---

## 6. CAMADA 4: NEGÓCIO

### Node 13: Processar Receita

**Tipo**: `n8n-nodes-base.code`

**Código JavaScript**:
```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const aiResponse = item.json.message.content;
  const subIntent = aiResponse.sub_intent || {};
  const tipoReceita = subIntent.tipo_receita;
  const localizacao = subIntent.localizacao;
  
  let mensagem = '';
  let trelloList = null;
  
  if (tipoReceita === 'branca') {
    mensagem = 'Receita branca confirmada! O Dr. Tiago enviará sua receita diretamente pelo aplicativo da clínica em até 24 horas. 📋';
  } else if (tipoReceita === 'controlada') {
    if (!localizacao) {
      mensagem = 'Para continuar com sua receita controlada, por favor, me informe sua cidade e estado. 📍';
    } else {
      const protocolo = `REC-${Date.now().toString().slice(-6)}`;
      
      if (localizacao === 'df') {
        mensagem = `Receita controlada confirmada para Brasília/DF! ✅\n\nSua receita será preparada e você pode retirá-la na clínica em até 48 horas.\n\n📋 Protocolo: ${protocolo}`;
        trelloList = 'Receitas DF';
      } else if (localizacao === 'rj') {
        mensagem = `Receita controlada confirmada para Rio de Janeiro/RJ! ✅\n\nSua receita será preparada e você pode retirá-la na clínica em até 48 horas.\n\n📋 Protocolo: ${protocolo}`;
        trelloList = 'Receitas RJ';
      } else {
        mensagem = `Receita controlada confirmada! ✅\n\nSua receita será enviada pelos Correios em até 5 dias úteis.\n\n📋 Protocolo: ${protocolo}`;
        trelloList = 'Receitas Correios';
      }
    }
  } else {
    mensagem = 'Qual tipo de receita você precisa?\n\n1️⃣ Receita Branca\n2️⃣ Receita Controlada (Azul/Amarela)';
  }
  
  results.push({
    json: {
      ...item.json,
      responseMessage: mensagem,
      trelloList: trelloList
    }
  });
}

return results;
```

**Posição**: `[2250, 200]`

---

### Node 14: Processar Agendamento

**Tipo**: `n8n-nodes-base.code`

**Código JavaScript**:
```javascript
const items = $input.all();
const results = [];

const especialidadesComProfissionais = {
  'cardiologia': 'Excelente escolha! A especialidade de Cardiologia é atendida pela **Dra. Mariana Leão, MD.**, uma profissional de referência na área. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'nefrologia': 'Excelente escolha! A especialidade de Nefrologia é atendida pelo **Dr. Diêgo Figueiredo, MD.**, Responsável Técnico da clínica. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'endocrinologia': 'Excelente escolha! A especialidade de Endocrinologia é atendida pela **Dra. Anna Gabriela, MD.**. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'psicologia': 'Excelente escolha! A especialidade de Psicologia é atendida por **Juliana Nogueira, MSc.** e **Ana Paula Gheller**. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'fonoaudiologia': 'Excelente escolha! A especialidade de Fonoaudiologia é atendida por **Ana Carolina Caputo Aucélio Rabetti**. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'terapia_ocupacional': 'Excelente escolha! A Terapia Ocupacional é atendida por **Wesley Oliveira**. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅',
  'educacao_parental': 'Excelente escolha! A Educação Parental é atendida por **Gleisse Nunes Pires da Silva**, certificada em Parentalidade Consciente. Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅'
};

for (const item of items) {
  const aiResponse = item.json.message.content;
  const subIntent = aiResponse.sub_intent || {};
  const especialidade = subIntent.especialidade;
  
  let mensagem = '';
  
  if (!especialidade) {
    mensagem = 'Para qual especialidade você gostaria de agendar?\n\n🩺 **Médicas**: Psiquiatria, Cardiologia, Nefrologia, Endocrinologia, Cirurgia Vascular\n\n🧠 **Terapias e Avaliações**: Psicologia, Educação Parental, Fonoaudiologia, Neuropsicologia, Neuropsicopedagogia, Terapia Ocupacional, Avaliação do Perfil Cognitivo, Avaliação da Cognição Social';
  } else if (especialidade === 'psiquiatria' || especialidade === 'avaliacao_neuropsicologica') {
    mensagem = 'Entendi! Para agendamentos de Psiquiatria e Avaliação Neuropsicológica, vou transferir você para um de nossos atendentes que poderá te ajudar melhor. Aguarde um momento! 🙋‍♀️';
  } else {
    mensagem = especialidadesComProfissionais[especialidade] || 'Excelente escolha! Para agendar sua consulta, entre em contato pelo telefone (61) 3964-9899 ou pelo WhatsApp (61) 99999-9999. 📅';
  }
  
  results.push({
    json: {
      ...item.json,
      responseMessage: mensagem
    }
  });
}

return results;
```

**Posição**: `[2250, 300]`

---

### Node 15: Processar Terapia

**Tipo**: `n8n-nodes-base.code`

**Código JavaScript**:
```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const aiResponse = item.json.message.content;
  const responseText = aiResponse.response_text;
  
  // Se a IA gerou uma resposta, usamos ela
  let mensagem = responseText || 'A Clínica Cognus oferece:\n\n🗣️ **Fonoaudiologia** (com Ana Carolina Rabetti)\n🧠 **Psicoterapia** (com Juliana Nogueira e Ana Paula Gheller)\n📚 **Psicopedagogia**\n🎯 **Terapia Ocupacional** (com Wesley Oliveira)\n👨‍👩‍👧 **Educação Parental** (com Gleisse Nunes Pires da Silva)\n🧩 **Neuropsicologia** (com Karla Sanches e Camila Ribeiro)\n\n🚀 **Programas Especiais**:\n• **VOAR**: Para crianças e adolescentes com TDAH\n• **Leitura e Escrita**: Para dificuldades de aprendizagem\n\nSobre qual você gostaria de saber mais?';
  
  results.push({
    json: {
      ...item.json,
      responseMessage: mensagem
    }
  });
}

return results;
```

**Posição**: `[2250, 400]`

---

### Node 16: Informar E-mail

**Tipo**: `n8n-nodes-base.set`

**Configuração**:
```json
{
  "values": {
    "string": [
      {
        "name": "responseMessage",
        "value": "Para solicitar relatórios e laudos, por favor, envie um e-mail para **contato@clinicacognus.com.br** com os seguintes dados:\n\n📧 Nome completo\n📧 CPF\n📧 Tipo de documento solicitado\n\nO prazo de entrega é de até 7 dias úteis. ⏰"
      }
    ]
  },
  "options": {
    "keepOnlySet": false
  }
}
```

**Posição**: `[2250, 500]`

---

### Node 17: Handoff Humano

**Tipo**: `n8n-nodes-base.set`

**Configuração**:
```json
{
  "values": {
    "string": [
      {
        "name": "responseMessage",
        "value": "Entendi! Vou transferir você para um de nossos atendentes que poderá te ajudar melhor. Aguarde um momento! 🙋‍♀️"
      }
    ]
  },
  "options": {
    "keepOnlySet": false
  }
}
```

**Posição**: `[2250, 600]`

---

### Node 18: Create Card in Trello

**Tipo**: `n8n-nodes-base.trello`

**Configuração**:
```json
{
  "operation": "create",
  "boardId": "{{ID_DO_BOARD}}",
  "listId": "={{ $json.trelloList }}",
  "name": "Receita - {{ $json.dataw.fromName }}",
  "description": "Paciente: {{ $json.dataw.fromName }}\nTelefone: {{ $json.dataw.from }}\nTipo: {{ $json.message.content.sub_intent.tipo_receita }}\nLocalização: {{ $json.message.content.sub_intent.localizacao }}\nData: {{ $now }}"
}
```

**Posição**: `[2450, 200]`

**Conexão**: Recebe de `Processar Receita` (apenas quando `trelloList` não é null)

---

## 7. CAMADA 5: SAÍDA

### Node 19: Enviar texto (Evolution API)

**Tipo**: `n8n-nodes-base.httpRequest`

**Configuração**:
```json
{
  "url": "{{EVOLUTION_API_URL}}/message/sendText/{{INSTANCE_NAME}}",
  "method": "POST",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpHeaderAuth",
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [
      {
        "name": "apikey",
        "value": "={{$env.EVOLUTION_API_KEY}}"
      }
    ]
  },
  "sendBody": true,
  "bodyParameters": {
    "parameters": [
      {
        "name": "number",
        "value": "={{ $json.dataw.from }}"
      },
      {
        "name": "text",
        "value": "={{ $json.responseMessage }}"
      }
    ]
  },
  "options": {
    "timeout": 10000
  }
}
```

**Posição**: `[2650, 300]`

**Conexão**: Recebe de todos os nodes de negócio (Processar Receita, Processar Agendamento, Processar Terapia, Informar E-mail, Handoff Humano)

---

## 8. CONEXÕES DO WORKFLOW

```json
{
  "Webhook WhatsApp Evolution": {
    "main": [[{"node": "playload", "type": "main", "index": 0}]]
  },
  "playload": {
    "main": [[{"node": "Switch1", "type": "main", "index": 0}]]
  },
  "Switch1": {
    "main": [
      [{"node": "push", "type": "main", "index": 0}],
      [{"node": "Download Audio", "type": "main", "index": 0}]
    ]
  },
  "Download Audio": {
    "main": [[{"node": "transcript", "type": "main", "index": 0}]]
  },
  "transcript": {
    "main": [[{"node": "Set Transcription", "type": "main", "index": 0}]]
  },
  "Set Transcription": {
    "main": [[{"node": "push", "type": "main", "index": 0}]]
  },
  "push": {
    "main": [[{"node": "Wait", "type": "main", "index": 0}]]
  },
  "Wait": {
    "main": [[{"node": "msgs", "type": "main", "index": 0}]]
  },
  "msgs": {
    "main": [[{"node": "junta_msgs", "type": "main", "index": 0}]]
  },
  "junta_msgs": {
    "main": [[{"node": "AI Agent", "type": "main", "index": 0}]]
  },
  "AI Agent": {
    "main": [[{"node": "Switch por Intenção", "type": "main", "index": 0}]]
  },
  "Switch por Intenção": {
    "main": [
      [{"node": "Processar Receita", "type": "main", "index": 0}],
      [{"node": "Processar Agendamento", "type": "main", "index": 0}],
      [{"node": "Processar Terapia", "type": "main", "index": 0}],
      [{"node": "Informar E-mail", "type": "main", "index": 0}],
      [{"node": "Handoff Humano", "type": "main", "index": 0}]
    ]
  },
  "Processar Receita": {
    "main": [[{"node": "Enviar texto", "type": "main", "index": 0}]]
  },
  "Processar Agendamento": {
    "main": [[{"node": "Enviar texto", "type": "main", "index": 0}]]
  },
  "Processar Terapia": {
    "main": [[{"node": "Enviar texto", "type": "main", "index": 0}]]
  },
  "Informar E-mail": {
    "main": [[{"node": "Enviar texto", "type": "main", "index": 0}]]
  },
  "Handoff Humano": {
    "main": [[{"node": "Enviar texto", "type": "main", "index": 0}]]
  }
}
```

---

## 9. VARIÁVEIS DE AMBIENTE NECESSÁRIAS

```
EVOLUTION_API_URL=https://sua-evolution-api.com
EVOLUTION_API_KEY=sua-chave-api
INSTANCE_NAME=cognus
OPENAI_API_KEY=sua-chave-openai
REDIS_HOST=localhost
REDIS_PORT=6379
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_KEY=sua-chave-supabase
TRELLO_API_KEY=sua-chave-trello
TRELLO_TOKEN=seu-token-trello
```

---

## 10. INSTRUÇÕES FINAIS

1. Gere o JSON do workflow com todos os 19 nodes especificados acima.
2. Use as posições fornecidas para organizar os nodes visualmente.
3. Configure todas as conexões conforme a seção 8.
4. Certifique-se de que o campo `sessionId` está presente em todos os fluxos.
5. Teste o workflow com mensagens de texto e áudio.

---

**FIM DO GUIA TÉCNICO**
