---
impacto: capacidade_nova
secao: adicionado
titulo: O agente consulta os produtos de uma loja WooCommerce pelo banco externo
---

Em Integração de dados, uma conexão MySQL com uma loja WooCommerce ganha a receita pronta "Produtos da loja WooCommerce". O agente passa a responder no WhatsApp com nome, preço, estoque e link dos produtos, sem que ninguém escreva SQL. Ele separa cada item pedido pelo cliente ("camisetas e sorvetes" vira dois termos), aceita plural, busca também no resumo do produto, filtra por preço máximo e por "só em estoque", e pode trazer primeiro os mais recentes. A receita só lê as tabelas que o administrador liberou na conexão e respeita o limite de linhas dela. Fica desligada até alguém adicioná-la à conexão; nada muda para quem não usa o banco externo.
