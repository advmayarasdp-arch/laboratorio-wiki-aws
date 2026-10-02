# 📚 Wiki Inteligente AWS — Arquitetura de Conhecimento com RAG

Proposta de arquitetura Serverless e orientada a eventos na AWS para transformar acervos corporativos desorganizados (PDFs digitais, documentos escaneados com escrita manual e exports tabulares em CSV) em uma base de conhecimento consultável em linguagem natural via IA Generativa (RAG), garantindo governança, segurança e citação auditável de fontes.

### 🎯 O Problema Resolvido
Empresas frequentemente acumulam documentos vitais espalhados em formatos heterogêneos dentro de repositórios não estruturados (como pastas genéricas de landing zone). Essa falta de padronização gera três gargalos principais:

Ineficiência Operacional: Tempo excessivo gasto por equipes na busca manual por decisões, pautas ou dados de vendas.

Perda de Contexto: Dados tabulares (como relatórios de CRM) perdem o significado do cabeçalho de colunas quando armazenados ou buscados de forma ingênua.

Falta de Rastreabilidade: Consultas informais não garantem que a resposta esteja respaldada no documento oficial mais recente.

Esta solução resolve o problema ao implementar uma pipeline automatizada de ingestão, OCR especialista e indexação vetorial, permitindo que qualquer colaborador pergunte à Wiki em linguagem natural e receba respostas precisas com a indicação exata do arquivo e página de origem.

### 🏗️ Arquitetura da Solução
A arquitetura adota o padrão RAG (Retrieval-Augmented Generation) de forma 100% Serverless, separando a ingestão imutável do processamento semântico.

### Diagrama da Arquitetura

[ S3 Landing Zone (raw/) ]
                                                  │
                                                  ▼ (EventBridge)
                                       [ AWS Step Functions ]
                                                  │
                     ┌────────────────────────────┼────────────────────────────┐
                     ▼                            ▼                            ▼
            (Imagens / Scans)              (PDFs Digitais)                 (CSV do CRM)
             Amazon Textract                 AWS Lambda                   AWS Lambda
           (AnalyzeDocument)                (pdfplumber)                   (Pandas)
                     │                            │                            │
                     └────────────────────────────┼────────────────────────────┘
                                                  ▼
                                      [ Amazon Bedrock (Haiku) ]
                                    (Enriquecimento de Metadados)
                                                  │
                                   ┌──────────────┴──────────────┐
                                   ▼                             ▼
                        [ Amazon DynamoDB ]             [ S3 Processed Zone ]
                        (Store Metadados)                        │
                                                                 ▼
                                                    [ Bedrock Knowledge Bases ]
                                                 (Titan Embeddings v2 + OpenSearch)
                                                                 │
[ Usuário ] <─> [ Cognito + Amplify ] <─> [ API Gateway / Lambda ] <─> [ Bedrock (Claude 3.5 Sonnet) ]


### Fluxo de Dados de Ponta a Ponta

1. Ingestão & Preservação: Arquivos chegam ao raw/ no Amazon S3 (Landing Zone). O bucket mantém o Versioning e Object Lock ativos, garantindo imutabilidade e auditabilidade por hash SHA-256.

2. Triagem & Processamento Automatizado: O Amazon EventBridge aciona o AWS Step Functions, que identifica dinamicamente o formato e invoca o serviço especialista sem mover o arquivo de diretório.

3. Extração de Texto Customizada:

° Imagens e Scans: O Amazon Textract extrai textos impressos e anotações manuscritas (handwriting).

° PDFs Digitais: O AWS Lambda realiza a extração direta de texto em milissegundos.

° CSVs Tabulares: O AWS Lambda (com pandas) converte linhas em registros narrativos contextuais.

4. Estruturação de Metadados: Um modelo leve no Amazon Bedrock (Claude 3 Haiku) analisa o texto e gera metadados em JSON (título, autor, datas, decisões e tags), salvando-os no Amazon DynamoDB.

5. Vetorização e Indexação: O Amazon Bedrock Knowledge Bases divide o texto limpo em blocos semânticos (chunks), gera vetores via Amazon Titan Text Embeddings v2 e os indexa no Amazon OpenSearch Serverless.

6. Consulta & Citação com IA: O usuário consulta a interface Web (autenticada via Amazon Cognito). O Amazon Bedrock (Claude 3.5 Sonnet) recupera os trechos mais relevantes do OpenSearch e constrói a resposta em linguagem natural, citando a fonte e a página/linha correspondente.

### 🛠️ Serviços AWS Utilizados e Justificativa

* **Amazon S3:** Atua como o Object Storage para as zonas de armazenamento (`raw/` e `processed/`). Oferece alta durabilidade, versionamento automático e gatilhos de eventos nativos para disparar a pipeline.
* **AWS Step Functions:** Orquestra o workflow de ingestão. Gerencia os estados de execução, aplica regras de retentativas para falhas e direciona os arquivos para a rota de processamento adequada.
* **Amazon Textract:** Realiza OCR avançado e análise documental em imagens e PDFs escaneados. Extrai tabelas, formulários e escrita manual (*handwriting*) com suporte especializado.
* **AWS Lambda:** Processador Serverless que executa scripts em Python para parsing leve de PDFs digitais e conversão de registros tabulares do CSV em narrativas de texto.
* **Amazon DynamoDB:** Banco NoSQL de chave-valor e baixa latência que armazena o catálogo de metadados enriquecidos de cada documento processado.
* **Amazon Bedrock Knowledge Bases:** Serviço gerenciado de RAG que automatiza o fatiamento de texto (*chunking*), a geração de embeddings e a integração com a base vetorial.
* **Amazon OpenSearch Serverless:** Vector Database escalável de alta performance para armazenamento de embeddings e execução de buscas vetoriais híbridas (similaridade + palavra-chave).
* **Amazon Bedrock (Claude 3.5 Sonnet & Haiku):** Plataforma de modelos de linguagem generativa. O Claude 3 Haiku é utilizado para extração rápida de metadados em JSON e o Claude 3.5 Sonnet para síntese de respostas em linguagem natural com citação de fontes.
* **Amazon Cognito & Amazon API Gateway:** Garantem a camada de segurança e acesso. O Cognito gerencia a autenticação dos usuários (RBAC) e o API Gateway expõe os endpoints REST de consulta de forma protegida.
* **AWS CloudTrail & Amazon CloudWatch:** Fornecem observabilidade e governança. Monitoram logs de erro, latências de invocação, alarmes de custo e auditam chamadas de API de todos os serviços.


### 📑 Tratamento por Formato de Arquivo
Ata de Reunião (PDF Digital de 5 Páginas):

Tratamento: Processado via AWS Lambda com bibliotecas leves (pypdf/pdfplumber).

Objetivo: Extrair o texto selecionável diretamente, preservando pautas, deliberações e datas sem consumir cotas desnecessárias de OCR.

Folha Digitalizada (Imagem PNG/JPG/Scan com Manuscrito):

Tratamento: Submetido ao Amazon Textract chamando a API AnalyzeDocument com a funcionalidade HANDWRITING ativada.

Objetivo: Decifrar e extrair tanto textos impressos quanto anotações feitas à mão e notas operacionais de campo.

Export do CRM (CSV de 19 Colunas):

Tratamento: Parseado no AWS Lambda com pandas / awswrangler para converter cada linha da tabela em uma estrutura narrativa em linguagem natural.

Objetivo: Garantir que a busca vetorial reconheça a relação entre o cliente, o valor (ARR/ACV) e o estágio do funil sem perder o contexto das colunas após o chunking.

### 💻 Exemplo de Consulta na Interface
Pergunta do Usuário:

"Qual foi a decisão tomada sobre o Projeto X e qual o valor da oportunidade no CRM?"

Resposta da Wiki Inteligente:

"Conforme registrado na Ata de Reunião [Fonte: ata_reuniao_projeto_x.pdf, Pág: 2], ficou decidido que o orçamento do Projeto X terá um reajuste de 15% para expansão da infraestrutura em nuvem. De acordo com os registros do CRM [Fonte: export_crm.csv, Linha: 42], a oportunidade vinculada à conta Acme Corp referente a este projeto está avaliada em R$ 150.000,00 e encontra-se no estágio 'Em Negociação'."

### 💡 Aprendizados
RAG Requer Engenharia de Ingestão: O sucesso de um modelo de Inteligência Artificial Generativa depende diretamente da qualidade da higienização, estruturação de metadados e escolha da estratégia de chunking no pipeline de dados.

Especialização de Serviços em Nuvem: Aplicar o serviço correto para cada tipo de arquivo (Textract para imagens/manuscritos e Lambda leve para PDFs digitais e CSVs) otimiza os custos operacionais e reduz drasticamente a latência do sistema.

Rastreabilidade e Governança: Citar obrigatoriamente a fonte do documento em arquiteturas RAG corporativas é indispensável para evitar alucinações e garantir a auditabilidade técnica exigida por equipes de compliance e governança.
