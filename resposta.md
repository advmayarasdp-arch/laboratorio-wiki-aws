👤 Identificação
•	Nome: Mayara Santos Diniz Porfirio

•	Data: 02/10/2026

•	Link do repositório: https://github.com/advmayarasdp-arch/laboratorio-wiki-aws

✅ Quest 1: O Mapa dos Arquivos Perdidos
1.1 Formatos encontrados na pasta raw/
•	PDF (Ata de Reunião de 5 páginas): Nasce digital (selectable text). Permite extração direta de texto via bibliotecas ou parsers leves, preservando a paginação original.

•	PNG / JPG (Folha Digitalizada): Precisa de OCR. Imagem matricial contendo texto impresso misturado a anotações manuscritas (handwriting). Requer reconhecimento óptico especializado para extrair o texto impresso e a caligrafia.

•	CSV (Export do CRM): Nasce digital em formato estruturado. Contém centenas de linhas e 19 colunas. Requer transformação de registros tabulares em narrativas de texto para que cada linha não perca o contexto do cabeçalho.

1.2 Principais desafios encontrados
•	Ausência de organização por subpastas: Todos os arquivos residem no mesmo diretório raiz sem padronização de nomenclatura.

•	Heterogeneidade de formatos: Mistura de arquivos legíveis por máquina, imagens de baixa/média resolução com escrita manual e dados tabulares.

•	Perda de contexto em tabelas: Se o CSV for submetido a um chunking ingênuo por tamanho fixo de caracteres, as linhas são separadas de suas respectivas colunas.

•	Dificuldade de OCR em manuscritos: Caligrafia humana e anotações marginais exigem modelos de visão computacional treinados especificamente para handwriting.

1.3 Informações importantes a serem extraídas
•	Atas de Reunião: Título do projeto, data da reunião, participantes/stakeholders, pauta, deliberações oficiais e ações futuras/prazos.

•	Documentos Digitalizados: Data das anotações, autor/responsável, notas operacionais, alertas, status de pendências e carimbos.

•	Export do CRM: ID da oportunidade, Nome da Conta/Cliente, Valor Financeiro (ARR/ACV), Estágio do Funil de Vendas, Data de Fechamento e Vendedor Responsável.

1.4 Estratégia de classificação inicial
Sem utilizar subpastas na landing zone, a classificação é realizada dinamicamente em tempo de execução (runtime) por um serviço Serverless (AWS Lambda):


1.	Inspeção de MIME Type e Extensão: Identificação primária do arquivo (.pdf, .png/.jpg, .csv).

2.	Avaliação de Camada de Texto (para PDFs): O Lambda faz uma verificação rápida para checar se o PDF possui camada de texto nativa ou se é apenas uma imagem encapsulada.

3.	Injeção de Metadados de Tipo: O arquivo recebe tags no Amazon S3 (document_type = ata | imagem_digitalizada | crm_export), direcionando o fluxo do pipeline sem mover o arquivo de pasta.

✅ Quest 2: O Portal de Entrada na AWS
2.1 Armazenamento dos arquivos brutos
•	Amazon S3 (Bucket Landing Zone): Ponto de entrada unificado com acesso restrito.

•	AWS IAM: Políticas de privilégio mínimo que permitem apenas às rotinas automatizadas da pipeline ler e gravar no bucket.

•	AWS KMS: Criptografia em repouso padrão Server-Side Encryption (SSE-KMS) com chaves gerenciadas pelo cliente (CMK).

•	Amazon S3 Lifecycle: Regras para transição automática dos arquivos brutos para classes de armazenamento de menor custo (como S3 Glacier Flexible Retrieval) após 30 dias de ingestão.

2.2 Preservação dos arquivos originais
•	Amazon S3 Versioning: Ativado no bucket de origem para evitar a sobrescrita acidental ou deleção de arquivos históricos.

•	S3 Object Lock (Write Once, Read Many - WORM): Ativado em modo Governance para impedir alterações ou exclusões maliciosas por um período determinado.

•	Imutabilidade e Separação: A pasta raw/ permanece estritamente em modo Read-Only para o pipeline. Todo texto extraído e metadado gerado são salvos em um bucket separado (S3 Processed Zone), mantendo a rastreabilidade por Hash SHA-256 do arquivo original.

2.3 Extração de texto dos documentos
•	PDFs Escaneados e Imagens (PNG/JPG): Processados pelo Amazon Textract invocando a API AnalyzeDocument com a feature HANDWRITING ativada, garantindo a extração precisa do texto impresso e da escrita manual.

•	PDFs Digitais: Processados via AWS Lambda utilizando bibliotecas Python leves (como pypdf ou pdfplumber) para extração direta de texto em milissegundos a baixo custo.

•	Arquivos .txt, .docx e .md: Extraídos via AWS Lambda convertendo a estrutura diretamente para texto plano limpo (UTF-8).

•	Arquivos CSV: Ingeridos via AWS Lambda com pandas / awswrangler, convertendo cada linha em um objeto estruturado em linguagem natural (ex: "Oportunidade ID 102 referente ao Cliente X no valor de Y está no estágio Z").

•	Orquestração e Monitoramento: O AWS Step Functions coordena a chamada assíncrona do Textract ou Lambda, enquanto o Amazon CloudWatch monitora logs e métricas de execução.

2.4 Tratamento de falhas
•	Dead-Letter Queues (DLQ) no Amazon SQS: Arquivos que falham no processamento (ex: arquivo corrompido, imagem totalmente ilegível) após 3 tentativas de reprocessamento são redirecionados para uma SQS DLQ.

•	CloudWatch Alarms: Alertas que disparam notificações via Amazon SNS para a equipe de Engenharia de Dados quando a DLQ recebe mensagens.

•	Logging Estruturado: Todos os eventos de erro são gravados no CloudWatch Logs registrando o S3 Object Key, timestamp e mensagem técnica da exceção.

✅ Quest 3: A Relíquia dos Metadados
3.1 Padronização dos textos processados
•	Normalização UTF-8: Remapeamento de caracteres especiais e remoção de null bytes.

•	Remoção de Ruído de OCR: Eliminação de quebras de linha quebras no meio de palavras e remoção de caracteres estranhos provenientes de falhas de escaneamento.

•	Estruturação Padrão: O texto de todos os formatos é convertido em arquivos formato JSON Lines ou JSON Schema, contendo o conteúdo textual e o cabeçalho de metadados unificado.

3.2 Metadados propostos
Metadado	Por que ele é importante?
Nome do documento	Identifica a origem do arquivo (source_file).
Tipo do documento	Classifica entre Ata, Imagem Digitalizada, CRM Export, Contrato, etc.
Data identificada	Define a linha temporal do fato/decisão, fundamental para ordenação lógica.
Tema principal	Categoriza o assunto (ex: Orçamento, Arquitetura Cloud, Vendas).
Participantes	Mapeia pessoas mencionadas ou envolvidas nas deliberações.
Decisões tomadas	Isola pontos de virada e deliberações para rápida consulta executiva.
Responsáveis	Define os donos das tarefas atreladas às decisões.
Próximos passos	Rastreia pendências e prazos de execução.
Nível de confidencialidade	Permite aplicar filtros de segurança e RBAC (Role-Based Access Control).
Caminho do arquivo original	Mantém o link do S3 URI para auditoria e rastreabilidade da fonte.
Hash SHA-256	Garante a integridade e unicidade do arquivo processado.
3.3 Uso de IA para enriquecimento dos documentos
Um modelo de linguagem leve e veloz no Amazon Bedrock (como o Anthropic Claude 3 Haiku) lê o texto limpo extraído na Quest 2 e gera automaticamente em formato JSON:


•	Um resumo executivo do documento (até 3 parágrafos).

•	A extração de entidades (nomes, datas, valores monetários, termos técnicos).

•	A identificação de pendências e responsáveis.

•	A sugestão de tags de categorização.

3.4 Armazenamento dos metadados
•	Amazon DynamoDB: Armazena os metadados estruturados de cada documento como um banco de dados NoSQL de altíssima velocidade, usando o document_id como Partition Key.

•	Amazon Bedrock Knowledge Bases: Importa os metadados e os associa aos chunks de texto para permitir filtragem híbrida durante as consultas (ex: buscar apenas decisões onde document_type = ata e data >= 2026-01-01).

✅ Quest 4: O Oráculo da Wiki Inteligente
4.1 Estratégia de indexação
•	Semantic Chunking (Divisão Semântica): Os documentos são fatiados em blocos de texto (chunks) de 300 a 500 tokens, mantendo uma sobreposição (overlap) de 10% a 15% para preservar o contexto entre frases contíguas.

•	Preservação de Estrutura: Parágrafos de atas não são cortados ao meio. Registros de CSV formam chunks autocontidos por linha/oportunidade.

4.2 Busca semântica e base vetorial
•	Modelo de Embeddings: Uso do Amazon Titan Text Embeddings v2 acessado via Amazon Bedrock, convertendo os chunks de texto em vetores de 1024 dimensões.

•	Vector Database: Os vetores gerados são indexados e armazenados no Amazon OpenSearch Serverless (ou nativamente no repositório gerenciado pelo Amazon Bedrock Knowledge Bases), permitindo busca por similaridade de cosseno e busca híbrida (vetores + termos-chave).

4.3 Geração de respostas com IA
1.	Recebimento: O usuário envia uma pergunta em linguagem natural através do portal web ou chat.

2.	Recuperação Vetorial: O Amazon Bedrock Knowledge Bases gera o embedding da pergunta e realiza uma consulta k-NN no OpenSearch Serverless, resgatando os 5 chunks mais relevantes e seus metadados.

3.	Construção do Prompt Augmented: Os chunks recuperados são injetados no contexto do modelo Anthropic Claude 3.5 Sonnet junto com a pergunta do usuário.

4.	Geração com Citação OBRIGATÓRIA: O modelo compõe a resposta em linguagem natural e adiciona citações diretas ao final das afirmações, no formato [Fonte: nome_do_arquivo, Pág/Linha: X].

4.4 Interface de consulta
•	Amazon Q Business / AWS Amplify: Interface Web responsiva construída com React via AWS Amplify e conectada a uma API no Amazon API Gateway.

•	Autenticação: Gerenciada pelo Amazon Cognito, garantindo que apenas usuários autenticados acessem a Wiki.

•	Roteamento Backend: AWS Lambda recebe a requisição do API Gateway, valida a sessão do Cognito e aciona a API do Amazon Bedrock Agents / Knowledge Bases.

4.5 Segurança, auditoria e monitoramento
•	Controle de Acesso: AWS IAM para permissões entre serviços e Amazon Cognito para perfis de usuário.

•	Criptografia: AWS KMS para dados em trânsito (TLS 1.3) e em repouso (KMS Customer Managed Keys).

•	Auditoria de Operações e Acesso: AWS CloudTrail registra todas as chamadas de API feitas nos serviços AWS. Amazon Macie analisa o S3 em busca de dados sensíveis não intencionais (PII/CPF/Cartões).

•	Observabilidade e Custos: Amazon CloudWatch monitora latências e falhas do Lambda/Bedrock. AWS Cost Explorer e AWS Budgets acompanham a evolução do consumo financeiro da infraestrutura.

🧩 Arquitetura Final da Solução
1. Visão geral
A arquitetura é 100% Serverless, auditável e orientada a eventos. Ela transforma documentos brutos em conhecimento consultável ao isolar o armazenamento original no S3, utilizar pipelines customizadas de extração (Textract / Lambda / Pandas), enriquecer textos com metadados estruturados no DynamoDB via Bedrock, e indexar chunks no OpenSearch Serverless para servir uma interface RAG alimentada pelo Claude 3.5 Sonnet com citação rigorosa de fontes.


2. Serviços AWS utilizados
Serviço AWS	Papel na Solução
Amazon S3	Landing Zone (raw/) e Processed Zone (processed/) imutáveis.
Amazon Textract	OCR avançado e extração de manuscritos (handwriting) em imagens.
Amazon Bedrock	Execução de modelos de Embedding (Titan) e GenAI (Claude 3.5 / Haiku).
Amazon Bedrock Knowledge Bases	Orquestrador de RAG, chunking, vetorização e recuperação contextual.
Amazon OpenSearch Serverless	Banco de dados vetorial de alta performance (Vector Store).
AWS Lambda	Processador Serverless, roteador de eventos e extrator para PDFs/CSVs.
AWS Step Functions	Orquestrador de fluxos assíncronos e tratador de retentativas.
Amazon DynamoDB	Banco NoSQL para armazenamento rápido de metadados dos documentos.
Amazon Cognito	Autenticação, autorização e gerenciamento de sessões de usuários.
Amazon CloudWatch	Monitoramento de métricas, retenção de logs de erro e alarmes.
AWS IAM / AWS KMS	Governança de acessos e criptografia das chaves corporativas.
3. Fluxo de dados de ponta a ponta
1.	Arquivos são carregados na pasta raw/ no Amazon S3.

2.	S3 Event Notification aciona o AWS Step Functions.

3.	O fluxo identifica o tipo do arquivo e roteia: imagens com manuscritos vão para o Amazon Textract; PDFs digitais e CSVs são processados por funções AWS Lambda especializadas.

4.	O texto limpo é gerado e um LLM no Amazon Bedrock extrai o JSON de metadados.

5.	Metadados são salvos no Amazon DynamoDB e os textos processados vão para a Processed Zone no S3.

6.	O Amazon Bedrock Knowledge Bases realiza o chunking, gera vetores com o Titan Text Embeddings v2 e salva no Amazon OpenSearch Serverless.

7.	O usuário autentica via Amazon Cognito no portal Web e faz uma pergunta.

8.	A consulta passa pelo API Gateway / Lambda e ativa o Bedrock Knowledge Bases.

9.	O Anthropic Claude 3.5 Sonnet compõe a resposta citando o nome exato do documento fonte e disponibiliza para o usuário na interface.

4. Diagrama textual da arquitetura
[Pasta raw/] ──> [S3 Landing Zone] ──> [EventBridge] ──> [Step Functions]
                                                              │
                ┌─────────────────────────────────────────────┼─────────────────────────────────────────────┐
                │ (Imagens / Scans)                           │ (PDFs Digitais)                             │ (CSV CRM)
                ▼                                             ▼                                             ▼
       [Amazon Textract]                             [Lambda (pdfplumber)]                       [Lambda (Pandas)]
                │                                             │                                             │
                └─────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                                              │
                                                              ▼
                                               [Bedrock (Haiku Metadata)]
                                                              │
                                        ┌─────────────────────┴─────────────────────┐
                                        ▼                                           ▼
                              [DynamoDB (Metadados)]                      [S3 Processed Zone]
                                                                                    │
                                                                                    ▼
                                                                     [Bedrock Knowledge Bases]
                                                                   (Titan Embeddings v2 + OpenSearch)
                                                                                    │
[Usuário] <──> [Cognito + Amplify] <──> [API Gateway / Lambda] <──> [Bedrock (Claude 3.5 Sonnet)]
5. Riscos e limitações
•	Documentos com severo grau de degradação física: Manuscritos extremamente borrados ou rasgados podem exigir validação ou digitação manual corretiva.

•	Custos de vetorização e busca: Alto volume de documentos reindexados com frequência pode impactar os custos do OpenSearch Serverless e chamadas de API do Bedrock.

•	Alucinação Residual em Prompts: Embora o RAG reduza drasticamente alucinações, prompts mal estruturados do usuário podem levar a interpretações ambíguas se não houver um System Prompt rígido.

•	Manutenção do CSV: Alterações no cabeçalho ou adição de novas colunas no CSV do CRM exigem atualização do script do Lambda extrator.

6. Melhorias futuras
•	Implementação de Bedrock Guardrails: Filtro de políticas de segurança para bloquear conteúdo inadequado e mascarar automaticamente dados sensíveis (PII) nas respostas.

•	Mecanismo de Feedback do Usuário: Botões de "Útil / Não Útil" na interface do chat para registrar o nível de precisão das respostas e refinar os prompts continuamente.

•	Painel Executivo de Pendências: Dashboard no Amazon QuickSight lendo o DynamoDB para listar todas as ações e decisões em aberto extraídas das atas.

•	Suporte Multi-modalidade: Expandir o pipeline para processar e responder perguntas sobre áudios e gravações de reuniões.

🧠 Checklist Final
Antes de entregar, confirme se sua solução responde:


•	[x] Como transformar documentos escaneados em texto? (Amazon Textract com suporte a Handwriting)

•	[x] Como lidar com diferentes formatos dentro da mesma pasta raw/? (Roteamento Serverless dinamizado por Lambda/Step Functions)

•	[x] Como armazenar os documentos originais? (Amazon S3 com Versionamento e KMS)

•	[x] Como preservar a rastreabilidade entre resposta e documento fonte? (RAG com citações explícitas de metadados no Bedrock)

•	[x] Como organizar metadados? (Esquema padronizado JSON armazenado em Amazon DynamoDB)

•	[x] Como criar busca semântica? (Titan Embeddings v2 + OpenSearch Serverless)

•	[x] Como usar Amazon Bedrock na solução? (Enriquecimento de metadados, embeddings e geração de respostas com Claude 3.5)

•	[x] Como proteger documentos sensíveis? (IAM, KMS, Cognito e Macie)

•	[x] Como monitorar falhas? (CloudWatch Alarms, SQS Dead-Letter Queues e SNS)

•	[x] Como a empresa usaria essa Wiki no dia a dia? (Interface Web conversacional para buscas corporativas e tomada de decisão)

🏁 Conclusão
A solução apresentada transforma um passivo de dados ocultos e desorganizados em um ativo estratégico de conhecimento acessível. Ao utilizar uma arquitetura 100% Serverless na AWS, garantimos escalabilidade total, pagamento estritamente pelo uso e segurança de nível corporativo. A combinação de OCR especializado para manuscritos, pipelines customizadas para dados estruturados/não estruturados e o padrão RAG via Amazon Bedrock assegura respostas precisas, contextualizadas e, acima de tudo, completamente auditáveis. Esta proposta resolve o problema operacional de busca de dados, reduz o tempo de tomada de decisão executiva e posiciona a empresa na vanguarda do uso responsável e prático de IA Generativa.
