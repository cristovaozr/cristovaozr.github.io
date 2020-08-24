---
layout: post
title: Comandos que uso no GDB
---

# Introdução

Como primeira postagem quero falar dos comandos que uso para fazer debugging de firmware. Não costumo usar interface gráfica para fazer debugging. Acho que na época que tentei usar não deu muito certo e desenvolvi um certo trauma.

Antes de começar algumas observações aqui:
* Eu utilizo (e gosto bastante) de microcontroladores da ST. Tenho vários deles. Em virtude disso utilizo o software [stlink](https://github.com/stlink-org/stlink) originalmente desenvolvido por `texane` para fazer debugging dos projetos;
* Usei pouco o `openocd` portanto não utilizarei aqui. Meu foco é o `stlink`;
* Os comandos utilizados aqui se aplicam (em sua maioria) a qualquer outro microcontrolador e até mesmo em debugging de software no PC;
* O stlink só se aplica a microcontroladores da ST portanto nada de utilizá-lo em outro fabricante. Talvez seja possível usar outra interface que nos dê algo para usar o GDB;
* Como utilizo Linux para fazer meus projetos (pra ser mais sincero não tenho nenhum PC com Windows em casa...) não conheço peculiaridades de instalação do GDB e etc. no Windows ou MacOS.

## Compilando em preparação para o debugging

É extremamente importante compilar o código-fonte sem otimização e com símbolos de debug. Em outras palavras: use as flags `-O0` para indicar "sem otimização" (para mais informações veja [aqui](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html#Optimize-Options)) e `-ggdb` para colocar no ELF os símbolos utilizados pelo GDB (para mais informações veja [aqui](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html#Debugging-Options)).

Uma vez compiado com essas flags (talvez tenha que modificar o Makefile ou coisa assim) estaremos prontos! Nos meus projetos costumo colocar uma flag para indicar que o build é feito em modo *release*. Quando isso acontece eu removo os símbolos de debug e coloco otimização por tamanho `-Os`. Isso posso discutir em outra postagem no futuro.

Só a título de curiosidade um *firwmare* compilado com otimização de tamanho chega a ficar consideravelmente menor que aquele fazendo sem otimização. Nunca cheguei a usar outras otimizações como `-O1` e `-O2` em firmware. Posso experimentar isso mais tarde.

## Inicializando o stlink

Suponho que você já tenha o `stlink` instalado e funcionando no seu computador. Para iniciar uma sessão de debugging será necessário executar o `st-util`:
```bash
$ st-util -n
```

Invocar o `st-util` com a flag `-n` diz para não resetar o microcontrolador ao iniciar a sessão. Desse modo você começa a debugar exatamente onde o microcontrolador foi parado pelo debugger. Caso deseje-se reiniciar o microcontrolador ao iniciar a sessão de debugging basta não especificar essa flag.

Por padrão o `st-utl` cria um *GDB server* na porta `4242`. Pode-se mudar essa porta com a flag `-p` indicando logo em seguida o número da porta. Só tive que mudar isso uma vez, quando um software que tinha em meu PC usava essa porta pra não-lembro-o-que.

Feito isso estamos prontos para começar uma sessão do GDB!

# Iniciando o debug

A versão do GDB que utilizo é chamado na minha distribuição GNU/Linux por `arm-none-eabi-gdb`. Em outras distribuições (já vi a Canonical fazer isso com o GDB) pode ser chamado de outra maneira. De qualquer maneira para iniciar invoca-se o GDB passando o ELF original de compilação:
```bash
$ arm-none-eabi-gdb /path/to/your/file/project.elf
```

Desse modo o GDB deve subir com uma mensagem mais ou menos assim:
```
GNU gdb (GDB) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /path/to/your/file/project.elf...
(gdb) 
```

Para conectar com o `st-util` que já deve estar executando basta colocar o comando:
```
(gdb) target extended-remote :4242
```

Esse `4242` é exatamente a porta que o `st-util` foi instruído a usar. Quando isso acontecer o microcontrolador para de executar e aparecerá no *prompt* do GDB o local exato onde o microcontrolador foi parado. Para contunar a execução pode-se enviar o comando `continue` seguido de *enter* ou simplesmente `c` também seguido de *enter*.

Para parar o microcontrolador e voltar ao controle do GDB basta apertar `Ctrl + C` a qualquer momento. Para sair do GDB basta digitar `exit` ou `Ctrl + D` no Linux que é essencialmente a mesma coisa.

## Definindo breakpoints

Uma das coisas mais "bacanas" de debugging é a possibilidade de instruir o debugger a parar a execução da CPU quando o programa chamar alguma função ou chegar em alguma linha de código. Isso se consegue usando o comando `break` no GDB:
```
(gdb) break function_name
```
ou ainda
```
(gdb) break main.c:75
```

No primeiro exemplo o GDB parará a execução quando a função `function_name()` for executada. No segundo exemplo o breakpoint está no arquivo `main.c` na linha 75. Cuidado ao usar breakpoints com linhas para colocar em algo que seja realmente executável. Normalmente o GDB não aceita linhas com comentários ou vazias: ele estima a próxima linha executável e coloca o breakpoint lá.

Assim como o comando `continue` pode-se usar uma abreviação para a inserção de breakpoints:
```
(gdb) b summation_function
```
parando a execução quando a função `summation_function` for executada.

### Uma observação sobre breakpoints

O número de breakpoints varia de microcontrolador para microcontrolador. Os que eu utilizo costumam ter uns quatro breakpoints em hardware (*hardware breakpoints*). Nunca excedi esse número para ver o que acontece. Talvez teste isso um dia.

## Visualizando e removendo breakpoints

Para visualizar breakpoints basta usar o comando `info`:
```
(gdb) info breakpoints
```

O comando exbibirá todos os breakpoints configurados. Um exemplo segue:
```
(gdb) b main
Breakpoint 1 at 0x8000340: file core/src/main.c, line 21.
(gdb) b USART_IRQHandler 
Breakpoint 2 at 0x8006804: file dal/stm_impl/usart_driver.c, line 149.
(gdb) info breakpoints 
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x08000340 in main at core/src/main.c:21
2       breakpoint     keep y   0x08006804 in USART_IRQHandler 
                                           at dal/stm_impl/usart_driver_impl.c:149
```

Também existe um atalho para `info breakpoints`: `i b`

Para remover breakpoints basta fazer:
```
(gdb) delete 1
```
Remove o breakpoint listado como "1" (no caso do exemplo, o breakpoint em `main()`). (OBS: o número dos breakpoints nunca são reaproveitados. Uma vez removido pode-se configurá-lo novamente mas o número será outro)

Se quiser remover **todos** os breakpoints basta fazer:
```
(gdb) delete breakpoints
(gdb) d breakpoints
```

## Colocando breakpoints condicionais

Sim! É possível colocar um breakpoint para ativar em uma condição específica! E isso é **OTIMO**!

Imagine a seguinte situação: você tem um *for-loop* que conta de 0 até 999. Você deseja parar quando o contador for 750 (por exemplo). A melhor maneira de fazer isso é usando um breakpoint condicional.

Para fazer isso basta colocar a condição após o breakpoint. Exemplo:
```
(gdb) b my_source.c:75 if counter == 750
```

O condicional é feito como um código em C

## Navegando o código

É possível, depois de parado, acompanhar a execução do firmware. Para tal utilizam-se os seguintes comandos:

* `next` ou `n` para ir para a próxima linha de código. As chamadas de função são executadas sem entrar na função;
* `step` ou `s` para prosseguir uma etapa de execução. As chamadas de função são executadas e entra-se na função;
* `stepi` ou `si` para prosseguir para a próxima instrução. Para se ver a instrução sendo executada pode-se usar o comando `disassemble`. A instrução é apontada com uma seta `=>`;
* `finish` ou `f` para finalizar a execução de uma função. Exemplo: suponha que você deu um step para uma função. Se desejar sair dela sem precisar fazer o processo completo basta digitar `finish` que o debugger volta a parar a execução assim que a função finalizar.

## Lendo valores de variáveis

Primeira coisa a se lembrar: as variáveis dependem de contexto: portanto se uma variável só existe em uma função ou até mesmo em um bloco de código (um laço *for*, um *if*, etc.) ela só é legível dentro dessa região a menos que uma variável seja global (ou `static` em um arquivo de código-fonte).

Para imprimir o valor de uma variável basta usar o comando `print`:
```
(gdb) print my_var
```

Se quiser imprimir o valor em hexadecimal basta fazer:
```
(gdb) print/x my_var
```

Se uma variável for um ponteiro (exceto ponteiro do tipo `void *`) ele pode ser deferenciado como se faz em C:
```
(gdb) print *my_non_void_pointer
```

Estruturas podem ser impressas da mesma maneira. Os campos dessa estrutura serão impressas assim como os valores.

Uma coisa bacana do GDB é a possibilidade de fazer *cast* na hora de imprimir. Imagine um código que possui um ponteiro `void *` mas que você sabe com certeza que é um ponteiro do tipo `int *`. Para fazer o cast basta:
```
(gdb) print *(int *)my_void_pointer
```
e voilá! O valor representado em formato `int` será impresso. Pode-se fazer isso para qualquer coisa em C.

### Periféricos do ARM

Como os Cortex-M são *cores* ARM eles tem os periféricos mapeados em RAM a partir do endereço `0x40000000`. Para ver onde os periféricos estão basta ler o manual. De qualquer forma usando o CMSIS é possível usar estruturas e a região de memória para ver a configuração dos periféricos de um dado microcontrolador.

O seguinte exemplo mostra o conteúdo dos registradores da USART2 de um microcontrolador `STM32F103`:
```
(gdb) print *(USART_TypeDef *)0x40004400
```

## Escrevendo valores em tempo de execução

Também é possível escrever valores em variáveis. Os princípios se aplicam a tudo que foi dito de impressão (ou seja: estruturas, variáveis, ponteiros, etc.).

Para escrever um valor específico em uma variável que está no contexto basta:
```
(gdb) set my_variable = new_value
```
e voilá! O novo valor estará disponível para ser utilizado

## Imprimindo a pilha de chamadas de função

A pilha pode nos dar bastante informação de *onde* a execução está. Para imprimir a pilha de chamadas de função (também conhecida como *stacktrace*) basta chamar o comando `backtrace` ou sua abreviação `bt`:
```
(gdb) backtrace
```

## Descompilando funções

Para descompilar uma função (ou o código onde estamos agora) basta usar o comando `disassemble`:
```
(gdb) disassemble main
(gdb) disassemble
```

No primeiro caso a função `main` vai ser descompilada. No segundo caso a descompilação será do trecho em que estamos executando agora.

## Imprimindo informações dos registradores

Também é possível ver o estado dos registradores:
```
(gdb) info registers
(gdb) i r
```

Com esse comando todo o estado dos registradores é impresso em tela.

# Conclusão

Esses são os comandos que mais uso ao fazer debugging de firmware. Se houver alguma sugestão, correção ou perguntas basta me contactar!