# Checklist de Testes PIX — Asaas Mobile

> **Como usar este documento**: marque apenas as seções relacionadas à área que seu PR alterou. Esta checklist é referenciada pelo Pull Request Template (seção "Checklist de Testes PIX") e pela regra de revisão **FND-008** (`.claude/rules/review-rules-en.md` / `.amazonq/rules/review-rules-en.md`), que solicita o preenchimento sempre que o diff tocar a jornada de Pix.

## Pré-requisitos antes de testar

- [ ] Testar em pelo menos uma conta Pessoa Física (PF) e uma Pessoa Jurídica (PJ)
- [ ] Testar com conta com saldo suficiente e com saldo insuficiente/zerado
- [ ] Testar em conta com Pix já habilitado e em conta sem Pix habilitado (fluxo de adesão)
- [ ] Confirmar se a alteração depende de alguma feature flag / Remote Config e testar com o flag ligado e desligado
- [ ] Testar em sandbox e, quando aplicável, validar a jornada crítica também em produção
- [ ] Testar em Android e iOS — divergências de comportamento entre plataformas são comuns em fluxos com biometria/autenticação crítica

---

## 1. Introdução e onboarding do Pix

- [ ] Primeiro acesso à aba/seção Pix (carrossel de introdução)
- [ ] Conta ainda sem chave cadastrada — CTA de cadastro em destaque
- [ ] Fechar/pular a introdução e reabrir mais tarde (não deve travar em loop)

## 2. Chaves Pix (cadastro, portabilidade, exclusão)

- [ ] Cadastrar chave do tipo CPF/CNPJ (deve usar o documento do titular da conta)
- [ ] Cadastrar chave do tipo e-mail
- [ ] Cadastrar chave do tipo celular (com verificação de código/SMS)
- [ ] Cadastrar chave aleatória (EVP)
- [ ] Tentar cadastrar uma chave já existente em outra instituição → fluxo de portabilidade/reivindicação
- [ ] Tentar cadastrar chave já usada em outra conta Asaas
- [ ] Cadastrar o número máximo de chaves permitido e tentar cadastrar mais uma (deve bloquear com mensagem clara)
- [ ] Excluir uma chave existente e confirmar que some da listagem e do fluxo de recebimento
- [ ] **Portabilidade recebida** (reivindicação externa): outra instituição reivindica uma chave cadastrada no Asaas — aceitar e recusar
- [ ] **Portabilidade solicitada** (reivindicação Asaas): solicitar a portabilidade de uma chave de outra instituição para o Asaas — acompanhar status pendente/concluído/expirado
- [ ] Confirmação de posse da chave (etapa de segurança) com dado correto e incorreto
- [ ] Cancelar uma reivindicação em andamento
- [ ] Deixar uma reivindicação expirar (prazo do BACEN) e validar o estado exibido ao usuário

## 3. Cobrança via QR Code

- [ ] Gerar QR Code **estático** (sem valor fixo, reutilizável)
- [ ] Gerar QR Code **dinâmico** com valor definido
- [ ] Gerar QR Code dinâmico com vencimento/expiração
- [ ] Compartilhar o QR Code gerado (imagem, copia e cola)
- [ ] Visualizar um QR Code já gerado a partir do histórico
- [ ] Pagar lendo um QR Code pela câmera
- [ ] Pagar colando o código Pix copia e cola
- [ ] Pagar um QR Code expirado (deve bloquear com mensagem clara)
- [ ] Pagar um QR Code de valor já alterado/inválido
- [ ] Pagar QR Code que excede o limite disponível do usuário

## 4. Enviar Pix

- [ ] Enviar por chave (CPF/CNPJ, e-mail, celular, aleatória) — busca do favorecido e confirmação dos dados retornados
- [ ] Enviar por dados bancários (agência/conta), sem usar chave
- [ ] Tela de confirmação (checkout) exibindo corretamente favorecido, valor e mensagem/descrição opcional
- [ ] Cancelar o envio na tela de confirmação antes de concluir
- [ ] Enviar valor acima do saldo disponível
- [ ] Enviar valor acima do limite diurno/noturno configurado
- [ ] Agendar um Pix para data futura
- [ ] Cancelar um Pix agendado antes da execução
- [ ] Acompanhar o Pix agendado sendo executado na data prevista
- [ ] Autenticação crítica (biometria/PIN) sendo exigida antes da confirmação do envio
- [ ] Perda de conexão durante o envio — validar que o app não duplica a transação nem deixa o usuário em estado ambíguo

## 5. Pix Automático (recorrente)

- [ ] Onboarding do Pix Automático para quem ainda não usou
- [ ] Autorizar uma nova recorrência via QR Code
- [ ] Autorizar definindo o limite de valor por cobrança e periodicidade
- [ ] Recusar/cancelar uma solicitação de autorização
- [ ] Alterar o limite de uma recorrência já autorizada
- [ ] Cancelar uma recorrência ativa
- [ ] Tela de "aguardando retorno" enquanto a autorização é processada pelo banco/BACEN
- [ ] Selecionar entre múltiplas autorizações ativas (tela de seleção)
- [ ] Consultar o recibo/histórico de uma cobrança recorrente já paga
- [ ] Cobrança recorrente falha por saldo insuficiente na data — validar aviso ao usuário

## 6. Limites de Pix e dispositivo autorizado

- [ ] Consultar limites atuais (diurno, noturno, por transação)
- [ ] Alterar limite por período (ex.: reduzir limite noturno) e validar prazo de carência do BACEN até a alteração valer
- [ ] Alterar limite por conta bancária de destino
- [ ] Cadastrar um novo dispositivo autorizado
- [ ] Tentar transacionar acima do limite de dispositivo não autorizado (deve bloquear/pedir liberação)
- [ ] Solicitar desbloqueio de dispositivo autorizado
- [ ] Revogar/remover um dispositivo autorizado
- [ ] Validar o banner de aviso quando o app estiver operando com limites reduzidos (dispositivo novo/não confiável)

## 7. Extrato e transações Pix

- [ ] Consultar detalhe de uma transação recebida
- [ ] Consultar detalhe de uma transação enviada
- [ ] Consultar/baixar/compartilhar o recibo (comprovante) de uma transação
- [ ] Consultar transações de cobranças recorrentes (Pix Automático) no extrato
- [ ] Consultar Pix agendados e seu status (pendente, executado, cancelado)
- [ ] Filtrar/buscar transações Pix no extrato geral

## 8. Devolução de valores (MED)

- [ ] Abrir uma solicitação de devolução (Mecanismo Especial de Devolução) a partir de uma transação recebida
- [ ] Selecionar o motivo da devolução (ex.: fraude, coação, erro operacional)
- [ ] Validar o carrossel/tela de introdução explicando o mecanismo antes da abertura
- [ ] Acompanhar o status da solicitação (em análise, aceita, recusada)
- [ ] Consultar o detalhe de uma devolução já concluída
- [ ] Tentar abrir uma solicitação fora do prazo permitido pelo BACEN (deve bloquear com mensagem explicativa)
- [ ] Validar o feedback exibido quando a devolução é parcial vs. total

## 9. Infração / disputa

- [ ] Reportar uma infração sobre uma transação (ex.: suspeita de fraude)
- [ ] Selecionar o motivo do relato na tela de seleção
- [ ] Consultar o detalhe/status de um relato já enviado
- [ ] Receber e responder a um relato de infração aberto por outra parte, se aplicável no app

## 10. Feedback assíncrono e estados intermediários

- [ ] Tela de carregamento/aguardando confirmação durante o processamento de um envio ou de uma autorização
- [ ] Fechar o app durante um processamento assíncrono e reabrir — validar que o status é retomado corretamente
- [ ] Transação que fica "em análise" (débito em análise) — validar o estado exibido e a atualização quando o resultado chega
- [ ] Timeout do backend durante o processamento — validar mensagem de erro e possibilidade de nova tentativa

## 11. Cenários transversais (aplicar a qualquer fluxo de Pix alterado)

- [ ] Estados de erro de rede (sem internet, API indisponível, timeout)
- [ ] Acessibilidade: leitor de tela (VoiceOver/TalkBack) e fonte do sistema ampliada
- [ ] Regressão: fluxos vizinhos não alterados pelo PR continuam funcionando (ex.: alterar envio não deve quebrar QR Code)
- [ ] Tracking: eventos de analytics da jornada alterada disparam corretamente (ver `Features/Pix/Trackers`)

---

## Histórico

| Data | Alteração | Ticket |
| --- | --- | --- |
| 2026-08-26 | Criação da checklist inicial | [MBT-4429](https://asaasdev.atlassian.net/browse/MBT-4429) |
