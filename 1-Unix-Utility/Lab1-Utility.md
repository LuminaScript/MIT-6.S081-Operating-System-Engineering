## Lab 1 - Utility

### sleep `EASY`
```c 
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void error_exit(char *error_msg) {
    fprintf(2, error_msg);
    exit(1);
}

int main(int argc, char *argv[])
{

    if (argc != 2) {
        error_exit("[Usage]: sleep <sleep_time_in_seconds>\nExample: sleep 10");
    }

    int sec = atoi(argv[1]);
    sleep(sec);
    exit(0);
}
```

### pingpong `EASY`
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    int fds[2];
    pipe(fds);
    int pid = fork();

    if (pid == 0) {
        char buf[2];
        int child_pid = getpid();
        read(fds[1], buf, 1);
        fprintf(0, "%d: received ping\n", child_pid);
        write(fds[1], buf, 1);
    } else {
        char buf[2];
        int par_pid = getpid();
        write(fds[0], "x", 1);
        read(fds[0], buf, 1);
        fprintf(0, "%d: received pong\n", par_pid);
    }
    exit(0);
}
```

### primes `hard`
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int SIZE = 34;
int fds[2]; // 0 - read, 1 - write

void child() {
    int buf[SIZE];
    memset(buf, 0, sizeof(buf));

    // process data from left neighbour
    int bytes = read(fds[0], buf, SIZE * (sizeof(int)));
    if (bytes == 0) {
        close(fds[0]); // close child pipe
        return ;
    }

    // drop data
    int min_num = buf[0], buf_out_idx = 0, n = bytes/sizeof(int);
    printf("prime %d\n", min_num);
    for (int i = 1; i < n; i++) {
        int num = buf[i];
        if(num % min_num) {
            buf[buf_out_idx++] = num;
        }
    }

    // feed data to right neighbour 
    if (buf_out_idx) {
        if (fork() == 0) { // child
            child(); // recursive
        } else {
            write(fds[1], buf, buf_out_idx * sizeof(int));
            wait((int *) 0);
        }
    } 
}

int main(int argc, char *argv[])
{
    int buf[SIZE], pid;
    for (int i = 2; i <= 35; i++) {
        buf[i - 2] = i;
    }

    pipe(fds);
    if ((pid = fork()) == 0) {
        child();
        close(fds[0]);
    } else {
        write(fds[1], buf, sizeof(buf));
        wait((int *) 0);
        close(fds[1]);
    }

    exit(0);
}
```

### find `moderate`
```c 
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* fmtname(char *path)
{
    static char buf[DIRSIZ+1];
    char *p;

    // Find first character after last slash.
    for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
    p++;

    // Return blank-padded name.
    if(strlen(p) >= DIRSIZ)
    return p;
    memmove(buf, p, strlen(p));
    memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));
    return buf;
}

void concat_path(char *result, const char *path, const char *dname) {
    while (*path) {
        *result++ = *path++;
    }
    if (*(result - 1) != '/') {
        *result++ = '/';
    }
    while (*dname && *dname != ' ') { // Stop at null or padding spaces in DIRSIZ
        *result++ = *dname++;
    }
    *result = '\0';
}

void iterate_dir(char *path, char *target_file)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "ls: cannot open %s\n", path);
        return;
    }
    
    if(fstat(fd, &st) < 0){
        fprintf(2, "ls: cannot stat %s\n", path);
        close(fd);
        return;
    }

    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
        printf("ls: path too long\n");
        return ;
    }

    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';

    while(read(fd, &de, sizeof(de)) == sizeof(de)){
        if(de.inum == 0 || strcmp(de.name, "..") == 0 || strcmp(de.name, ".") == 0) {
            continue;
        }

        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = 0;
        if(stat(buf, &st) < 0){
            printf("find: cannot stat %s\n", buf);
            continue;
        }
        char *dname = fmtname(buf);

        if (st.type == T_DIR ) { 
            char child_dir[512];
            concat_path(child_dir, path, dname);
            iterate_dir(child_dir, target_file);
            printf("dir_name: %s\n", child_dir);
        } else if (st.type == T_FILE && strcmp(de.name, target_file) == 0){
                printf("%s/%s\n", path, dname);
        }
    }
  
  close(fd);
}

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(2, "Usage: find <directory tree> <file_name>\nExample: find . b");
        exit(1);
    }

    char *path = argv[1];
    char *targed_dname = argv[2];
    iterate_dir(path, targed_dname);

    exit(0);
}
```

### xargs `moderate`
```c 
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

#define MAX_LEN 120
#define MAXAGR 120

void copy_argument(char **dest, const char *src) {
    *dest = malloc(strlen(src) + 1);
    if (*dest == 0) {
        fprintf(2, "Error: Memory allocation failed\n");
        exit(1);
    }
    strcpy(*dest, src);
}

int parse_line(char *paramv[], int start_index) {
    char buf[MAX_LEN];
    int buf_index = 0;

    // Read input until a newline or EOF
    while (read(0, &buf[buf_index], 1) > 0) {
        if (buf[buf_index] == '\n') {
            buf[buf_index] = '\0';
            break;
        }
        buf_index++;
        if (buf_index >= MAX_LEN) {
            fprintf(2, "Error: Argument too long\n");
            exit(1);
        }
    }

    // Check if we reached EOF with no input
    if (buf_index == 0) {
        return -1; // Signal EOF or no input
    }

    // Parse the line into arguments
    int param_index = start_index;
    int token_start = 0;

    while (token_start < buf_index) {
        // Skip leading spaces
        while (buf[token_start] == ' ' && token_start < buf_index) {
            token_start++;
        }

        // Find the end of the token
        int token_end = token_start;
        while (token_end < buf_index && buf[token_end] != ' ') {
            token_end++;
        }

        if (token_start < token_end) { // Valid token found
            if (param_index >= MAXAGR - 1) {
                fprintf(2, "Error: Too many arguments\n");
                exit(1);
            }
            buf[token_end] = '\0'; // Null-terminate the token
            copy_argument(&paramv[param_index], &buf[token_start]);
            param_index++;
        }

        token_start = token_end + 1; // Move to the next token
    }

    return param_index;
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(2, "[Usage] xargs <command_name> <arg1> ...\n");
        exit(1);
    }

    char *paramv[MAXAGR];
    for (int i = 1; i < argc; i++) {
        copy_argument(&paramv[i - 1], argv[i]);
    }

    int param_start = argc - 1; // Start filling arguments after provided ones
    int param_end;

    while ((param_end = parse_line(paramv, param_start)) != -1) {
        paramv[param_end] = 0; // Null-terminate the argument list

        if (fork() == 0) {
            exec(paramv[0], paramv);
            fprintf(2, "Error: exec failed\n");
            exit(1);
        } else {
            wait(0); // Wait for the child process to finish
        }

        // Free dynamically allocated arguments after execution
        for (int i = param_start; i < param_end; i++) {
            free(paramv[i]);
        }
    }

    // Free initial arguments
    for (int i = 0; i < param_start; i++) {
        free(paramv[i]);
    }

    exit(0);
}
```
