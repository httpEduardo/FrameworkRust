# Framework para Alocação de Memória em Módulos Rust

- Requisitos: Rust 1.6

## Descrição:
Este framework fornece uma solução para a alocação de memória em módulos #![no_std], onde atualmente não existe um método padrão. Ele permite descrever alocações de memória que podem ser realizadas completamente na pilha, através de um link inseguro para calloc, ou por meio de uma referência insegura a uma variável global mutável. É importante notar que a memória pode vazar se free_cell não for explicitamente chamado.

## Integração com stdlib:
Quando vinculado a uma biblioteca que depende da stdlib, esta pode passar alocadores e utilizar a alocação padrão Box, permitindo liberação automática de memória.

## Isolamento de Aplicativos:
A biblioteca também visa isolar aplicações Rust que requerem alocações dinâmicas. Isso é feito pré-alocando um limite máximo de dados antecipadamente usando calloc e empregando seccomp para prevenir futuras chamadas de sistema.

## Na Pilha: Viável sem a stdlib. No entanto, isso impacta o limite natural de ulimit na profundidade da pilha e geralmente restringe o programa a apenas alguns megabytes de dados alocados dinamicamente.
