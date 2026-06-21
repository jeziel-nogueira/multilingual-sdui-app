# Voo Idiomas — Aplicativo Mobile de Prática Complementar Multilíngue

Este repositório contém os artefatos conceituais, de requisitos, arquitetura e documentação acadêmica do **Aplicativo Mobile de Prática de Língua Estrangeira (Inglês e Espanhol Castelhano)**. O projeto é desenvolvido de forma individual como portfólio acadêmico para o **Projeto de Aprendizagem Colaborativa Extensionista VII (PAC ESOFT)** e Trabalho de Conclusão de Curso (TCC) em Engenharia de Software na **Faculdade Católica de Santa Catarina**.

A solução foi projetada em cooperação e parceria piloto direta com a escola de idiomas **Voo Idiomas Jaraguá do Sul**.

---

## O Projeto e Objetivo Geral
O objetivo principal deste ecossistema é apoiar alunos da escola parceira na prática complementar e autônoma de conversação, listening e gramática extraclasse. O aplicativo atua como um **Thick Client** multiplataforma móvel que consome lições dinâmicas e fornece relatórios diagnósticos de desempenho para os professores.

---

## Arquitetura do Ecossistema (Serverless / No-Backend)

A arquitetura foi estruturada para garantir escalabilidade, latência zero de processamento e baixíssimo custo operacional de infraestrutura (aproximadamente R$ 0,71 mensais por aluno ativo em produção comercial).

```
                  ┌────────────────────────────────────────────────────────┐
                  │               CAMADA DE SERVIÇOS (NUVEM)               │
                  │                                                        │
                  │  ┌──────────────┐ (Webhook)  ┌──────────────┐          │
                  │  │ Headless CMS ├───────────►│  BaaS NUVEM  │          │
                  │  │ (Strapi CMS) │            │  (Firebase)  │          │
                  │  └──────┬───────┘            └──────▲───────┘          │
                  │         │                           │                  │
                  │         │ (JSON Conteúdo)           │ (Auth, XP,       │
                  │         │                           │  Telemetria)     │
                  └─────────┼───────────────────────────┼──────────────────┘
                            ▼                           ▼
                  ┌────────────────────────────────────────────────────────┐
                  │             FRONTEND MÓVEL (THICK CLIENT)              │
                  │                                                        │
                  │  ┌──────────────────────┐   ┌───────────────────────┐  │
                  │  │    Mecanismo SDUI    │   │    Motor Fonético     │  │
                  │  │   (Render Dinâmico)  │   │      (Strategy)       │  │
                  │  └──────────────────────┘   └───────────▲───────────┘  │
                  └─────────────────────────────────────────┼──────────────┘
                                                            │ 
                                                            │ (1) Envia Áudio
                                                            │ (4) Retorna Transcrição
                                                            │
                  ┌─────────────────────────────────────────│──────────────┐
                  │                 CAMADA SERVERLESS PROXY │              │
                  │                                         │              │
                  │                                ┌────────▼─────────┐    │
                  │                                │    Serverless    │    │
                  │                                │ Cloud Function   │    │
                  │                                └────────┬─────────┘    │
                  │                                         │              │
                  │                 (2) Envia Áudio + Chave │              │
                  │                 (3) Retorna Transcrição │              │
                  │                                         ▼              │
                  │                                ┌──────────────────┐    │
                  │                                │   Whisper API    │    │
                  │                                │    (Serviço)     │    │
                  │                                └──────────────────┘    │
                  │                                                        │
                  └────────────────────────────────────────────────────────┘
```

### 1. Camada Curricular (Headless CMS)
O **Strapi CMS** é hospedado em PaaS (Render/Railway) e atua como a retaguarda pedagógica. Os professores inserem novos cenários do cotidiano, gabaritos de texto e arquivos de listening diretamente no painel administrativo do Strapi, que os expõe como JSON REST.

### 2. Apresentação Móvel (Server-Driven UI)
O aplicativo Thick Client consome as rotas REST do CMS e reconstrói dinamicamente os layouts e fluxos das lições em tempo de execução. Isso permite que a escola atualize a interface, altere lições ou mude textos de exercícios instantaneamente, sem a necessidade de novos builds móveis ou aprovações demoradas nas lojas Google Play e App Store.

### 3. Persistência de Dados (BaaS)
O **Firebase** é utilizado como BaaS para gerenciar:
* Autenticação segura de usuários.
* Armazenamento NoSQL (Cloud Firestore) para pontuações de XP, ofensivas diárias (Streaks) e telemetrias de exercícios concluídos.
* Segurança baseada em **Regras de Segurança Firestore** declarativas atreladas ao UID autenticado (`auth.uid`).

---

## Diferenciais de Inovação Pedagógica e Engenharia

### A. Motor Fonético Polimórfico (Strategy Pattern)
A correção de fala é processada localmente na borda usando conceitos de **Profundidade Ortográfica (*Orthographic Depth*)**:
* **Inglês (Ortografia Profunda/Opaca):** A transcrição de voz (OpenAI Whisper) e o gabarito textual são normalizados e convertidos em chaves fonéticas pelo algoritmo **Double Metaphone** para neutralizar homófonos (ex: *"their"* e *"there"* são mapeados uniformemente para a assinatura `0R`). Em seguida, aplica-se a **Distância de Levenshtein** sobre as chaves fonéticas geradas.
* **Espanhol Castelhano (Ortografia Rasa/Translúcida):** Como as palavras são faladas exatamente como são escritas, o sistema aplica o cálculo de **Distância de Levenshtein Ortográfica Direta** sobre o texto sanitizado, identificando com precisão os desvios de pronúncia exigidos pela fonologia castelhana da escola parceira.

### B. Heurística Adaptativa determinística (Média Móvel)
Como alternativa viável à complexidade estatística e necessidade de calibração populacional em massa do Modelo de Rasch (Teoria de Resposta ao Item), a aplicação móvel executa uma heurística na borda que calcula a taxa de acerto ponderada das últimas 5 lições de uma competência específica (ex: *Simple Past*). Se a proficiência cair abaixo do limiar de 70\%, a interface (Server-Driven UI) altera dinamicamente o fluxo curricular para injetar exercícios de fixação.

### C. Segurança Contratual e Registro por Código de Ativação
Para impedir cadastros não autorizados nas lojas de aplicativos, o cadastro de estudantes exige um **Código de Ativação Único (Token)** gerado pela secretaria da escola no Strapi CMS.
1. O aluno insere o token no cadastro móvel.
2. Uma **Firebase Cloud Function transacional** valida o token em nuvem, cria o registro do aluno no Firebase Auth, associa os idiomas ativamente contratados ao perfil do usuário no Firestore (`/users/{uid}`) e consome a chave de ativação para prevenir reutilização.
3. Regras de segurança no Firestore bloqueiam qualquer requisição a lições ou exercícios cujo idioma não conste no vetor de matrículas ativas do usuário logado:

```javascript
match /lessons/{lessonId} {
  allow read: if request.auth != null && 
    (resource.data.language in get(/databases/$(database)/
     documents/users/$(request.auth.uid)).data.enrolled_languages);
}
```

### D. Higienização e Privacidade de Dados (LGPD)
Para garantir conformidade legal com a Lei Geral de Proteção de Dados (LGPD - Lei 13.709/18), a exclusão de conta pelo usuário impõe fricção de segurança e dispara um gatilho em nuvem (`onDelete` trigger): os dados identificáveis (PII) são removidos permanentemente, e as telemetrias de logs de progresso pedagógico são higienizadas (substituindo o identificador pessoal `userId` por uma hash desvinculada e irreversível como `"user_deleted_anonymous"`), preservando o ativo histórico de Learning Analytics para a instituição escolar.

---

## Organização de Pastas do Repositório

O repositório está estruturado conforme abaixo, contendo o código/configurações do projeto e a documentação das entregas acadêmicas (`atividade_academica/`):

```
multilingual-sdui-app/
├── .gitignore                           # Regras de ignores do Git (com ignores do LaTeX)
├── README.md                            # Documentação principal (este arquivo)
└── atividade_academica/                 # Arquivos acadêmicos oficiais
    ├── Atividade_N1/                    # PDF da Proposta Inicial
    ├── Atividade_N2/                    # LaTeX e rascunhos da entrega N2
    ├── Atividade_N3/                    # Artigo LaTeX final completo (N3) e Roteiro do Pitch
    └── artigos_referencia/              # PDFs de artigos científicos citados no artigo
```

> [!NOTE]
> Os documentos de suporte originais (`fonte_notion/` e o playbook do TCC) são mantidos externamente ao repositório público de produção.

---

## Referências Acadêmicas Principais
* **ÇAKMAK, F. (2019):** *"Mobile Learning and Mobile Assisted Language Learning in Focus"*. Language and Technology, v. 1, n. 1, p. 30–48.
* **BERNARDO, J. C. O. (2013):** *"Dispositivos móveis digitais na incrementação do processo de ensino e aprendizagem: mobile-learning no rompimento de paradigmas"*. Revista EDaPECI, v. 13, n. 1, p. 141-157.
* **KIM, H.; KWON, Y. H. (2012):** *"Exploring Smartphone Applications for Effective Mobile-Assisted Language Learning"*. Multimedia-Assisted Language Learning, v. 15, n. 1, p. 31-57.
