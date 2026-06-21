# TCC: Especificação Arquitetural, Gestão de Infraestrutura e Análise de Riscos

Este documento centraliza toda a engenharia de software complexa do aplicativo de idiomas. Ele consolida os diagramas de fluxo de dados, a modelagem de classes e estratégias de processamento de fala, as políticas de segurança da informação, a justificativa de infraestrutura em nuvem (*PaaS* vs. *IaaS*), a matriz de riscos técnicos e as planilhas de estimativa de custos do projeto.

---

## 1. Visão Geral da Arquitetura do Ecossistema

Para atender aos requisitos de escalabilidade, performance de usabilidade e viabilidade de tempo de desenvolvimento para o MVP móvel, a solução adota uma **Arquitetura Baseada em Serviços e APIs (No-Backend / Serverless)** orientada ao paradigma de cliente centrado em regras (**Thick Client**).

O objetivo primordial desta abordagem é desacoplar totalmente a camada de apresentação das regras rígidas de layout (via *Server-Driven UI*), eliminando a necessidade de codificação de uma API proprietária do zero e mitigando gargalos operacionais de deploy e aprovação em lojas de aplicativos (Google Play e App Store).

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
                  └─────────────────────────────────────────│──────────────┘
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

---

## 2. Abordagem Fonética Híbrida: Ortografias Profundas vs. Ortografias Rasas

Um dos maiores desafios de aplicativos móveis de prática de fala (*speaking*) é a calibração da rigidez do corretor de pronúncia, evitando falsos negativos na avaliação de sotaques.

Para resolver essa limitação com alto rigor acadêmico, o motor de fala foi desenhado sob o paradigma linguístico de **Profundidade Ortográfica (*Orthographic Depth*)**.

```
                           ┌────────────────────────────────┐
                           │    Motor de Avaliação Híbrido  │
                           └───────────────┬────────────────┘
                                           │
                         [Verifica Metadado de Idioma (Strapi)]
                                           │
                   ┌───────────────────────┴───────────────────────┐
                   ▼                                               ▼
          [Idioma: Inglês]                                [Idioma: Espanhol]
   (Ortografia Profunda - Opaca)                     (Ortografia Rasa - Translúcida)
                   │                                               │
                   ▼                                               ▼
   [Camada Double Metaphone]                         [Camada de Sanitização Ortográfica]
                   │                                               │
                   ▼                                               ▼
      [Levenshtein Fonético]                          [Levenshtein Ortográfico Direto]
```

### 2.1 Idioma Inglês: Ortografia Profunda (Opaca)
A relação grafema-fonema no inglês é irregular: palavras com grafias diferentes podem ter a mesma pronúncia (*homófonos*).
* **Solução de Contorno Híbrida (PED — *Phonetic Edit Distance*):** O aplicativo converte as palavras da transcrição do Whisper e do gabarito em chaves sonoras baseadas no algoritmo **Double Metaphone** (ex: *"their"* e *"there"* mapeiam uniformemente para a chave fonética `0R`). Em seguida, executa o cálculo de **Distância de Levenshtein** sobre as chaves fonéticas concatenadas. Isso impede que o aluno seja penalizado por pequenos desvios ortográficos em palavras cujas assinaturas sonoras são idênticas.

### 2.2 Idioma Espanhol Castelhano: Ortografia Rasa (Simétrica)
A relação grafema-fonema no espanhol é linear e transparente: as palavras são faladas exatamente como são escritas.
* **Solução de Contorno Ortográfica Direta:** Como a representação escrita é simétrica à falada, qualquer desvio de articulação do estudante (capturado pela IA de Speech-to-Text Whisper) é refletido diretamente como um desvio de caracteres na transcrição ortográfica (ex: o erro na vibrante múltipla de *"perro"* gera *"pero"*).
* **Vantagem e sotaque Castelhano:** O aplicativo dispensa traduções fonéticas, aplicando a **Distância de Levenshtein Ortográfica Direta** sobre o texto sanitizado. Isso simplifica o código do frontend e capta desvios de sotaques regionais da Espanha de forma natural (ex: avaliando rigidamente a distinção típica de Castela entre /s/ na letra 'S' e /θ/ em 'Z' e 'C' antes de E/I).

---

### 2.3 Modelagem Arquitetural do Motor de Fala (Strategy Pattern)

Para evitar acoplamentos rígidos e ramificações complexas no código, a rotina de avaliação delega a extração das strings de similaridade para uma estrutura baseada no padrão de projeto **Strategy** no frontend móvel:

```javascript
// Interface Base da Estratégia
class PhoneticEvaluationStrategy {
    evaluate(gabarito, transcricao) {
        throw new Error("Método evaluate() deve ser implementado.");
    }
}

// Estratégia para Inglês (Fonética Profunda - Requer Double Metaphone)
class EnglishEvaluationStrategy extends PhoneticEvaluationStrategy {
    evaluate(gabarito, transcricao) {
        const chavesGabarito = normalizar(gabarito).split(' ').map(w => doubleMetaphone(w)[0]).join(' ');
        const chavesTranscricao = normalizar(transcricao).split(' ').map(w => doubleMetaphone(w)[0]).join(' ');
        
        return calcularLevenshteinPercentual(chavesGabarito, chavesTranscricao);
    }
}

// Estratégia para Espanhol (Fonética Rasa - Ortografia Direta Sanitizada)
class SpanishEvaluationStrategy extends PhoneticEvaluationStrategy {
    evaluate(gabarito, transcricao) {
        const textoGabarito = normalizarEspanhol(gabarito);
        const textoTranscricao = normalizarEspanhol(transcricao);
        
        return calcularLevenshteinPercentual(textoGabarito, textoTranscricao);
    }
}
```

---

## 3. Políticas de Segurança e Proteção de Chaves

Como a arquitetura prescinde de um backend proprietário tradicional corporativo intermediário para gerenciar permissões, a integridade e segurança do ecossistema dependem de três mecanismos rígidos:

### A. Encapsulamento de Chaves em Serverless Functions (Firebase Cloud Functions)
As chaves de acesso privadas de faturamento de APIs cognitivas externas (OpenAI Whisper) **nunca** são inseridas, compiladas ou expostas no código binário do aplicativo móvel. 
* O aplicativo envia o áudio gravado e o metadado do idioma (ex: `"es"` ou `"en"`) para uma **Serverless Cloud Function** protegida no lado do servidor.
* A Cloud Function recupera com segurança a chave da OpenAI a partir do Secret Manager, insere a flag de idioma de forma estrita no payload da requisição (forçando o Whisper a buscar o dicionário correto e evitando que tente traduzir foneticamente espanhol como inglês) e dispara a chamada. A Cloud Function retorna apenas a string textual limpa para o aplicativo, blindando a chave contra engenharia reversa do APK.

### B. Regras de Segurança Declarativas no Banco de Dados (BaaS Security Rules)
A proteção dos dados e a prevenção de fraudes no ganho de XP e Streaks são tratadas no nível do BaaS (Firestore/Supabase RLS). São configuradas regras declarativas atreladas à identidade criptográfica única (`auth.uid`) do estudante. Um aluno autenticado terá permissão restrita de leitura e escrita exclusivamente sobre seus próprios caminhos e registros de progresso, impedindo a injeção ou adulteração de pontuações de terceiros.

### C. Ofuscação de Código Binário
O build final de distribuição do aplicativo móvel aplicará ferramentas de ofuscação de código e compilação binária (ProGuard, R8 no Android ou obfuscadores nativos do compilador Flutter/React Native). Isso mascara referências internas, nomes de classes e endpoints da Cloud Function contra decompilação (*apktool* ou *Frida*).

### D. Adequação à LGPD via Anonimização de Dados (onDelete Triggers)
Para assegurar a conformidade legal com o direito de eliminação de dados (Direito ao Esquecimento) previsto na Lei Geral de Proteção de Dados (LGPD) sem corromper o ativo analítico da instituição parceira:
* A exclusão de conta solicitada pelo aluno no frontend móvel (`currentUser.delete()`) dispara um gatilho assíncrono no servidor Firebase (`functions.auth.user().onDelete()`).
  * **Fricção UX de Segurança:** Para mitigar totalmente cliques acidentais, o botão de deleção no aplicativo móvel só será ativado se o Aluno preencher um campo de texto com a chave confirmatória estrita: `"EXCLUIR MINHA CONTA"`.
  * **Reautenticação Nativa (Firebase Security):** O SDK do Firebase Auth implementa nativamente uma política rígida de segurança que impede a exclusão de contas com sessões "frias" (antigas). Se a última autenticação do usuário tiver ocorrido há mais de 5 minutos, a chamada `delete()` falhará no client-side com o erro `auth/requires-recent-login`. O aplicativo capturará este erro e exibirá obrigatoriamente um fluxo de reautenticação ativa (exigindo que o aluno digite suas credenciais novamente) para validar a identidade e confirmar a intenção voluntária e consciente de exclusão.
* A Cloud Function executa a exclusão permanente dos dados pessoais identificáveis (PII) do usuário na coleção `users` (nome, e-mail, foto).
* Em seguida, ela realiza a varredura e a higienização de todos os registros na coleção de telemetrias (`exercise_logs`), substituindo o campo `userId` por uma hash desvinculada e irreversível (ex: `"user_deleted_anonymous"`).
* Isso garante a privacidade do titular do dado ao mesmo tempo em que preserva a integridade estatística global das curvas de aprendizado exibidas no painel de *Learning Analytics* dos professores.

### E. Controle de Acesso Baseado em Matrícula (CMS-BaaS Sync e Firestore Security Rules)
Como a escola de idiomas não liberará indiscriminadamente todas as trilhas linguísticas para todos os alunos matriculados, a arquitetura implementa um controle de acesso estrito atrelado ao plano contratual do estudante.

1. **Modelagem de Matrículas no Strapi CMS:**
   No painel administrativo do CMS, é criado o schema de conteúdo de Coleção `Estudante` (ou `Enrollment`) com os seguintes atributos de interesse:
   * `nome` (String)
   * `email` (String, Chave de Identificação única)
   * `status_matricula` (Enum: "Ativo", "Inativo")
   * `idiomas_autorizados` (Array/Enum: `["en", "es"]`)

2. **Fluxo Assíncrono de Sincronização via Webhooks:**
   Sempre que um registro de `Estudante` for criado ou atualizado no Strapi (ex: inserção de um aluno em uma nova trilha de Espanhol), o Strapi aciona um **Webhook** nativo apontando para um endpoint HTTPS exposto por uma **Firebase Cloud Function** (`syncEnrollment`).
   * A Cloud Function recebe o payload seguro, busca pelo e-mail do aluno na coleção Firestore `/users` e localiza o UID criptográfico do Firebase Auth.
   * Se o usuário existir, a Cloud Function atualiza o campo `enrolled_languages` (vetor contendo strings de identificação de idiomas, ex: `["en"]`) e o campo `is_active` dentro do documento do estudante `/users/{userId}`.

3. **Validação e Restrição Estrita no Servidor (Firestore Security Rules):**
   Para impedir que alunos tentem realizar requisições de API a exercícios ou lições de idiomas que não contrataram (manipulando parâmetros de rede ou burlam o frontend do Thick Client), o banco de dados implementa regras de segurança declarativas no servidor.
   As regras leem dinamicamente o documento do usuário antes de liberar a leitura das coleções de lições e exercícios:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Função auxiliar que lê a lista de idiomas matriculados no documento do próprio usuário autenticado
    function getEnrolledLanguages() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.enrolled_languages;
    }
    
    // Validação de acesso para leitura das lições Server-Driven UI
    match /lessons/{lessonId} {
      allow read: if request.auth != null && 
                   (resource.data.language in getEnrolledLanguages());
    }
    
    // Validação de acesso para execução de exercícios interativos
    match /exercises/{exerciseId} {
      allow read: if request.auth != null && 
                   (resource.data.language in getEnrolledLanguages());
    }
  }
}
```

4. **Tratamento no Thick Client Mobile:**
   O aplicativo móvel, ao carregar a interface (Server-Driven UI), consulta o perfil do estudante armazenado em cache local ou no Firestore. O motor SDUI lê o vetor `enrolled_languages`. Se o vetor contém apenas `"en"`, o app renderiza apenas os cards de lições de inglês, apresentando a trilha de Espanhol Castelhano com um aspecto visual desativado (com uma mensagem educativa "Adquira este idioma com a secretaria da escola" e link direto para contato), garantindo uma experiência de usuário polida e prevenindo requisições rejeitadas pelo Firestore.

### F. Fluxo de Registro Seguro com Código de Ativação Transacional (Prevenção de Cadastro Irrestrito)
Como o aplicativo móvel estará disponível publicamente em lojas oficiais, faz-se necessário impedir o auto-registro de usuários não autorizados que inflariam os custos de APIs cognitivas e de infraestrutura. A solução adota um **Mecanismo de Ativação Baseado em Token Único**.

1. **Modelagem de Chaves de Ativação no Strapi CMS (Coleção `ActivationKeys`):**
   Ao matricular um aluno presencialmente, o coordenador escolar acessa a interface administrativa do CMS e gera uma chave alfanumérica única. O schema de dados consiste em:
   * `key` (String, única, gerada de forma segura na secretaria, ex: `VOO-2026-X8Y9`).
   * `is_used` (Boolean, padrão `false`).
   * `enrolled_languages` (Array: `["en"]` ou `["es"]`, mapeia os contratos do aluno).
   * `used_by` (String, UID de referência ao Firebase Auth do aluno após o consumo).
   * `assigned_email` (String, e-mail do aluno que utilizou a chave).
   * `used_at` (Timestamp, momento exato da ativação).

2. **Criação de Conta via Serverless Cloud Function Transacional:**
   A criação direta de usuários via SDK Client do Firebase Auth é bloqueada nas regras do BaaS. A criação de novos alunos é delegada exclusivamente a uma **Firebase Cloud Function** executada no lado do servidor (`registerWithActivationKey`).
   Para evitar **condições de corrida (race conditions)** — por exemplo, dois usuários tentando enviar simultaneamente a mesma chave alfanumérica —, a Cloud Function realiza todas as leituras e escritas dentro de uma **transação atômica do Firestore**.
   
   Abaixo é apresentada a modelagem conceitual do código Javascript da Cloud Function de registro:

```javascript
const functions = require('firebase-functions');
const admin = require('firebase-admin');
admin.initializeApp();

exports.registerWithActivationKey = functions.https.onCall(async (data, context) => {
  const { email, password, name, activationKey } = data;
  
  if (!email || !password || !name || !activationKey) {
    throw new functions.https.HttpsError('invalid-argument', 'Parâmetros obrigatórios ausentes.');
  }

  const db = admin.firestore();
  const keyRef = db.collection('activation_keys').doc(activationKey);

  try {
    // Executa uma transação atômica no banco de dados Firestore
    return await db.runTransaction(async (transaction) => {
      const keyDoc = await transaction.get(keyRef);
      
      if (!keyDoc.exists) {
        throw new functions.https.HttpsError('not-found', 'O código de ativação fornecido é inválido.');
      }
      
      const keyData = keyDoc.data();
      if (keyData.is_used) {
        throw new functions.https.HttpsError('already-exists', 'Este código de ativação já foi utilizado.');
      }

      // 1. Cria o usuário com segurança no Firebase Auth (Server-side)
      const userRecord = await admin.auth().createUser({
        email: email,
        password: password,
        displayName: name
      });

      // 2. Registra o documento do usuário no Firestore (/users/{uid})
      const userRef = db.collection('users').doc(userRecord.uid);
      transaction.set(userRef, {
        name: name,
        email: email,
        enrolled_languages: keyData.enrolled_languages, // Herda as permissões do token
        is_active: true,
        total_xp: 0,
        streak: 0,
        created_at: admin.firestore.FieldValue.serverTimestamp()
      });

      // 3. Marca o token como consumido e vincula os metadados do aluno correspondente
      transaction.update(keyRef, {
        is_used: true,
        used_by: userRecord.uid,
        assigned_email: email,
        used_at: admin.firestore.FieldValue.serverTimestamp()
      });

      return { success: true, uid: userRecord.uid };
    });
  } catch (error) {
    console.error("Erro crítico na transação de registro:", error);
    throw new functions.https.HttpsError('internal', error.message || 'Erro ao processar o registro.');
  }
});
```

Essa abordagem garante integridade total de dados, previne ataques de repetição e garante conformidade absoluta entre o contrato de estudos assinado com a escola parceira e as credenciais fornecidas em nuvem de forma assíncrona.

---

## 4. Inovação Pedagógica: Diagnóstico Adaptativo e Learning Analytics

A concepção original de sistemas especialistas na educação apoia-se em modelos probabilísticos como a **Teoria de Resposta ao Item (TRI)** e, mais especificamente, o **Modelo de Rasch** para calibrar a dificuldade de itens ($b_i$) e estimar a habilidade latente do estudante ($\theta_j$).

### 4.1 A Inviabilidade Estatística e Computacional do Modelo de Rasch no MVP
Durante a modelagem arquitetural, identificou-se que a implantação matemática estrita das equações de verossimilhança do Modelo de Rasch em tempo real na borda ou nuvem híbrida seria **tecnicamente inviável** para o escopo de um MVP de TCC pelos seguintes fatores:
1. **Sobrecarga de Calibração Populacional:** O modelo de Rasch exige que os itens (exercícios) sejam previamente calibrados estatisticamente. Para que as estimativas dos parâmetros de dificuldade convirjam com significância estatística, faz-se necessário expor as questões a amostras populacionais massivas (geralmente centenas ou milhares de respondentes ativos), o que é impossível em um teste piloto com turmas de controle pequenas (10 a 20 alunos).
2. **Overhead Computacional:** A resolução iterativa de estimativas de máxima verossimilhança para recalcular a proficiência do aluno a cada tela resolvida exigiria poder de processamento inadequado para celulares modestos ou inflaria desnecessariamente a fatura de servidores serverless em nuvem.

---

### 4.2 A Solução de Contorno: Heurística Adaptativa Baseada em Regras Lógicas

Como uma **solução de contorno viável e elegante**, a arquitetura implementa uma **Heurística Adaptativa Determinística** que replica a essência pedagógica de Rasch (casamento entre habilidade do aluno e dificuldade do item) sem a sobrecarga estatística, operando em duas frentes:

1. **Calibração Pedagógica (CMS):** A calibração estatística do parâmetro de dificuldade é substituída por uma **calibração pedagógica realizada pelo professor** no Strapi CMS. O docente define explicitamente o peso/dificuldade de cada exercício (Básico, Intermediário ou Avançado) e associa tags de competência (ex: *Simple Past*).
2. **Estimação Dinâmica de Habilidade (Sliding Scale):** A proficiência do estudante é estimada por meio de uma média móvel ponderada (*rolling average*) dos seus últimos 5 exercícios de uma determinada tag gramatical.
3. **Encaminhamento Heurístico:** Um motor de regras lógico determinístico na borda analisa a média móvel do aluno. Se a proficiência estimada cair abaixo de um limiar crítico (ex: < 70% de acerto), o sistema aciona gatilhos que reconfiguram dinamicamente o fluxo de telas (Server-Driven UI), injetando exercícios de fixação da mesma tag sob uma dificuldade inferior (Básica). Se o desempenho for superior (> 90%), o sistema puxa lições avançadas.

Essa abordagem preserva o rigor do **Aprendizado Adaptativo (*Adaptive Learning*)**, garante latência zero na borda, custa R$ 0,00 e fornece alta previsibilidade de comportamento pedagógico para o coordenador da escola.

### 4.3 Learning Analytics (Dashboard do Professor)
Através do cruzamento das telemetrias de progresso no BaaS, o coordenador e o professor acessam um painel web que consolida relatórios gerenciais das turmas, ranqueando as maiores dificuldades gramaticais coletivas e estimando o nível de proficiência de cada estudante no padrão internacional do Quadro Europeu Comum de Referência para Línguas (CEFR: A1, A2, B1), permitindo intervenções docentes em sala de aula de alta precisão baseadas em dados.

---

## 5. Justificativa de Infraestrutura: PaaS (Render/Railway) e não IaaS (AWS)

Uma defesa acadêmica de TCC robusta deve pautar a escolha de infraestrutura de nuvem sob o conceito de **Adequação ao Propósito (*Fit for Purpose*)**. Para o cenário do MVP e da escala da escola, escolher plataformas de **PaaS (Plataforma como Serviço - Render ou Railway)** em detrimento de gigantes de **IaaS (Infraestrutura como Serviço - AWS)** justifica-se por quatro pilares técnicos essenciais:

1. **Minimização do DevOps Overhead:** Na AWS, a implantação do Headless CMS (Strapi) exigiria gerenciar instâncias EC2, VPCs, grupos de roteamento de redes, chaves SSH e Linux. Na Render/Railway, adota-se o conceito de **Zero Ops** com Continuous Deployment nativo atrelado ao Git: a plataforma lê o repositório, provisiona os containers e o deploy leva 3 minutos automaticamente.
2. **Previsibilidade Financeira Absoluta:** A AWS possui cobranças dinâmicas complexas baseadas em IOPS de disco, dados transferidos por giga e requisições, podendo gerar contas surpresa de centenas de dólares no cartão pessoal do estudante por erro de configuração. Na Render/Railway, os planos pagos de containers são fixos ($5.00 a $7.00), e em caso de estouro de cota, o serviço pausa de forma controlada em vez de cobrar excedentes ilimitados.
3. **Prevenção do Efeito Cold Start (Partida Fria):** A utilização de modelos serverless de IaaS (como AWS Lambda) para hospedar o banco de dados e o CMS geraria latências severas de partida fria (de 5 a 10 segundos na primeira requisição após inatividade), violando diretamente o RNF03 (Performance de até 2 segundos). Na Render/Railway, o banco PostgreSQL e a API do CMS rodam em instâncias contínuas dedicadas no plano básico pago, fornecendo respostas rápidas instantâneas.

---

## 6. Projeção de Custos e Viabilidade Financeira

### 6.1 Detalhamento Operacional de Infraestrutura
* **Google Play Store:** Taxa de ativação única de **$25,00 dólares** (aproximadamente R$ 140,00) cobrada pelo console de desenvolvedor do Google, permitindo atualizações e deploys vitalícios sem cobranças recorrentes.
* **Mídias e Armazenamento (Cloudinary integrado ao Strapi):** Plano gratuito permanente que oferece **25 GB** de armazenamento em cache para mídias pesadas. Com áudios de listening comprimidos em cache a 250 KB por exercício, a cota comporta até **40.000 reproduções mensais** sem custos extras.
* **Inteligência de Transcrição (OpenAI Whisper):** Cobrança estritamente dinâmica baseada em minutos de áudio processados ($0.006 por minuto). 

---

### 6.2 Planilhas de Projeção Financeira

#### **Cenário A: Período de Desenvolvimento, Testes e Defesa (TCC)**
*Foco em custo mínimo de validação sob grupo de controle acadêmico da instituição.*

| **Item de Infraestrutura** | **Provedor sugerido** | **Tipo de Custo** | **Valor (R$)** |
| --- | --- | --- | --- |
| Publicação Mobile (Android) | Google Play Console | Taxa Única | R$ 140,00 |
| Hospedagem do CMS (Backend) | Render (Free Tier) | Mensal | R$ 0,00 |
| Banco de Dados, Auth e Proxy | Firebase (Blaze Plan - Free Tier) | Mensal | R$ 0,00 |
| Storage de Áudios e Imagens | Cloudinary (Free Plan) | Mensal | R$ 0,00 |
| API Cognitiva (Speaking) | OpenAI Whisper API | Por Consumo | R$ 0,00 (Uso controlado) |
| **INVESTIMENTO INICIAL TOTAL** | | | **R$ 140,00** |
| **CUSTO MENSAL DE MANUTENÇÃO** | | | **R$ 0,00** |

*Nota sobre o Firebase:* O Plano Blaze exige cartão de crédito ativo no console Google Cloud para habilitar as Cloud Functions, mas nenhuma cobrança de rede ou invocação ocorrerá, pois o tráfego de testes consumirá menos de 0,1% da cota gratuita mensal (2 milhões de invocações e 5GB de rede).

#### **Cenário B: Lançamento e Operação Comercial Semestral (Escala Comercial)**
*Projeção financeira baseada em uma volumetria contínua de **100 alunos ativos** praticando constantemente.*

| **Item de Infraestrutura** | **Provedor e Cobrança** | **Custo Mensal (USD)** | **Custo Mensal (BRL)** |
| --- | --- | --- | --- |
| Servidor Dedicado Strapi | Render/Railway (Fixo Pago) | $7.00 | R$ 38,50 |
| Firebase (Auth, Telemetria, Proxy)| Google Cloud Blaze (Free Tier) | $0.00 | R$ 0,00 |
| Cloudinary (Assets/Mídia) | Cloudinary Free Tier | $0.00 | R$ 0,00 |
| OpenAI Whisper (100 alunos) | OpenAI API (~1000 minutos/uso) | $6.00 | R$ 33,00 |
| **CUSTO MENSAL ESTIMADO** | **Para 100 alunos ativos** | **$13.00** | **R$ 71,50 / mês** |

📌 **Nota de Custo Unitário:** Sob operação comercial em larga escala, o custo de sustentação da infraestrutura de tecnologia de ponta é de aproximadamente **R$ 0,71 ao mês por aluno ativo**.

---

## 7. Matriz de Riscos de Implantação e Desenvolvimento

| Risco Técnico Identificado | Impacto | Probabilidade | Nível de Risco | Momento e Estratégia de Mitigação |
| :--- | :---: | :---: | :---: | :--- |
| **Sobrecarga de Escopo SDUI** | Médio | Alta | **Alto** | Fase de Planejamento: Adotar um esquema JSON minimalista estrito com componentes fechados na Factory. |
| **Falsos Negativos no Speaking** | Médio | Alta | **Alto** | Fase de Testes: Pipeline de normalização estrita de strings na borda e divisão polimórfica (inglês/espanhol). |
| **Vazamento de Chaves de API** | Alto | Média | **Alto** | Fase Inicial: Implementar a Serverless Function (Firebase Functions) como proxy, ocultando chaves na nuvem. |
| **Latência de Transmissão** | Médio | Alta | **Médio** | Fase de Implementação: Otimização de gravação de áudio em Opus mono 16kHz e pré-carregamento assíncrono local em cache. |
| **Adulteração de Pontuação (BaaS)**| Baixo | Média | **Baixo** | Fase de Implantação: Regras declarativas rígidas atreladas ao UID autenticado (`Firebase Security Rules`). |

---

## 8. Análise de Viabilidade Real e Cronograma de Desenvolvimento (7º ao 8º Semestre)

Para garantir que o projeto de TCC seja concluído dentro do prazo regulamentar das disciplinas de TCC 1 (7º semestre) e TCC 2 (8º semestre), apresenta-se abaixo o estudo de viabilidade prática e o cronograma detalhado de execução.

### 8.1 Análise de Viabilidade Prática
O escopo proposto conta com 18 Requisitos Funcionais e 7 Requisitos Não Funcionais. Para um único estudante atuando como desenvolvedor solo part-time (dividindo tempo com aulas e estágio/trabalho), o escopo original seria de alto risco caso adotasse tecnologias corporativas tradicionais. 

A viabilidade técnica e temporal do projeto sustenta-se estritamente nas seguintes **escolhas estratégicas de arquitetura (Workarounds de Tempo de Desenvolvimento)**:

* **Abordagem No-Backend (Firebase BaaS):** Elimina a necessidade de codificar, testar e implantar um sistema de banco de dados relacional proprietário, rotinas de login/cadastro, gerenciamento de tokens de sessão e infraestrutura de APIs REST. **Economia estimada de tempo: 25% do cronograma.**
* **Infraestrutura baseada em PaaS (Render/Railway):** O provisionamento do Strapi CMS e banco PostgreSQL é resolvido em 3 minutos via integrações Git nativas (Zero Ops), eliminando a necessidade de estudar complexidades de DevOps da AWS (EC2, redes, VPCs, Docker, Nginx). **Economia estimada de tempo: 15% do cronograma.**
* **Mecanismo de Fala Híbrido (Whisper API + Lógica Local):** Evita que o estudante tenha que coletar datasets acústicos massivos e treinar modelos proprietários de machine learning para reconhecimento de fala na borda. **Economia estimada de tempo: 40% do cronograma.**
* **Motor Heurístico Baseado em Regras:** A substituição do modelo econométrico iterativo do Rasch (TRI) por uma lógica determinística baseada em médias móveis por tags de conteúdo simplifica drasticamente a codificação e elimina a necessidade de coleta prévia de amostras populacionais gigantescas de calibração. **Economia estimada de tempo: 20% do cronograma.**

**Conclusão de Viabilidade:** Diante das otimizações arquiteturais adotadas, o projeto classifica-se como **Altamente Viável** para conclusão por um único desenvolvedor dentro do calendário acadêmico padrão.

---

### 8.2 Cronograma Detalhado do Projeto (TCC 1 e TCC 2)

O planejamento está estruturado em 5 fases sequenciais mapeadas ao longo do 7º semestre, período de férias intermediário e 8º semestre letivo.

```
7º SEMESTRE (TCC 1)              FÉRIAS             8º SEMESTRE (TCC 2)
  Meses 1 a 3                   Mês 4                 Meses 5 a 8
┌──────────────┐             ┌──────────┐          ┌──────────────┐
│  Concepção,  │ ──────────> │ Setup da │ ───────> │ Codificação, │
│ Modelagem e  │             │  Infra   │          │  Piloto e    │
│  Defesa TCC1 │             │  Local   │          │ Defesa TCC2  │
└──────────────┘             └──────────┘          └──────────────┘
```

#### **Fase 1: TCC 1 — Concepção, Requisitos e Defesa da Proposta (Meses 1 a 3 — 7º Semestre Letivo)**
* **Mês 1:**
  * Levantamento de escopo e elicitação inicial de requisitos.
  * Reunião inicial com a gerência e coordenação pedagógica da Voo Idiomas para firmar a parceria piloto.
  * Modelagem dos Casos de Uso (UML) e diagramação da arquitetura conceitual serverless.
* **Mês 2:**
  * Estudo aprofundado do paradigma de *Server-Driven UI* e das regras de *Profundidade Ortográfica* para suporte multilíngue.
  * Elaboração do relatório de estimativa de custos, viabilidade de nuvem (*PaaS* vs. *IaaS*) e matriz de riscos.
  * Escrita da primeira versão do documento do TCC 1 (Fundamentação Teórica e Modelagem).
* **Mês 3:**
  * Prototipagem estática de média fidelidade das interfaces visuais do aplicativo móvel (Figma/Wireframes).
  * Revisão da escrita científica do TCC 1 e submissão do relatório.
  * Apresentação e defesa da Proposta de TCC 1 perante a banca examinadora.

#### **Fase 2: Férias Acadêmicas — Setup de Infraestrutura (Mês 4 — Período Inter-semestral)**
* **Mês 4:**
  * Criação do repositório oficial no GitHub e configuração do ambiente local de desenvolvimento.
  * Instalação local do Headless CMS Strapi em localhost e modelagem dos schemas de dados de lições.
  * Setup da conta Firebase/Supabase e instalação das ferramentas de CLI locais (Emuladores).
  * Estudo e testes práticos de gravação e compressão de áudio nativo (formatos Opus/OGG e MP3) no framework móvel escolhido (Flutter ou React Native).

#### **Fase 3: TCC 2 Letivo — Implementação do Core do Software (Meses 5 e 6 — Início do 8º Semestre)**
* **Mês 5 (O Core de Interface e CMS):**
  * Implantação e hospedagem do Strapi CMS e banco PostgreSQL em produção na Render/Railway.
  * Desenvolvimento do parser de *Server-Driven UI* no aplicativo móvel para renderização assíncrona do JSON vindo do CMS.
  * Integração da autenticação segura no aplicativo móvel com o serviço de identidade do BaaS.
* **Mês 6 (A Inteligência Fonética e Segurança):**
  * Desenvolvimento e implantação da Serverless Cloud Function (Proxy Whisper) e configuração de variáveis de ambiente seguras (Secret Manager).
  * Codificação do motor polimórfico de fala local (Double Metaphone + Levenshtein para o inglês, e Levenshtein Ortográfico Direto para o espanhol castelhano).
  * Codificação da lógica de gamificação light (Streaks, XP e Badges) e regras de segurança de banco (*Security Rules*).
  * Codificação heurística do sistema adaptativo baseada em regras de médias móveis.

#### **Fase 4: TCC 2 Letivo — Integração, Teste Piloto e Coleta de Dados (Mês 7 — Meio do 8º Semestre)**
* **Mês 7:**
  * Treinamento rápido de 30 minutos dos professores e coordenação da Voo Idiomas Jaraguá do Sul.
  * Inserção do conteúdo das 3 unidades pedagógicas piloto (inglês/espanhol) no Strapi CMS.
  * Geração do APK oficial homologado de produção e distribuição direta para os alunos selecionados (turma controle).
  * **Execução do Teste Piloto de 4 semanas** com coleta contínua e assíncrona de telemetria pedagógica no Firebase.
  * Extração e análise de dados gerenciais pelo Dashboard do Professor.

#### **Fase 5: TCC 2 Letivo — Redação Final e Defesa do TCC 2 (Mês 8 — Final do 8º Semestre)**
* **Mês 8:**
  * Consolidação científica dos resultados do piloto no capítulo de "Resultados e Discussões" da monografia (métricas de usabilidade, taxas de acerto, análise de engajamento).
  * Escrita final da fundamentação teórica, descrevendo o sucesso da heurística adaptativa de Rasch e da análise fonética baseada em profundidade ortográfica.
  * Formatação final da monografia nos padrões ABNT e submissão ao orientador e à banca.
  * Defesa final do TCC 2 (Monografia e Apresentação do Software) perante a banca examinadora.
