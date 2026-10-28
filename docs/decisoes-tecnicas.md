# Decisões Técnicas

Este documento explica as principais decisões tomadas na construção do fluxo de RPA + IA para recomendação de investimentos.

## Visão geral do fluxo

```
Colab (RPA em Python)
   │  extrai clientes de uma página HTML via requests + BeautifulSoup
   ▼
Webhook (n8n)
   │
HTTP Request ──► busca o CSV de produtos de investimento
   │
Code (JavaScript) ──► transforma o CSV em uma lista de objetos (investimentos)
   │
Merge ──► junta a lista de clientes com a lista de investimentos
   │
Code (JavaScript) ──► para cada cliente, escolhe o produto adequado ao perfil
   │                   e ao saldo, e gera 3 variações de mensagem
   │
Code (JavaScript) ──► desempacota o array em um item por cliente
   │
Basic LLM Chain + Groq Chat Model ──► reescreve a mensagem como um e-mail
   │                                   (assunto, corpo texto e corpo HTML)
   │
Merge ──► junta a resposta da IA de volta com os dados do cliente
   │
Code (JavaScript) ──► interpreta o JSON devolvido pela IA
   │
If (e-mail válido?)
   ├─ true  ──► Gmail: envia a recomendação ao cliente ──► Respond to Webhook
   └─ false ──► Code (monta resumo) ──► Gmail: envia um alerta único ao responsável
```

## 1. RPA em Python no Google Colab

O Colab foi escolhido por não exigir instalação local e por já vir com `requests` e `beautifulsoup4` disponíveis. O script baixa a página HTML publicada no GitHub Pages, usa o `BeautifulSoup` para localizar a tabela de clientes (`#clientes tbody tr`) e monta uma lista de dicionários com nome, e-mail, saldo e perfil de cada cliente.

## 2. Comunicação entre Colab e n8n local via ngrok

O n8n roda localmente (`localhost:5678`), mas o Colab executa em servidores do Google, que não enxergam `localhost` da máquina do desenvolvedor. Para resolver isso, foi usado o **ngrok** para expor o n8n local em uma URL pública temporária (`https://...ngrok-free.dev`), usada no `requests.post` do Colab para chamar o Webhook do n8n.

Essa URL muda a cada reinício do túnel ngrok, então não é fixa no código — é um valor de exemplo que deve ser substituído pela URL atual sempre que o túnel for reaberto. Em um cenário de produção, o ideal seria usar um n8n hospedado em nuvem com URL fixa, dispensando o túnel.

O ambiente local também exigiu adaptações nas credenciais OAuth2 (Google Sheets e Gmail): a URL de redirecionamento aponta para `http://localhost:5678/rest/oauth2-credential/callback`, foi necessário criar manualmente um cliente OAuth2 no Google Cloud Console, e, enquanto o app permanecer em modo de teste, o token expira a cada 7 dias — meio-termo escolhido para evitar a verificação completa do app pelo Google, desnecessária para um projeto de uso pessoal/educacional.

## 3. Webhook em modo de teste

Durante o desenvolvimento, o Webhook do n8n foi usado em modo de teste (`/webhook-test/...`), que só aceita uma chamada por vez e exige clicar em "Execute workflow" antes de cada execução. Essa limitação é aceitável para fins de demonstração do lab; em produção, o workflow seria publicado/ativado, passando a usar a URL `/webhook/...`, que funciona de forma contínua sem precisar desse clique manual.

## 4. Lógica de recomendação de investimento

Para cada cliente, o fluxo filtra os produtos de investimento com o mesmo perfil do cliente (Conservador, Moderado ou Arrojado) e, entre eles, seleciona o produto de **maior valor mínimo que o saldo do cliente ainda consegue pagar**. A ideia é sugerir a opção mais robusta que o cliente já tem capacidade de aplicar, em vez de sempre sugerir a opção mais barata.

Quando o saldo do cliente não atinge o mínimo de nenhum produto do seu perfil, o fluxo sugere o produto de menor valor mínimo daquele perfil, como uma meta a ser alcançada, e sinaliza isso no campo `motivo` (`saldo_insuficiente`).

## 5. Geração de mensagens com IA generativa

Inicialmente a geração de e-mails foi configurada com a API da OpenAI (GPT-4o), mas a conta de testes ficou sem créditos. Diante da necessidade de concluir o desafio, o provedor foi trocado para a **Groq**, que oferece um nível gratuito e respostas rápidas, usando o modelo `openai/gpt-oss-20b` rodado na infraestrutura da Groq, através do nó **Basic LLM Chain** do n8n.

O prompt enviado à IA inclui os dados do cliente, a recomendação já calculada previamente pela lógica determinística (não pela IA) e um conjunto de regras de estilo (idioma, tom, tamanho, termos a evitar). Isso mantém o cálculo financeiro sob controle do código, e usa a IA apenas para a tarefa de redação do texto, pedindo a resposta em formato JSON estruturado (assunto, corpo em texto simples e corpo em HTML).

## 6. Tratamento de erros e validação de e-mail

Como a resposta da IA é um texto, e não um JSON garantido, o fluxo faz um `JSON.parse` dentro de um bloco de tratamento de erro: se a IA responder algo fora do formato esperado, o item recebe uma mensagem padrão de erro em vez de quebrar o restante do fluxo.

Antes do envio, um nó **If** valida o formato do e-mail do cliente com uma expressão regular (`^[^@\s]+@[^@\s]+\.[^@\s]+$`). E-mails válidos seguem para o envio individual da recomendação via Gmail. E-mails inválidos são agrupados em um único e-mail de alerta, enviado ao responsável pela operação, listando quais clientes precisam ter o cadastro corrigido — evitando tentativas de envio fadadas ao erro e mantendo a rastreabilidade dos problemas.

## 7. Envio de e-mail via Gmail

O envio final usa o nó nativo **Gmail** do n8n, autenticado via OAuth2 com um cliente criado no Google Cloud Console (o mesmo cliente reaproveitado da integração com Google Sheets, com a Gmail API ativada adicionalmente). Essa escolha evita custos de uma API de terceiros e aproveita a mesma credencial Google já usada em outras partes do fluxo.

## 8. JavaScript em vez de Python nos nós Code

Os nós **Code** do fluxo (leitura do CSV, cálculo da recomendação, parse da resposta da IA, montagem do resumo de erros) foram escritos em **JavaScript**, e não em Python, por três motivos:

- É a linguagem nativa do nó Code no n8n: o JavaScript roda no motor V8 já embutido no próprio n8n, enquanto o modo Python (Beta) depende do Pyodide (um interpretador Python compilado para rodar no navegador/worker), que é mais lento para iniciar e tem suporte limitado apenas à biblioteca padrão, sem acesso a pacotes externos.
- Variáveis como `$json` e `$input` já vêm prontas em JavaScript na documentação e nos exemplos do n8n, o que reduz o risco de erro ao escrever expressões dentro dos próprios campos dos nós (como `{{ $json.nome }}`), já que essas expressões do n8n também usam a sintaxe JavaScript.
- Maior estabilidade observada durante os testes: o modo Python (Beta) é mais recente e sujeito a instabilidades, enquanto o JavaScript é o modo padrão e mais maduro do nó Code.

O RPA de extração de clientes (fora do n8n, rodando no Google Colab) continuou em **Python**, já que é a linguagem usada no notebook e mais adequada para bibliotecas como `requests` e `BeautifulSoup`.

## Resumo das principais decisões

| Decisão | Motivo |
|---|---|
| ngrok para expor o n8n local | Colab não acessa `localhost` da máquina local |
| Groq em vez de OpenAI | Conta OpenAI sem créditos; Groq tem nível gratuito |
| Basic LLM Chain + Groq Chat Model | Padrão de nós do n8n para IA conversacional |
| Regra de produto = maior mínimo pagável | Sugere a melhor opção dentro da capacidade do cliente |
| Validação de e-mail com If | Evita falhas de envio e concentra erros em um só alerta |
| Gmail nativo do n8n | Reaproveita credencial Google já configurada, sem custo extra |
| JavaScript nos nós Code | Linguagem nativa do n8n, mais rápida e estável que o Python (Beta) |
