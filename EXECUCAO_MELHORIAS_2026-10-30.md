---
title: Evidências de execução das melhorias do Fin
updated_at: 2026-10-31
tags: [fin, execucao, deploy, testes]
status: entrega-publicada-roadmap-parcial
---

# Execução e publicação

Plano de desenvolvimento completo: [[PLANO_MELHORIA_EXECUTAVEL_2026-10-30]]. Arquitetura confirmada nos provedores: [[RUNTIME_ARCHITECTURE]]. O anexo foi tratado como referência de requisitos; escolhas finais foram confirmadas pelo usuário: Directus 11, Aiven editorial e publicação dos repositórios. Este relatório distingue entregas verificadas de critérios ainda abertos.

## Entregas verificadas

| Área | Resultado | Evidência |
|---|---|---|
| CMS | Directus 11.17.4 no Render com PostgreSQL Aiven dedicado e CA confiável | https://fin-cms-wo8q.onrender.com/admin ; login e ACLs remotos exercitados |
| Editorial | editor cria/submete, revisor publica, leitor técnico somente lê publicado; acesso anônimo negado | teste remoto com conteúdo e usuários temporários removidos |
| API | deploy Render com DEBUG=false, papel fin_api_rw, token Directus e webhook | https://fin-api-jbrz.onrender.com ; release ebfc59a |
| API saudável | processo, banco e Redis disponíveis | /liveness e /readiness HTTP 200 após deploy |
| Integração | home, artigos, navigation, glossary e flags via API | cinco endpoints editoriais HTTP 200 |
| Financeiro | catálogo, cotação/histórico PETR4, benchmarks, dividendos, detalhes e valuation disponíveis com papel restrito | smoke HTTP 200 nos oito caminhos financeiros exercitados |
| Frontend | Preview Vercel com API_BACKEND_URL configurado; tipos OpenAPI e tratamento editorial de erro/vazio/stale | status Ready; CI remoto aprovado |
| Segurança PG | runtime sem superuser/BYPASSRLS/membership de migrations; acesso sem tenant não vê carteiras | conexão real fin_api_rw; catálogo com 1.378 ativos; zero carteiras sem tenant |
| RLS | duas contas, registro/login HTTP, carteiras próprias, pool reutilizado e commit/rollback | script verify_runtime_rls.py: escrita cruzada, DDL e promoção de role negados |
| Fila | refresh persiste comando PostgreSQL antes de responder 202; status por UUID | testes de integração de idempotência e contratos; ARQ sem scraping financeiro |
| Recuperação CMS | dump AES-256-GCM do schema v11 ativo restaurado em Aiven descartável | 40 tabelas e contagens conferidas, backup 28,30 s, restore 59,54 s; banco descartável removido |
| Infraestrutura | Terraform AWS marcado como referência; removido apply fictício; bootstrap separado de runtime | fin-infra/e5a3df2 |
| Migrations | reconciliação com main schema-only, ambos os ledgers considerados, drift completo antes do apply, lock em conexão direta e manual/CMS bloqueados | seis testes do runner + CI SQL remoto aprovado; PR fin-db#2 |

## Publicação

- API: main/ebfc59a63bfb2c5d7dab7fe9f8fb94cd05c71236; Render confirmou serviço live e smoke passou.
- CMS: main/779c497095833bac3af79fab232034c8e75f538a; configuração da aplicação foi publicada em 4bef279; commit posterior atualiza os scripts de verificação/backup. CI de configuração e imagem aprovado.
- Web: codex/directus-aiven/a28c29fb2845a4f61870187f74dd7104c83fe7be; Preview https://fin-nf1jzv0ff-barrosrafaelrb-8511s-projects.vercel.app ; CLI inspect confirmou Ready. Publicação Preview preserva o domínio de produção existente.
- Infra: main/e5a3df2aa4f01cb2be19a3a0ec0b3848dfefdd11 enviado ao GitHub.
- Banco: branch reconciliada codex/runtime-isolation-integrated/b86df0d; https://github.com/barrosrafa/fin-db/pull/2 . Integrado ao main pelo commit 22c6f665d8d77ab0ba17aaeab91b73cfd3d20084. A migração 18 aplicada no Neon tem o mesmo SHA-256 do arquivo versionado. PR #1 foi substituído para preservar as alterações recentes do main.
- Ingestão: main/4c92fc0610f50122171a0b8e4288122b571a1497 já publicado por trabalho paralelo. ENGINE_ENABLED e ENGINE_RELEASE_SHA foram confirmados nas variáveis GitHub. Não alteramos seus secrets durante o trabalho paralelo.

## Validação

API: 168 testes passaram, mais o ensaio real de RLS e CI remoto com PostgreSQL/Redis/Docker. Web: 11 testes locais e CI remoto de lint, tipos, testes e build aprovados. CMS: três testes de configuração e CI de imagem aprovados, além de teste remoto de ACL/publicação. Ingestão: nesta execução, 161 testes passaram e 11 foram ignorados por dependência de servidor ausente; não apresentar esse resultado como cobertura integral. Banco: seis testes do runner reconciliado passaram e o CI SQL foi aprovado.

O backup ativo está em fin-run/backups/cms/fin-cms-20261031T163040Z.dump.aesgcm e seu manifesto. Secrets, chave de backup, CA e metadados de propriedade anteriores estão em fin-run/secrets com modo 0600. Nenhum secret foi incluído no Git. A senha administradora Directus fica em cms-aiven.json, para rafa.a.barros@hotmail.com.

Onze objetos vinculados ao Timescale gerenciado mantiveram o proprietário do provedor; a lista está no arquivo privado financial-runtime.json. A API não é proprietária desses objetos. Não foi necessária exclusão ou reimportação de dados financeiros.

## Critérios ainda abertos do plano abrangente

Os sete sprints do plano não estão integralmente concluídos. Permanecem requisitos de aceite:

1. Consolidar credenciais de ingestão e transformação em processos distintos, coordenando o trabalho paralelo já publicado; o papel fin_ingestion_rw foi criado, mas não foi comprovada sua adoção por todos os jobs remotos.
2. Validar object storage cloud durável e replay do raw; artefato Actions de sete dias não equivale a política de arquivo de longo prazo.
3. Catálogo transacional de publicação, SLA de freshness por calendário e metadados consistentes em todos os contratos financeiros.
4. Auditoria completa de operações privadas, logs/trace correlacionados, métricas, alertas com canal/dono e ensaio de recebimento.
5. Restore financeiro e ensaios de incidente de Neon/Redis/fonte/worker; o restore CMS foi comprovado separadamente.
6. Benchmark representativo de queries, pool e percentis; ajustes somente após medição.
7. Remover todos os arrays financeiros de demonstração restantes e cobrir estados/e2e das telas críticas; a entrega atual removeu os fallbacks editoriais falsos.
8. Completar gates de contrato incompatível, SBOM/scan por digest, proteção de branch e ambiente staging integral separado.
9. Ampliar a integração visual da home/navigation/flags e das demais coleções editoriais. Os endpoints estão disponíveis, mas sua mera existência não prova substituição de todo conteúdo estático do site.

## Operação e reversão

Migrations futuras usam o papel de migrations e conexão direta; runtime não recebe senha administrativa. Reverter uma aplicação mantém o schema expandido e as cercas de isolamento. O estado anterior da propriedade foi registrado para análise, sem gerar script automático de reversão de segurança. Não executar todo o histórico financeiro novamente: o main passou por reorganização schema-only e drift histórico exige conciliação explícita.

Render Free pode suspender por inatividade; o primeiro acesso pode demorar mais de 50 segundos. Os deadlines e o último snapshot editorial protegem chamadas posteriores, sem transformar ausência de dados em sucesso fictício. O Preview Vercel usa a API remota existente; não constitui staging financeiro isolado.
