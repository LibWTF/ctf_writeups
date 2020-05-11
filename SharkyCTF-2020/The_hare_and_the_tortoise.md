# Intro
The challenge is described as follows "Do you know Jean de La Fontaine? A friend of mine created a program mimicking the hare and the tortoise. He told me that smart tortoises always wins. I want you to be that tortoise." We can login with ssh as user tortoise 
```bash
ssh tortoise@172.30.0.2 Password : tortoise
```

## Initial Exploration
As user tortoise we can list files in the home directory:

```bash
tortoise@the_hare_and_the_tortoise$ ls -l
-r-------- 1 hare hare   77 May 11 09:04 flag.txt
-r--r--r-- 1 root root 2360 May 11 09:04 main.c
-r--r--r-- 1 root root  687 May 11 09:04 semaphores.h
-r-sr-xr-x 1 hare hare 2754 May 11 09:04 the_hare_and_the_tortoise
```

We can see the the permissions for the following files are limited for us as user tortoise. We can run the program the_hare_and_the_tortoise, read main.c and semaphores.h, and we do not have access to flag.txt.

After running the program we get the following output:
```bash
tortoise@the_hare_and_the_tortoise$ ./the_hare_and_the_tortoise flag.txt
The tortoise, progressing slowly... : "s"
The hare says : "Do you ever get anywhere?"
The tortoise, progressing slowly... : "h"
The tortoise, progressing slowly... : "k"
The tortoise, progressing slowly... : "C"
The tortoise, progressing slowly... : "T"
The tortoise, progressing slowly... : "F"
The hare says : "Hurry up tortoise !"
The tortoise, progressing slowly... : "{"
The tortoise, progressing slowly... : "r"
The tortoise, progressing slowly... : "4"
The tortoise, progressing slowly... : "c"
Killed
```

This looks interesting, but we need to learn more about the c program:
```bash
tortoise@the_hare_and_the_tortoise$ cat main.c
```

```c
#include <stdlib.h>
#include <stdio.h>
#include <fcntl.h>
#include <signal.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <sys/ipc.h>
#include "semaphores.h"

// The Hare and the Tortoise


#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)


pid_t ppid;
int sem = -1;
char* sem_name;
char temp_dir[60] = {0};
char lock = 0;
char ppid_dir[30] = {0};

void cleanup(){
  if(sem != -1){
    SEM_DEL(sem);
  }
  rmdir(ppid_dir);
  rmdir(sem_name);
}

void sigint_handler(int signo){
  cleanup();
  exit(1);
}

void alarm_handler(int signo){
  cleanup();
  kill(ppid, SIGKILL);
  exit(1);
}


void random_string(){
  /* Only one execution should be allowed per term */
  sprintf(ppid_dir, "/tmp/%d", getppid());
  if(mkdir(ppid_dir, 0700) == -1){
    puts("There is no need for bruteforce");
    exit(1);
  }
  sprintf(temp_dir, "/tmp/%d/XXXXXX", getppid());
  sem_name = mkdtemp(temp_dir);
  if(sem_name == NULL){ perror("mkdtemp failed: "); exit(1); }
}

int main(int argc, char** argv){

  if(argc != 2){
    printf("Usage : %s <file to read>\n", argv[0]);
    exit(1);
  }
  atexit(cleanup);
  signal(SIGINT, sigint_handler);
  signal(SIGALRM, alarm_handler);
  random_string();
  sem = semget(ftok(sem_name, 1337 & 1), 1, IPC_CREAT | IPC_EXCL | 0600);

  if(sem == -1) handle_error("semget");

  SEM_SET(sem, 1);

  int hare = open (argv[1], O_RDONLY);
  int tortoise = open (argv[1], O_RDONLY);
  if(hare == -1 || tortoise == -1) handle_error("open");
  ppid = getpid();
  int pid;
  pid = fork();

  int cnt = 0;
  if(pid == 0) { // The hare
    puts("The hare says : \"Do you ever get anywhere?\"");
    char c;
    int y = 1;
    while(y == 1){
      SEM_WAIT(sem);
      y = read(hare, &c, sizeof(char));
      if(y == -1){ alarm(0.1);handle_error("read"); }
      usleep(100 * 750);
      SEM_POST(sem);
    }
    puts("The hare says : \"Hurry up tortoise !\"");
    alarm(5);
    sleep(10);

	} else { // The tortoise
    char c;
    int y = 1;
    while(y == 1){
      SEM_WAIT(sem);
      y = read(tortoise, &c, sizeof(char));
      printf("The tortoise, progressing slowly... : \"%c\"\n", c);
      if(y == -1){ handle_error("read"); }
      SEM_POST(sem);
      sleep(1);
    }
    puts("Slow but steady wins the race!");
  }
}
```

## Solution
As you can see in the code the fork function is called. The fork system call is used for creating a new process, which is called a child process, which runs concurrently with the process that makes the fork() call (parent process). After a new child process is created, both processes will execute the next instruction following the fork() system call. A child process uses the same pc(program counter), same CPU registers, same open files which are used in the parent process.
```c
pid = fork();
```

We now know that a parent and child process will be created (the tortoise and the hare). We can also infer a couple more things.

The hare is the child process:
```C
  if(pid == 0) { // The hare
    puts("The hare says : \"Do you ever get anywhere?\"");
```

The tortoise is the parent process. Apart from the C code you can also see it in the output:
```bash
tortoise@the_hare_and_the_tortoise$ ./the_hare_and_the_tortoise flag.txt
The tortoise, progressing slowly... : "s"
The hare says : "Do you ever get anywhere?"
The tortoise, progressing slowly... : "h"
```

We see here that after the hare is spawned it will eventually call the following and kill the program:
```c
 puts("The hare says : \"Hurry up tortoise !\"");
    alarm(5);
    sleep(10);
```

What if we try to kill the hare child process?
``` bash
tortoise@the_hare_and_the_tortoise$ ps  xao pid,ppid,pgid,sid,comm | grep the_hare_and_the_tortoise
 PID   PPID   PGID    SID COMMAND
3355   2274   3355   2274 the_hare_and_the_tortoise
3356   3355   3355   2274 the_hare_and_the_tortoise
```

From the ps output we see that the parent process ID for the second process of "the_hare_and_the_tortise" is PID 3355. So, lets try to kill the child process (the hare), thus stopping the alarm from being triggered.
``` bash
kill -9 3356
```

Finally we get the flag:
```bash
The tortoise, progressing slowly... : "s"
The hare says : "Do you ever get anywhere?"
The tortoise, progressing slowly... : "h"
The tortoise, progressing slowly... : "k"
The tortoise, progressing slowly... : "C"
The tortoise, progressing slowly... : "T"
The tortoise, progressing slowly... : "F"
The hare says : "Hurry up tortoise !"
The tortoise, progressing slowly... : "{"
The tortoise, progressing slowly... : "r"
The tortoise, progressing slowly... : "4"
The tortoise, progressing slowly... : "c"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "5"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "4"
The tortoise, progressing slowly... : "r"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "s"
The tortoise, progressing slowly... : "i"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "r"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "w"
The tortoise, progressing slowly... : "h"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "n"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "y"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "u"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "4"
The tortoise, progressing slowly... : "r"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "l"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "n"
The tortoise, progressing slowly... : "e"
The tortoise, progressing slowly... : "_"
The tortoise, progressing slowly... : "6"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "2"
The tortoise, progressing slowly... : "6"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "2"
The tortoise, progressing slowly... : "6"
The tortoise, progressing slowly... : "c"
The tortoise, progressing slowly... : "5"
The tortoise, progressing slowly... : "7"
The tortoise, progressing slowly... : "f"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "1"
The tortoise, progressing slowly... : "2"
The tortoise, progressing slowly... : "e"
The tortoise, progressing slowly... : "d"
The tortoise, progressing slowly... : "6"
The tortoise, progressing slowly... : "6"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "b"
The tortoise, progressing slowly... : "8"
The tortoise, progressing slowly... : "c"
The tortoise, progressing slowly... : "2"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "e"
The tortoise, progressing slowly... : "5"
The tortoise, progressing slowly... : "3"
The tortoise, progressing slowly... : "8"
The tortoise, progressing slowly... : "a"
The tortoise, progressing slowly... : "0"
The tortoise, progressing slowly... : "4"
The tortoise, progressing slowly... : "}"
The tortoise, progressing slowly... : "}"
Slow but steady wins the race!
```

shkCTF{r4c35_4r3_3asi3r_wh3n_y0u_4r3_al0ne_6a26a26c57f0012ed66ab8c20e538a04}
