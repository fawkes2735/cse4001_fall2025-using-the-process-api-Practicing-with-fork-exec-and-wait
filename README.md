# Assignment: Practicing the Process API
Practicing with fork, exec, wait. 

### Overview

In this assignment, you will practice using the Process API to create processes and run programs under Linux. The goal is to gain hands-on experience with system calls related to process management. Specifically, you will practice using the unix process API functions 'fork()', 'exec()', 'wait()', and 'exit()'. 

⚠️ Note: This is not an OS/161 assignment. You will complete it directly on Linux. 

Use the Linux in your CSE4001 container. If you are using macOS, you may use the Terminal (you may need to install development tools with C/C++ compilers). 

**Reference Reading**: Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*, Chapter 5 (Process API Basics)
 👉 [Chapter 5 PDF](http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)

---

### **Steps to Complete the Assignment**

1. **Accept the GitHub Classroom Invitation**
    [GitHub Link](https://classroom.github.com/a/FZh4BrQG)
2. **Set up your Repository**
   - Clone the assignment repository.
3. **Study the Reference Materials**
   - Read **Chapter 5**.
   - Download and explore the sample programs from the textbook repository:
      [OSTEP CPU API Code](https://github.com/remzi-arpacidusseau/ostep-code/tree/master/cpu-api).
4. **Write Your Programs**
   - Adapt the provided example code to answer the assignment questions.
   - Each program should be clear, well-commented, and compile/run correctly.
   - Add your solution source code to the repository.

5. **Prepare Your Report**
   - Answer the questions in the README.md file. You must edit the README.md file and not create another file with the answers. 
   - For each question:
     - Include your **code**.
     - Provide your **answer/explanation**.
6. **Submit Your Work via GitHub**
   - Push both your **program code** to your assignment repository.
   - This push will serve as your submission.
   - Make sure all files, answers, and screenshots are uploaded and rendered properly.








---
### Questions
1. Write a program that calls `fork()`. Before calling `fork()`, have the main process access a variable (e.g., x) and set its value to something (e.g., 100). What value is the variable in the child process? What happens to the variable when both the child and parent change the value of x?


```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main(void) {
    int x = 100;
    printf("[before fork pid=%d] x=%d\n", getpid(), x);

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    if (pid == 0) {
        printf("[child  pid=%d] sees x=%d\n", getpid(), x);
        x = 200;
        printf("[child  pid=%d] changed x=%d\n", getpid(), x);
        _exit(0);
    } else {
        x = 300;
        printf("[parent pid=%d] changed x=%d\n", getpid(), x);
    }
    return 0;
}
```

2. Write a program that opens a file (with the `open()` system call) and then calls `fork()` to create a new process. Can both the child and parent access the file descriptor returned by `open()`? What happens when they are writing to the file concurrently, i.e., at the same time?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/wait.h>

int main(void) {
    int fd = open("q2_output.txt", O_CREAT | O_TRUNC | O_WRONLY, 0644);
    if (fd < 0) { perror("open"); exit(1); }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    const char *who = (pid == 0) ? "child" : "parent";
    for (int i = 0; i < 8; i++) {
        char buf[64];
        int n = snprintf(buf, sizeof(buf), "[%s pid=%d] line %d\n", who, getpid(), i);
        if (write(fd, buf, n) != n) perror("write");
        sleep(10000);
    }

    if (pid == 0) _exit(0);
    wait(NULL);
    close(fd);
    return 0;
}
```

3. Write another program using `fork()`.The child process should print “hello”; the parent process should print “goodbye”. You should try to ensure that the child process always prints first; can you do this without calling `wait()` in the parent?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    int p[2];
    if (pipe(p) == -1) { perror("pipe"); exit(1); }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    if (pid == 0) {
        close(p[0]);
        puts("hello");
        if (write(p[1], "x", 1) != 1) perror("write");
        close(p[1]);
        _exit(0);
    } else {
        close(p[1]);
        char d;
        if (read(p[0], &d, 1) != 1) perror("read");
        close(p[0]);
        puts("goodbye");
    }
    return 0;
}
```


4. Write a program that calls `fork()` and then calls some form of `exec()` to run the program `/bin/ls`. See if you can try all of the variants of `exec()`, including (on Linux) `execl()`, `execle()`, `execlp()`, `execv()`, `execvp()`, and `execvpe()`. Why do you think there are so many variants of the same basic call?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

extern char **environ;

static void die(const char *m) { perror(m); exit(1); }

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "usage: %s {execl|execle|execlp|execv|execvp|execvpe}\n", argv[0]);
        return 1;
    }
    pid_t pid = fork();
    if (pid < 0) die("fork");

    if (pid == 0) {
        if (!strcmp(argv[1], "execl")) {
            execl("/bin/ls", "ls", "-l", (char*)NULL);
        } else if (!strcmp(argv[1], "execle")) {
            char *envp[] = {"FOO=bar", "LC_ALL=C", NULL};
            execle("/bin/ls", "ls", "-l", (char*)NULL, envp);
        } else if (!strcmp(argv[1], "execlp")) {
            execlp("ls", "ls", "-l", (char*)NULL);
        } else if (!strcmp(argv[1], "execv")) {
            char *args[] = {"ls", "-l", NULL};
            execv("/bin/ls", args);
        } else if (!strcmp(argv[1], "execvp")) {
            char *args[] = {"ls", "-l", NULL};
            execvp("ls", args);
        } else if (!strcmp(argv[1], "execvpe")) {
            char *args[] = {"ls", "-l", NULL};
            char *envp[] = {"FOO=bar", "LC_ALL=C", NULL};
            execvpe("ls", args, envp);
        } else {
            fprintf(stderr, "unknown: %s\n", argv[1]);
            _exit(2);
        }
        perror("exec failed");
        _exit(1);
    }
    return 0;
}
```

5. Now write a program that uses `wait()` to wait for the child process to finish in the parent. What does `wait()` return? What happens if you use `wait()` in the child?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <errno.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    if (pid == 0) {
        _exit(42);
    } else {
        int st = 0;
        pid_t done = wait(&st);
        if (done == -1) { perror("wait"); exit(1); }
        if (WIFEXITED(st))
            printf("wait() -> pid=%d, exit=%d\n", done, WEXITSTATUS(st));
        else if (WIFSIGNALED(st))
            printf("wait() -> pid=%d, signal=%d\n", done, WTERMSIG(st));
    }

    int st2;
    pid_t r = wait(&st2);
    if (r == -1) perror("wait with no children (expected ECHILD)");
    return 0;
}
```

6. Write a slight modification of the previous program, this time using `waitpid()` instead of `wait()`. When would `waitpid()` be useful?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t c1 = fork();
    if (c1 < 0) { perror("fork c1"); exit(1); }
    if (c1 == 0) { usleep(150000); _exit(11); }

    pid_t c2 = fork();
    if (c2 < 0) { perror("fork c2"); exit(1); }
    if (c2 == 0) { usleep(300000); _exit(22); }

    int st = 0;
    waitpid(c1, &st, 0);
    printf("waited for c1 exit=%d\n", WIFEXITED(st) ? WEXITSTATUS(st) : -1);

    int st2 = 0;
    pid_t r = waitpid(c2, &st2, WNOHANG);
    if (r == 0) {
        printf("c2 not ready, doing other work...\n");
        r = waitpid(c2, &st2, 0);
    }
    printf("waited for c2 exit=%d\n", WIFEXITED(st2) ? WEXITSTATUS(st2) : -1);
    return 0;
}

```

7. Write a program that creates a child process, and then in the child closes standard output (`STDOUT FILENO`). What happens if the child calls `printf()` to print some output after closing the descriptor?

```cpp
// Add your code or answer here. You can also add screenshots showing your program's execution.  
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    if (pid == 0) {
        if (close(STDOUT_FILENO) == -1) perror("close stdout");

        printf("child: you should NOT see this\n");
        if (fflush(stdout) == EOF) perror("fflush error (expected)");

        int fd = open("child_output.txt", O_CREAT | O_WRONLY | O_TRUNC, 0644);
        if (fd == -1) { perror("open"); _exit(1); }
        dprintf(fd, "child: writing to file (fd=%d)\n", fd);
        close(fd);
        _exit(0);
    } else {
        puts("parent: this prints to terminal as usual");
    }
    return 0;
}
```

