## problem reqs
so when we log in, there are typicall three files - flag, input, input.c

```
// input.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}
```

we can see there are 5 stages to clear.

## thought process
For stage1, I could see that I will pass 99 arguments with argv['A']('A' is 65) and argv['B'] set to required value in the program.

Cool!

`echo` would work here. So here is the input.
<input>

But their is one probelm. bash doens't allow `\x00` input in the arguments. But we'll figure out it later for now. let's move on to next stage. Stage 2!

Stage 2 requires us to read from stdin and stderr. Stdin should be easy.

`echo -ne "\x00\x0a\x00\xff" | ./input <same argument as lne 84>`
For second part of the stage 2 we could use something like `2 < <(echo -ne "\x00\x0a\x02\xff")`. 2 is file descriptor for `stderr`. And `2 < (file or any command)` is a syntax which reads data from `file` and put in the `2` descriptor. read more by googling on command substitution.

Ok-- so we are ready for our input to clear stage 2. Here is the input:

<echo-command>
  
 
let' go to stage 3.

here we have to set the env variable to clear this stage.

Now here I tried following:
- set env "\xde\xad\xbe\xef"="\xca\xfe\xba\xbe" && <input to clear stage 2>

After this I was lost on how to pass the env variable using command line.

Then after some googling, I found out that this problem was intended to be solved by writing the C wrapper or for that matter using any program. Wow!! I learnt something new here. Cool!

Let's start

We'll use `execve()` in C to replace the current process with another process.

#### stage1
```
void setArgs() {
	args[0]=PATH;
	for (int i = 1; i < 100; ++i)
		args[i]="0";

	args['A']="\x00";
	args['B']="\x20\x0a\x0d";
	args['C']="9001";
	args[100]=NULL;
}

```

eazy peazy!

#### stag2
here we would like the read from input/stderr of a process. I tried to write to `fd=0` and hoping that new process of `input` will read from `fd=0`. Well not a bad appoach, but if you read the man of `execve`, it says `Any open directory streams are closed`. Darn it!
So there ought to be some other way to write to stdin of `input` process.

Here we introduce `pipes`. Yes this is same as the `|` we use to pass the output of one process as an input to other in terminal. Idea is the following:
- create two pipes - stdinpipe, other stderrpipe. 
- create a fork. Now there will be two processes of this wrapper.
- We'll be writing from child to parent. So close the write end of stdinpipe and stderrpipe in parent.And close the read end of both of these in child.
- `dup2()` the read end of pipe to `fd=0`. Remember `fork` inherit the open file descriptors of it's parent process.

For more information on pipes, read  http://unixwiz.net/techtips/remap-pipe-fds.html

Code listing all the above principles

```
void main_main() {
	
	int pipe2stdin[2], pipe2stderr[2];

	if(pipe(pipe2stderr) < 0 || pipe(pipe2stdin) < 0) {
		perror("can't create pipe");
		exit(1);
	}

	forkAndPipe(pipe2stdin, pipe2stderr, args);
}

void forkAndPipe(int *pipe2stdin, int *pipe2stderr, char **args) {
	pid_t c_pid; 
	if((c_pid=fork()) == -1) {
		perror("fork error");
		exit(1);
	}

	char *envp[] = {
		"\xde\xad\xbe\xef=\xca\xfe\xba\xbe",
		0
	};

	if (c_pid == 0)
	{
		close(pipe2stdin[0]); close(pipe2stderr[0]);
		write(pipe2stdin[1], "\x00\x0a\x00\xff", 4);
		write(pipe2stderr[1], "\x00\x0a\x02\xff", 4);
		stage5();
		
	} else {
		// parent process 
		close(pipe2stdin[1]); close(pipe2stderr[1]);
		dup2(pipe2stdin[0], 0);
		dup2(pipe2stderr[0], 2);
		execve(PATH, args, envp);
	}
}


```

gotcha!! stage 2

#### stag3
seems easy. could be done by passing the env in the argument of `execve()`. Let's just do that only

```
char *envp[] = {
		"\xde\xad\xbe\xef=\xca\xfe\xba\xbe",
		0
	};
	execve(PATH, args, envp);

```

#### stage4

again just simple file operation. 

```
void stage4() {
	FILE *ptr = fopen("\x0a", "w");
	if(ptr == NULL) {
		perror("file not able to create");
		exit(1);
	}
	fwrite("\x00\x00\x00\x00", 4, 1, ptr);

    fclose(ptr);
    
}
```

voila!

#### stage5

socket related. I didn't had a working knowledge of sockets till now.
So went I through this tutorial: https://www.cs.rpi.edu/~moorthy/Courses/os98/Pgms/socket.html

basically the idea is that if you are a server: 
- you would create a socket
- bind to that socket
- listen on that socket
- finally accept/receive/read() data on that socket.

And if you are client:
- create a socket
- connect to that socket
- send/write data to that socket.

In the original function, stage5 is listening, so for our purposes it's a server and we'll be writing a client for it.

```
void stage5() {
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		exit(1);
	}
	
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(args['C']));

	while(connect(sd,(struct sockaddr *)&saddr,sizeof(saddr))<0);

    char *buf = "\xde\xad\xbe\xef";

    if (write(sd,buf,strlen(buf)) < 0)
         perror("ERROR writing to socket");
}
```

#### conclusion

Add all the function together

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <string.h>

#define PATH "/home/ubuntu/ctf/pwnable/input/input"

void forkAndPipe(int *pipe2stdin, int *pipe2stderr, char **args);
void main_main();
void stage4();
void stage5();
void setArgs();

char* args[101];

int main() {
	setArgs();
	main_main();
	stage4();
	return 0;
}

void stage4() {
	FILE *ptr = fopen("\x0a", "w");
	if(ptr == NULL) {
		perror("file not able to create");
		exit(1);
	}
	fwrite("\x00\x00\x00\x00", 4, 1, ptr);

    fclose(ptr);
    
}

void setArgs() {
	args[0]=PATH;
	for (int i = 1; i < 100; ++i)
		args[i]="0";

	args['A']="\x00";
	args['B']="\x20\x0a\x0d";
	args['C']="9001";
	args[100]=NULL;
}

void main_main() {
	
	int pipe2stdin[2], pipe2stderr[2];

	if(pipe(pipe2stderr) < 0 || pipe(pipe2stdin) < 0) {
		perror("can't create pipe");
		exit(1);
	}

	forkAndPipe(pipe2stdin, pipe2stderr, args);
}

void forkAndPipe(int *pipe2stdin, int *pipe2stderr, char **args) {
	pid_t c_pid; 
	if((c_pid=fork()) == -1) {
		perror("fork error");
		exit(1);
	}

	char *envp[] = {
		"\xde\xad\xbe\xef=\xca\xfe\xba\xbe",
		0
	};

	if (c_pid == 0)
	{
		close(pipe2stdin[0]); close(pipe2stderr[0]);
		write(pipe2stdin[1], "\x00\x0a\x00\xff", 4);
		write(pipe2stderr[1], "\x00\x0a\x02\xff", 4);
		stage5();
		
	} else {
		// parent process 
		close(pipe2stdin[1]); close(pipe2stderr[1]);
		dup2(pipe2stdin[0], 0);
		dup2(pipe2stderr[0], 2);
		execve(PATH, args, envp);
	}
}

void stage5() {
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		exit(1);
	}
	
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(args['C']));

	while(connect(sd,(struct sockaddr *)&saddr,sizeof(saddr))<0);

    char *buf = "\xde\xad\xbe\xef";

    if (write(sd,buf,strlen(buf)) < 0)
         perror("ERROR writing to socket");
}

```
