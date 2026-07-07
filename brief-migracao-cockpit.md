# Cockpit Make Buzz · Brief de migração para versão multiusuário

**Para:** Claude Code
**De:** Adriano Ortolani, sócio-gestor da Make Buzz Comunicação
**Data:** julho de 2026

---

## O que existe hoje

Um app de gestão operacional da agência, em um único arquivo `index.html` (HTML + CSS + JS puro, sem framework), publicado em https://heroic-youtiao-936838.netlify.app/ via repositório GitHub `aortolani84/makebuzz` com deploy contínuo do Netlify (branch `main`, sem build). **Esse arquivo é a referência de produto: visual, textos, regras e fluxos já foram validados pelo time e devem ser preservados.** Os dados hoje vivem em `localStorage` — cada aparelho tem seu próprio quadro, e esse é o problema a resolver.

## O que esta migração precisa entregar

Transformar o app em ferramenta multiusuário de verdade: **qualquer alteração (status, comentário, tarefa nova, aprovação) aparece para todo o time em tempo real**, sem regenerar arquivo, sem exportar/importar. Mesmo endereço, mesmo visual, mesma lógica — dados centralizados e login.

## Stack definida

- **Backend:** Supabase (plano gratuito) — Postgres, Auth e Realtime.
- **Frontend:** o `index.html` atual evoluído. Sem framework novo; manter HTML/CSS/JS puro. Pode modularizar em poucos arquivos (`index.html`, `app.js`, `style.css`) se ajudar a manutenção.
- **Deploy:** o mesmo repositório GitHub + Netlify já configurados. Chaves do Supabase (URL e anon key) podem ficar no cliente — a segurança vem de RLS, não de esconder a anon key.
- **Login:** magic link por e-mail (sem senha). Restringir cadastro/entrada ao domínio `@makebuzz.com.br`.

## O time (usuários)

| Nome | E-mail | Papel |
|---|---|---|
| Adriano Ortolani | adriano@makebuzz.com.br | gestor |
| Daniela Loreto | daniela@makebuzz.com.br | gestor |
| Helder Daniel | helder@makebuzz.com.br | gestor |
| Cintia Esteves | cesteves@makebuzz.com.br | atendimento |
| Bruna | bruna@makebuzz.com.br | atendimento |
| Karol | karol@makebuzz.com.br | atendimento |
| Mateus | mateus@makebuzz.com.br | atendimento |
| Nayara | nayara@makebuzz.com.br | atendimento |
| Renan | renan@makebuzz.com.br | atendimento |

Papéis: **gestor** aprova/devolve entregas e edita tudo; **atendimento** cria e edita tarefas, comenta, envia para aprovação. Cores fixas por pessoa já definidas no código atual (objeto `PCOLORS`) — manter.

## Modelo de dados (migrar do código atual)

O `index.html` atual contém os seeds completos. Estruturas:

**frentes** (tarefas) — hoje ~115 registros no seed `SEED_FRENTES`:
`id, cliente (nome), demanda (texto), responsavel (texto, pode ter mais de um nome separado por vírgula), status, prazo (date|null), arquivada (bool), dataConclusao, criadaEm, resultado {link, nota, em}|null, aprovadoPor`
Status possíveis: `A fazer, Em andamento, Aguardando cliente, Aguardando veículo, Travado, Em aprovação, Concluído`. Os dois últimos não são selecionáveis à mão — só via fluxo.

**comentarios** — hoje array dentro de cada frente: `{autor, quando, txt}`. Na migração, virar tabela própria (`frente_id, autor_id, texto, criado_em`), com autor vindo do usuário logado (elimina o campo "quem comenta").

**pautas** (funil de divulgação) — seed `SEED_PAUTAS`, 17 registros:
`id, cliente, pauta (título), responsavel, obs, proc {release, estrategia, exclusiva, lista, envio, individual, abertura, consolidado, segundo, encerrar}, follows []`
As 9 etapas do processo estão no array `STEPS` do código — manter nomes, perguntas-guia e tipos (gate/choice/date), incluindo o desfazer por etapa.

**follows** — hoje array dentro da pauta: `{veiculo, jornalista, status}` com status em `FOLLOW_STATUS` (Enviado, Abriu, Respondeu, Em conversa, Publicou, Recusou, Sem retorno). Virar tabela própria.

**clientes** — o cadastro `CLIENTES` (19 contas com ficha editorial: resumo, gancho para podcasts, porta-voz, tags, pontos a confirmar, guardrail da Rivool). Virar tabela, com as fichas migradas como estão. A **nota interna** por cliente (hoje `mb_notas_v1`) vira campo/tabela compartilhada com autor e data.

**Migração dos dados:** escrever script que popula o banco a partir dos seeds do `index.html` atual. Nada se perde.

## Regras de negócio que não podem mudar

1. **Gate de aprovação.** Concluir passa por: responsável clica "Concluir…" → informa link do resultado (opcional, recomendado) → status vira "Em aprovação" → só usuário com papel gestor aprova (vai para Concluído + Histórico, registrando quem aprovou e quando) ou devolve (volta para Em andamento com comentário automático). Com login, o aprovador é o usuário autenticado — sem select de nome.
2. **Comentário vira tarefa** com um toque, herdando cliente e responsável e registrando a origem.
3. **TO DO do dia** gerado do estado vivo: agrupado por atendimento → cliente → tarefas, com marcas [VENCIDA] e [HOJE], mais blocos "para o gestor aprovar" e "pautas no funil (próximo passo)". **Status semanal** macro por cliente. Ambos com Copiar e WhatsApp (wa.me).
4. **Histórico e relatórios**: entregas aprovadas com filtro por período (semana/mês/semestre/tudo) e por pessoa; relatório em texto com panorama, por cliente, por pessoa, aprovações e lista de entregas.
5. **Bilhete por pauta**: gerador do modelo de bilhete individual usando porta-voz e cliente do registro (função `bilheteTexto`).
6. **Filtros da aba Hoje**: por status (incluindo Vencidas e Para hoje), por pessoa (chips coloridos) e visão por cliente/por atendimento. Botão flutuante para nova tarefa/pauta.
7. Aba Clientes: busca, filtro por tema, "só pendências", card com atendimento (cores) e as 2 prioridades; drawer com Ficha/Frentes/Pautas.

## O que a migração adiciona (além do tempo real)

1. **Login por magic link** com nome e cor do usuário; autor de comentário, criador de tarefa e aprovador passam a ser o usuário logado.
2. **Realtime**: assinar mudanças (Supabase Realtime) para o quadro atualizar sem recarregar.
3. **Notificação de comentário**: manter o selo "novo" por usuário (tabela de leitura `frente_id × user_id`), agora valendo entre aparelhos.
4. **Trilha de alterações mínima**: quem mudou status e quando (campo `atualizado_por/atualizado_em` basta nesta fase).
5. **RLS**: todo o time lê tudo; escrita conforme papel (aprovação só gestor). Nada público sem login.

## O que fica para depois (não fazer agora)

Push notification de verdade, disparo automático de e-mail, integração com transcrição de reunião, multiagência. Registrar como TODO no README.

## Critérios de aceite

1. Dois celulares logados com contas diferentes: status alterado em um aparece no outro em menos de 3 segundos, sem recarregar.
2. Atendimento não consegue aprovar entrega; gestor consegue, e o Histórico registra o nome real.
3. Os ~115 registros e as 17 pautas migrados batem com o seed do `index.html` atual.
4. TO DO do dia, status semanal e relatório saem idênticos em estrutura aos atuais.
5. App usável no Android (Samsung) — o time é mobile-first.
6. Deploy contínuo funcionando no mesmo repositório e endereço.

## Observações de estilo

Interface em português do Brasil, tom direto de redação (os textos atuais do app são a referência — não reescrever). Tipografia e identidade: Archivo, Newsreader, IBM Plex Mono, azul Make Buzz `#284888`, logo já embutida no HTML.
