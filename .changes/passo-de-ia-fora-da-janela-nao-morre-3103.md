---
impacto: nada_mudou
secao: corrigido
titulo: Follow-up escrito pela IA fora do horário de envio espera a janela em vez de morrer
---
Quando um passo de follow-up em que a própria IA escreve a mensagem caía fora do horário de envio do número (por exemplo, à noite com envio das 9h às 18h), o envio era guardado para a abertura, mas o follow-up não ficava sabendo da espera. Depois de umas 11 horas ele considerava o passo travado e encerrava a inscrição antes de a janela abrir, e o contato nunca recebia a mensagem. Agora o follow-up registra a espera e envia quando a janela abre. Nada a fazer na atualização; inscrições já encerradas por isso não voltam sozinhas.

Medido em produção e corrigido por @RenanzitoIA (#3103).
