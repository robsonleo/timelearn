# TimeLearn — Plataforma Educacional P2P Descentralizada

**Desenvolvido por Robson Leonardo** ✦ (assinatura visível no rodapé de todas as telas)

TimeLearn é um app Flutter educacional para **Educador / Responsável / Aprendiz** que funciona 100% offline-first e sincroniza dados diretamente entre dispositivos (sem servidor central), usando Nearby Connections como transporte e uma camada própria de identidade/roteamento/pubsub/armazenamento inspirada no protocolo libp2p. A persistência é um banco de dados local (Hive).

Este README é a documentação única e a fonte de verdade sobre arquitetura e comportamento do app.

## Como o aplicativo funciona (visão geral)

1. **Cadastro local** — o usuário cria a conta no próprio aparelho escolhendo um perfil: `aprendiz`, `tutor` (exibido na interface como **Responsável**) ou `educador`. A senha é armazenada com hash + salt no Hive local. Não existe servidor: a conta vive no dispositivo e é propagada aos outros aparelhos via P2P.
2. **Perguntas de segurança** — no cadastro o usuário responde 3 perguntas fixas (definidas em `lib/constants/security_questions.dart`), as mesmas usadas na recuperação de senha e na tela Configurações → Perguntas de Segurança. As respostas são comparadas sem diferenciar maiúsculas/espaços.
3. **Vinculação** — o aprendiz se vincula a um responsável/educador por **código único** (com botão de copiar 📋 na tela "Meus Aprendizes") ou **QR code** (`/learner_linking`). O responsável aceita ou recusa a solicitação.
4. **Conteúdo** — educadores e responsáveis criam questões (8 tipos, todos funcionais), listas, grupos, cursos, quizzes, pacotes e competições. Cada questão ganha um código de compartilhamento (`Q-XX-XX`) que pode ser digitado em outro aparelho para importá-la.
5. **Estudo** — o aprendiz responde quizzes por fase, acumula pontos, sobe de fase quando acerta tudo, ganha conquistas, certificados, metas e estatísticas (sequência de dias, acurácia, nível de desempenho).
6. **Controle parental** — o responsável configura travas de tempo, bloqueio por questão (o aparelho só desbloqueia quando o aprendiz responde corretamente), proteção contra desinstalação, monitoramento de bateria e localização.
7. **Sincronização** — todo `save`/`delete` no banco local também é publicado na malha P2P; qualquer aparelho conectado recebe e aplica a mudança localmente. Sem internet, sem nuvem.

## Sistema de questões (8 tipos, todos funcionais)

| Tipo | Como o criador monta | Como o aprendiz responde |
|---|---|---|
| **Múltipla escolha** | 2–6 opções, marca a correta | Cartões A/B/C/D |
| **Verdadeiro ou Falso** | Escreve a afirmação e escolhe V/F | Dois cartões grandes |
| **Preencher lacunas** | Texto com `___` (botão "Inserir lacuna"), resposta por lacuna; alternativas separadas por `;` | Campos embutidos no próprio texto |
| **Resposta curta** | Lista de respostas aceitas (qualquer uma vale) | Campo de texto livre |
| **Correspondência** | Pares esquerda ↔ direita (ex.: País → Capital) | Coluna esquerda fixa + opções embaralhadas |
| **Ordenação** | Itens na ordem correta | Itens embaralhados para arrastar na ordem |
| **Arrastar e soltar** | Categorias (caixas) + itens com sua categoria | Fichas arrastáveis para as caixas |
| **Hotspot** | Grade 2x2 / 2x3 / 3x3 com emoji/texto por célula, marca a correta | Toca na área correta |

**Arquitetura do sistema de questões:**
- `lib/widgets/question_editor_form.dart` — formulário único de criação/edição com editor específico e dica de preenchimento por tipo; usado pela tela `/create_edit_question`, pelo `QuestionDialog` (cursos, grupos, listas) e pela tela do responsável (com campo Fase).
- `lib/widgets/question_answer_widget.dart` — widget único que renderiza qualquer tipo de forma interativa (com modo escuro para telas de bloqueio) + `QuestionEvaluator`, que corrige a resposta normalizando maiúsculas/acentos/espaços.
- `lib/models/question.dart` — dados específicos por tipo em `extraData` (pares, ordem, categorias, lacunas, grade), mantendo `options`/`correctAnswerIndex` compatíveis para questões antigas; `correctAnswerText` fornece a resposta legível para TTS e mensagens de erro.
- Integrado em **todas** as telas de resposta: quiz do aprendiz (por fase), quiz de grupo, quiz de pacote, competição 1v1, questionários e as telas de bloqueio/desbloqueio do aparelho (onde as opções só são lidas em voz alta quando não revelam a resposta).

## Arquitetura

### Transporte: Nearby Connections (`lib/services/p2p_service.dart`)
- Descoberta/anúncio automático de dispositivos próximos (`Strategy.P2P_STAR`, múltiplas conexões simultâneas).
- CRUD com fila de retry (backoff) para todos os domínios de dados.
- Modo backup dispositivo-a-dispositivo, independente do backup completo via LibP2P.
- Handlers de bloqueio/desbloqueio remoto do aparelho do aprendiz pelo responsável (`device_lock_request` / `device_unlock_request`), registrados na inicialização do serviço.

### Camada LibP2P-style (`lib/services/libp2p_*.dart`)
Camada real (criptografia genuína, não simulada) inspirada nos conceitos centrais do protocolo libp2p, rodando sobre o transporte Nearby:

- **`libp2p_encryption_service.dart`** — identidade criptográfica por dispositivo (secp256k1), PeerId como multihash sha2-256 em base58btc, assinatura ECDSA, canal seguro par-a-par via ECDH + HKDF-SHA256 + AES-256-GCM.
- **`libp2p_network_service.dart`** — tabela de roteamento estilo Kademlia (k-buckets por distância XOR), diretório persistente de peers conhecidos (Hive), separando o PeerId estável do endpointId efêmero do Nearby.
- **`libp2p_storage_service.dart`** — armazenamento local endereçado por conteúdo (CID sha2-256, com dedup automático); sobrevive a reinícios e funciona sem nenhum peer conectado.
- **`libp2p_sync_service.dart`** — PubSub estilo GossipSub: publica para todos os peers conectados, retransmite em malha até um TTL, deduplica por id de mensagem, cifra ponta-a-ponta após o handshake de identidade.
- **`p2p_libp2p_bridge.dart`** — liga a pilha acima ao transporte, com handshake assinado ao conectar com peer novo (equivalente ao protocolo *Identify* do libp2p real).
- **`libp2p_manager.dart`** — fachada única usada pela UI e pelos demais serviços.
- **`libp2p_data_sync_service.dart`** — ponte genérica entre `DatabaseService` e a camada acima: cada domínio publica um ponteiro `{id, cid, lastModified}` via gossip após guardar o conteúdo por CID; quem recebe busca o conteúdo localmente ou pede ao peer de origem (similar a um Bitswap simplificado).

### Alcance da rede e QR code (importante)

- **Perto:** a sincronização acontece automaticamente entre aparelhos próximos via Bluetooth/Wi-Fi Direct (Nearby Connections). A malha gossip retransmite: se A encontra B, e B depois encontra C, os dados de A chegam a C.
- **Longe — transporte de internet (`libp2p_internet_transport.dart`):** dois aparelhos se conectam por WebSocket a qualquer distância em que o endereço seja alcançável (mesma rede Wi-Fi, ou internet com IP público/VPN). O fluxo é por QR code, na tela **LibP2P → Internet (longa distância)**:
  1. O aparelho A toca em **"Gerar QR (iniciar servidor)"** — sobe um servidor WebSocket local (porta 4001) e mostra um QR com seus endereços e PeerId.
  2. O aparelho B toca em **"Escanear QR"** e aponta a câmera (ou digita/cola o endereço `ws://...`).
  3. Os dois fazem o **mesmo handshake assinado** (ECDSA) do transporte Nearby; o peer entra na mesma tabela de roteamento e o gossip cifrado (AES-256-GCM) passa a fluir pela conexão.
  4. O endereço fica salvo e o app **reconecta sozinho** a cada 30 segundos quando a conexão cai.
- **Ponte entre redes:** um aparelho conectado ao mesmo tempo à malha local (Nearby) e a um peer remoto (WebSocket) retransmite o gossip entre as duas redes — os dados de toda a malha local chegam ao peer distante e vice-versa.
- **Requisito de rede:** na mesma rede Wi-Fi funciona direto. Pela internet, o aparelho "servidor" precisa de um endereço alcançável (redirecionamento de porta no roteador, ou VPN par-a-par como Tailscale/ZeroTier — que mantém o modelo sem servidor central). Não há interoperabilidade com a rede libp2p/IPFS pública (inviável em Dart/Flutter hoje).
- **Descoberta automática de IP público:** ao gerar o QR, o app também consulta um serviço externo leve e gratuito ("qual é o meu IP", tipo api.ipify.org — não é servidor nosso, não retransmite nada do app) e inclui esse endereço como mais uma opção de conexão, marcado com 🌍 na tela. Isso poupa o passo manual de descobrir o próprio IP quando a porta já está liberada no roteador — **não é hole-punching e não resolve NAT simétrico/CGNAT de operadora móvel**; continua precisando de porta liberada (ou VPN) para funcionar de verdade a longa distância.
- **IPv6 (dual-stack):** o servidor de longa distância aceita IPv4 e IPv6 no mesmo socket, e `listenAddresses()` inclui os endereços IPv6 globais do aparelho (marcados com ⚡ na tela) — link-local (`fe80::...`) fica de fora, não serve pra alcançar de fora da rede. A vantagem: **IPv6 normalmente não passa por NAT/CGNAT de operadora** — cada aparelho recebe um endereço público de verdade, sem tradução nenhuma. Se os dois lados tiverem IPv6 (comum em operadora móvel brasileira hoje), a conexão a longa distância pode simplesmente funcionar sem liberar porta nem VPN. Não é garantido — depende da rede de cada um ter IPv6 de ponta a ponta e não bloquear conexão de entrada — mas quando tem, resolve o problema de NAT na raiz, ao contrário da descoberta de IP público (que continua sendo IPv4 e sofre com CGNAT).
- **QR/códigos como transporte de dados:** independentemente da conexão, o QR de backup (`Conectar/Parear`) carrega dados exportados, e códigos de questão/quiz/grupo/pacote podem ser enviados por qualquer meio e digitados no destino.

### Persistência local (`lib/services/database_service.dart`)
Todos os dados vivem numa única Hive box (`timelearn_box`), como mapas JSON sob uma chave por domínio: usuários, questões, listas, grupos, questões-de-grupo, cursos, quizzes, bloqueios, visitas de site, competições, pacotes, grupos compartilhados, travas periódicas, proteção contra desinstalação, alertas de bateria, notificações, saúde do sistema, logs de atividade, conquistas, estatísticas, preferências, certificados, metas, cartões de perfil, sessões de estudo, perfis/ações/badges de educador.

Todo `save*`/`delete*` grava no Hive **e** publica na camada LibP2P (`broadcast: false` evita eco ao aplicar dados recebidos da rede). O `main.dart` assina todos os domínios na inicialização e aplica localmente o que chega dos peers.

Consultas prontas: busca de questões por texto (`searchQuestions`), por fase, educador, curso, grupo, quiz; contagem por fase (`countQuestionsByPhase`); quizzes por status/criador; grupos e cursos filtrados por criador ou participação (`getGroupsByUser`/`getCoursesByUser`).

### Backup completo (`exportAllData()` / `importAllData()`)
- Exporta/restaura todos os domínios de uma vez.
- Guardado como conteúdo endereçado por CID via LibP2P e também como arquivo `.json` local, compartilhável via share sheet do sistema (tela **Backup**).
- O diálogo **Conectar/Parear** gera QR temporário (expira em 2 minutos) para transferir backup entre aparelhos.

## Autenticação e recuperação de conta

- **Login** por nickname + senha (hash + salt no Hive local).
- **Perguntas de segurança unificadas** (`lib/constants/security_questions.dart`): as mesmas 3 perguntas aparecem no cadastro, em Configurações → Perguntas de Segurança (que carrega e salva de verdade as respostas, com sincronização P2P) e na recuperação de senha. As respostas são salvas com as chaves `q1`/`q2`/`q3` e comparadas sem diferenciar maiúsculas nem espaços.
- **Recuperar senha**: nickname + 3 respostas corretas → define nova senha.
- **Troca de perfil** após login (`/select_profile`), com nomes exibidos traduzidos (o perfil interno `tutor` sempre aparece como **Responsável**).

## Funcionalidades por perfil

### Educador
- Criar/editar/excluir questões nos **8 tipos** (tela dedicada ou diálogo), com seleção de tipo ilustrada.
- Listas de questões, grupos, grupos compartilhados (com ranking de aprendizes), cursos com questões próprias, quizzes publicáveis e pacotes de questões vinculáveis.
- Repositório de conteúdo, importação de questões por código, relatórios, perfil com reputação/badges.
- Rede LibP2P (`/libp2p`) e Backup completo.

### Responsável (perfil interno `tutor`)
- **Meus Aprendizes**: código de responsável com botão de copiar 📋, aceitar/recusar solicitações de vínculo, remover aprendizes; Gerenciar Aprendizes (visão avançada).
- **Gerenciar Questões / Grupos / Cursos**: organização em pastas e subpastas (árvore própria para cada tipo), criação nos 8 tipos com campo **Fase** (1–5), busca e filtro por tipo.
- **🛡️ Painel de Controle Parental** (`/parental_hub`) — hub único com todas as ferramentas para o aprendiz selecionado:
  - **Pausar agora**: trava o aparelho do aprendiz na hora, com botão de retomar direto na notificação.
  - **⏱️ Pergunta Periódica**: trava a tela a cada X minutos até o aprendiz responder corretamente (áudio TTS) ou aguardar o tempo configurado.
  - **🌙 Modo Soninho**: horário de silêncio/bloqueio configurável.
  - **🎯 Metas Semanais**: define quantidade de acertos na semana; ao bater a meta o aprendiz ganha bônus de minutos de uso e pontos automaticamente.
  - **📵 Bloquear Apps**: bloqueia apps específicos por pacote e/ou define limite diário de uso em minutos; o app bloqueado é interceptado e o aprendiz vê a tela de bloqueio.
  - **📊 Uso de Apps**: relatório de tempo de uso por aplicativo instalado no aparelho do aprendiz.
  - **📍 Cercas Virtuais (geofencing)**: cadastra zonas seguras; o responsável recebe notificação quando o aprendiz entra ou sai de cada zona; histórico de localização.
  - **🗺️ Localização e Histórico** do aprendiz em tempo real.
  - **🔒 Anti-Desinstalação**: liga/desliga por aprendiz, aprova/nega pedidos cooperativos (ver seção dedicada abaixo).
  - **🎁 Loja de Recompensas** e **📈 Relatório de Aprendizado** (atalhos, ver abaixo).
- **🎁 Loja de Recompensas**: cadastra prêmios do mundo real (nome, emoji, custo em pontos); aprova ou recusa os resgates pedidos pelos aprendizes (recusa devolve os pontos automaticamente).
- **📊 Relatório de Aprendizado**: desempenho e reforço de todos os aprendizes vinculados; **relatório semanal automático** por notificação (resumo de questões respondidas, acurácia e bloqueios de cada aprendiz, uma vez por semana).
- **🆘 SOS**: recebe alerta com localização quando um aprendiz aciona o botão de emergência.
- Bloqueio/desbloqueio remoto do aparelho via P2P, além do bloqueio por questão.
- Dashboard de Sincronização P2P, Dashboard de Saúde do Sistema (com exportação CSV), central de notificações, histórico de atividades, agendamento de questões, recebimento de questões do educador, competições.
- Rede LibP2P e Backup completo.

### Aprendiz
- **Quiz por fase** com TTS (leitura em voz alta), pontos, avanço de fase ao gabaritar.
- **✏️ Minhas Questões**: cria e organiza as próprias questões em pastas (mesma experiência de "Gerenciar Questões" do responsável).
- Grupos compartilhados (quiz com ranking), pacotes de questões, cursos, biblioteca de questões.
- **💰 Ganhar Tempo**: tela dedicada onde cada acerto vale X minutos de uso do aparelho (valor definido pelo responsável) — separada do desbloqueio por questão.
- **🎁 Loja de Recompensas**: troca os pontos ganhos estudando por prêmios cadastrados pelo responsável (resgate fica pendente até aprovação).
- **⏱️ Pergunta Periódica**: pode ativar por conta própria como autodesafio, estudando a cada X minutos.
- **🆘 SOS**: envia alerta de emergência (com localização, se disponível) direto ao responsável.
- **🔒 Proteção contra desinstalação**: quando o responsável liga essa proteção, o app pede para o aprendiz ativá-la no próprio aparelho (ver seção dedicada abaixo).
- **Competição 1v1** via P2P em tempo real (convite, aceite, pontuação por velocidade) e histórico de competições com ranking.
- Conquistas, metas, certificados, cartão de perfil, sessões de estudo, estatísticas (sequência de dias, acurácia, nível de desempenho); resumo semanal automático do próprio desempenho.
- Vinculação a um responsável/educador por código ou QR no onboarding; central de notificações; Rede LibP2P.

### Admin
- Menu com atalhos administrativos, incluindo Rede LibP2P.

## Tela LibP2P (`/libp2p`)

Painel de diagnóstico da camada descentralizada, 100% local:
- Identidade do peer (PeerId, chave pública, algoritmo).
- Peers conectados e tabela de roteamento (estilo Kademlia).
- **Internet (longa distância)** — gerar QR/servidor, escanear QR ou digitar endereço `ws://`, conexões ativas e endereços salvos com reconexão automática.
- Tópicos PubSub e ping de teste.
- Contadores de sincronização por domínio na sessão.
- Armazenamento endereçado por conteúdo (blocos, tamanho, CIDs).
- Atalho para o Backup completo.

## Proteção Anti-Desinstalação (aprendiz)

Como funciona, incluindo o limite real da plataforma Android (documentado aqui de propósito, para não passar a impressão de um bloqueio inquebrável que o Android não permite):

- **O que o Android permite para um app comum**: nenhum app instalado normalmente (baixado/instalado como qualquer outro, sem provisionar o aparelho do zero como "dono do dispositivo") consegue impedir a própria desinstalação de verdade — isso é uma restrição de segurança do próprio Android, para impedir exatamente esse tipo de abuso por malware. O que um app pode fazer é virar **administrador do dispositivo**: aí o Android passa a **exigir que o admin seja desativado em Configurações antes de permitir desinstalar**, mostrando um aviso forte nesse caminho.
- **Como o TimeLearn usa isso** (`TimeLearnDeviceAdminReceiver.kt`):
  1. O responsável liga a proteção na tela **🔒 Anti-Desinstalação** (dentro do Painel de Controle Parental), por aprendiz.
  2. No aparelho do aprendiz, isso sincroniza via LibP2P e o app pede para o aprendiz confirmar a ativação (tela dedicada) — o Android exige que seja o próprio usuário do aparelho a conceder, nenhum app consegue ativar sozinho.
  3. Se o aprendiz tentar desativar a proteção pelas Configurações do Android, o sistema mostra o aviso "o responsável será avisado agora mesmo" — e o TimeLearn já registra a tentativa e notifica o responsável nesse instante (`onDisableRequested`), mesmo que o aprendiz desista.
  4. Se o aprendiz confirmar a desativação, o TimeLearn grava isso (`onDisabled`), marca a proteção como **desativada pelo aprendiz** e manda um alerta mais forte ao responsável — a partir daí a desinstalação já é possível.
  5. Quando o responsável desliga a proteção pela tela (não precisa ser por tentativa do aprendiz), o próprio app libera o administrador no aparelho do aprendiz sozinho, sem precisar de nenhuma ação da criança.
- **O que isso garante, na prática**: atrito real (uma tela extra + aviso assustador) e **aviso confiável no exato momento da tentativa**, não um bloqueio impossível de contornar. Um aprendiz determinado e com acesso físico ao aparelho ainda consegue desinstalar — só não consegue fazer isso sem o responsável ficar sabendo.
- Existe também um fluxo cooperativo (`requestUninstall`/`approveUninstall`/`denyUninstall`) para quando o aprendiz prefere pedir permissão em vez de desativar a proteção sozinho — aparece na mesma tela do responsável.

## Padrões de interface

- **Assinatura** "✦ Robson Leonardo ✦" no rodapé de todas as telas (aplicada uma única vez no `builder` do `MaterialApp`).
- **Respiro inferior**: todas as telas roláveis têm pelo menos 48px (≈ duas linhas) de espaço no fim do conteúdo, para nada ficar cortado.
- Em toda a interface o perfil é exibido como **"Responsável"**; internamente (banco, rotas, sincronização) o valor continua `tutor` para manter compatibilidade com dados já existentes.

## Validações e regras de negócio (lógica pura, coberta por testes)

- **CPF** com validação real de dígitos verificadores (rejeita sequências repetidas e dígitos inválidos); campo opcional.
- **Question**: `hasValidAnswer` / `isCorrectAnswer(i)` — protege contra índice de resposta fora das opções; `QuestionEvaluator` corrige os 8 tipos normalizando maiúsculas/acentos.
- **Quiz**: `questionCount`, `hasQuestions`, `isPublished`, `isPlayable`.
- **User**: `isCurrentlyBlocked` (bloqueio expira sozinho quando `blockedUntil` passa), `isLearner`/`isEducator`/`isTutor`, `totalLearners`, `hasPendingLearners`.
- **LearningStats**: `totalIncorrectAnswers`, `weeklyActivityTotal`, `isStreakActive` (estudou hoje ou ontem), `performanceLevel` (Excelente/Bom/Regular/Precisa melhorar).
- **LearnerGoal**: `progressPercent`, `isOverdue`, `remainingValue`.

## Coisas que NÃO estão no escopo (deliberadamente fora)

- **Autenticação descentralizada** (`decentralized_auth_service.dart`/`credential_sync_service.dart`) — inicializados no boot mas não usados pelo fluxo real de login; mantidos assim para não mexer no login de todos os usuários.
- **Interoperabilidade com a rede libp2p/IPFS pública** — inviável em Dart/Flutter hoje; a longa distância é atendida pelo transporte WebSocket próprio (ver "Alcance da rede").

## Serviços principais

| Serviço | Responsabilidade |
|---|---|
| `database_service.dart` | Persistência local (Hive) de todos os domínios + backup completo |
| `p2p_service.dart` | Transporte Nearby, discovery/advertising, backup dispositivo-a-dispositivo, bloqueio remoto |
| `libp2p_manager.dart` | Fachada da camada libp2p-style (identidade, DHT, pubsub, storage, sync) |
| `libp2p_data_sync_service.dart` | Publica/assina todos os domínios de dados via a camada libp2p-style |
| `libp2p_internet_transport.dart` | Transporte WebSocket de longa distância (servidor + cliente + QR + reconexão) |
| `encryption_service.dart` | Criptografia legada do transporte Nearby (tamper-detection local) |
| `location_service.dart` | Localização GPS em tempo real |
| `geofence_monitor_service.dart` | Roda no aparelho do aprendiz: grava histórico de localização e notifica o responsável ao entrar/sair de uma cerca virtual |
| `app_block_service.dart` | Roda no aparelho do aprendiz: detecta apps bloqueados/limite diário estourado e mostra a tela de bloqueio |
| `parental_control_service.dart` | Comandos "pausar agora"/"retomar" e SOS via P2P; verificação de meta semanal e concessão de bônus |
| `weekly_report_service.dart` | Relatório semanal automático (notificação) para responsável/educador e aprendiz |
| `uninstall_protection_service.dart` | Liga/desliga a Proteção Anti-Desinstalação por aprendiz e o fluxo cooperativo de pedido/aprovação (ver seção dedicada) |
| `device_admin_channel.dart` | Ponte com o Android para o administrador do dispositivo usado na Proteção Anti-Desinstalação |
| `screen_time_service.dart` | Conta de minutos de uso ganhos (tela Ganhar Tempo) e consumo desses minutos |
| `battery_monitoring_service.dart` | Monitoramento de bateria dos aprendizes com alertas ao responsável |
| `notification_service.dart` | Notificações locais |
| `system_health_service.dart` | Métricas de saúde do sistema |
| `qr_code_service.dart` | Geração/leitura de QR para vínculo responsável-aprendiz |
| `audio_service.dart` / TTS | Leitura em voz alta de questões (pt-BR) |

## Como executar

```bash
flutter pub get
flutter run
```

### Build APK
```bash
flutter build apk --debug   # ou --release
```
O APK é gerado em `build/app/outputs/flutter-apk/`.

### Qualidade
```bash
flutter analyze   # 0 problemas
flutter test      # 98 testes, todos passando
```

Suítes: `database_service_test.dart` (persistência), `libp2p_encryption_service_test.dart` (identidade/assinatura/AES-GCM), `libp2p_internet_transport_test.dart` (conexão WebSocket real com handshake e gossip), `logic_helpers_test.dart` (regras de negócio e validadores), `question_core_test.dart` (correção dos 8 tipos de questão e repetição espaçada), `rewards_and_screentime_test.dart` (loja de recompensas e economia de minutos), `parental_models_test.dart` (modelos de controle parental), `auth_consistency_test.dart` (consistência de autenticação), `user_persistence_restart_test.dart` (regressão do bug de dados sumindo após reiniciar o app), `widget_test.dart` (smoke test de UI).

## Estrutura de pastas

- `lib/main.dart` — inicialização, rotas, rodapé global com assinatura e wiring da sincronização LibP2P de todos os domínios.
- `lib/constants/` — constantes compartilhadas (perguntas de segurança).
- `lib/screens/` — telas da aplicação.
- `lib/widgets/` — componentes reutilizáveis: editor e respondedor universais de questões, diálogos e drawers por perfil.
- `lib/services/` — serviços (P2P, LibP2P, banco, localização, notificações).
- `lib/providers/` — gerenciamento de estado (`AuthProvider`).
- `lib/models/` — modelos de dados.
- `test/` — testes automatizados.
