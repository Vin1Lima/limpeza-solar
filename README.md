# Limpeza Solar: acesso pela internet (ESP32 + MQTT/EMQX + GitHub Pages)

## Como funciona

Pense no EMQX Serverless como uma **central dos Correios** na internet. O ESP32 e o site nunca falam direto um com o outro. Os dois só deixam e retiram cartas na central, cada carta endereçada a um "tópico":

```
 [ESP32 no telhado] --Wi-Fi do local--> [  EMQX Cloud  ] <--internet-- [ Site no GitHub Pages ]
        |   publica:  limpezasolar/placa01/status  (estado em JSON)    ->  o site mostra
        |   publica:  limpezasolar/placa01/online  ("1" ou "0")         ->  "Online / ESP32 offline"
        |   assina:   limpezasolar/placa01/cmd     <- o site manda "a=limpar", "tm=10", ...
        |
   [rede própria "LimpezaSolar"] ← continua no ar como reserva (http://192.168.4.1)
```

- **De qualquer lugar:** o ESP32 *sai* para a internet e se conecta ao EMQX. Por isso não precisa abrir porta no roteador nem ter IP fixo.
- **Protegido:** as duas conexões usam TLS (criptografia, o mesmo cadeado do HTTPS) e pedem usuário e senha.
- **Reserva local:** sem internet, tudo continua funcionando como antes pela rede "LimpezaSolar". O horário programado e a chuva rodam no próprio ESP32 e não dependem da nuvem.

## O que tem nesta pasta

| Arquivo | Para quê |
|---|---|
| `firmware/limpeza_solar_mqtt/limpeza_solar_mqtt.ino` | Código do ESP32-S3 |
| `firmware/limpeza_solar_mqtt/segredos.h` | **Suas senhas** (Wi-Fi e EMQX). Fica só no seu PC |
| `firmware/limpeza_solar_mqtt/segredos.exemplo.h` | Modelo do arquivo acima (este pode ir para o GitHub) |
| `docs/index.html` + `docs/mqtt.min.js` | Site que vai para o GitHub Pages |
| `.gitignore` | Impede que o `segredos.h` vá para o GitHub |

---

## Passo 1: criar o servidor no EMQX Serverless (grátis)

> Não use o HiveMQ Cloud: o plano grátis (Serverless) não aceita mais cadastros e será desligado em 31/12/2026.

1. Crie uma conta em **emqx.com** (não pede cartão) e entre no console.
2. Se ainda não tiver um projeto, crie um. Clique em **New Deployment** e escolha **Serverless**.
3. **Deixe o Spend Limit em 0.** Assim ele usa só a cota grátis. Se a cota do mês acabar, o servidor para, sem cobrança.
4. Dê um nome, aceite os termos e clique em **Deploy**.
5. Abra o deployment. Em **MQTT Connection Information** estão o endereço (**Address**), as portas **8883** (ESP32) e **8084** (site) e o **CA Certificate** (`emqxsl-ca.crt`). Esse certificado (DigiCert Global Root G2) já está dentro do firmware.
6. Em **Access Control → Authentication**, clique em **Add** e crie **dois usuários**:
   - `esp32`, que vai no firmware;
   - `site`, que você digita no celular.

   Use senhas diferentes e fortes.

> O EMQX Serverless só aceita conexões criptografadas: porta 8883 (MQTT com TLS) e 8084 (WebSocket com TLS).

## Passo 2: gravar o ESP32

1. Na Arduino IDE, abra o **Gerenciador de Bibliotecas** e instale **PubSubClient** (de Nick O'Leary).
2. Abra `limpeza_solar_mqtt.ino`. O `segredos.h` aparece como uma aba. Preencha:
   - `WIFI_CASA_NOME` / `WIFI_CASA_SENHA`: Wi-Fi do local onde a placa fica. **Tem que ser 2,4 GHz**, porque o ESP32 não enxerga 5 GHz.
   - `MQTT_HOST`: já vem preenchido com o seu endereço (`m182fe11.ala.us-east-1.emqxsl.com`).
   - `MQTT_USUARIO` / `MQTT_SENHA`: o usuário `esp32` do passo 1.
3. Placa: **ESP32S3 Dev Module**. A partição padrão serve (o programa ocupa cerca de 81%).
4. Grave e abra o **Monitor Serial a 115200**. O esperado é isto:

```
Rede criada: LimpezaSolar
Site local: http://192.168.4.1
Internet: Wi-Fi "MinhaRede" + EMQX m182fe11.ala.us-east-1.emqxsl.com (tópicos limpezasolar/placa01/*)
Wi-Fi: procurando "MinhaRede"...
Wi-Fi "MinhaRede" conectado. IP 192.168.0.50, sinal -61 dBm
Relógio acertado: 14:32
EMQX: conectando... ok!
```

Como o relógio agora acerta sozinho pela internet (NTP), não precisa mais deixar o site local aberto para isso.

## Passo 3: colocar o site no ar (GitHub Pages)

1. Crie um repositório **público** no GitHub (no plano grátis, o Pages só funciona em repositório público).
2. Envie a pasta do projeto. Se usar "Add file → Upload files", mande pelo menos `docs/index.html` e `docs/mqtt.min.js`. **Nunca envie o `segredos.h`.**
3. No repositório, vá em **Settings → Pages → Build and deployment**. Em *Source* escolha **Deploy from a branch**, branch `main`, pasta **`/docs`**, e clique em **Save**.
4. Em 1 ou 2 minutos o site fica em `https://SEU-USUARIO.github.io/NOME-DO-REPO/`.
5. Abra no celular e entre com o usuário `site` e a senha. Com "Lembrar neste aparelho" marcado, você só faz isso uma vez.
6. Dica: no menu do navegador, use **"Adicionar à tela inicial"** para abrir o painel como se fosse um app.

O endereço do servidor já está em `docs/index.html` (`const CLUSTER`). Ele não é segredo, por isso pode ficar no arquivo publicado. Se um dia criar outro deployment, troque ali e no `segredos.h`.

---

## Quando algo não funciona

| O que aparece | Causa provável | O que fazer |
|---|---|---|
| Serial fica em `Wi-Fi: procurando...` | Nome ou senha do Wi-Fi errados, rede 5 GHz ou sinal fraco | Confira `segredos.h`. Pela rede LimpezaSolar, a aba Manual mostra o estado do Wi-Fi |
| `EMQX: ... falhou (-2: não conectou...)` com erro TLS | `MQTT_HOST` errado ou relógio ainda sem acertar | Confira a URL. O ESP32 tenta de novo sozinho, com espera crescente até 60 s |
| `falhou (4/5: usuário ou senha...)` | Usuário `esp32` errado | Confira em Access Control → Authentication |
| Site: "Usuário ou senha incorretos" | Usuário `site` errado | Confira, ou crie outro usuário |
| Site: "Não consegui falar com o servidor" | Deployment parado (cota do mês acabou) ou rede que bloqueia a porta 8084 (algumas redes corporativas) | Veja o status no console do EMQX; teste no 4G do celular |
| Site: **ESP32 offline** | Placa sem energia ou sem Wi-Fi | O aviso aparece cerca de 20 s depois da queda, porque o EMQX publica o "último desejo" (LWT) do ESP32 |

Comportamentos normais:
- **O celular cai da rede "LimpezaSolar" quando o ESP32 entra no Wi-Fi do local.** O ESP32 tem um rádio só, e a rede própria passa a usar o mesmo canal do roteador. Basta reconectar.
- **O ESP32 não tenta se conectar à nuvem com bomba ligada.** Conectar trava o programa por 1 a 3 s, então ele espera o ciclo acabar.
- **Comandos enviados com o ESP32 offline não ficam guardados para depois.** Isso evita a bomba ligar horas mais tarde, quando a internet voltar. Por isso o site trava os botões enquanto o ESP32 está fora do ar.

## Aviso de sol forte (choque térmico)

Entre **10h e 17h**, no horário do local da placa (relógio do ESP32), os dois sites, o local e o da internet, pedem confirmação antes de:
- **Iniciar limpeza**;
- **ligar a Bomba 1 ou a Bomba 2** na aba Manual (desligar nunca pergunta);
- **programar o horário automático** dentro dessa faixa. Se você confirmar, fica um alerta fixo embaixo do relógio.

O motivo: com sol forte o vidro da placa fica muito quente. A água fria pode causar choque térmico (microfissuras no vidro e nas células) e manchas, porque evapora rápido demais. Depois de confirmar, o site não pergunta de novo por 10 minutos, para você poder testar as bombas.

É só um aviso: se você confirmar, a limpeza acontece. O **botão BOOT** da placa não avisa, porque não tem tela.

### Limpeza por chuva entre 10h e 17h

Na aba **Automático**, dentro de "Quando chover", o interruptor **Permitir entre 10h e 17h** decide se a chuva pode disparar a limpeza nesse período.
- **Desligado (padrão):** se chover entre 10h e 17h, o ESP32 **não** limpa. Se ainda estiver molhado às 17h, ele limpa nessa hora. Enquanto espera, o site mostra "Chovendo agora: a limpeza espera passar das 17h".
- **Ligado:** a chuva dispara a limpeza a qualquer hora. Para ligar, o site pede confirmação.

Essa regra roda no próprio ESP32 e fica salva na memória: vale mesmo com o site fechado e depois de reiniciar. Se o relógio da placa ainda não tiver acertado, ela não sabe que horas são e deixa a chuva limpar normalmente.

A faixa de horário fica em `SOL_INI` e `SOL_FIM` no JavaScript das duas páginas (para os avisos) e em `SOL_INI_H` e `SOL_FIM_H` no firmware (para a regra da chuva). Se mudar, mude nos três lugares.

## Registros para os testes (TCC v2.1)

O ESP32 grava cada evento numa planilha CSV dentro da própria memória flash. Os registros continuam lá depois de desligar a placa.

**Onde baixar:**
- **Site local** (rede LimpezaSolar ou IP do ESP32 no Wi-Fi do local): aba **Registros → Baixar todos os registros (CSV)**, ou direto em `http://192.168.4.1/registros.csv`.
- **Site da internet:** aba **Registros → Carregar histórico do ESP32** (últimos 500 eventos) **→ Baixar eventos (CSV)**. Na mesma aba ficam o **tempo de resposta** de cada comando e o botão **Baixar tempos de resposta (CSV)**.

Os arquivos usam `;` como separador e abrem direto no Excel em português.

**Colunas:** `data_hora; uptime_s; evento; origem; detalhe; leitura_chuva; rssi_dbm`
- `uptime_s` são os segundos desde que o ESP32 ligou. Serve para os eventos de antes do relógio acertar, que ficam com `data_hora` vazia.
- `origem` diz quem causou o evento: `auto`, `botao`, `site_local` ou `internet`.

| evento | quando acontece | detalhe |
|---|---|---|
| `ligou` | o ESP32 ligou ou reiniciou | versão, **motivo do reset** (energia, watchdog, **queda de tensão**...), memória livre |
| `wifi_on` / `wifi_off` | conectou / caiu o Wi-Fi do local | IP e **tempo para conectar** |
| `nuvem_on` / `nuvem_off` / `nuvem_falha` | conexão com o EMQX | **tempo da conexão TLS**, código de erro |
| `relogio_ok` | relógio acertado | fonte: `internet_ntp` ou `celular` |
| `limpeza_inicio` | começou um ciclo | motivo (manual, horário, chuva, botão), tempo de bomba, duração prevista |
| `limpeza_fim` | ciclo terminou | **tempo medido de cada etapa**, total, previsto e **erro em ms** |
| `limpeza_interrompida` | ciclo parado no meio | etapa e quanto tempo já tinha rodado |
| `bomba` / `bombas_desligadas` | bomba ligada/desligada à mão | qual bomba |
| `seguranca` | bomba desligada pelo limite de 60 s | qual bomba |
| `chuva_molhou` / `chuva_secou` | sensor mudou de estado (após 3 s estável) | leitura na coluna `leitura_chuva` |
| `chuva_bloqueada` | choveu entre 10h e 17h com a opção desligada | |
| `leitura` | a cada 5 min | leitura do sensor (só no CSV, não aparece na lista) |
| `config` | alguma configuração mudou | o que mudou (ex.: `horario=06:30 tc=10`) |
| `comando_ignorado` | pediu limpeza com outra em andamento | |

**Ideias de testes** (cada um sai pronto de um filtro na planilha):
1. **Precisão do ciclo:** faça 10 limpezas de 5 s e 10 de 10 s, filtre `limpeza_fim` e calcule média e desvio do `erro_ms`.
2. **Tempo de resposta pela internet:** faça 20 comandos pelo site no 4G e no Wi-Fi e compare média, mínimo e máximo do CSV de tempos.
3. **Detecção de chuva:** pingue água e compare a hora de `chuva_molhou` com a de `limpeza_inicio` (motivo=chuva). O esperado é cerca de 10 s.
4. **Bloqueio por sol forte:** entre 10h e 17h, com a opção desligada, molhe o sensor. Deve aparecer `chuva_bloqueada` e **nenhuma** `limpeza_inicio`.
5. **Segurança:** ligue uma bomba manualmente e espere. Deve aparecer `seguranca` em cerca de 60 s.
6. **Queda e recuperação:** desligue o roteador por 1 minuto e religue. Meça o tempo entre `wifi_off`, `wifi_on` e `nuvem_on`.
7. **Horário programado:** programe para daqui a 2 minutos e confira `limpeza_inicio` com motivo=horário no minuto certo.

> A memória guarda cerca de 600 KB, o que dá semanas de uso normal. Quando enche, os registros mais antigos saem primeiro. Antes de uma bateria de testes, use **Apagar registros** para começar com a planilha limpa. Na Arduino IDE, mantenha um *Partition Scheme* que tenha SPIFFS (o padrão tem).

## Segurança

- Senhas só no `segredos.h` (ESP32) e no navegador de quem faz login. O site publicado não contém nenhuma.
- Quem tiver o usuário `site` consegue ligar as bombas. Não compartilhe.
- Troque a senha `12345678` da rede LimpezaSolar no `.ino` (`AP_SENHA`).
- Se uma senha vazar, apague o usuário no EMQX e crie outro.

## Limites do plano grátis

A cota grátis do EMQX Serverless por mês é **1 milhão de minutos de sessão e 1 GB de tráfego**.
- **Minutos de sessão:** o ESP32 conectado o mês inteiro gasta cerca de 43.200. Cada pessoa com o site aberto gasta 1 por minuto.
- **Tráfego:** cada mensagem de estado tem cerca de 350 bytes. Parado, o ESP32 publica a cada 15 s (ou quando algo muda), e durante a limpeza a cada 1 s. Isso dá algo na faixa de 50 a 200 MB por mês.

As duas contas ficam bem abaixo do limite. O uso aparece na página do deployment, com cerca de 1 h de atraso.

---

## Para o TCC: conceitos usados

- **Publish/Subscribe (MQTT):** quem envia não precisa saber quem recebe. Dá para colocar um segundo celular, um painel ou um banco de dados só assinando os mesmos tópicos, sem mexer no ESP32.
- **Mensagem retida (retained):** o broker guarda a última mensagem de `status` e `online`. Quem abre o site recebe o estado na hora, sem esperar a próxima publicação.
- **LWT (Last Will and Testament):** ao conectar, o ESP32 deixa com o broker a mensagem `online = "0"`. Se ele sumir sem se despedir (falta de energia, Wi-Fi caiu), o próprio broker publica essa mensagem.
- **QoS 1:** os comandos são entregues *pelo menos uma vez*. Os comandos foram feitos para que repetir não cause problema: `b1on` duas vezes dá no mesmo, e `limpar` é ignorado se já estiver limpando.
- **Confirmação (ack):** cada comando do site leva um número (`id`), e o ESP32 devolve esse número no estado (`ack`). Assim o site sabe qual estado já inclui o comando e não "pisca" de volta.
- **TLS com certificado raiz:** o ESP32 confere o certificado do servidor contra a raiz DigiCert Global Root G2 gravada no firmware. Isso impede que alguém se passe pelo EMQX.
- **Mesmo protocolo em dois caminhos:** o site local (HTTP) e o site da internet (MQTT) mandam os mesmos textos de comando (`a=limpar`, `tm=10`...), tratados por uma única função no firmware (`aplicar()`).

Biblioteca do site: [MQTT.js](https://github.com/mqttjs/MQTT.js) 5.16.1 (licença MIT), incluída como `docs/mqtt.min.js` para o site não depender de CDN.
