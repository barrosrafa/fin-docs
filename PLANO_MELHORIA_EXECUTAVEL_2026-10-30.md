---
title: Plano executável de melhoria do Fin
updated_at: 2026-10-31
tags: [fin, arquitetura, seguranca, aiven, plano]
status: em-execucao
---

# Plano executável de melhoria do Fin

## Escopo e decisões

Este plano traduz o anexo `plano-melhoria-sistema-fin.md` em entregas de desenvolvimento, evidências e operações. O anexo é material de referência; sugestões de tecnologia e descrições de produção precisam ser confrontadas com o código e os serviços reais. Não é evidência de que uma infraestrutura esteja implantada.

Manter os dados financeiros no PostgreSQL financeiro existente. Usar a Aiven fornecida exclusivamente para conteúdo editorial, em banco e credencial próprios. Não apontar o runner financeiro para a Aiven CMS. Redis continua efêmero. Objetos raw devem ter arquivo imutável e metadados, com replay testado. A credencial administrativa fornecida serve somente para bootstrap, nunca para runtime, commits ou logs.

Foi encontrada implementação local de Payload CMS no `fin-web`; a decisão do usuário foi substituí-la pelo Directus 11; o Payload local não é a arquitetura CMS de destino. Alterações locais preexistentes no frontend, ingestão e gateway devem ser preservadas. Não criar outro ETL nem outro histórico de migrations.

Fonte operacional: [[RUNTIME_ARCHITECTURE]]. Evidências e pendências desta execução: [[EXECUCAO_MELHORIAS_2026-10-30]].

## Método de execução e dependências

1. Inventariar árvore Git, implementações existentes, conexões e processos sem publicar secrets.
2. Registrar diferença entre comprovado localmente, comprovado remotamente e apenas declarado em README.
3. Preparar cada mudança compatível com a versão atual e plano de reversão.
4. Rodar verificações locais; integração de banco usa papel runtime e banco descartável.
5. Executar operações remotas aditivas autorizadas, com backup antes de alterar dados existentes.
6. Publicar somente com secrets, acesso ao provedor e validação do artefato. Deploy não é inferido de um build local.
7. Registrar evidência, resultado, limitação e próximo passo. Um item só é concluído com aceite exercitado.

Dependências: segurança e inventário → migrations/restore → fila única → qualidade/publicação → CMS/contratos → frontend/performance → observabilidade/incidentes. Conteúdo CMS pode avançar após isolamento de banco, sem esperar a migração financeira.

## Sprint 1 — Arquitetura e segurança (fases 0 e 1)

1. Criar `RUNTIME_ARCHITECTURE.md`: serviços, provedor declarado, processo, banco, tabelas, variáveis, timeouts e rollback. Rotular AWS como referência até existir deploy comprovado; registrar scheduler e worker como não verificados quando não houver acesso operacional.
2. Remover agenda e registro de jobs de setores, rankings, token e purge que retornam valores fixos. Chamadas legadas devem falhar explicitamente. Cache warm-up deve ler publicação financeira, sem scraper. Sincronizar catálogo com `core.ativo`, sem universo fixo como suposta sincronização B3/CVM.
3. CORS configurável por allowlist exata; sem regex global de previews nem credenciais para Bearer. Validar origem, esquema e ausência de wildcard. Documentar previews com banco/credenciais próprios.
4. Produção deve rejeitar JWT de teste, URL de banco placeholder, usuário administrador, debug e secrets obrigatórios ausentes. Secrets não aparecem em repr nem erro. Billing só pode ser ativado com secrets reais.
5. Rate limit: confiar em forwarded IP somente quando o socket pertence a proxy explicitamente configurado; validar cadeia da direita para esquerda. Manter fallback limitado com alerta Redis. Separar limites de autenticação, exportação, busca e mercado. Identidade autenticada requer validação de JWT antes de usá-la como chave.
6. Mapear SQL real da API e do ingestion; criar grants específicos para `fin_api_rw`, `fin_ingestion_rw`, `fin_migration_owner`, `fin_readonly_public`. Não conceder todos os schemas por conveniência. CMS tem proprietário de migrations e papel runtime separados.
7. Testar RLS com dois usuários, tenant ausente e reutilização de conexão após commit/rollback. Testar negação de DDL e escrita de billing pelo ingestion. FORCE RLS depende da revisão dos triggers e jobs reais.
8. Auditar carteira, transações, billing, permissões, reprocessamento e publicação: actor, recurso, ação, request_id, IP, user-agent e data; remover dados privados dos metadados.

Aceite: jobs incompletos não retornam sucesso; origem não autorizada não recebe CORS; configuração insegura é rejeitada; papéis runtime sem owner/BYPASSRLS e tenancy comprovada no banco financeiro. Validação: pytest de config/CORS/worker/proxy + integração real de RLS. Rollback: reverter código mantendo jobs incompletos desagendados; grants anteriores somente após revisão.

## Sprint 2 — Migrations, infraestrutura e recuperação (fase 2)

1. Eleger checkout financeiro canônico: existem `/fin` e `/fin/fin-db`; comparar histórico antes de alterar runner duplicado. Preservar migrations aplicadas.
2. Reutilizar `system.schema_migrations`; adicionar metadados compatíveis (nome, executor, ambiente, duração) em migration nova. Nunca atualizar checksum histórico para ocultar drift.
3. Runner: plano offline com checksums; plano conectado somente leitura; advisory lock de sessão, transação por arquivo, timeout, ordem explícita, validação prévia de todos os checksums antes do primeiro SQL. Migration manual nunca entra no automático. Não adicionar `19_merge_base_b` se existir como manual.
4. Registrar migrations automáticas, manuais e data-fixes em manifesto; manuais fora do pipeline. Não mover arquivos históricos sem estratégia para chaves do ledger.
5. Marcar Terraform AWS como referência, remover falsos apply/echo e OIDC fictício. Pipeline de referência valida estrutura sem alegar provisionamento. Inventariar configuração efetiva Vercel/Render/Neon/Redis/Aiven por ambiente.
6. Isolar CMS em banco Aiven próprio, com runtime sem CREATEDB/CREATEROLE/BYPASSRLS, sem acesso ao financeiro e sem senha administrativa. Migrations CMS têm ciclo próprio. Revogar CREATE em public e PUBLIC CONNECT no banco CMS quando compatível com o serviço.
7. Backup: metas iniciais, a confirmar com os planos contratados: financeiro RPO 15 min/RTO 2 h; editorial RPO 24 h/RTO 4 h. Não apresentar metas como capacidade comprovada.
8. Criar dump customizado criptografado, retenção 30 dias e restore em banco descartável; verificar contagens, permissões e tempo. Backup do provedor e PITR precisam ser verificados no painel/API. Ensaiar replay raw com hash/parser version.
9. Benchmark versionado: queries de cotação, intraday, dividendos, ranking, screener, carteira e lateral; medir EXPLAIN ANALYZE BUFFERS em ambiente representativo, p50/p95/p99, pool wait e cardinalidade. Não executar consultas caras sobre produção sem limites.

Aceite: runner recusa manual/drift e falha atomicamente; apply fictício removido; isolamento CMS testado; restore executado com tempo registrado. Rollback: compensação de migrations aditivas e restauração validada; nunca DROP de produção para testar.

## Sprint 3 — Caminho único de ingestão (fase 3.1–3.4 e 3.6)

1. Reutilizar `ingestion.ingestion_task`, `ingestion_run`, `ingestion_checkpoint` e `ingestion_issue` já implementados. Não criar fila paralela para obter nomes idênticos ao anexo.
2. Documentar equivalência de estados: COMPLETED → SUCCEEDED; FAILED terminal → FAILED_PERMANENT; PENDING com erro/tentativa → FAILED_RETRYABLE; registrar quarentena separadamente até migration compatível ampliar enum.
3. API valida ticker em catálogo autoritativo, parâmetros e permissão; gera chave natural dataset/ticker/janela e insere comando idempotente. Commit antes de HTTP 202; retornar UUID real e Location. Falha de INSERT não pode retornar `sync_pending` como sucesso.
4. Adicionar GET de tarefa com UUID, estado, tentativas, timestamps e erro sanitizado. Não expor stack, senha ou tarefas privadas de outros usuários.
5. Remover chamada ao scraper e promoção canônica do worker API; compatibilidade legada apenas encaminha ao PostgreSQL. ARQ fica com cache e notificações.
6. Claim SKIP LOCKED com lease; renovação/fencing em todas as escritas e checkpoints; worker morto tem tarefa recuperável e limite de tentativas. Classificar erro permanente/transitório, backoff exponencial com jitter, Retry-After limitado e justiça entre fontes.
7. Eleger operação inicial como batch GitHub Actions com concorrência exclusiva, pois worker persistente remoto não foi demonstrado. Se ativar consumidor contínuo depois, Actions apenas enfileira e scheduler Docker fica desabilitado. Registrar run e métricas verdadeiras em ambos.
8. Testar duplicação, duas conexões em claim, morte no lease, token vencido, publicação anterior ao checkpoint e retry após commit parcial.

Aceite: um pedido de refresh leva a tarefa durável, execução real e status consultável; zero scraping pesado na API. Rollback: manter fila e tarefas; desativar produtor se consumidor não estiver saudável.

## Sprint 4 — Qualidade, raw, freshness e publicação (fases 2.4, 3.5 e 4)

1. Reutilizar validadores temporais e OHLC, raw sink e tabelas de qualidade existentes. Cobrir timezone BRT/UTC, preço/volume negativos, ajuste corporativo, duplicidade, calendário, gaps e divergências entre fontes.
2. Armazenar payload grande em object storage por hash; persistir chave, SHA-256, content-type, bytes, coleta, run, parser e retenção. Arquivo local não é considerado object storage durável em cloud. Provedor precisa de credencial e teste upload/download/checksum.
3. Quarentena deve manter evidência imutável e motivo; promoção falha se gate ou reconciliação falhar. Testar que linha rejeitada não aparece no dataset publicado.
4. Implementar catálogo de datasets com última coleta/sucesso/publicação, as_of, qualidade, contagem, versão e run. Atualização do catálogo na mesma transação da publicação ou outbox reconciliável.
5. SLA por calendário: intraday somente no pregão, EOD após fechamento, dividendos diário e rankings após cálculo. Monitorar falha versus mercado fechado.
6. Respostas preservam contrato atual e acrescentam metadata versionada (as_of, collected_at, published_at, source, quality_status, dataset_version, schema_version). Ausência de proveniência não é preenchida com horário do cache.
7. Redis com namespaces finance/content/rate/lock; TTL e tamanho máximos; alias de versão só muda após publicação concluída. Conservar versão anterior e testar rollback.
8. Fallback conserva último conteúdo/dado válido com stale explícito. Banco indisponível sem snapshot é 503; coleção realmente vazia é 200 com empty. Nunca inventar zero para mascarar falha.

Aceite: teste coleta→raw→qualidade→publicação→API prova rastreabilidade e dado rejeitado ausente. Replay reconstrói dataset em banco descartável. Rollback: alias para versão válida anterior.

## Sprint 5 — CMS na Aiven (fase 5)

1. Consolidar CMS escolhido após decisão sobre Payload existente versus Directus. Evitar dois painéis e dois modelos editoriais concorrentes.
2. Banco editorial exclusivo `fin_cms`; schema exclusivo; papel runtime `cms_runtime_rw` e proprietário `cms_migration_owner`. Conectar com TLS, connection timeout e pool compatível com limite do plano Aiven.
3. Versionar migrations e modelo de site/home/navigation/footer/flags/pages/articles/glossary/announcements/authors/media. Reutilizar artigos, livros, glossário e usuários editoriais existentes se Payload permanecer.
4. Separar editor, revisor e administrador; draft→review→published; publicar somente com campos obrigatórios, autor, data e versão; administração não é autenticação dos usuários financeiros.
5. Toggles editoriais têm allowlist; billing, saques, manutenção, validação JWT, ingestion e read-only ficam na configuração operacional da API.
6. Webhook autenticado, idempotente e limitado invalida alias por versão; sem revalidar cache para rascunhos. Registrar auditoria e retries. Não expor segredo no browser.
7. API entrega `/api/v1/content/home`, `/navigation`, `/feature-flags`, `/glossary` com contratos próprios, timeout e último snapshot válido. Web usa API; preview editorial autenticado pode acessar CMS deliberadamente.
8. Seed idempotente sem contas/senhas padrão. Primeiro admin por email real aprovado e senha aleatória guardada no mecanismo de secrets. Não inventar identidade do usuário.
9. Gerar migration pelo CLI do CMS instalado, inspecionar SQL e aplicar com owner. Runtime não usa auto schema push em produção. Restore e negativa de DDL precisam passar antes de deploy.

Aceite: publicar altera site via API; rascunho fica privado; CMS fora do ar serve último snapshot stale; banco CMS sem dados financeiros; migrations não dependem de runtime administrador. Rollback: versão editorial anterior e release anterior compatível.

## Sprint 6 — Contratos, frontend e performance (fases 6 e 9)

1. Exportar OpenAPI estável da API e versionar diff; gerar TypeScript com ferramenta fixada no lockfile. Manter `/api/v1` para não quebrar clientes só para espelhar `/v1` do anexo.
2. Envelope de erros com code, message, request_id, retryable e details sanitizados; migrar consumidores antes de remover campos legados. Testar 401/403/404/409/422/429/503.
3. Precisão financeira explícita por campo (decimal string onde necessário), timestamps timezone e paginação limitada. CI detecta campo removido, alteração de tipo e unidade.
4. Inventariar arrays hardcoded por rota; classificar conteúdo editorial, fixtures e dados financeiros. ETFs/rankings/setores/filtros recebem fonte real ou indisponibilidade clara; nunca substituir stub por outra fixture no runtime.
5. Unificar fetch server/client com URL, auth, deadline, cancelamento, retry apenas seguro, request_id e política documentada de CDN/Redis/IndexedDB.
6. Cada tela crítica testa loading/empty/stale/error/partial/success, timestamp e acessibilidade de avisos. Preservar a reestruturação local `(frontend)`/`(payload)` existente.
7. Rodar benchmark e corrigir somente planos medidos; índices criados com operação compatível. Não adicionar Timescale/ClickHouse/Kafka/Kubernetes para compensar falta de medição.

Aceite: build/type/lint, contrato gerado e e2e das rotas críticas passam; nenhum dado financeiro de teste entra em produção. Rollback: manter contratos aditivos e cliente anterior.

## Sprint 7 — Operação, observabilidade e CI (fases 7 e 8)

1. Logs JSON correlacionam request_id/trace_id/run_id, rota, serviço, ambiente, estado, latência e código; redigir tokens e URLs com credenciais. Nunca logar carteira ou payload pessoal.
2. Métricas API: latência por rota, 4xx/5xx, pool wait, timeout upstream, Redis hit, queries/request e bytes. Ingestion: backlog, idade, retries, qualidade, linhas, watermark e freshness. Infra: conexões, locks, bloat, armazenamento, memória/eviction, custo e deploy.
3. `/liveness` e aliases legados provam processo; `/readiness` testa dependências com deadline; `/freshness` consulta publicação e SLA. Dado atrasado não muda artificialmente para fresh após cache warm-up.
4. Alertas com dono/canal e runbook: freshness, fila antiga, 5xx, auth anormal, Redis, crescimento PG, queda de coleta, quarentena, CMS/webhook e backup. Configurar no provedor só após identificar conta e canal; arquivo de regra não comprova alerta ativo.
5. CI API: lint/type/unit/integration PG+Redis/RLS/OpenAPI/image/smoke. Ingestion: fixtures/schema/lease/idempotência/quality/replay/dbt/publicação. DB: plan/empty/representative/drift/grants. Web: lint/type/build/contratos/a11y/e2e. CMS: migration/seed/permissões/webhook/snapshot.
6. Fixar terceiros por SHA verificado (não inventar digest), menor GITHUB_TOKEN, gates que falham, SBOM e scan da imagem por digest. Branch protection e secrets de ambiente são operações de provedor separadas de editar YAML.
7. Release: build→test→plan→migration compatível→deploy→smoke→monitoramento; expansão/contração antes de remover legado. Job inexistente não deve constar como deploy bem-sucedido.
8. Exercitar Neon/Redis/CMS indisponíveis, fonte bloqueada, migration interrompida, duplicação, morte de worker, schema incompatível, perda de cache e restore. Registrar gatilho, resultado, duração e compensação.

Aceite: evidência de alerta recebido no teste, restore cronometrado e incidente com recuperação; CI obrigatório e sem gates permissivos. Rollback: código e alias de versão; schema expandido preservado.

## Matriz de conclusão

| Domínio | Evidência mínima | Condição que impede marcar concluído |
|---|---|---|
| Segurança | RLS/grants com runtime real + config/CORS/proxy | teste somente com admin |
| Migrations | drift/manual/lock/atomicidade em banco novo e existente | apenas imprimir plano |
| Recuperação | dump + restore + hashes/contagens + tempo | backup não restaurado |
| Ingestão | API→fila→worker→publicação com lease | apenas mocks ou batch sem run |
| Qualidade | rejeição e replay com proveniência | dados suspeitos publicados |
| CMS | publicação + ACL + snapshot + outage | painel instalado sem fluxo |
| Contratos/web | OpenAPI/cliente + e2e de estados | build sem integração |
| Operação | alerta exercitado + incidente + métricas | regras sem provedor ativo |

Estimativa: sete sprints sequenciais, ajustadas pelos testes e acesso aos provedores. Não há promessa de concluir sete sprints apenas com uma URL de banco. A execução começa imediatamente pelo que é verificável; credenciais ausentes e decisões externas são pendências explícitas, sem sucesso fictício.


## Acompanhamento da execução

Escolhas confirmadas pelo usuário: Directus 11, PostgreSQL editorial dedicado na Aiven e administrador rafa.a.barros@hotmail.com. A solicitação inclui push e deploy. O inventário efetivo está em [[RUNTIME_ARCHITECTURE]]; evidências e critérios ainda abertos estão em [[EXECUCAO_MELHORIAS_2026-10-30]]. A presença de uma etapa neste plano não significa que tenha sido concluída.
