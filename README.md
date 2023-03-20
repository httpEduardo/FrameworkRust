#Framework para alocar memória em módulos #!

Requisitos Rust 1.6

Atualmente, não há uma maneira padrão de alocar memória em um módulo que seja #![no_std]. Este framework fornece um mecanismo para descrever uma alocação de memória que pode ser atendida totalmente na pilha, por meio de um link inseguro para o calloc, ou por meio de uma referência insegura a uma variável global mutável. Esta biblioteca atualmente vazará memória se free_cell não for especificamente invocado na memória.

No entanto, se for vinculado por uma biblioteca que realmente pode depender da stdlib, essa biblioteca pode simplesmente passar alguns alocadores e usar a alocação padrão Box e liberar automaticamente.

Essa biblioteca também deve tornar possível isolar completamente um aplicativo Rust que precisa de alocações dinâmicas, pré-alocando um limite máximo de dados antecipadamente usando calloc e usando seccomp para impedir futuras chamadas de sistema.

Uso Existem 3 modos para alocar memória, cada um com vantagens e desvantagens.

Na pilha Isso é possível sem a stdlib. No entanto, isso reduz o limite natural de ulimit na profundidade da pilha e geralmente limita o programa a apenas alguns megabytes de dados alocados dinamicamente.
