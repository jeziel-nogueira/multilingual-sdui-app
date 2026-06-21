# TCC: Escopo Pedagógico, Casos de Uso e Modelo de Requisitos

Este documento consolida o escopo pedagógico, a especificação textual de casos de uso (UML) e o modelo formal de requisitos funcionais e não funcionais do **Aplicativo Mobile de Prática de Língua Estrangeira**. Ele atua como o alicerce de escopo e especificação de requisitos para a monografia do TCC.

---

## 1. Objetivos do Projeto

### 1.1 Objetivo Geral
Desenvolver um aplicativo mobile educacional voltado para a prática complementar de língua estrangeira, permitindo que alunos de uma escola de idiomas realizem exercícios interativos de gramática, compreensão auditiva e expressão oral, contribuindo para o reforço do aprendizado fora do ambiente de sala de aula.

### 1.2 Objetivos Específicos
* Desenvolver uma aplicação mobile acessível e de fácil utilização para alunos da escola de idiomas.
* Implementar exercícios de prática gramatical por meio de perguntas de múltipla escolha e atividades de completar frases.
* Disponibilizar exercícios de compreensão auditiva (*listening*) baseados em áudios com lacunas para preenchimento.
* Implementar atividades de prática de fala (*speaking*) com gravação de áudio e verificação da resposta por meio de reconhecimento de fala.
* Criar módulos de aprendizagem baseados em situações do cotidiano, como interações em restaurantes, hotéis e outros contextos comuns de comunicação em inglês e espanhol castelhano.
* Fornecer feedback imediato ao aluno sobre suas respostas e desempenho nas atividades.
* Registrar o progresso do aluno nas atividades realizadas, permitindo acompanhamento da evolução no aprendizado.
* Projetar a arquitetura do sistema de forma que permita expansão futura de funcionalidades e conteúdos educacionais.

---

## 2. Escopo do Projeto

A aplicação será projetada para funcionar como ferramenta complementar ao ensino presencial, oferecendo atividades interativas que estimulem o contato frequente com o idioma.

O sistema disponibilizará diferentes tipos de exercícios, incluindo atividades de gramática, exercícios de compreensão auditiva e práticas de fala baseadas em repetição de frases nos idiomas inglês e espanhol castelhano. Também serão incluídos módulos baseados em situações do cotidiano, permitindo que os alunos pratiquem expressões e diálogos comuns utilizados em contextos reais de comunicação nos respectivos idiomas.

O aplicativo permitirá que os alunos acessem atividades educacionais, respondam exercícios e recebam feedback imediato sobre seu desempenho. Além disso, o sistema registrará informações básicas de progresso do usuário, permitindo acompanhar sua evolução ao longo das atividades.

Para a implementação da funcionalidade de reconhecimento de fala, o sistema utilizará serviços externos de processamento de áudio e conversão de fala em texto encapsulados em nuvem.

Não fazem parte do escopo inicial do projeto funcionalidades avançadas como sistemas completos de gamificação competitiva (ligas interativas), chat entre múltiplos usuários ou integração direta com sistemas administrativos de cobrança e notas da escola, podendo essas funcionalidades ser consideradas para versões futuras do sistema.

### Atores do Sistema
* **Aluno:** Usuário final da aplicação mobile que interage com os módulos pedagógicos. Suas responsabilidades incluem a realização de exercícios interativos de gramática, compreensão auditiva e expressão oral, além de gerenciar o próprio perfil, acompanhar sua evolução por meio de relatórios de proficiência e interagir com as mecânicas de engajamento e gamificação.
* **Professor / Administrador:** Agente responsável pela gestão pedagógica e estratégica do ecossistema. Suas responsabilidades concentram-se na modelagem e inserção de conteúdos educacionais (exercícios, textos, trilhas e mídias) por meio do painel web do Headless CMS, bem como no monitoramento analítico do desempenho dos estudantes e das turmas através de relatórios gerenciais e de Business Intelligence.

---

## 3. Especificação Textual de Casos de Uso (UML)

### 3.1 Atores: Aluno

#### **CSU01 — Registrar Conta (Fricção Contratual via Token)**
* **Descrição:** Permite que um novo estudante crie suas credenciais de acesso locais informando dados pessoais básicos e um **Código de Ativação Único** alfanumérico fornecido pela secretaria da escola. O sistema valida transacionalmente o token antes de criar a conta, habilitando seu perfil e definindo os idiomas autorizados de estudo no banco de dados.

#### **CSU02 — Efetuar Autenticação**
* **Descrição:** Permite a validação de identidade do usuário por meio de e-mail e senha para liberação dos recursos da aplicação móvel de acordo com seu perfil de acesso (Role).

#### **CSU03 — Selecionar Módulo de Conteúdo**
* **Descrição:** Permite que o estudante navegue pelos cenários do cotidiano e trilhas de idiomas disponíveis e selecione a unidade pedagógica que deseja praticar (inglês ou espanhol castelhano), sendo essa navegação restrita estritamente ao(s) idioma(s) no(s) qual(is) possui matrícula ativa e autorização cadastrada no seu perfil de usuário.

#### **CSU04 — Realizar Exercício de Gramática**
* **Descrição:** Permite a execução de atividades de múltipla escolha e preenchimento de lacunas textuais, recebendo correção automatizada imediata do sistema.

#### **CSU05 — Realizar Exercício de Compreensão Auditiva (Listening)**
* **Descrição:** Permite o acionamento de arquivos de áudio por streaming e a posterior resolução de questionários de interpretação associados ao conteúdo ouvido.

#### **CSU06 — Realizar Exercício de Expressão Oral (Speaking)**
* **Descrição:** Permite a gravação de voz pelo microfone do dispositivo para repetição de frases propostas, disparando a rotina de transmissão do áudio para a Serverless Function (proxy seguro), transcrição externa parametrizada por idioma (OpenAI Whisper) e a avaliação polimórfica de pronúncia local baseada na Profundidade Ortográfica de cada idioma (Double Metaphone + Levenshtein para o inglês, e Levenshtein direto sanitizado para o espanhol castelhano).

#### **CSU07 — Consultar Painel de Desempenho (Dashboard do Aluno)**
* **Descrição:** Permite a visualização de gráficos e métricas consolidadas sobre a evolução do usuário, segmentada por tempo gasto, taxa de acerto semanal e proficiência por macro-habilidade linguística.

#### **CSU13 — Excluir Conta (Fluxo Seguro LGPD)**
* **Descrição:** Permite que o Aluno solicite a eliminação do seu perfil de usuário através de uma jornada de fricção projetada de UX no aplicativo, exigindo digitação de frase de controle e reautenticação ativa de credenciais. A ação dispara em nuvem o descarte permanente de seus dados identificáveis (PII) e a higienização irreversível (anonimização) de suas telemetrias pedagógicas.

#### **CSU14 — Gerenciar Perfil e Preferências**
* **Descrição:** Permite que o Aluno selecione o seu idioma ativo de estudo (limitado estritamente àqueles aos quais possui autorização de matrícula no banco), altere sua senha de acesso, configure lembretes diários de lembrete e efetue o logout seguro.

---

### 3.2 Atores: Professor / Administrador

#### **CSU09 — Estruturar Componentes de Interface (Server-Driven UI)**
* **Descrição:** Permite que o docente utilize a interface visual do Headless CMS para agrupar e ordenar os blocos de componentes dinâmicos que ditarão o layout das lições no aplicativo móvel.

#### **CSU10 — Gerenciar Exercícios e Lições**
* **Descrição:** Permite a inclusão, alteração e exclusão de enunciados textuais, alternativas, marcações gramaticais de erro, strings de gabarito técnico de voz para o motor de speaking e a respectiva tag de parametrização de idioma (EN ou ES) no Headless CMS.

#### **CSU11 — Gerenciar Biblioteca de Ativos Digitais (Media Library)**
* **Descrição:** Permite o upload, organização e vinculação de arquivos de áudio de listening e imagens ilustrativas aos respectivos módulos conceituais hospedados no servidor.

#### **CSU12 — Consultar Painel Gerencial Docente (Dashboard do Professor)**
* **Descrição:** Permite o acompanhamento analítico do progresso pedagógico das turmas, incluindo gráficos de desempenho coletivo, identificação de alunos estagnados na curva de aprendizado e mapeamento das maiores dificuldades gramaticais da turma.

#### **CSU15 — Gerenciar Matrículas e Acessos Linguísticos de Estudantes (Via Headless CMS)**
* **Descrição:** Permite que o Professor/Administrador gerencie as permissões pedagógicas e a ativação seletiva de idiomas (EN e/ou ES) de cada estudante a partir do painel administrativo do Headless CMS (Strapi). Essa alteração dispara automaticamente um Webhook que invoca uma Firebase Cloud Function, a qual atualiza o perfil do Aluno no banco Firestore, garantindo conformidade contratual em tempo real.

---

## 4. Modelo Formal de Requisitos

### 4.1 Requisitos Funcionais (RF)

* **RF01 — Cadastro de Alunos (Validação de Código de Ativação):** O sistema deve permitir que o Aluno realize o seu cadastro pela tela inicial do aplicativo móvel informando nome, e-mail, senha e um **Código de Ativação Único** (token) fornecido pela escola. A criação da conta é intermediada por uma Firebase Cloud Function transacional que valida o token, vincula os idiomas contratados ao perfil do usuário no Firestore, e consome o token de forma irreversível para evitar fraudes ou reutilizações.
* **RF02 — Autenticação de Usuários:** O sistema deve permitir que Alunos e Professores realizem a autenticação na aplicação por meio de e-mail e senha, direcionando o fluxo de telas de acordo com o perfil de acesso (Role) do usuário logado.
* **RF03 — Gerenciamento de Conteúdo Pedagógico:** O sistema deve fornecer uma interface web (via Headless CMS) para que o Administrador/Professor cadastre, edite e remova módulos de conteúdo, cenários do cotidiano, perguntas de múltipla escolha, frases de preenchimento, gabaritos de fala e a parametrização obrigatória de idioma correspondente (EN ou ES) para cada lição/exercício.
* **RF04 — Renderização Dinâmica de Exercícios (Server-Driven UI):** O aplicativo móvel deve consumir as estruturas de dados em formato JSON vindas do servidor e renderizar dinamicamente a interface e o layout das atividades pedagógicas em tempo de execução.
* **RF05 — Prática de Gramática:** O sistema deve apresentar ao Aluno exercícios de gramática baseados em múltipla escolha em conformidade com o módulo selecionado.
* **RF06 — Exercícios de Completar Frases:** O sistema deve permitir que o Aluno realize atividades de preenchimento de lacunas (*fill-in-the-blank*) inserindo os termos gramaticais faltantes.
* **RF07 — Exercícios de Compreensão Auditiva (Listening):** O sistema deve permitir que o Aluno reproduza arquivos de áudio sob demanda e responda a questionários de interpretação associados à mídia executada.
* **RF08 — Exercícios de Expressão Oral (Speaking):** O sistema deve permitir que o Aluno capture e grave sua voz através do microfone do dispositivo móvel para repetir frases em inglês ou espanhol castelhano propostas pelo exercício.
* **RF09 — Transcrição de Áudio (Speech-to-Text):** O sistema deve transmitir o arquivo de áudio gravado pelo Aluno para uma Serverless Function (proxy seguro de APIs) que insere de forma segura a chave de acesso, encaminha a chamada para o serviço cognitivo externo (OpenAI Whisper) parametrizado com o respectivo idioma (EN ou ES) e retorna a transcrição literal da fala como string.
* **RF10 — Algoritmo de Avaliação de Pronúncia (Edge Computing):** O sistema deve executar localmente de forma polimórfica (Strategy Pattern) o algoritmo de similaridade baseado em Profundidade Ortográfica. Para o inglês (ortografia profunda), deve calcular a distância de Levenshtein sobre as chaves Double Metaphone das palavras. Para o espanhol castelhano (ortografia rasa), deve aplicar a distância de Levenshtein ortográfica direta sobre o texto sanitizado, capturando os desvios fonéticos do estudante diretamente nas variações dos grafemas.
* **RF11 — Feedback Pedagógico Imediato:** O sistema deve validar as respostas do Aluno instantaneamente e emitir um retorno visual de acerto ou erro. Em caso de erro gramatical mapeado, o sistema deve exibir uma orientação qualitativa contextualizada sobre a falha (ex: revisão de tempo verbal).
* **RF12 — Telemetria de Progresso e Métricas:** O sistema deve registrar de forma persistente o desempenho do Aluno ao final de cada atividade, salvando o identificador do usuário, identificador do exercício, percentual de acerto, tempo gasto na execução e data da atividade.
* **RF13 — Motor Adaptativo de Recomendação:** O sistema deve analisar o histórico de métricas acumuladas do Aluno e, por meio de uma heurística adaptativa baseada em regras lógicas determinísticas (atuando como solução de contorno viável frente a modelos estatísticos complexos como a Teoria de Resposta ao Item / Modelo de Rasch), injetar automaticamente sugestões de revisões e exercícios de reforço focados nas competências de menor desempenho do usuário.
* **RF14 — Painel Analítico do Estudante (Dashboard Aluno):** O sistema deve exibir ao Aluno gráficos consolidados de sua evolução pedagógica semanal, indicando o tempo médio por questão, percentual de acerto geral e o desempenho isolado por macro-habilidade (Gramática, Listening, Speaking e Vocabulário).
* **RF15 — Painel Gerencial Docente (Dashboard Professor):** O sistema deve disponibilizar ao Professor uma visão analítica consolidada das turmas, permitindo visualizar o ranking de maiores dificuldades coletivas, alunos em situação de estagnação na curva de aprendizado e a estimativa de nível de proficiência (padrão CEFR) de cada estudante.
* **RF16 — Sistema de Ofensiva (Streaks / Dias Consecutivos):** O sistema deve monitorar os dias consecutivos de acesso do Aluno e exibir um contador visual de engajamento na interface principal do aplicativo, salvando o timestamp do último acesso no Firebase. Se ele praticar no dia seguinte, o contador incrementa. Se passar de 24 horas sem acessar, o contador zera de forma matemática simples.
* **RF17 — Sistema de Experiência e Nível (XP e Level Up):** O sistema deve acumular pontos de experiência (XP) na conta do aluno proporcionalmente à complexidade da lição concluída (ex: 10 XP para gramática, 20 XP para speaking). O nível do estudante (ex: Nível 1 - *Beginner Traveler*, Nível 5 - *Fluent Negotiator*) será calculado no frontend móvel com base no campo `total_xp` gravado no banco de dados.
* **RF18 — Mural de Conquistas (Badges / Insígnias Estéticas):** O aplicativo móvel deve acender insígnias estéticas fixas no perfil do usuário de forma automática quando ele atingir marcos de engajamento ou proficiência armazenados na sua telemetria local (ex: Medalha "Tagarela" ao concluir 5 exercícios de speaking, ou "Madrugador" ao realizar uma tarefa antes das 7h).
* **RF19 — Fluxo Seguro de Exclusão de Conta (Fricção UX):** Para evitar deleções acidentais e irreversíveis pelo usuário, o sistema deve impor um fluxo de alta fricção projetada de UX: o aplicativo exibirá um modal de aviso explícito detalhando o descarte permanente do perfil (XP, Ofensivas, Medalhas) e a anonimização das telemetrias. O botão de confirmação final só será habilitado após o Aluno digitar exatamente a frase confirmatória "EXCLUIR MINHA CONTA" em um campo de entrada de texto, exigindo ainda a reautenticação de credenciais ativa (e-mail/senha) antes da invocação do trigger no BaaS.
* **RF20 — Controle de Acesso e Sincronização de Idiomas (CMS-BaaS Sync):** O sistema deve sincronizar as permissões de idiomas ativos do Aluno cadastrados na retaguarda (Strapi CMS) com o Firebase Firestore de forma automatizada por meio de Webhooks acionando uma Firebase Cloud Function. O aplicativo móvel deve ler o vetor de idiomas ativos no perfil do usuário (`enrolled_languages`) e ocultar ou desabilitar o acesso às trilhas pedagógicas de idiomas para os quais o aluno não possui matrícula ativa.

---

### 4.2 Requisitos Não Funcionais (RNF)

* **RNF01 — Plataforma e Compatibilidade:** O sistema de uso do aluno e visualização de relatórios do professor deve ser desenvolvido como uma aplicação móvel multiplataforma, compatível com o sistema operacional Android (versão 8.0 ou superior).
* **RNF02 — Usabilidade e Acessibilidade:** O aplicativo móvel deve seguir as diretrizes de design de interface estabelecidas pelo Material Design (Android), garantindo contraste de componentes, legibilidade e navegação fluida em telas de diferentes resoluções.
* **RNF03 — Performance e Tempo de Resposta:** A validação de respostas locais e o processamento do algoritmo de similaridade de fala na borda devem ser concluídos em um tempo de resposta inferior a 2 segundos a partir do clique de envio do usuário.
* **RNF04 — Escalabilidade da Infraestrutura:** A arquitetura do backend deve ser baseada em microsserviços integrados e provisionada em uma plataforma como serviço (PaaS), garantindo o suporte ao crescimento de requisições concorrentes sem degradação do tempo de resposta da API.
* **RNF05 — Segurança, Privacidade e LGPD:** A persistência de dados em nuvem deve utilizar um mecanismo de Regras de Segurança no Servidor (*Security Rules*) atrelado à identidade criptográfica do usuário (`auth.uid == userId`). Para conformidade estrita com a Lei Geral de Proteção de Dados (LGPD - Lei 13.709/18), a exclusão de conta solicitada pelo Aluno executará uma rotina automatizada de anonimização via Serverless Cloud Function (onDelete trigger): os dados pessoais identificáveis (PII) serão excluídos permanentemente do banco, enquanto os logs de telemetria pedagógica de acertos/erros serão higienizados (substituindo o `userId` por uma hash anônima desvinculada), garantindo a privacidade absoluta do titular e preservando o ativo histórico do Learning Analytics para a instituição parceira.
* **RNF06 — Integridade de Chaves de API:** As chaves de faturamento e acesso utilizadas para consumo de APIs cognitivas de terceiros devem ser protegidas no lado do servidor por meio do encapsulamento em Serverless Functions (Firebase Cloud Functions / proxy seguro), garantindo que nenhuma chave de API secreta seja compilada ou armazenada estaticamente no código binário do aplicativo móvel distribuído nas lojas oficiais.
* **RNF07 — Disponibilidade:** Os serviços em nuvem utilizados (Headless CMS e BaaS) devem garantir uma taxa de disponibilidade (*Uptime*) mínima de 99,5% para assegurar o acesso dos alunos fora do ambiente de sala de aula em qualquer período do dia.
