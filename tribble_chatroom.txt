#define _GNU_SOURCE
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <unistd.h>
#include <string.h>
#include <signal.h>
#include <sched.h>

typedef struct {
  int uid[10];
  int upid[10];
  int inidx;
  char msg[100];
} chatmem;

chatmem *ptr;
int usernumber=-1;
int outidx;
int monitorPid;
char line[1024];
char* lineptr = line;
size_t lineSize = sizeof(line);
char childStack[4000000];
char username[500];
char filler[50] = " has entered the chat";
char nothing[550];

void putString(char *str) {
  while (*str != '\0') {
    ptr->msg[ptr->inidx++] = *str++;
    if (ptr->inidx>=sizeof(ptr->msg)) ptr->inidx=0;
  }
  ptr->msg[ptr->inidx] = 0;
}


// Child process to monitor for new mesaages
int monitor() {
  // Code for child process that will monitor for new messages
  
printf("\n[Warning Big Brother is watching (monitor process)]: %de\n",getpid());
  while (1) {
    while (outidx!=ptr->inidx) {
      putchar(ptr->msg[outidx++]);
      if (outidx > sizeof(ptr->msg)) outidx=0;
    }
    sleep(1);
  }
  //return;   // Terminate the child process
}

int main() {
  int shmid;
  int key=34;
  char buffer[4096];
  int i;
  printf("Please enter your username: ");
  fgets(username,sizeof(username),stdin);
  username[strcspn(username, "\n")] = 0;
  strcat(nothing,username);
  strcat(nothing,filler);
  shmid = shmget(key,4096*1024,IPC_CREAT|IPC_EXCL|0666);
  if (shmid>0) {
    printf("Created new shared memory.  Initializing\n");
    printf("Shared mem id: %d\n",shmid);
    ptr = shmat(shmid,NULL,0);
    ptr->inidx = 0;
    ptr->msg[ptr->inidx] = '\0';
    putString("Freshly initialized\n\n");
    for (i=0; i<10; i++) {
      ptr->uid[i] = 0;
      ptr->upid[i] = 0;
    }
  }
  else {
    shmid = shmget(key,4096*1024,0);
    if (shmid>0)
      printf("Shared memory already exists, Shared mem id: %d\n",shmid);
    else {
      printf("You lose, no shared memory is accessible.\nProgram terminated\n\n");
	return 1;
    }
    ptr = shmat(shmid,NULL,0);
    putString(nothing);
  }
  for (i=0; i<10; i++){
    if (ptr->uid[i]==0) {
      ptr->uid[i] = getuid();
      ptr->upid[i] = getpid();
      usernumber = i;
      break;
    }
}
  printf("Current user list\n_________________\n");
  for (i=0; i<10; i++)
    if (ptr->uid[i]!=0) {
      printf("User id: %d  pid: %d\n",ptr->uid[i],ptr->upid[i]);
    }
  // Set the out index to the shared memory in index
  outidx = ptr->inidx;
  printf("Address of shared memory: ptr= %lx\n",(long)ptr);
  printf("Shm contains: %s\n\n",ptr->msg);
  printf("Current message below:\n");
  monitorPid = clone(monitor,&childStack[4000000-8],CLONE_VM,NULL);


  while (1) {
    printf("%s:",username);
    getline(&lineptr,&lineSize,stdin);
    if (strcmp(line,"exit\n")==0) break;
    putString(line);
    outidx = ptr->inidx;
  }

  kill(monitorPid,9);
  // Remove this user from the user tables
  ptr->uid[usernumber] = 0;
  ptr->upid[usernumber] = 0;

}
