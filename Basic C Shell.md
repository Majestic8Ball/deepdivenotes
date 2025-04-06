>Using this repo: https://github.com/codecrafters-io/build-your-own-x
>I found and created using this guide: https://brennan.io/2015/01/16/write-a-shell-in-c/
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
