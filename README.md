# O Pensador — README

## Visão geral

**O Pensador** é uma aplicação inovadora que transforma qualquer acervo de conhecimento em uma mesa de debate entre especialistas virtuais. Utilizando o GPT-5 como motor central, ele não apenas responde perguntas, mas constrói análises profundas, cruzando informações e gerando múltiplas perspectivas sobre qualquer tema. Com isso, pesquisadores, empresas e estudantes ganham um aliado estratégico para explorar ideias, tomar decisões embasadas e acelerar descobertas. Tudo em poucos cliques, com uma experiência intuitiva e interativa.

Uma aplicação web (Streamlit) que transforma um acervo de conhecimento em uma "mesa de debate" entre especialistas virtuais. Utiliza a API OpenAI (ex.: GPT-5) como motor de geração e um back-end próprio para armazenar conversas, gerenciar pagamentos e buscar/streamar resultados de pesquisa em artigos ou obras de pensadores.

O frontend (este código) faz o papel de interface do usuário, orquestrando chamadas para um servidor remoto (via `requests`) e para a OpenAI SDK (via `openai.OpenAI`). Ele também gerencia sessões, créditos, histórico de pagamentos e múltiplas conversas.

---

## Principais funcionalidades

* Autenticação de usuário (integração com `st.user`).
* Seleção de modelos (ex.: `gpt-5-nano` gratuito e `gpt-5` Pro).
* Busca/analise em base de artigos científicos ou obras de pensadores (streaming de múltiplas etapas).
* Sistema de créditos / loja (geração de links de pagamento via backend).
* Histórico de conversas e múltiplos chats (multichat com UUIDs).
* Uploads aceitos no `st.chat_input` (imagens, PDF, MP3).
* Integração com backend via vários endpoints REST.

---

## Arquivos e dependências recomendadas

Crie um arquivo `requirements.txt` com, pelo menos, as dependências abaixo:

```
streamlit>=1.24
openai>=1.0.0
requests
pytz
Pillow
```

> Ajuste versões conforme necessário (este projeto usa recursos recentes do Streamlit – badges, chat APIs, etc.).

---

## Configuração (secrets / ambiente)

No `streamlit secrets` (ou `secrets.toml`) defina as chaves usadas pelo app. Exemplo de `secrets.toml`:

```toml
# .streamlit/secrets.toml
IP = "meu.servidor.externo:porta"
CHAVE = "MINHA_CHAVE_SHARED_COM_BACKEND"
IDMASTER = "id_do_admin_para_acesso_avancado"
# (opcional) OPENAI_API_KEY = "sk-..."  # se desejar usar st.secrets para chave OpenAI
```

Além disso, o app também tenta usar `st.session_state.openai_api_key` como fallback se `st.secrets` não contiver a chave.

Observações de ambiente:

* O código define o fuso-horário `TZ = America/Sao_Paulo` (usa `time.tzset()` — funciona em sistemas Unix).
* A autenticação espera que `st.user` esteja disponível (integração com mecanismo de identidade do Streamlit/Workspace). Garanta que sua instância de Streamlit ofereça esse objeto.

---

## Como executar

1. Criar ambiente virtual Python e instalar dependências:

```bash
python -m venv .venv
source .venv/bin/activate   # ou .\venv\Scripts\activate no Windows
pip install -r requirements.txt
```

2. Configurar `secrets.toml` conforme o exemplo acima.

3. Rodar o Streamlit:

```bash
streamlit run app.py
```

(Substitua `app.py` pelo nome do arquivo que contém o código.)

---

## Endpoints do backend usados (expectativas de payload/resposta)

O frontend faz `POST` para várias rotas no host `http://{IP}`. Abaixo estão as rotas e o formato esperado pelo frontend (resumido):

* `POST /produto/post/filosofo/retornarconversa/`
  Payload: `{"data": {"usuario": <usuario>}, "chave": <chave>}`
  Resposta esperada: JSON com `saida` sendo lista de mensagens da conversa.

* `POST /produto/post/filosofo/retornarusuario/`
  Payload: `{"data": {"usuario": <usuario>, "email": <email>}, "chave": <chave>}`
  Resposta esperada: JSON com `saida` contendo dados do usuário (ex.: créditos, etc.).

* `POST /produto/post/filosofo/addartigostream/`
  Usado para enviar a argumentação/resultado final (stream) ao backend.

* `POST /produto/post/filosofo/addusuario/`
  Usado para salvar interação usuário -> resposta no banco.

* `POST /produto/post/filosofo/criarpagamento/`
  Gera link de pagamento (Boleto/PIX/Cartão). Retorna lista `saida` de pagamentos.

* `POST /produto/post/filosofo/retornarpagamentos/`
  Retorna histórico de pagamentos para o `idCliente`.

* `POST /produto/post/filosofo/removercreditos/`
  Atualiza contagem de créditos do usuário (quando usa o plano Pro).

* `POST /produto/post/filosofo/recomecarconversa/`
  Reseta / reinicia a conversa do usuário.

* `/produto/post/artigos/iniciar/` e `/produto/post/artigos/continuar/` (ou o mesmo caminho com pensadores)
  Endpoints para iniciar a busca/streaming em artigos ou continuar a pipeline (streams 1..5). Devem retornar estruturas que o frontend utiliza como `mensagem` para enviar ao OpenAI.

> **Importante**: Ajuste o backend para devolver as estruturas que o frontend espera (normalmente `saida` contendo `mensagem`, `usuario`, `mensagem` compostas por objetos compatíveis com o SDK OpenAI chat messages).

---

## Estrutura de threads / principais classes

* `RetornoMensagens(Thread)`: solicita ao backend as mensagens (histórico) do usuário e popula `mensagens`.
* `RetornoUsuario(Thread)`: busca informações do usuário (créditos, etc.).
* `EnviarArgumentacao(Thread)`: envia as argumentações / referências geradas para o backend (para armazenamento).
* `EnviarUsuario(Thread)`: envia a interação (entrada/saída) do usuário para persistência.

Essas threads são usadas para evitar bloqueio da UI enquanto as chamadas de rede ocorrem.

---

## Fluxo de consulta (resumido)

1. Usuário envia prompt no `st.chat_input`.
2. Se ativado `marcar_artigos` ou `marcar_pensadores`, o app chama o endpoint de `iniciar/` no backend para obter mensagens iniciais e `usuario` para streaming; em seguida faz 4 chamadas em sequência (`stream=1..4`) coordenando chamadas ao OpenAI para produzir as etapas: leitura, sabatina, preparo de contra-argumento, contra-argumentação.
3. O resultado final é processado, referências são mostradas e o backend é notificado (via `EnviarArgumentacao`). Créditos são removidos conforme o plano.
4. Para o modo padrão (sem artigos/pensadores), o app monta `provisorio` — um histórico de mensagens — e envia diretamente para o OpenAI para resposta.



* Corrigir os bugs apontados (fazer um PR com correções).

