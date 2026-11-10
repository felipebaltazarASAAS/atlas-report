# Como usar: build via Azure Pipelines

Dispara o build do app no Azure DevOps pelo GitHub Actions e acompanha até o fim.

## Como rodar

Actions → **Azure Pipeline Build** → **Run workflow**.

| Input | Valores | Efeito |
|---|---|---|
| `branch` | nome simples da branch (padrão `release`) | Branch que o Azure vai buildar |
| `android` | checkbox | Builda o Android |
| `ios` | checkbox | Builda o iOS |

Marque pelo menos uma plataforma, senão o workflow falha logo no início. Marcando as duas, os builds rodam em paralelo, um job por plataforma.

## O que você vê

- A posição na fila enquanto o build espera por um agente.
- As etapas ao vivo, conforme começam e terminam, e o resultado de cada stage/job ao terminar.
- Um resumo no fim do job.
- Se o build falhar: as mensagens de erro e o final do log dos steps que falharam.

## Onde o build chega (App Tester)

Quando o pipeline termina, o DevOps envia o app para o **App Tester** do Firebase (Firebase App Distribution), grupo `Testers`. Quem está no grupo recebe o build e instala por lá.

## Cancelar

Cancelar o workflow cancela também o build no Azure. O mesmo acontece se o acompanhamento estourar o tempo limite (cerca de 83 min) ou perder a conexão de forma persistente com o Azure.

O cancelamento é melhor esforço: se a conexão com o Azure estiver totalmente fora, o pedido pode falhar e o build continua no Azure.

## Resultado do job

| Resultado do job | Significado |
|---|---|
| Sucesso | O build terminou com `succeeded` |
| Falha | O build falhou, terminou com sucesso parcial ou foi cancelado, houve erro de HTTP (PAT inválido, branch inexistente) ou o tempo limite estourou |
