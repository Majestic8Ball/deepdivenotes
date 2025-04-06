>Using this repo: https://github.com/codecrafters-io/build-your-own-x
>I found and created using this guide: https://brennan.io/2015/01/16/write-a-shell-in-c/
>Essentially this is just so I can have my own version of this and add my own notes and comments all for my own reading and learning purposes.
## Life time of a shell (basic)
---
- **Initialize:** Typical shells read and execute configuration files.
- **Interpret:** Shell reads commands from stdin and executes them.
	- This means, when it is opened (by opening terminal or other initiates), if you type a command or feed one it. It just reads it and interprets it.
- **Terminate:** After execution, the shell executes shutdown commands, frees up memory, and terminates
## Basic loop of a shell
---
- **Read:** Read command from standard input.
- **Parse:** Separate the command string into a program and arguments.
- **Execute:** Run the parsed command.

```C
void lsh_loop(void)
{
  char *line;
  char **args;
  int status;

  do {
    printf("> ");
    line = lsh_read_line();
    args = lsh_split_line(line);
    status = lsh_execute(args);

    free(line);
    free(args);
  } while (status);
}
```

## Reading a line in C
---
>You can't just determine a memory block and hope the user doesn't exceed it. So, we have to make it so when reading in the data, if the memory block gets exceeded we can make that block bigger.
#### EOF
---
You have to be careful in C when trying to find EOF. In C, EOF is an integer, not a character.
```C
int c;
...
while (1) {

c = getchar();
if (c == EOF || c == '\n') {...}

...
}
```
When we reach EOF, null terminate our current string and return it, or you can add the character to our existing string.
#### Checking if input goes beyond buffer
---
You need to check if the NEXT character goes beyond the buffer. If so, reallocate buffer before continuing.
#### Full Sample Code
---
```C
#define LSH_RL_BUFSIZE 1024
char *lsh_read_line(void)
{
  int bufsize = LSH_RL_BUFSIZE;
  int position = 0;
  char *buffer = malloc(sizeof(char) * bufsize);
  int c;

  if (!buffer) {
    fprintf(stderr, "lsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

  while (1) {
    // Read a character
    c = getchar();

    // If we hit EOF, replace it with a null character and return.
    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;

    // If we have exceeded the buffer, reallocate.
    if (position >= bufsize) {
      bufsize += LSH_RL_BUFSIZE;
      buffer = realloc(buffer, bufsize);
      if (!buffer) {
        fprintf(stderr, "lsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}
```

## Parsing the line
---
Now, the arguments have to be parsed. We can do that by "tokenizing" the arguments and delimiting by the whitespace. So,  ```this message``` is actually two arguments; this and message. We can use strtok to do the tokenizing for us.
#### strtok
---
It literally takes in a string and based off a delimiter separates it for us.
```C
#include <stdio.h>
#include <string.h>

int main() {
    char str[] = "This is a sample string";
    char *token;

    token = strtok(str, " ");
    while (token != NULL) {
        printf("%s\n", token);
        token = strtok(NULL, " ");
    }
    return 0;
}
```

This grabs the first token.
```C 
token = strtok(str, " ");
```

Subsequent calls are done doing this, so you pass in null to go to the next token. This is because it returns a pointer within the string and places a `\0` bytes at the end of each token.
```C
token = strtok(NULL, " ");
```
#### Full Sample Code
---
```C
#define LSH_TOK_BUFSIZE 64
#define LSH_TOK_DELIM " \t\r\n\a"
char **lsh_split_line(char *line)
{
  int bufsize = LSH_TOK_BUFSIZE, position = 0;
  char **tokens = malloc(bufsize * sizeof(char*));
  char *token;

  if (!tokens) {
    fprintf(stderr, "lsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

  token = strtok(line, LSH_TOK_DELIM);
  while (token != NULL) {
    tokens[position] = token;
    position++;

    if (position >= bufsize) {
      bufsize += LSH_TOK_BUFSIZE;
      tokens = realloc(tokens, bufsize * sizeof(char*));
      if (!tokens) {
        fprintf(stderr, "lsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }

    token = strtok(NULL, LSH_TOK_DELIM);
  }
  tokens[position] = NULL;
  return tokens;
}
```
## How shells start processes
---
>Starting processes is the whole point of shells. You need to know exactly what processes are and how they start to write a shell.
#### Init
---
When a Unix computer boots, its kernel is loaded. Once loaded, the kernel starts only one process, init. This process runes for the entire time the computer is on. It manages loading up all the processes your computer needs.
#### fork()
---
Most programs are not init. Because of this the computer also uses ``fork()`` system call. When called, the operating system makes a duplicate of the process and starts running them both. The original process is the parent and the new one is the child. ``fork()`` returns 0 to the child process, and it returns to the parent the process ID number (PID) of its child. This means that the only way for new processes to start is by an existing one duplicating itself.
#### exec()
---
Though, when you want to start a new process you don't want a copy of the same program, rather you want to run a different program. ``exec()`` solves this. It replaces the running program with a new one entirely. When you call ``exec()`` the operating system stops your process, loads the new program, and starts a new one in its place. A process will never return from an ``exec()`` call.
#### Full Example Code
---
With those two system calls, this is the basis of how programs are run on Unix. First, a program separates/clones itself into two. Then ``exec()`` replaces the child with a different process. The parent process can continue doing other things, and can even keep tabs on its children, using ``wait()`` call.

To clear up some confusion about what exactly this means, init runs creating the very first running process with what we will call PID 1. Then every other process ``fork()`` and ``exec()`` from that. 
```
				Init (PID 1)
                 /    |    \
                /     |     \
           Service1 Service2 Login_Manager
              /         \         \
             /           \         \
        Subprocess    Webserver    User_Session
                        /              \
                       /                \
                  Web_Worker         Terminal
                                        |
                                        |
                                      Shell
                                     /  |  \
                                    /   |   \
                               Program1 Program2 Program3
```
Some examples of this in work:
- A shell forks when you type a command
- A web server forks when a new connection arrives
- A service might fork to perform background work
```C
int lsh_launch(char **args)
{
  pid_t pid, wpid;
  int status;

  pid = fork();
  if (pid == 0) {
    // Child process
    if (execvp(args[0], args) == -1) {
      perror("lsh");
    }
    exit(EXIT_FAILURE);
  } else if (pid < 0) {
    // Error forking
    perror("lsh");
  } else {
    // Parent process
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (!WIFEXITED(status) && !WIFSIGNALED(status));
  }

  return 1;
}
```
This program takes the list of arguments that we had parsed earlier. It then forks the process, and saves the return value. Once ``fork()`` returns we will have two processes running concurrently. The child process will take the ``pid == 0`` condition. 

In this child process we want to run the command given by the user. 
>We are literally running the new process, since the command its self is a process. That child will become the process, whether it gets cleaned up right after or not is dependent on the process.

So, we use a variant of the ``exec()`` system call, ``execvp()``. The different variants do slightly different things. Some might take a variable number of string arguments. Others take a list of strings. Still others let you specify the environment that the process runs with. This particular variant expects a program name and an array of string arguments. The 'p' means that instead of providing the full file path of the program to run, we are going to give its name, and let the operating system search for the program in the path. 

If ``exec()`` returns -1, (or if it returns at all) we know there was an error. So, ``perror()`` prints the system's error message, along with the program name, so users know where the error came from. Then, we exit so the shell can continue. 

The second condition ``pid < 0`` checks if ``fork()`` had an error. If so, we just print it and keep going.

The third condition means that ``fork()`` executed successfully. The parent process lands there. In this condition we know that the child is going to execute the process, so the parent needs to wait for the command to finish running. We use ``waitpid()`` to wait for the process's state in lots of ways, and not all of them mean that the process has ended. ``waitpid()`` has a lot of options, and the process can change state in lots of ways, not all of them mean the program has ended. A process can either exit (normally, or with an error code), or it can be killed by a signal. So, we use the macros provided with ``waitpid()`` to wait until either the processes are exited or killed. Then, the function finally returns a 1, as a signal to the calling function that we should prompt for input again.

To visualize this further, imagine we have two terminals open. Terminal 1 (PID 1000) and Terminal 2 (PID 2000). Shell 1 calls ``fork()`` after a command was run through it, thus creating a child (PID 1001). Now Shell 1 has to wait because of ``waitpid()`` since it is listening for a return value. The child now calls ``execvp()`` and transforms into what the command typed was. Terminal 1 throughout this process looks frozen, but Terminal 2 is free to do whatever since it is not a part of that process. Once, the command finishes and Terminal 1 gets the return value from ``waitpid()`` it is free to prompt again.

This is also why you can run a command with a ``&`` at the end. Lets say you run the command ``ls -la`` in your terminal, even though it is fast there is still a period of time there where the terminal is block, because of ``waitpid()``. If we ran the command as ``ls -la &`` It would NOT wait for the child to finish. It will just immediately continue and shows a new prompt. Once child is done, the terminal will still be notified. 

## Shell Builtins
---
If you paid close attention you would have noticed that the initial ``lsh_loop()`` function calls ``lsh_execute()``, but above the function is titled ``lsh_launch()``. This is intentional. Most commands a shell executes are programs, but not all of them. Some are built right into the shell. 

The reason for this is easy to understand. For example, if you want to change directory you use the function ``chdir()``. The current directory though is a property of a process. So, if you wrote a program called ``cd`` that changed directory, it would just change its own current directory, then terminate. The parent process directory would be unchanged. Instead, the shell process itself needs to execute ``chdir()``, so that is own current directory is updated. Then, when it launches child processes, they too will inherit that directory. 

In that same vein, if there was a program called ``exit``, it would not be able to exit the shell that called it. That command also needs to be built into the shell. Also, most shells are configured by running configuration scripts like ``~/.bashrc``. Those scripts use commands that change the operation of the shell. These commands could only change the shell's operation if they were implemented within the shell process itself.

So, it makes sense that we need to add some commands to the shell itself. The ones added are ``cd, exit, and help``.
#### Full Sample Implementations
---
```C
/*
  Function Declarations for builtin shell commands:
 */
int lsh_cd(char **args);
int lsh_help(char **args);
int lsh_exit(char **args);

/*
  List of builtin commands, followed by their corresponding functions.
 */
char *builtin_str[] = {
  "cd",
  "help",
  "exit"
};

int (*builtin_func[]) (char **) = {
  &lsh_cd,
  &lsh_help,
  &lsh_exit
};

int lsh_num_builtins() {
  return sizeof(builtin_str) / sizeof(char *);
}

/*
  Builtin function implementations.
*/
int lsh_cd(char **args)
{
  if (args[1] == NULL) {
    fprintf(stderr, "lsh: expected argument to \"cd\"\n");
  } else {
    if (chdir(args[1]) != 0) {
      perror("lsh");
    }
  }
  return 1;
}

int lsh_help(char **args)
{
  int i;
  printf("Stephen Brennan's LSH\n");
  printf("Type program names and arguments, and hit enter.\n");
  printf("The following are built in:\n");

  for (i = 0; i < lsh_num_builtins(); i++) {
    printf("  %s\n", builtin_str[i]);
  }

  printf("Use the man command for information on other programs.\n");
  return 1;
}

int lsh_exit(char **args)
{
  return 0;
}
```

## Putting together builtins and processes
---
The last missing piece to implement ``lsh_execute()``, is the function that will either launch a builtin, or a process. We already set ourselves up for a really simple function:
```C
int lsh_execute(char **args)
{
  int i;

  if (args[0] == NULL) {
    // An empty command was entered.
    return 1;
  }

  for (i = 0; i < lsh_num_builtins(); i++) {
    if (strcmp(args[0], builtin_str[i]) == 0) {
      return (*builtin_func[i])(args);
    }
  }

  return lsh_launch(args);
}
```
All this is doing is checking if the command equals each builtin, and if so, run. If it does not match, it calls ``lsh_launch(args)`` to launch the process. But, args might contain ``NULL``, so we check that at the beginning.

## Putting it all together
---
That is all the code that goes into the shell. To try it out, you would just copy all of these into a file (main.c), and compile. Make sure to only have one ``lsh_read_line()``. And add these includes.
- `#include <sys/wait.h>`
	- `waitpid()` and associated macros
- `#include <unistd.h>`
	- `chdir()`
	- `fork()`
	- `exec()`
	- `pid_t`
- `#include <stdlib.h>`
	- `malloc()`
	- `realloc()`
	- `free()`
	- `exit()`
	- `execvp()`
	- `EXIT_SUCCESS`, `EXIT_FAILURE`
- `#include <stdio.h>`
	- `fprintf()`
	- `printf()`
	- `stderr`
	- `getchar()`
	- `perror()`
- `#include <string.h>`
	- `strcmp()`
	- `strtok()`
Once these headers are including, it should be as simple as running ``gcc -o main main.c`` to compile and ``./main`` to run it.

## Wrap up
---
Use commands like ``man 3p`` to find through documentation on every system call. Some issues with this shell implementation:
- Only whitespace separating arguments, no quoting or backslash escaping.
- No piping or redirection.
- Few standard builtins.
- No globbing.
