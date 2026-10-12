# 🤖 Assistente de Investimentos com RPA e IA Generativa

Pipeline automatizado de ponta a ponta que simula um cenário real do mercado financeiro: coleta dados cadastrais e financeiros de clientes de uma interface web via **RPA (Robotic Process Automation)**, processa e cruza dados com o catálogo de investimentos em um fluxo orquestrado no **n8n**, e formula e-mails consultivos personalizados em conformidade com as diretrizes regulatórias via **IA Generativa (LLMs)** e envio automático pelo **Gmail**.

## 📌 Sumário

* [Visão Geral e Objetivos](#-visão-geral-e-objetivos)

* [Arquitetura da Solução](#-arquitetura-da-solução)

* [Tecnologias Utilizadas](#-tecnologias-utilizadas)

* [Estrutura do Repositório](#-estrutura-do-repositório)

* [Guia de Configuração e Execução](#-guia-de-configuração-e-execução)

  * [1. Instalação e Execução do n8n (Fedora Linux)](#1-instalação-e-execução-do-n8n-fedora-linux)

  * [2. Configuração do Túnel Reverso Seguro](#2-configuração-do-túnel-reverso-seguro)

  * [3. Execução da Automação RPA (Google Colab)](#3-execução-da-automação-rpa-google-colab)

  * [4. Importação e Credenciais no n8n](#4-importação-e-credenciais-no-n8n)

* [Detalhamento do Workflow no n8n](#-detalhamento-do-workflow-no-n8n)

  * [Tratamento e Normalização de Dados](#tratamento-e-normalização-de-dados)

  * [Versão Mínima (MVP): Templates Estáticos e Humanizados](#versão-mínima-mvp-templates-estáticos-e-humanizados)

  * [Versão Completa: Integração com IA Generativa](#versão-completa-integração-com-ia-generativa)

* [Engenharia de Prompt e Conformidade Regulatória](#-engenharia-de-prompt-e-conformidade-regulatória)

* [Decisões Técnicas e Solução de Problemas](#-decisões-técnicas-e-solução-de-problemas)

* [Exemplo de Mensagem Produzida](#-exemplo-de-mensagem-produzida)

* [Licença](#-licença)

## 💡 Visão Geral e Objetivos

Instituições financeiras frequentemente enfrentam o desafio de integrar sistemas legados ou interfaces sem APIs nativas às suas plataformas modernas de atendimento ao cliente.

Este projeto aborda esse desafio com um fluxo estruturado:

1. **Robotic Process Automation (RPA):** Um script Python substitui a operação manual de leitura e extração de dados da tela (`docs/index.html`), enviando-os a um orquestrador.

2. **Orquestração e Transformação (n8n):** Recebimento do evento via Webhook, download e parsing de regras de produtos (`docs/data.csv`), e junção de dados por perfil de risco (*Suitability*).

3. **Hiperpersonalização com IA:** O modelo de linguagem redige um comunicado sob medida para a situação financeira e o perfil de cada cliente, sem fazer promessas de ganhos e sem violar regras de compliance.

4. **Disparo em Lote:** Envio de e-mail em HTML para os clientes e retorno imediato do status da execução para a esteira RPA.

## 🏗 Arquitetura da Solução

```
[ Página Web: docs/index.html ]
             │
             ▼ (RPA: Web Scraping com BeautifulSoup)
[ Google Colab / Notebook Python ]
             │
             ▼ (HTTP POST via Túnel Reverso / Localtunnel)
[ n8n: Whk_Busca_Clientes ] ──────────────► [ n8n: Busca_Dados_Investimentos (HTTP Request) ]
             │                                                   │
             │                                                   ▼
             │                                      [ n8n: Organiza_Dados (Code Node) ]
             │                                                   │
             └──────────────────────┬────────────────────────────┘
                                    ▼
               [ n8n: Merge_Combina_Clientes_Investimentos ]
                                    │
                                    ▼
               [ n8n: Gera_mensagem_Fixa (Regras & MVP) ]
                                    │
                                    ▼
               [ n8n: Message a model (OpenAI / LLM) ]
                                    │
                                    ▼
               [ n8n: Cria_Estrutura_Gmail (Sanitização JSON) ]
                                    │
                                    ▼
               [ n8n: Envia_Mensagem_Investimentos (Gmail API) ]
                                    │
                                    ▼
               [ n8n: Respond to Webhook (HTTP 200 de volta ao Colab) ]

```

## 🛠 Tecnologias Utilizadas

| Camada | Tecnologia | Finalidade | 
| ----- | ----- | ----- | 
| **Hospedagem Estática** | GitHub Pages | Distribuição da tabela HTML de clientes e da base de investimentos em CSV. | 
| **Robô de Coleta (RPA)** | Python 3, BeautifulSoup4, Requests | Automação da extração tabular de clientes simulando comportamento humano. | 
| **Ambiente de Execução RPA** | Google Colab | Execução em nuvem do script de raspagem de dados. | 
| **Infraestrutura Local** | Fedora Linux, Node.js, Localtunnel | Execução self-hosted do n8n com exposição da porta 5678 para a internet. | 
| **Orquestrador de Processos** | n8n (Community / Self-hosted) | Coordenação dos nós, scripts JavaScript, fusão de dados e integrações. | 
| **Inteligência Artificial** | OpenAI (`@n8n/n8n-nodes-langchain.openAi`) | Geração contextual de mensagens orientadas a conformidade financeira. | 
| **Canal de Comunicação** | Gmail Node (n8n) | Disparo das comunicações formatadas em HTML. | 

## 📂 Estrutura do Repositório

```
├── docs/
│   ├── index.html               # Página web simulada com tabela de clientes
│   └── data.csv                 # Catálogo de produtos (perfil, produto, mínimo, rentabilidade)
├── n8n/
│   └── RPA_N8N_Python_1.json    # Workflow completo exportado do n8n
├── src/
│   └── extrair_clientes.ipynb    # Notebook com o script RPA de extração
└── README.md                    # Documentação oficial do projeto

```

## 🚀 Guia de Configuração e Execução

### 1. Instalação e Execução do n8n (Fedora Linux)

No Fedora, instale o Node.js e o n8n pelo terminal:

```
sudo dnf install -y nodejs
npm install n8n -g
n8n start

```

O painel de controle ficará acessível no navegador em `http://localhost:5678`.

### 2. Configuração do Túnel Reverso Seguro

Como o Google Colab opera na nuvem do Google, ele não consegue alcançar a porta local `5678` diretamente via `localhost`. Para permitir a comunicação temporária com tempo de encerramento programado no Fedora:

```
# Mantém o túnel ativo por 2 horas e encerra o processo automaticamente
timeout 2h npx localtunnel --port 5678

```

O comando exibirá uma URL pública (exemplo: `https://sua-url-temporaria.loca.lt`). Guarde este endereço.

### 3. Execução da Automação RPA (Google Colab)

1. Abra o arquivo `src/extrair_clientes.ipynb` no Google Colab.

2. Localize a variável de configuração da URL do webhook e insira a URL pública gerada:

   ```
   WEBHOOK_URL = "https://sua-url-temporaria.loca.lt/webhook-test/clientes"
   
   ```

3. Execute o notebook. O script irá:

   * Ler a tabela de clientes da página web usando `BeautifulSoup`;

   * Converter os campos em um payload JSON estruturado;

   * Enviar a requisição POST diretamente para o webhook do n8n.

### 4. Importação e Credenciais no n8n

1. No n8n, clique em **Workflows** > **Add Workflow** > menu **Import from File**.

2. Selecione o arquivo `n8n/RPA_N8N_Python_1.json`.

3. Configure as credenciais necessárias:

   * **OpenAi account:** Chave de API da OpenAI no nó `Message a model`.

   * **Gmail OAuth2:** Autenticação de conta Google no nó `Envia_Mensagem_Investimentos`.

## ⚙️ Detalhamento do Workflow no n8n

O workflow está estruturado em blocos modulares identificados por notas (*Sticky Notes*):

### Tratamento e Normalização de Dados

* **`Whk_Busca_Clientes`:** Webhook de entrada configurado como método `POST` no caminho `clientes`, com modo de resposta `responseNode`.

* **`Busca_Dados_Investimentos`:** Nó HTTP Request que consome o arquivo `data.csv`.

* **`Organiza_Dados`:** Nó Code (JavaScript) que faz o parsing do CSV, remove quebras de linha e converte a coluna de aporte mínimo para tipo numérico:

```
const csvRaw = $json.data ?? $json.body ?? $json;
const text = (typeof csvRaw === "string") ? csvRaw : JSON.stringify(csvRaw);

const lines = text.split(/\r?\n/).map(l => l.trim()).filter(Boolean);
const headers = lines[0].split(",").map(h => h.trim());

const investimentos = lines.slice(1).map(line => {
    const cols = line.split(",").map(c => c.trim());
    const obj = {};
    headers.forEach((h, i) => obj[h] = cols[i] ?? "");

    // Transforma mínimo de texto para número puro
    obj.minimo = Number(String(obj.minimo).replace(/[^\d]/g, "")) || 0;

    return obj;
});

return [{ json: { investimentos } }];

```

* **`Merge_Combina_Clientes_Investimentos`:** Mescla a entrada de clientes com a base de investimentos via `combineByPosition`.

### Versão Mínima (MVP): Templates Estáticos e Humanizados

O nó `Gera_mensagem_Fixa` implementa as diretrizes do MVP, filtrando as alternativas compatíveis com o perfil de risco do cliente, aplicando formatação monetária brasileira (`R$ 0,00`) e estruturando a recomendação:

* **Conservador:** Foco na preservação de capital e previsibilidade da renda fixa.

* **Moderado:** Equilíbrio entre a segurança da renda fixa e o potencial da renda variável.

* **Arrojado:** Foco no crescimento patrimonial a longo prazo via renda variável.

### Versão Completa: Integração com IA Generativa

1. **`Message a model`:** Nó de LLM configurado para receber o contexto de cada cliente e redigir o e-mail em formato JSON.

2. **`Cria_Estrutura_Gmail`:** Nó Code que processa o retorno da IA, eliminando delimitadores de markdown (```` ```json ````), realizando o parse seguro e mapeando os campos para o Gmail:

````
return $input.all().map(item => {
  let rawText = item.json.output?.[0]?.content?.[0]?.text || "";
  rawText = rawText.replace(/```json/gi, "").replace(/```/g, "").trim();

  let emailData = {};
  try {
    emailData = JSON.parse(rawText);
  } catch (error) {
    emailData = {
      to: "",
      subject: "Informações sobre seus investimentos",
      text_body: rawText,
      html_body: `<p>${rawText}</p>`
    };
  }

  return {
    json: {
      to: emailData.to || "",
      subject: emailData.subject || "",
      message: emailData.html_body || emailData.text_body || "",
      html: emailData.html_body || "",
      text: emailData.text_body || ""
    }
  };
});

````

3. **`Envia_Mensagem_Investimentos`:** Dispara os e-mails com corpo em HTML via Gmail API.

4. **`Respond to Webhook`:** Emite um HTTP 200 de conclusão de volta ao Google Colab.

## 🧠 Engenharia de Prompt e Conformidade Regulatória

No mercado de capitais, comunicações de assessoria de investimento não podem prometer retornos futuros nem omitir riscos. O prompt configurado no nó de IA em modo *Expression* foi elaborado para garantir esse alinhamento:

```
Escreva um e-mail curto em pt-BR para o cliente abaixo, sem prometer ganhos e sem garantias.

Cliente:
Nome: {{ $json.nome }}
E-mail: {{ $json.email }}
Perfil: {{ $json.perfil }}
Saldo: {{ $json.saldo }}

Recomendação:
Produto: {{ $json.produto }}
Rentabilidade: {{ $json.rentabilidade }}
Motivo: {{ $json.motivo }}

Retorne APENAS JSON com:
{
  "to": "...",
  "subject": "...",
  "text_body": "...",
  "html_body": "..."
}

```

## 🔍 Decisões Técnicas e Solução de Problemas

1. **`ConnectionRefusedError: [Errno 111]` (Google Colab ➔ n8n Local):**

   * *Problema:* O Colab roda em uma VM na nuvem do Google e interpretava `localhost` como a sua própria máquina virtual interna.

   * *Solução:* Criação de um túnel reverso com **Localtunnel** e gestão do ciclo de vida da porta com o utilitário nativo `timeout` do Fedora Linux.

2. **Divisão de listas em itens independentes (*Fan-Out*):**

   * *Problema:* Nós visuais como o *Edit Fields* mantêm arrays encapsulados dentro de uma linha única.

   * *Solução:* Uso de nós **Code** mapeando itens com `.map(item => ({ json: item }))`, garantindo que cada cliente seja tratado como uma execução individual nos nós subsequentes.

3. **Tratamento de Saída da LLM:**

   * *Problema:* Modelos de linguagem frequentemente inserem blocos markdown (```` ```json ... ``` ````) em volta de strings de saída, quebrando o `JSON.parse` convencional.

   * *Solução:* Sanitização por expressão regular (```` replace(/```json/gi, "") ````) antes do processamento do objeto no nó `Cria_Estrutura_Gmail`.

## 📩 Exemplo de Mensagem Produzida

Abaixo está um exemplo de comunicação gerada pela IA e enviada via Gmail:

> **Destinatário:** `henrique@email.com`
>
> **Assunto:** Próximos passos para sua carteira de investimentos
>
> Olá, Henrique!
>
> Com base no seu perfil **Moderado** e no saldo atual de **R\$ 22.300,00**, podemos avaliar alternativas que combinem segurança com oportunidades graduais de valorização.
>
> Ressaltamos que todo investimento está sujeito a oscilações de mercado e não há garantia de rentabilidade futura. Ficamos à total disposição para detalhar os produtos disponíveis e auxiliar você na sua tomada de decisão.
>
> Atenciosamente,
>
> **Equipe de Assessoria Financeira**

## 📜 Licença

Projeto desenvolvido para fins didáticos e demonstrativos como parte do desafio prático **Assistente de Investimentos com RPA e IA Generativa**.