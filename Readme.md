# Nautilus Shell

Nautilus es un shell de Unix escrito en C que permite a los usuarios interactuar con el sistema operativo mediante comandos. Incluye varios features que son comunmente encontrados en los shells de Unix, como redireccion de entrada y salida y multituberia. Este proyecto forma parte de nuestro curso de Sistemas Operativos en 3er año de Ciencias de la Computación

## Indice

- [Colaboradores](#colaboradores)
- [Features](#features)
- [Uso](#uso)
- [Detalles Tecnicos](#detalles-técnicos)
  - [Basic](#basico)
  - [Ctrl+C](#ctrlc)
  - [History](#history)
  - [Multipipe](#multipipe)

## Colaboradores

- Hector Miguel Rodriguez Sosa - Grupo C311
- Sebastian Suarez Gomez - Grupo C311

## Features
- Redireccion de entrada desde un archivo en vez del teclado (<)
- Redireccion de salida a un archivo en vez de la consola (>)
- Redireccion de la salida a un archivo pero agregando al contenido en vez de remplazandolo (>>)
- Multituberias que permiten enlazar varios comandos tomando la salida de un comando como la entrada del siguiente (|)
- cd: permite al usuario cambiar el directorio de trabajo
- help: permite al usuario conocer mas sobre las funcionalidades del shell
- exit: termina el shell
- history: permite ver los 10 ultimos comandos que se uso en pantalla
- again: dado un indice 0-9 ejecuta el comando indexado del history
- Ctrl+C: mata el proceso actual usando signals

## Uso
Para usar nautilus simplemente clone el repositorio, compile el archivo main.c y use el ejecutable creado en la carpeta raiz. Un ejemplo de como hacerlo usando GCC es:

```bash
gcc -o main main.c && ./main
```

El shell mostrara un prompt del estilo:

```c
nautilus $ 
```

y ahi debe escribir sus comandos. Para informacion mas detallada escriba help

## Detalles técnicos

### Basico

La lectura del comando se hace en el metodo ntl_loop. El cual imprime el prompt de la shell mientras q el comando que entremos sea vacio o cuando termine un commando. Luego se pasa al metodo ntl_execute **(L532)**, el cual su funcion es manejar algun que otro caso extremo y por encima de todo separar la linea que se leyo en comandos y separadores. Todo eso para que el metodo ntl_parsing **(L391)** vaya por cada uno de los comandos y separadores y dependencia de cuales separadores rodean a un comando los ejecuta.

Se puede ejecutar cualquier comando que se le pase usando la funcion execvp(...)
Esto se puede ver con la funcion ntl_launch **(L337)**. Lo que hace es crear un fork del commando principal, el shell y pasarselo a la funcion execvp. Si esta devuelve un error entonces ese mismo error se devuelve para atras

El resto de argumentos de la funcion ntl_launch son usados en el piping y en la redireccion de entrada y salida del comando.
En cuanto a opciones se pueden recibir 6 opciones:
0. El comando es raso (salida y entrada normal)
1. La salida debe ser a un archivo (operador >)
2. La entrada debe ser de un archivo (operador <)
3. La salida se le debe acoplar al final del archivo (operador >>)
5. La entrada y salida son de archivos (imaginese que sea command1 < file1 > file2)
7. Lo mismo que la 5 pero con un append (command1 < file1 >> file2)

El piping se explica en help multi-pipe

### Ctrl+C

Nuestro shell permite que cuando un comando se esté ejecutando se le pueda enviar un Ctrl+C.

Para implementar eso, tuvimos que primero crear un manejador de señarles (sig_handler) que captura cualquier señal SIGINT que venga. Luego la manda al padre del proceso (el shell) y le dice que le de una advertencia al hijo mandandole la misma señal SIGINT. Si este decide ignorarla por cualquier razon se queda guardada como variable que ya se mando un sigint. Si se vuelve a mandar otro sigint, entonces se manda a matar completo usando un SIGTERM

El sighandler se asigna en el metodo que ademas hace otras funciones como asegurarse que esta corriendo en el foreground o poner al proceso del shell como el padre de todos los procesos. Es el metodo en **(L24)**.

En concreto el sighandler se asigna en **(L32)** y **(L33)** y el metodo del sighandler esta en **(L56)**.

El programa tiene un pequeño bug que es que cuando se manda un SIGINT, si este mata al proceso entonces cuando sale el prompt de nuevo sale doble pero es algo minimo lo que escurridizo. Tambien parece que esta asociado a lo mismo es que los metodos builtin se mandan a matar ponen un prompt por cada uno q haya abierto.


### History

Se implementaron los builtin history y again. El builtin history **(L184)** lo que hace es leer de un archivo ya creado sus contenidos. En este archivo están los 10 ultimos comandos.

Estos comandos se guardan una vez leida la línea usando append_to_history **(L215)**. Como solo puedo guardar a lo sumo los 10 ultimos comandos en el history, este metodo de lo que se encarga es de mantener el archivo como si fuera una cola circular. Si llega el momento de que hay 10 elementos al momento de añadir en la cola entonces se popea el fondo y se pushea el nuevo comando.

Luego está el builtin again **(L289)** el cual si el usuario usa como comando again {índice}, busca en el archivo history y manda a ntl_execute() lo que se encuentra en el índice. Luego guarda ese comando en el historial de nuevo.

### Multipipe

Para empezar, aclarar que no se usa la funcion pipe() para implementar la tuberia. Lo que se usan son archivos de buffer que sirven tanto para enviar la salida de un comando y recibir la entrada del otro.

Aclarar q esto funciona bien como un piping normal. En terminos sencillos. Cuando se hace command1 | command2 como en realidad lo esta interpretando es como command1 > buffer_file1 ; command2 < buffer_file1. Al final siempre se elimina el archivo de buffer.

Esto hace la implementacion comoda ya que a la hora de ejecutar el comando simplemente estoy haciendo las mimsmas funciones ya implementadas para los operadores > y <. Es similar a la implementacion normal de la funcion pipe(), con la diferencia de que el file descriptor es un archivo real dentro de la carpeta (efimero).

Para implementar el piping se empieza en **(L458)**. Cuando se detecta que el siguiente operador es una tuberia entonces entra en un "modo tuberia" en el cual pueden pasar 3 cosas: es un comando al principio, es un comando en el medio o es el ultimo comando.
1. El comando del principio simplemente se le manda a ntl_launch **(L337)** con opcion 1 de escribir en el buffer file que toca.
2. El comando del final es con opcion 2 de leer el buffer file que toca.
3. Si esta en el medio ya es mas turbio. Como mismo pasa en la funcion pipe(), si intentamos leer y escribir del mismo archivo nos da error. Asi que tengo 2 buffer file. Estos se intercalan de forma que si 1 esta leyendo en este momento entonces 2 esta escribiendo. Cada vez que itero de nuevo estos se deben intercalar ya que si en el anterior escribi en 1, lo que quiero leer esta alli.

