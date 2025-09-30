
# Tinder de Vagas 🎯 (Análise via Formulário n8n)

Um workflow em n8n que automatiza por completo o processo de triagem de candidatos, desde a submissão do currículo em um formulário até a pontuação de aderência com as vagas abertas, utilizando o poder do Google Gemini.

-----

## 📜 Visão Geral

Este projeto elimina a triagem manual de currículos. Quando um candidato preenche o formulário n8n e anexa seu CV, o workflow executa uma série de ações automáticas: salva o CV, extrai e estrutura todas as informações relevantes, cadastra o candidato em uma base de dados, resume as vagas em aberto (se ainda não tiverem sido resumidas) e, por fim, utiliza um agente de IA especializado em RH para calcular um **score de compatibilidade** entre o candidato e as vagas para as quais ele se aplicou.

## ✨ Funcionalidades Principais

  * **Formulário de Inscrição Nativo:** Utiliza o nó `Form Trigger` do n8n para criar um formulário web personalizável para os candidatos.
  * **Backup Automático de CVs:** Salva uma cópia de cada currículo (PDF) enviado em uma pasta específica no **Google Drive**.
  * **Extração Dupla com IA:** Emprega o Google Gemini para fazer duas extrações de dados em paralelo do texto do CV:
    1.  **Dados Pessoais:** Nome, contato, email, LinkedIn, etc.
    2.  **Qualificações:** Formação acadêmica, histórico profissional e competências técnicas.
  * **Banco de Dados de Candidatos:** Alimenta automaticamente uma planilha do **Google Sheets** com os dados estruturados de cada candidato que se inscreve, criando uma base de talentos.
  * **Resumo Inteligente de Vagas (com Cache):**
      * Quando o candidato seleciona uma área, o workflow busca as vagas correspondentes.
      * Se uma vaga ainda não foi resumida pela IA, ele cria um resumo e o salva de volta na planilha, marcando-a como "resumida". Isso **evita reprocessamento** e economiza custos de API.
  * **Agente de RH Virtual:** Um prompt especializado instrui a IA a agir como um recrutador, analisando o perfil do candidato contra o perfil da vaga e atribuindo um **score de 0 a 10**.
  * **Relatório Final:** Consolida os scores de compatibilidade em uma planilha de resultados, pronta para a análise do time de recrutamento.

## 🤖 Como o Workflow Funciona

O fluxo é robusto e dividido em dois ramos principais de processamento que ocorrem em paralelo antes de se unirem para o "match" final.

**[Início] -\> Form Trigger -\> [RAMO A: Processamento do Candidato] & [RAMO B: Processamento das Vagas] -\> [Junção] -\> Análise Final -\> [Fim]**

-----

### **Etapa 1: Entrada do Candidato (`Form Trigger`)**

1.  **`On form submission`:** Um formulário web nativo do n8n é o ponto de partida. Ele coleta **Nome**, **Email**, o arquivo do **CV (.pdf)** e a **área de interesse (`Job Role`)** através de um menu dropdown.

-----

### **Etapa 2 (Ramo A): Processamento e Extração de Dados do Candidato**

1.  **`Upload CV`:** O PDF do currículo é enviado para uma pasta no **Google Drive**.
2.  **`Extract from File`:** O texto puro é extraído do PDF.
3.  **`Personal Data` & `Qualifications` (Execução em Paralelo):** O texto extraído é enviado para dois nós de extração da IA (Gemini) simultaneamente:
      * Um extrai os dados de contato (nome, email, etc.).
      * O outro extrai os dados profissionais (experiência, formação, skills).
4.  **`Merge` & `Code in JavaScript1`:** Os dados extraídos são unificados. Um script gera um **ID único** para o candidato (baseado no timestamp `Date.now()`) e formata tudo em um objeto JSON limpo.
5.  **`Append row in sheet`:** O candidato é salvo na planilha "Candidatos Cadastrados", criando um registro permanente.
6.  **`Summarization Chain`:** A IA cria um resumo conciso (até 150 palavras) do perfil completo do candidato.

-----

### **Etapa 3 (Ramo B): Filtragem e Resumo das Vagas**

1.  **`job roles`:** O workflow lê a planilha "vagas" e filtra apenas as vagas que correspondem ao `Job Role` selecionado pelo candidato no formulário.
2.  **`If "Resumo_ia" == "NÃO"`:** O sistema verifica se as vagas filtradas já possuem um resumo gerado pela IA (através de uma coluna de controle).
3.  **Se NÃO tiver resumo:**
      * **`Summarization Chain1`:** A IA é acionada para criar um resumo da vaga.
      * **`Code in JavaScript`:** O código garante que a saída da IA esteja em um formato JSON limpo.
      * **`Append or update row in sheet`:** O resumo é salvo de volta na planilha da vaga, e a coluna de controle `Resumo_ia` é atualizada para "SIM".
4.  **`job roles2`:** O fluxo lê novamente a planilha de vagas para garantir que possui os resumos atualizados de todas as vagas relevantes.

-----

### **Etapa 4: O "Match" - Análise de Aderência**

1.  **`Code in JavaScript3`:** Este nó é o coração da junção. Ele pega o resumo do candidato (do Ramo A) e o replica para cada uma das vagas relevantes (do Ramo B), criando os pares de "candidato vs. vaga" que serão analisados.
2.  **`HR Expert` (IA):** Cada par é enviado para o Gemini com um prompt que o instrui a agir como um **Especialista em RH**. Ele analisa a aderência e atribui um **score de 0 a 10**.
3.  **`Structured Output Parser`:** Garante que a resposta da IA seja um JSON estruturado contendo `id_vaga`, `candidate_id` e `score`.

-----

### **Etapa 5: Registro do Resultado Final**

1.  **`Plan. Final`:** O resultado final (os scores do candidato para cada vaga) é adicionado à planilha "resultado\_tinder\_vagas", concluindo o processo.

## 🛠️ Configuração e Instalação

1.  **Importe o Workflow:** Baixe o arquivo `.json` deste projeto e importe-o para sua instância n8n.
2.  **Credenciais:** Você precisará configurar três credenciais no seu n8n:
      * **Google Drive** (OAuth2)
      * **Google Sheets** (OAuth2)
      * **Google Gemini (PaLM) API** (sua chave de API do Google AI Studio)
3.  **Planilhas Google Sheets:** Crie 3 planilhas (ou abas em uma única planilha) e configure as colunas:
      * **Planilha de Vagas:** Colunas para `id_vaga`, `job_role`, detalhes da vaga (ex: `perfil_nivel_academico`, `perfil_competencia_tecnicas_e_comportamentais`), e a coluna de controle **`Resumo_ia`** (preenchida com "NÃO" para novas vagas).
      * **Planilha de Candidatos:** Colunas para `ID`, `nome candidato`, `email`, `linkedin`, `formacao academica`, etc.
      * **Planilha de Resultados:** Colunas para `id_vaga`, `id_candidato`, `score`.
4.  **Google Drive:** Crie uma pasta para armazenar os CVs.
5.  **Atualize os Nós:**
      * Em cada nó do **Google Sheets** e **Google Drive**, associe a credencial correta e insira os IDs da sua planilha/pasta.
      * Nos nós de IA (`Google Gemini Chat Model`), associe sua credencial do Gemini.
6.  **URL do Formulário:** Clique no nó inicial **`On form submission`** e copie a URL do formulário (`Form URL`). É este link que você enviará aos candidatos.

## 🚀 Como Usar

1.  **Cadastre as Vagas:** Mantenha sua planilha de **Vagas** atualizada, lembrando de colocar "NÃO" na coluna `Resumo_ia` para qualquer vaga nova ou alterada.
2.  **Ative o Workflow:** Ligue o botão "Active" no canto superior direito da tela do n8n.
3.  **Divulgue o Formulário:** Compartilhe o link do formulário de inscrição com os candidatos.
4.  **Acompanhe os Resultados:** À medida que os candidatos se inscrevem, a planilha de **Resultados** será preenchida automaticamente com os scores de compatibilidade, permitindo que sua equipe de RH foque apenas nos melhores talentos.
