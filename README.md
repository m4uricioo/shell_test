# shell_test
testing make own shell 
Componentes de un shell de Linux
El shell es una pieza compleja de software que contiene muchas partes diferentes.

La parte central de cualquier shell de Linux es el intérprete de línea de comandos o CLI . Esta parte tiene dos propósitos: lee y analiza los comandos del usuario, luego ejecuta los comandos analizados. Puede pensar que la CLI en sí misma tiene dos partes: un analizador (o front-end) y un ejecutor (o back-end).

El analizador analiza la entrada y la divide en tokens. Un token consta de uno o más caracteres (letras, dígitos, símbolos) y representa una sola unidad de entrada. Por ejemplo, un token puede ser un nombre de variable, una palabra clave, un número o un operador aritmético.

El analizador toma estos tokens, los agrupa y crea una estructura especial que llamamos Árbol de sintaxis abstracta , o AST . Puede pensar en el AST como una representación de alto nivel de la línea de comando que le dio al shell. El analizador toma el AST y lo pasa al ejecutor , que lee el AST y ejecuta el comando analizado.

Otra parte del shell es la interfaz de usuario, que normalmente funciona cuando el shell está en modo interactivo , por ejemplo, cuando ingresa comandos en el indicador del shell. Aquí, el shell se ejecuta en un bucle, que conocemos como Read-Eval-Print-Loop o REPL.

Como indica el nombre del ciclo, el shell lee la entrada, la analiza y la ejecuta, luego realiza un ciclo para leer el siguiente comando, y así sucesivamente hasta que ingrese un comando comoexit , shutdown, oreboot.

La mayoría de los shells implementan una estructura conocida como tabla de símbolos , que el shell utiliza para almacenar información sobre las variables, junto con sus valores y atributos. Implementaremos la tabla de símbolos en la parte II de este tutorial.

Los shells de Linux también tienen una función de historial, que permite al usuario acceder a los comandos ingresados ​​más recientemente, luego editar y volver a ejecutar los comandos sin escribir mucho. Un shell también puede contener utilidades integradas , que son un conjunto especial de comandos que se implementan como parte del propio programa shell.

Las utilidades integradas incluyen comandos de uso común, comocd, fg, ybg. Implementaremos muchas de las utilidades integradas a medida que avanzamos en este tutorial.

Ahora que conocemos los componentes básicos de un shell típico de Linux, comencemos a construir nuestro propio shell.

Nuestra primera concha
Nuestra primera versión del caparazón no hará nada elegante; simplemente imprimirá una cadena de solicitud, leerá una línea de entrada y luego repetirá la entrada en la pantalla. 

Lo primero que haremos será escribir nuestro bucle REPL básico. Crear un archivo llamado main.c

#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include "shell.h"

int main(int argc, char **argv)
{
    char *cmd;

    do
    {
        print_prompt1();

        cmd = read_cmd();

        if(!cmd)
        {
            exit(EXIT_SUCCESS);
        }

        if(cmd[0] == '\0' || strcmp(cmd, "\n") == 0)
        {
            free(cmd);
            continue;
        }

        if(strcmp(cmd, "exit\n") == 0)
        {
            free(cmd);
            break;
        }

        printf("%s\n", cmd);

        free(cmd);

    } while(1);

    exit(EXIT_SUCCESS);
}


Nuestro main()La función necesita implementar el bucle REPL. Primero imprimimos el indicador del shell, luego leemos un comando (por ahora, definamos un comando como una línea de entrada que termina con\n). Si hay un error al leer el comando, salimos del shell. Si el comando está vacío (es decir, el usuario presionóENTERsin escribir nada), salteamos esta entrada y continuamos con el bucle.

Si el comando esexit, salimos del shell. De lo contrario, hacemos eco del comando, liberamos la memoria que usamos para almacenar el comando y continuamos con el bucle. 

Nuestromain()función llama a dos funciones personalizadas,print_prompt1()yread_cmd(). La primera función imprime la cadena de solicitud y la segunda lee la siguiente línea de entrada. Echemos un vistazo más de cerca a esas dos funciones.

Impresión de cadenas de solicitud
Dijimos que el shell imprime una cadena de solicitud antes de leer cada comando. De hecho, hay cinco tipos diferentes de cadena de solicitud: PS0 , PS1 , PS2 , PS3 y PS4 . La cadena cero, PS0 , solo la usa bash , por lo que no la consideraremos aquí. Las otras cuatro cadenas se imprimen en ciertos momentos, cuando el shell quiere transmitir ciertos mensajes al usuario.

En esta sección, hablaremos de PS1 y PS2 . El resto vendrá más adelante cuando discutamos temas de shell más avanzados.

Ahora crea el archivo fuenteprompt.ce ingrese el siguiente código:

#include <stdio.h>
#include "shell.h"

void print_prompt1(void)
{
    fprintf(stderr, "$ ");
}

void print_prompt2(void)
{
    fprintf(stderr, "> ");
}
La primera función imprime la primera cadena de solicitud, o PS1 , que generalmente ve cuando el shell está esperando que ingrese un comando. La segunda función imprime la segunda cadena de solicitud, o PS2 , que imprime el shell cuando ingresa un comando de varias líneas (más sobre esto a continuación).

A continuación, leamos algunas entradas de los usuarios.

Lectura de la entrada del usuario
Abre el archivomain.ce ingrese el siguiente código al final, justo después delmain()función:

char *read_cmd(void)
{
    char buf[1024];
    char *ptr = NULL;
    char ptrlen = 0;

    while(fgets(buf, 1024, stdin))
    {
        int buflen = strlen(buf);

        if(!ptr)
        {
            ptr = malloc(buflen+1);
        }
        else
        {
            char *ptr2 = realloc(ptr, ptrlen+buflen+1);

            if(ptr2)
            {
                ptr = ptr2;
            }
            else
            {
                free(ptr);
                ptr = NULL;
            }
        }

        if(!ptr)
        {
            fprintf(stderr, "error: failed to alloc buffer: %s\n", strerror(errno));
            return NULL;
        }

        strcpy(ptr+ptrlen, buf);

        if(buf[buflen-1] == '\n')
        {
            if(buflen == 1 || buf[buflen-2] != '\\')
            {
                return ptr;
            }

            ptr[ptrlen+buflen-2] = '\0';
            buflen -= 2;
            print_prompt2();
        }

        ptrlen += buflen;
    }

    return ptr;
}
Aquí leemos la entrada de stdin en fragmentos de 1024 bytes y almacenamos la entrada en un búfer. La primera vez que leemos la entrada (el primer fragmento del comando actual), creamos nuestro búfer usandomalloc(). Para fragmentos posteriores, ampliamos el búfer usandorealloc(). No deberíamos encontrar ningún problema de memoria aquí, pero si sucede algo incorrecto, imprimimos un mensaje de error y devolvemosNULL. Si todo va bien, copiamos el fragmento de entrada que acabamos de leer del usuario a nuestro búfer y ajustamos nuestros punteros en consecuencia.

El último bloque de código es interesante. Para entender por qué necesitamos este bloque de código, consideremos el siguiente ejemplo. Digamos que desea ingresar una línea de entrada muy, muy larga:

echo "This is a very long line of input, one that needs to span two, three, or perhaps even more lines of input, so that we can feed it to the shell"
Este es un ejemplo tonto, pero demuestra perfectamente de lo que estamos hablando. Para ingresar un comando tan largo, podemos escribir todo en una línea (como hicimos aquí), lo cual es un proceso engorroso y feo. O podemos cortar la línea en pedazos más pequeños y alimentar esos pedazos al caparazón, una pieza a la vez:

echo "This is a very long line of input, \
      one that needs to span two, three, \
      or perhaps even more lines of input, \
      so that we can feed it to the shell"
Después de escribir la primera línea, y para que el shell sepa que no terminamos nuestra entrada, terminamos cada línea con un carácter de barra invertida.\\, seguido de nueva línea (también puse sangría en las líneas para que fueran más legibles). A esto lo llamamos escapar del carácter de nueva línea. Cuando el shell ve la nueva línea escapada, sabe que necesita descartar los dos caracteres y continuar leyendo la entrada.

Ahora volvamos a nuestroread_cmd()función. Estábamos discutiendo el último bloque de código, el que dice:

        if(buf[buflen-1] == '\n')
        {
            if(buflen == 1 || buf[buflen-2] != '\\')
            {
                return ptr;
            }

            ptr[ptrlen+buflen-2] = '\0';
            buflen -= 2;
            print_prompt2();
        }
Aquí, verificamos si la entrada que tenemos en el búfer termina con\ny, de ser así, si el\nes escapado por un carácter de barra invertida\\. si el ultimo\nno se escapa, la línea de entrada está completa y la devolvemos a lamain()función. De lo contrario, eliminamos los dos caracteres (\\y\n), imprima PS2 y continúe leyendo la entrada.

Compilando el Shell
Con el código anterior, nuestro shell de nicho está casi listo para compilarse. Solo agregaremos un archivo de encabezado con nuestros prototipos de funciones, antes de proceder a compilar el shell. Este paso es opcional, pero mejora en gran medida la legibilidad de nuestro código y evita algunas advertencias del compilador.

Crear el archivo fuenteshell.h, e ingrese el siguiente código:

#ifndef SHELL_H
#define SHELL_H

void print_prompt1(void);
void print_prompt2(void);

char *read_cmd(void);

#endif
Ahora vamos a compilar el shell. Abra su emulador de terminal favorito (yo pruebo mis proyectos de línea de comandos usando GNOME Terminal y Konsole , pero también puede usar XTerm , otros emuladores de terminal o una de las consolas virtuales de Linux ). Navegue a su directorio de origen y asegúrese de tener 3 archivos allí:



imagen

Ahora compile el shell usando el siguiente comando:

gcc -o shell main.c prompt.c




imagen

Ahora invoque el shell ejecutando./shelle intente ingresar algunos comandos:



imagen

En el primer caso, el shell imprime PS1 , que por defecto es$y un espacio Entramos en nuestro comando,echo Hello World, que el shell nos devuelve (extenderemos nuestro shell en la parte II para permitirle analizar y ejecutar este y otros comandos simples).

En el segundo caso, el shell vuelve a hacer eco de nuestro comando (ligeramente largo). En el tercer caso, dividimos el comando largo en 4 líneas. Observe cómo cada vez que escribimos una barra invertida seguida deENTER, el shell imprime PS2 y continúa leyendo la entrada. Después de ingresar la última línea, el shell fusiona todas las líneas, elimina todos los caracteres de nueva línea escapados y nos devuelve el comando.

Para salir del shell, escribaexit, seguido porENTER:



imagen

¡Y eso es! Acabamos de terminar de escribir nuestro primer shell de Linux.
