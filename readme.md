# OpenAI Responses API

Esse tutorial ilustra com usar a API `responses` da OpenAI para a geração de textos. Apesar dessa API ser multimodal, ou seja, permitir também manipular imagens, nesse tutorial será tratada apenas a geração de textos.

No momento em que esse tutorial foi elaborado, a API `responses` era o recurso mais atualizado da OpenAI para a geração de textos. Ele substitui os recursos anteriores, como `chat.completion` e `assistant`, os quais não devem mais ser empregados em novos projetos. Inclusive, deve-se ter cautela com o vibecoding, pois o agente de IA usualmente se baseia em documentações antigas e sugere com frequência o `chat.completion`, que já está depreciado.

Além disso, esse tutorial irá ilustrar apenas como fazer as inferências na API, não tem por objetivo explicar como fazer um chatbot. Para saber como fazer um chatbot usando os recursos da OpenAI, veja aqui: [https://github.com/mgarlabx/chatkit](https://github.com/mgarlabx/chatkit).

# Chamada básica

A sintaxe básica das chamadas dessa API (usando o SDK da OpenAI) é:

```
response = client.responses.create(
    model="gpt-5-nano",
    input="Quem descobriu a América?",
)
print(response.output_text)
```

(ver [example_01_basic.ipynb](example_01_basic.ipynb))

# O objeto `response`

O "response" é um objeto bem complexo, com vários atributos. Esses são alguns exemplos:

```
print(response.id)
print(response.model)
print(response.temperature)
print(response.top_p)
print(response.status)
```

O atributo `output_text` contém a resposta final da inferência:

```
print(response.output_text)
```

Um atributo interessante para analisar, todavia, é o `output`, que contém o resultado da inferência. Ele é uma lista de eventos, contendo todas as etapas do raciocínio do LLM. Veja como explorá-los:

```
for output in response.output:
    print(output.type)
```

Essa é uma lista com exemplos dos tipos de outputs:

- `reasoning`: sinaliza uma etapa de raciocínio do modelo, sempre começa com um evento desse tipo.
- `file_search_call`: informa a chamada da busca. O atributo "status" informa se houve sucesso ou não.
- `function_call`: informa a chamada da function. O atributo "status" informa se houve sucesso ou não.
- `mcp_list_tools`: devolve todas as "tools" que o mcp acessado possui.
- `mcp_call`: informa a chamada do mcp. O atributo "status" informa se houve sucesso ou não.
- `message`: contém a mensagem final, é sempre o último evento. 

Assim, dependendo da aplicação, pode ser interessante percorrer os outputs, ao invés de buscar diretamente o `output_text`.

# Streaming

Ao se acrescentar o parâmetro `stream=True` na chamada da API, a resposta será gerada em pedaços ("chunks"). Isso pode ser interessante no desenvolvimento de chatbots, para que o usuário não tenha que esperar ser gerada a resposta de uma só vez. Ele pode ir lendo aos poucos, conforme ela vai sendo gerada, melhorando a sua experiência com o bot.

(ver [example_02_streaming.ipynb](example_02_streaming.ipynb))

# Web Search

Quando se faz uma chamada simples usando a API, todo o conteúdo será buscado no próprio LLM. Todavia, os modelos possuem cortes temporais, isto é, seu conhecimento está limitado até a data em que foi gerado. Por exemplo, o [GPT-5](https://platform.openai.com/docs/models/gpt-5) foi treinado com dados até 30/09/2024. Tudo que ocorreu depois dessa data não será informado na consulta. Assim, ao se adicionar o parâmetro abaixo, o modelo irá fazer uma busca na web e ampliar o contexto da consulta:

```
tools=[{
    "type": "web_search"
}]
```

(ver [example_03_web_search.ipynb](example_03_web_search.ipynb))

Observar que esse parâmetro `tools` introduz o conceito de ferramentas usadas na API. Há outras opções de ferramentas, conforme será ilustrado a seguir. Repare, ainda, que esse parâmetro é uma lista, ou seja, podem ser usadas várias ferramentas ao mesmo tempo.

# File Search

É possível acrescentar ao contexto um arquivo, como fonte de informação. Para tanto, é necessário inicialmente criar uma "vector store". Isso pode ser feito diretamente no console da API (veja [aqui](https://platform.openai.com/storage/vector_stores)), mas pode também ser feito programaticamente.

Ao se criar uma vector store, deve-se anexar o arquivo e copiar o ID da store. Esse ID será informado nos parâmetros da chamada:

```
tools=[{
    "type": "file_search",
    "vector_store_ids": [<VECTOR_STORE_ID>]
}]
```

(ver [example_04_file_search.ipynb](example_04_file_search.ipynb))

Repare que o parâmetro é uma lista, ou seja, podem ser passados vários IDs, acessando múltiplas stores. Além disso, em uma mesma store é possível ter vários arquivos.

Essa funcionalidade implementa automaticamente um recurso conhecido como "Retrieval Augmented Generation" (RAG).

# Output estruturado

Dependendo do uso, pode ser interessante receber a resposta da chamada da API no formato `json`, ao invés de um texto corrido. Isso é útil nos casos em que o resultado da chamada será usado em algum fluxo de trabalho ("workflow"), tal como ocorre em automações.

Para tanto, é necessário informar na chamada o layout de saída do `json`. Há duas formas disso ser feito:

1. Informando o "schema", usando o parâmetro `text`.

   (ver [example_11_struc_output_schema.ipynb](example_11_struc_output_schema.ipynb))   

2. Usando a biblioteca "Pydantic", com o parâmetro `text_format`. Nesse caso, deve-se usar o método `parse` da API.   

   (ver [example_12_struc_output_pydantic.ipynb](example_12_struc_output_pydantic.ipynb))   

# Function call

A API do `responses` permite auxiliar no desenvolvimento de fluxos de trabalho ("workflows"), através de chamadas de funções.

**Importante**: a função NÃO é executada pelo LLM, ele apenas retorna se a mensagem enviada pelo usuário se encaixa em alguma função prevista no workflow.

Funciona assim. O LLM inicialmente analisa a mensagem enviada pelo usuário. Se ele entender que a mensagem se enquadra em alguma função prevista no workflow, ele informa a função identificada e extrai da consulta os parâmetros que essa função demanda. Caso contrário, o LLM irá processar como uma chamada simples e retornará uma mensagem que considerar relevante.

Mas para que o LLM saiba qual ou quais funções estão previstas no workflow, é preciso passar uma lista dessas funções na chamada da API, incluindo os parâmetros requeridos:

```
tools = [{
    "type": "function",
    "name": "get_capital",
    "description": "Retorna a capital de um país dado o nome do país.",
    "parameters": {
        "type": "object",
        "properties": {
            "country": {
                "type": "string",
                "description": "O nome do país cuja capital foi solicitada.",
            }
        },
        "required": ["country"],
        "additionalProperties": False,
    },
    "strict": True,
}]
```
(ver [example_13_function_call.ipynb](example_13_function_call.ipynb))

Notar que é uma lista e nesse caso foi informada apenas uma função, mas poderia haver mais. Reparar também que os "parameters" seguem o mesmo padrão do "schema" dos outputs estruturados. Veja mais [aqui](https://json-schema.org/learn/getting-started-step-by-step).

O parâmetro `strict: True` garante que o modelo siga exatamente o schema JSON que você definiu. Se estiver `False`, o modelo tenta seguir o schema, mas pode haver pequenas variações.

# MCP

O MCP (Model Context Protocol) é um protocolo que permite aos LLM executar funções. Ou seja, ao contrário da "function call" citada anteriormente, em que o LLM apenas identifica a função, mas não a executa, a "mcp call" permite tanto identificar quanto executar funções.

Para tanto, deve-se passar os parâmetros do mcp:

```
tools=[{
    "type": "mcp",
    "server_label": "aloyoga",
    "server_url": "https://www.aloyoga.com/api/mcp",
    "require_approval": "never"
}]
```
(ver [example_14_mcp.ipynb](example_14_mcp.ipynb))

Para saber mais sobre os usos dos mcps, veja aqui: [https://github.com/mgarlabx/mcp](https://github.com/mgarlabx/mcp).

.