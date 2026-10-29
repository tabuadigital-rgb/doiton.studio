# Do It On Studio: instruções para o Claude Code

## O que é

O Do It On Studio é a ferramenta interna da Do It On (agência de Digital PR e comunicação integrada, Lisboa) para planear, produzir, aprovar e medir conteúdo. Os clientes entram num portal com a marca da Do It On, onde consultam, comentam e aprovam o trabalho. Não é um produto para vender: não há vários clientes-agência, instâncias dedicadas nem API pública.

Foi inspirado no ContentLine, mas é uma construção de raiz e original. Nunca copiar nome, marca, textos, ilustrações, código ou desenho de interface do ContentLine.

## Fonte de verdade

O documento "Do It On Studio: produto, arquitectura e plano de construção" decide tudo o que está aqui resumido: https://claude.ai/artifact/TeLXXYkNzJA3dDRK1GkRn7

Ler o documento inteiro antes de escrever código. Se este ficheiro e o documento divergirem, o documento ganha, e a divergência comunica-se a quem encomendou. O que não estiver decidido pergunta-se; não se assume.

## Primeira tarefa: análise técnica (antes de escrever código)

Nesta sessão não se escreve código de produto. Entrega-se uma análise técnica, em ficheiro `docs/analise-tecnica.md` no repositório, com estas partes:

1. Leitura do documento fonte de verdade e da maquete, e lista das dúvidas ou contradições encontradas, com a proposta de resolução de cada uma.
2. Verificação da stack: pontos de atrito entre NestJS, Prisma, RLS no PostgreSQL e JSONB; escolha e justificação para Tiptap com Yjs; leitura de Word e PDF (e a decisão sobre OCR); geração de PDF com Chromium.
3. Modelo de dados proposto em detalhe (entidades do documento, mais Tema, Versão de conteúdo, Indicador, Tarefa com quadro kanban), com o desenho das políticas RLS e dos testes de isolamento.
4. Desenho do quadro kanban ligado à aprovação: máquina de estados (a fazer, em curso, revisão interna, aprovação do cliente, aprovado, publicado), quem pode mover cada cartão e o que o portal mostra ao cliente.
5. Estimativa de esforço por parte e por fase, com os riscos principais e o que deve ser decidido por quem encomenda antes de a parte A começar.
6. Plano da parte A em tarefas pequenas, cada uma com o seu teste de aceitação.

Maquete dos ecrãs principais (dados de exemplo, não é a interface final): https://claude.ai/artifact/UroJrwgB9gMpbSkJ7NSAWq. Mostra a listagem de campanhas, projectos e temas, o quadro kanban, a pasta de história com histórico de versões, a biblioteca de activos e o portal do cliente com aprovação e relatório. Serve para validar fluxos e vocabulário; o aspecto final segue o sistema de design.

Regras desta tarefa: não instalar nem criar infra-estrutura sem pedir; não alterar o documento fonte de verdade; cada afirmação técnica que dependa de documentação ou de versão deve citar a fonte. Termina com a lista das perguntas que precisam de resposta de quem encomenda.

## Stack (decidida, não se reabre)

- TypeScript em modo estrito, de ponta a ponta. Repositório único com pnpm.
- API: Node.js (LTS) com NestJS, um módulo por capacidade.
- Base de dados: PostgreSQL com Prisma, migrações versionadas, JSONB para blocos e segurança ao nível da linha (RLS).
- Web: React com Vite, para a equipa e para o portal do cliente, que partilham componentes.
- Editor: Tiptap (ProseMirror) com Yjs sobre WebSocket para edição simultânea.
- Fila e agendamento: Redis com BullMQ. Pesquisa: texto do PostgreSQL na fase 1; OpenSearch só quando o volume o exigir.
- Ficheiros: disco do servidor com cópia externa, ou armazenamento compatível com S3; FFmpeg para vídeo e áudio.
- PDF: Chromium sem interface, a partir do conteúdo.
- Identidade: Microsoft Entra ID (OIDC) para a equipa; clientes por convite, com ligação enviada por e-mail.
- IA: SDK de IA da Vercel, um adaptador por fornecedor (OpenAI, Anthropic, Google); Copilot pela Chat API do Microsoft 365.
- Interface: i18next, com um ficheiro de tradução por idioma.
- Qualidade: Vitest e Playwright, integração contínua no GitHub Actions. Observabilidade: OpenTelemetry e Grafana.
- Infra: Docker Compose num Cloud Compute da HawkHost, em Londres, com cópias de segurança diárias e restauro testado.
- Peças em Python (relatórios PESO, clipping) correm como trabalhadores isolados, ligados pela fila, até serem migradas para TypeScript.

## Estrutura do repositório

- `apps/api`: NestJS, módulos de planeamento, conteúdo, activos, automação e identidade.
- `apps/web`: React com Vite, equipa e portal.
- `packages/shared`: tipos e validações comuns entre API e interface.
- `workers`: trabalhadores em segundo plano; as peças em Python ficam aqui, isoladas.
- `infra`: Docker Compose, configuração do servidor e cópias de segurança.

O repositório chama-se `do-it-on-studio`.

## Regras que não se quebram

1. Toda a tabela leva o identificador do cliente e tem RLS. Nenhuma consulta depende só do código para isolar clientes. Cada módulo fecha com um teste de isolamento que prova que um cliente não vê dados de outro.
2. O portal é uma vista limitada da mesma aplicação, não um sistema à parte. O cliente vê o trabalho, comenta e aprova; nunca vê material interno, notas de equipa nem outras contas.
3. As permissões vivem na pasta de história e são herdadas por tudo o que ela contém.
4. Migrações sempre versionadas, nunca alteradas à mão. Segredos só em variáveis de ambiente: nenhuma credencial no código, nos registos nem nos testes.
5. Interface e e-mails em português europeu, inglês, espanhol e francês, sempre em ficheiros de tradução desde o primeiro dia, com o português europeu como língua de origem. O idioma escolhe-se num seletor no cabeçalho, visível também no início de sessão e na página do convite, e fica guardado no perfil. Ordem de decisão: escolha da pessoa, idioma do convite ou da conta de cliente, idioma do navegador, português europeu.
6. Nada gerado por IA chega a um cliente ou a um canal sem revisão humana. Cada texto gerado fica identificado como tal, com fornecedor, modelo, versão da receita, custo e resultado da verificação do tom de voz.
7. As chaves dos fornecedores de IA existem só no servidor, cifradas, e nunca chegam ao navegador. Todo o pedido sai de um trabalhador e fica registado. Antes de enviar, retira-se do contexto o que o briefing não precisa, como dados pessoais de contactos.
8. A memória de estilo de um cliente é um documento versionado, editado pela equipa, e entra só nos pedidos feitos para esse cliente. Não é aprendizagem escondida do modelo.
9. Autenticação em dois passos (TOTP e códigos de recuperação) oferecida aos clientes no primeiro início de sessão e sempre opcional para o cliente. Registo só de acréscimo de quem viu o quê, conservado 12 meses. Exportação da auditoria em CSV, reservada a administradores, e a própria exportação fica registada.
10. As ligações às redes sociais, ao Google Analytics e ao Google Ads são só de leitura, por conta de cliente, e não podem gerar custos. Onde o acesso por interface for pago (parece ser o caso do X), a rede entra pela importação de ficheiros exportados da própria plataforma.
11. O valor equivalente de publicidade mantém-se no relatório mensal, com os valores recolhidos do Carma. Se o Carma não os fornecer, o sistema emite um alerta e o relatório só segue para aprovação depois de uma pessoa validar esses valores.
12. Nenhuma importação grava sem a pessoa ver uma pré-visualização e confirmar o mapeamento de colunas. Ficheiros grandes correm em segundo plano e registam os erros por linha.

## Interface e design

Seguir o sistema de design oficial da Do It On, o artefacto "Do It On Design System": https://claude.ai/artifact/VBWoaxPMu8kcav1ZaAHxbK. Importar os valores do ficheiro de tokens desse sistema em vez de os copiar à mão. Referência: Comfortaa em títulos e texto; rosa #B03090 como cor primária, azul #1090D0 como secundária, verde #30C090 como acento e sucesso; fundo claro #F7F7FA; botões em forma de pílula; cartões com raio de 16 px. O branco sobre o verde não cumpre o contraste mínimo (2,3 para 1): o verde usa-se em ícones e estados de sucesso, nunca como fundo de texto.

## Fases e critérios de saída

Construir uma parte de cada vez, pela ordem abaixo. Nenhuma parte seguinte começa antes de o critério de saída da anterior estar verificado por testes e por uma demonstração.

1. **Fase 1, parte A, fundações.** Repositório e integração contínua, base de dados com isolamento por cliente, início de sessão da equipa e convite de clientes, contas de cliente, permissões e pastas de história. Saída: a equipa entra, cria uma conta de cliente com uma pasta de história e convida um cliente de teste, que vê essa pasta.
2. **Fase 1, parte B, conteúdo e portal.** Editor por blocos com histórico de versões do conteúdo, listagem de campanhas, projectos e temas, biblioteca de activos pesquisável por campanha, tema e metadados, portal de consulta com comentários e aprovação simples (aprovar ou pedir alterações), quatro idiomas, importação de Excel e CSV, autenticação em dois passos opcional, registo de quem viu o quê e primeira versão do relatório mensal (o PDF em PESO que hoje sai do Excel e do Python, publicado no portal sem automação). Saída: uma conta de cliente piloto vê no portal uma campanha planeada e escrita na aplicação, aprova-a ali e recebe o relatório do mês, sem receber ficheiros por e-mail. Medir antes e depois: percentagem de planos do mês feitos na aplicação em vez de Excel, e tempo de preparação do relatório mensal.
3. **Fase 2, planeamento, aprovação e impressão.** Calendário editorial, quadro kanban de actividades (título, descrição, prazo e responsável) ligado ao estado de aprovação, com as colunas iniciais a fazer, em curso, revisão interna, aprovação do cliente, aprovado e publicado (até existir o exportador da fase 4, "publicado" marca-se à mão), alocação da equipa a clientes por perfil, escalas e disponibilidade, notificações, histórico reversível, aprovação por níveis pelo cliente no portal (desenvolve a aprovação simples da fase 1), conversão de e-mails em pastas de história, importação de Word e PDF (PDF digitalizado exige OCR, a decidir), PDF para impressão e exportação da auditoria. Saída: o piloto atribui todas as tarefas de um mês no quadro kanban da aplicação e o cliente aprova no portal um dossiê que sai também em PDF.
4. **Fase 3, produção com IA.** Adaptadores para ChatGPT, Claude, Gemini e Copilot, ligações e regras por conta de cliente, receitas de instrução versionadas, memória de estilo por cliente, verificação do tom de voz, custo de IA por cliente e botão "Iniciar produção" nas pastas aprovadas. Úteis: tradução com revisão humana e biblioteca partilhada de receitas. Saída: depois de o cliente aprovar um briefing, a equipa inicia a produção com pelo menos dois fornecedores, compara os resultados e guarda o escolhido como variante em rascunho, com o registo completo do pedido.
5. **Fase 4, distribuição, automação e relatórios.** Exportador para WordPress (rascunho ou agendado), planos de distribuição, etiquetagem automática, relatório mensal automático, que substitui a versão manual da fase 1, com PESO e monitorização de imprensa, catálogo de KPIs padrão por disciplina, escolhidos por campanha e meio, com indicadores personalizados criados pela equipa e ligações de leitura às plataformas de dados. Útil: painel próprio de análise de redes. Saída: um conteúdo sai para o WordPress a partir de uma só pasta de história e o cliente recebe o relatório mensal no portal.

## Ordem de trabalho da parte A

1. Repositório, pnpm, integração contínua e Docker Compose com PostgreSQL e Redis.
2. Modelo de dados inicial (agência, conta de cliente, utilizador, pasta de história, permissão, evento de histórico) e RLS.
3. Início de sessão da equipa com Microsoft Entra ID e convite de clientes por e-mail.
4. Contas de cliente e permissões por papel, com herança pela pasta de história.
5. Pastas de história: criar, aninhar, partilhar com um cliente e ver o histórico de alterações.
6. Testes de isolamento e demonstração do critério de saída.

## Fora do âmbito (não construir)

- Publicação nativa em redes sociais com pré-visualizações por rede (decidido: não por agora, nem de raiz nem por serviço externo; o texto e os activos saem prontos a copiar).
- Registo de horas e rentabilidade por cliente (desenvolvimento futuro).
- API pública, instâncias dedicadas e vários clientes-agência.

## Por decidir (perguntar, não assumir)

- Preço do Cloud Compute da HawkHost e instalação do PostgreSQL (a confirmar com o suporte); política de cópias de segurança.
- Fundamento legal e aviso aos utilizadores do registo de consultas.
- Validação, com a equipa, do catálogo de KPIs proposto no documento.
- X: exportação de estatísticas da plataforma ou inserção manual.
- Quem mantém e opera a aplicação depois de entregue.
- Retenção e apagamento por conta de cliente (útil; sem fase atribuída).
- Leitura óptica (OCR) de PDF digitalizado: decidir se entra na fase 2.

## O que tem de vir de quem encomenda

- Parte A: registo da aplicação no Azure AD e o domínio; o Cloud Compute da HawkHost com acesso de administrador; ficheiros do sistema de design e do logótipo; um cliente de teste e um utilizador da equipa para a demonstração.
- Parte B: três a cinco ficheiros reais de planos em Excel, Word e PDF, anonimizados se for preciso.
- Fase 3: chaves de API dos fornecedores de IA a ligar (OpenAI, Anthropic, Google) e, se for para usar, licenças Microsoft 365 Copilot.
- Fase 4: acesso de administrador aos sites WordPress dos clientes piloto, autorizações de leitura do Google Analytics, do Google Ads e das redes sociais (LinkedIn, Meta, YouTube, TikTok, X e Pinterest), acesso aos dados do Carma. O registo das aplicações de leitura em cada plataforma deve começar cedo, porque a aprovação demora.

## Peças existentes a reaproveitar (por confirmar no código)

- Sistema de design da Do It On (interface do portal e dos PDF).
- Relatórios mensais de PR em PESO: lógica feita em Excel com Python, a passar a serviço.
- Cliente Microsoft Graph e sincronização Outlook para GHL (entregue, OAuth 2.0): base da conversão de e-mail em pasta de história.
- Skill de monitorização de imprensa da Carma (em evolução).

## Convenções de trabalho

- Commits pequenos e com mensagem clara. Cada alteração passa por testes antes de ir para o ramo principal.
- Cada módulo fecha com testes, e o critério de saída de cada parte demonstra-se antes de passar à seguinte.
- Comandos de arranque, teste e migração: acrescentar a este ficheiro assim que o repositório existir.
